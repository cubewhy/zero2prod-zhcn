# 控制流错误

## 分层

我们实现了想要的结果（有用的日志），但我不太喜欢这个解决方案：我们从 Web 框架中实现了一个trait (`ResponseError`), 用于处理由一个完全不了解 REST 或 HTTP 协议的操作（`store_token`）返回的错误类型。我们可以从其他入口点（例如 CLI）调用 `store_token`——它的实现应该没有任何改变。

即使假设我们只会在 REST API 上下文中调用 `store_token`，我们也可能会添加依赖于该例程的其他端点——它们可能不希望在失败时返回 500。

在发生错误时选择合适的 HTTP 状态码是请求处理程序需要考虑的问题，它不应该泄露到其他地方。

让我们删除

```rs
//! src/routes/subscriptions.rs
// [...]

// Nuke it!
impl ResponseError for StoreTokenError {}
```

为了强制执行适当的关注点分离，我们需要引入另一种错误类型: `SubscribeError`。

我们将使用它作为 `subscribe` 的失败变体，并负责 HTTP 相关的逻辑 (`ResponseError` 的实现)。

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(/* */) -> Result<HttpResponse, SubscribeError> {
    // [...]
}

#[derive(Debug)]
struct SubscribeError {}

impl std::fmt::Display for SubscriberError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "Failed to create a new subscriber."
        )
    }
}

impl std::error::Error for SubscriberError {}

impl ResponseError for SubscriberError {}
```

如果你运行 `cargo check`, 你会看到大量的 '?', 无法将错误转换为 `SubscribeError` ——我们需要实现函数返回的错误类型与 `SubscribeError` 之间的转换。

## 将错误建模为枚举

枚举是解决这个问题最常用的方法: 为我们需要处理的每个错误类型提供一个变体。

```rs
//! src/routes/subscriptions.rs
// [...]

#[derive(Debug)]
pub enum SubscribeError {
    ValidationError(String),
    DatabaseError(sqlx::Error),
    StoreTokenError(StoreTokenError),
    SendEmailError(reqwest::Error),
}

impl ResponseError for SubscribeError {}

impl std::error::Error for SubscribeError {}

impl std::fmt::Display for SubscribeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Failed to create a new subscriber.")
    }
}
```

然后，我们可以在处理程序中利用 ? 运算符，为每个包装的错误类型提供一个 From 实现:

```rs
//! src/routes/subscriptions.rs
// [...]

impl From<reqwest::Error> for SubscribeError {
    fn from(value: reqwest::Error) -> Self {
        Self::SendEmailError(value)
    }
}

impl From<sqlx::Error> for SubscribeError {
    fn from(e: sqlx::Error) -> Self {
        Self::DatabaseError(e)
    }
}
impl From<StoreTokenError> for SubscribeError {
    fn from(e: StoreTokenError) -> Self {
        Self::StoreTokenError(e)
    }
}
impl From<String> for SubscribeError {
    fn from(e: String) -> Self {
        Self::ValidationError(e)
    }
}
```

现在，我们可以通过删除所有 `match` / `if fallible_function().is_err()` 行来清理我们的请求处理程序:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(
    // [...]
) -> Result<HttpResponse, SubscribeError> {
    let new_subscriber = form.into_inner().try_into()?;
    let mut transaction = pool.begin().await?;
    let subscriber_id = insert_subscriber(&mut *transaction, &new_subscriber).await?;
    let subscription_token = generate_subscription_token();

    store_token(/*[...]*/).await?;

    transaction.commit().await?;

    // Pass the applicaiton url
    send_confirmation_email(/*[...]*/).await?;
    Ok(HttpResponse::Ok().finish())
}
```

代码可以编译，但是我们的一个测试失败了:

```text
thread 'subscriptions::subscribe_returns_a_200_when_fields_are_present_but_inval
id' panicked at tests/api/subscriptions.rs:93:9:
assertion `left == right` failed: The API did not return a 400 OK when the paylo
ad was empty name.
  left: 400
 right: 500
```

我们仍然使用 `ResponseError` 的默认实现——它总是返回 500。

这就是枚举的亮点：我们可以使用 `match` 语句来**控制流**——根据我们处理的失败场景，我们会采取不同的行为。

```rs
//! src/routes/subscriptions.rs
use actix_web::http::StatusCode;
// [...]

impl ResponseError for SubscribeError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        match self {
            SubscribeError::ValidationError(_) => StatusCode::BAD_REQUEST,
            SubscribeError::DatabaseError(_)
            | SubscribeError::StoreTokenError(_)
            | SubscribeError::SendEmailError(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

测试应该会再次通过。

## 错误类型还不足够

我们的日志怎么样?

我们再来看看:

```shell
export RUST_LOG="sqlx=error,info"
export TEST_LOG=enabled
cargo t subscribe_fails_if_there_is_a_fatal_database_error | bunyan
```

```text
    exception.details: StoreTokenError(A database error was encountered while tr
ying to store a subscription token.
    
    Caused by:
        error returned from database: column "subscription_token" of relation "s
ubscription_tokens" does not exist
```

我们仍然可以在 `exception.details` 中很好地表示底层的 `StoreTokenError`, 但它显示我们现在正在使用派生的 Debug 实现来处理 `SubscribeError`。不过，
信息没有丢失。

但 `exception.message` 的情况则不同——无论失败模式如何，我们总是会收到“无法创建新订阅者”的错误信息。这不太实用。

让我们改进一下 `Debug` 和 `Display` 的实现:

```rs
//! src/routes/subscriptions.rs
// [...]

// Remember to delete `#[derive(Debug)]`!
impl std::fmt::Debug for SubscribeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        error_chain_fmt(self, f)
    }
}

impl std::error::Error for SubscribeError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            // &str does not implement `Error` - we consider it the root cause
            SubscribeError::ValidationError(_) => None,
            SubscribeError::DatabaseError(e) => Some(e),
            SubscribeError::StoreTokenError(e) => Some(e),
            SubscribeError::SendEmailError(e) => Some(e),
        }
    }
}

impl std::fmt::Display for SubscribeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            SubscribeError::ValidationError(e) => write!(f, "{}", e),
            // What should we do here?
            SubscribeError::DatabaseError(_) => write!(f, "???"),
            SubscribeError::StoreTokenError(_) => write!(
                f,
                "Failed to store the confirmation token for a new subscriber."
            ),
            SubscribeError::SendEmailError(_) => {
                write!(f, "Failed to send a confirmation email.")
            }
        }
    }
}
```

`Debug` 很容易排序: 我们为 `SubscribeError` 实现了 `Error` trait，包括 `source`, 并且我们可以再次使用之前为 `StoreTokenError` 编写的辅助函数。

在 `Display` 方面，我们遇到了一个问题——相同的 `DatabaseError` 变体用于以下情况下遇到的错误:

- 从池中获取新的 Postgres 连接
- 在订阅者表中插入订阅者
- 提交 SQL 事务

在为 `SubscribeError` 实现 `Display` 时，我们无法区分我们正在处理的是这三种情况中的哪一种——底层错误类型是不够的。

让我们通过为每个操作使用不同的枚举变体来消除歧义:

```rs
//! src/routes/subscriptions.rs
// [...]

pub enum SubscribeError {
    // [...]
    // No more `DatabaseError`
    PoolError(sqlx::Error),
    InsertSubscriberError(sqlx::Error),
    TransactionCommitError(sqlx::Error),
}

impl std::error::Error for SubscribeError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            // [...]
            // No more DatabaseError

            SubscribeError::PoolError(e) => Some(e),
            SubscribeError::InsertSubscriberError(e) => Some(e),
            SubscribeError::TransactionCommitError(e) => Some(e),
        }
    }
}

impl std::fmt::Display for SubscribeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            // [...]
            // No more DatabaseError
            SubscribeError::PoolError(_) => {
                write!(f, "Failed to acquire a Postgres connection from the pool")
            }
            SubscribeError::InsertSubscriberError(_) => {
                write!(f, "Failed to insert new subscriber in the database.")
            }
            SubscribeError::TransactionCommitError(_) => {
                write!(
                    f,
                    "Failed to commit SQL transaction to store a new subscriber."
                )
            }
        }
    }
}

impl ResponseError for SubscribeError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        match self {
            SubscribeError::ValidationError(_) => StatusCode::BAD_REQUEST,
            SubscribeError::TransactionCommitError(_)
            | SubscribeError::InsertSubscriberError(_)
            | SubscribeError::PoolError(_)
            | SubscribeError::StoreTokenError(_)
            | SubscribeError::SendEmailError(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}
```

`DatabaseError` 还在另一个地方使用:

```rs
//! src/routes/subscriptions.rs
// [...]

impl From<sqlx::Error> for SubscribeError {
    fn from(e: sqlx::Error) -> Self {
        Self::DatabaseError(e)
    }
}
```

仅凭类型不足以区分应该使用哪种新变体；我们无法为 `sqlx::Error` 实现 `From`。

我​​们必须使用 `map_err` 在每种情况下执行正确的转换。

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
    let mut transaction = pool.begin().await
        .map_err(SubscribeError::PoolError)?;
    let subscriber_id = insert_subscriber(/* */).await
        .map_err(SubscribeError::InsertSubscriberError)?;
    // [...]

    store_token(/* */).await?;

    transaction.commit().await
        .map_err(SubscribeError::TransactionCommitError)?;

    // [...]
}
```

代码编译后, `exception.message` 再次变得有用:

```text
    exception.details: Failed to store the confirmation token for a new subscrib
er.
    
    Caused by:
        A database error was encountered while trying to store a subscription to
ken.
    Caused by:
        error returned from database: column "subscription_token" of relation "s
ubscription_tokens" does not exist
```

## 使用 thiserror 移除模板代码

我们花了大约 90 行代码来实现 `SubscriberError` 及其周围的所有机制，以便实现所需的行为并在日志中获得有用的诊断信息。

这代码量很大，包含大量样板代码（例如源代码或 From 的实现）。

我们能做得更好吗?

嗯，我不确定我们能否编写更少的代码，但我们可以找到另一种方法：我们可以使用宏来生成所有这些样板代码!

碰巧的是，生态系统中已经有一个很棒的 crate 用于此目的: `thiserror`。让我们将它添加到我们的依赖项中:

```shell
cargo add thiserror
```

它提供了一个 derive 宏，可以生成我们刚刚手写的大部分代码。

我们来看看它的实际效果:

```rs
//! src/routes/subscriptions.rs
// [...]
#[derive(thiserror::Error)]
pub enum SubscribeError {
    #[error("{0}")]
    ValidationError(String),
    #[error("Failed to store the confirmation token for a new subscriber.")]
    StoreTokenError(#[from] StoreTokenError),
    #[error("Failed to send a confirmation email.")]
    SendEmailError(#[from] reqwest::Error),
    #[error("Failed to acquire a Postgres connection from the pool")]
    PoolError(#[source] sqlx::Error),
    #[error("Failed to insert new subscriber in the database.")]
    InsertSubscriberError(#[source] sqlx::Error),
    #[error("Failed to commit SQL transaction to store a new  subscriber")]
    TransactionCommitError(#[source] sqlx::Error),
}

// We are still using a bespoke implementation of `Debug`
// to get a nice report using the error source chain
impl std::fmt::Debug for SubscribeError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        error_chain_fmt(self, f)
    }
}

pub async fn subscribe(
    // [...]
) -> Result<HttpResponse, SubscribeError> {
    let new_subscriber = form.into_inner().try_into()
        .map_err(SubscribeError::ValidationError)?;
    // [...]
}
```

我们把它精简到了 21 行——还不错!

让我们来分析一下现在的情况。

`thiserror::Error` 是一个通过 `#[derive(/* */)]` 属性使用的过程宏。

我们之前见过并使用过这些宏，例如 `#[derive(Debug)]` 或 `#[derive(serde::Serialize)]`。

该宏在编译时接收 `SubscribeError` 的定义作为输入，并返回另一个 token 流作为输出——它会生成新的 Rust 代码，然后将其编译成最终的二进制文件。
在 `#[derive(thiserror::Error)]` 的上下文中，我们可以访问其他属性来实现我们想要的行为:

- `#[error(/* */)]` 定义了它所应用到的枚举变量的 Display 表示形式。例如，当在 `SubscribeError::SendEmailError` 的实例上调用 `Display` 时，它将返回 `Failed to send a confirmed email.` 。你可以在最终表示形式中插入值，例如ValidationError 之上 `#[error("{0}")]` 中的 {0} 指的是包装的字符串字段，模仿了访问元组结构体（例如 self.0）字段的语法。
- `#[source]` 用于表示 Error::source 中应返回的根本原因；
- `#[from]` 自动将 From 的实现派生为所应用类型的顶级错误类型（例如，`impl From<StoreTokenError> for SubscribeError {/* */}`）。带有 `#[from]` 注解的字段也用作错误源，这样我们就不必在同一个字段上使用两个注解了（例如，`#[source]` `#[from] reqwest::Error`）。

我想提醒您注意一个小细节: 我们没有对 `ValidationError` 变体使用 `#[from]` 或 `#[source]`。这是因为 `String` 没有实现 `Error` trait，因此它无法在 `Error::source` 中返回——这与我们之前手动实现 `Error::source` 时遇到的限制相同，导致我们在 `ValidationError` 的情况下返回 `None`。
