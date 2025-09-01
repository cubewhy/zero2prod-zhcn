# 避免“泥球”错误枚举

在 `SubscribeError` 中我们使用枚举变量有两个目的:

- 确定应返回给 API 调用者的响应 (`ResponseError`)
- 提供相关的诊断信息 (`Error::source`、`Debug`、`Display`)。

目前定义的 `SubscribeError` 暴露了 `subscribe` 的大量实现细节：我们在请求处理程序中，每个可能出错的函数调用都有一个对应的变体!

这种策略的扩展性并不好。

我们需要从抽象层的角度来思考：subscribe 的调用者需要知道什么?

他们应该能够确定要返回给用户的响应（通过 `ResponseError`）。就是这样。

`subscribe` 的调用者不了解订阅流程的复杂性：他们对领域了解不够，无法对 `SendEmailError` 和 `TransactionCommitError` 做出不同的行为（设计如此！）。`subscribe` 应该返回一个在正确抽象层级上表达的错误类型。

理想的错误类型如下所示:

```rs
//! src/routes/subscriptions.rs
// [...]

#[derive(thiserror::Error)]
pub enum SubscribeError {
    #[error("{0}")]
    ValidationError(String),
    #[error(/* */)]
    UnexpectedError(/* */),
}
```

`ValidationError` 映射到 400 Bad Request, `UnexpectedError` 映射到不透明的 500 Internal Server Error。

我们应该在 `UnexpectedError` 变量中存储什么?

我们需要将多种错误类型映射到它——`sqlx::Error`、`StoreTokenError` 和 `reqwest::Error`。

我们不想暴露通过 `subscribe` 映射到 `UnexpectedError` 的易错例程的实现细节——它必须是不透明的。
在查看 Rust 标准库中的 `Error` trait 时，我们偶然发现了一个满足这些要求的类型: `Box<dyn std::error::Error>` !

让我们试一试:

```rs
//! src/routes/subscriptions.rss
#[derive(thiserror::Error)]
pub enum SubscribeError {
    #[error("{0}")]
    ValidationError(String),
    // Transparent delegates both `Display`'s and `source`s implementation
    // to the type wrapped by `UnexpectedError`
    #[error(transparent)]
    UnexpectedError(#[from] Box<dyn std::error::Error>)
}
```

我们仍然可以为呼叫者生成准确的响应:

```rs
//! src/routes/subscriptions.rs
// [...]
impl ResponseError for SubscribeError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        match self {
            SubscribeError::ValidationError(_) => StatusCode::BAD_REQUEST,
            SubscribeError::UnexpectedError(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

我们只需要在使用 `?` 运算符之前调整 `subscribe` 方法以正确转换我们的错误:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> Result<HttpResponse, SubscribeError> {
    let new_subscriber = form
        .into_inner()
        .try_into()
        .map_err(SubscribeError::ValidationError)?;
    let mut transaction = pool.begin()
        .await
        .map_err(|e| SubscribeError::UnexpectedError(Box::new(e)))?;
    let subscriber_id = insert_subscriber(/* */)
        .await
        .map_err(|e| SubscribeError::UnexpectedError(Box::new(e)))?;
    // [...]

    store_token(/**/)
        .await
        .map_err(|e| SubscribeError::UnexpectedError(Box::new(e)))?;

    transaction
        .commit()
        .await
        .map_err(|e| SubscribeError::UnexpectedError(Box::new(e)))?;

    send_confirmation_email(/* */)
    .await
    .map_err(|e| SubscribeError::UnexpectedError(Box::new(e)))?;
    Ok(HttpResponse::Ok().finish())
}
```

代码有些重复，但暂时先这样吧。

代码编译通过，测试也按预期通过。

让我们修改一下之前用来检查日志消息质量的测试: 在 `insert_subscriber` 中触发一个失败，而不是在 `store_token` 中。

```rs
//! tests/api/subscriptions.rs
// [...]

#[tokio::test]
async fn subscribe_fails_if_there_is_a_fatal_database_error() {
    // [...]
    sqlx::query!("ALTER TABLE subscriptions DROP COLUMN email;",)
        .execute(&app.db_pool)
        .await
        .unwrap();

    // [...]
}
```

测试通过了，但是我们可以看到我们的日志已经倒退了:

```text
    exception.details: error returned from database: column "email" of relation 
"subscriptions" does not exist
    
    Caused by:
        column "email" of relation "subscriptions" does not exist
```

我们再也看不到原因链了。

我们丢失了之前附加到 `InsertSubscriberError` 的、对操作员友好的错误消息，通过这个错误:

```rs
//! src/routes/subscriptions.rs
// [...]

#[derive(thiserror::Error)]
pub enum SubscribeError {
    #[error("Failed to insert new subscriber in the database.")]
    InsertSubscriberError(#[source] sqlx::Error),
    // [...]
}
```

这是意料之中的：我们现在将原始错误转发到 `Display` (通过 `#[error(transparent)]`)，

我们没有在 `subscribe` 中附加任何额外的上下文。

我们可以解决这个问题——让我们在 `UnexpectedError` 中添加一个新的字符串字段，以便将上下文信息附加到我们存储的不透明错误中:

```rs
//! src/routes/subscriptions.rs
// [...]

#[derive(thiserror::Error)]
pub enum SubscribeError {
    #[error("{0}")]
    ValidationError(String),
    // Transparent delegates both `Display`'s and `source`s implementation
    // to the type wrapped by `UnexceptedError`
    #[error("{1}")]
    UnexpectedError(#[source] Box<dyn std::error::Error>, String),
}

impl ResponseError for SubscribeError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        match self {
            // The cariant now has two fields, we need an extra `_`
            SubscribeError::UnexpectedError(_, _) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

我们需要相应地调整 `subscribe` 中的映射代码——我们将重用重构 `SubscribeError` 之前的错误描述:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> Result<HttpResponse, SubscribeError> {
    // [...]
    let mut transaction = pool.begin().await.map_err(|e| {
        SubscribeError::UnexpectedError(
            Box::new(e),
            "Failed to acquire a Postgres conneciton from the pool".into(),
        )
    })?;
    let subscriber_id = insert_subscriber(/* */)
        .await
        .map_err(|e| {
            SubscribeError::UnexpectedError(
                Box::new(e),
                "Failed to insert new subscriber in the database.".into(),
            )
        })?;
    // [...]

    store_token(/* */)
        .await
        .map_err(|e| {
            SubscribeError::UnexpectedError(
                Box::new(e),
                "Failed to store the confirmation token for a new subscriber.".into(),
            )
        })?;

    transaction.commit().await.map_err(|e| {
        SubscribeError::UnexpectedError(
            Box::new(e),
            "Failed to commit SQL transaction to store a new subscriber.".into(),
        )
    })?;

    send_confirmation_email(/* */)
    .await
    .map_err(|e| {
        SubscribeError::UnexpectedError(Box::new(e), "Failed to send a confirmation email.".into())
    })?;
    Ok(HttpResponse::Ok().finish())
}
```

虽然有点丑陋，但是可以工作:

```text
[2025-09-01T04:48:01.123Z] ERROR: test/76786 on qby-workspace: [HTTP REQUEST - E
VENT] Error encountered while processing the incoming HTTP request: Failed to in
sert new subscriber in the database.

Caused by:
        error returned from database: column "email" of relation "subscriptions"
 does not exist
Caused by:
        column "email" of relation "subscriptions" does not exist
```

## 使用 anyhow 作为不透明错误类型

我们可以花更多时间完善我们刚刚建造的机器，但事实证明这没有必要:

我们可以再次依靠生态系统。

`无论如何，thiserror` 的作者为我们准备了另一个 crate。

```shell
cargo add anyhow
```

我们正在寻找的类型是 `anyhow::Error`。引用文档:

> `anyhow::Error` 是一个动态错误类型的包装器。`anyhow::Error` 的工作原理与 `Box<dyn std::error::Error>` 非常相似，但有以下区别:
>
> - `anyhow::Error` 要求错误类型为 `Send`、`Sync` 和 `'static`。
> - `anyhow::Error` 保证即使底层错误类型未提供回溯，也提供回溯。
> - `anyhow::Error` 表示为一个窄指针——长度恰好为一个字，而不是两个。

额外的约束（Send、Sync 和 'static'）对我们来说不是问题。

如果我们感兴趣的话，我们很欣赏更紧凑的表示形式和访问回溯的选项。

让我们在 `SubscribeError` 中将 `Box<dyn std::error::Error>` 替换为 `anyhow::Error`:

```rs
//! src/routes/subscriptions.rs
// [...]
#[derive(thiserror::Error)]
pub enum SubscribeError {
    #[error("{0}")]
    ValidationError(String),
    // Transparent delegates both `Display`'s and `source`s implementation
    // to the type wrapped by `UnexceptedError`
    #[error(transparent)]
    UnexpectedError(#[from] anyhow::Error),
}

impl ResponseError for SubscribeError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        match self {
            SubscribeError::ValidationError(_) => StatusCode::BAD_REQUEST,
            // Back to a single field
            SubscribeError::UnexpectedError(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

我们还删除了 `SubscribeError::UnexpectedError` 中的第二个 `String` 字段——它不再是必需的。

`anyhow::Error` 提供了使用**额外上下文**来丰富错误信息的功能。

```rs
//! src/routes/subscriptions.rs
use anyhow::Context;
// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> Result<HttpResponse, SubscribeError> {
    // [...]
    let mut transaction = pool
        .begin()
        .await
        .context("Failed to acquire a Postgres connection from the pool")?;
    let subscriber_id = insert_subscriber(/* */)
        .await
        .context("Failed to insert new subscriber in the database.")?;
    let subscription_token = generate_subscription_token();

    store_token(/* */)
        .await
        .context("Failed to store the confirmation token for a new subscriber.")?;

    transaction
        .commit()
        .await
        .context("Failed to commit SQL transaction to store a new subscriber.")?;
    send_confirmation_email(/* */)
    .await
    .context("Failed to send a confirmation email.")?;
    Ok(HttpResponse::Ok().finish())
}
```

context 方法在这里执行双重任务:

- 它将我们方法返回的错误转换为 `anyhow::Error`
- 它围绕调用者的意图，为其添加额外的上下文。

`context` 由 `Context` trait 提供——无论如何，它为 `Result` 实现了它，让我们能够访问
流畅的 API, 从而轻松处理各种易出错的函数。

## anyhow 还是 thiserror

我们已经讨论了很多内容——现在是时候解决一个常见的 Rust 误区了:

> anyhow 针对应用程序的，而 thiserror 是针对库的。

现在讨论错误处理并不是一个合适的框架。

你需要思考调用者的意图。

你是否期望调用者根据他们遇到的故障模式做出不同的行为?

使用错误枚举，让他们能够匹配不同的变体。引入 `thiserror` 可以减少样板代码的编写。

你是否期望调用者在发生故障时就放弃? 他们主要关心的是将错误报告给运维人员还是用户?

使用不透明的错误，不要让调用者通过编程方式访问错误内部细节。如果你觉得他们的 API 方便，可以使用 `anyhow` 或 `eyre`。

误解源于观察到大多数 Rust 库返回一个错误枚举，
而不是 `Box<dyn std::error::Error>`（例如 `sqlx::Error`）。

库的作者不能（或者不想）对用户的意图做出假设。他们避免（在一定程度上）固执己见——如果需要，枚举可以为用户提供更多控制权。

自由是有代价的——界面更加复杂，用户需要筛选 10 多个变体，才能找出哪些（如果有的话）需要特殊处理。

仔细考虑你的用例以及你能做出的假设，以便设计出
最合适的错误类型——有时 `Box<dyn std::error::Error>` 或 `anyhow::Error` 是最合适的选择, 即使对于库来说也是如此。
