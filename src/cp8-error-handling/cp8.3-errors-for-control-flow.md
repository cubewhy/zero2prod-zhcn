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

// TODO: wip
```
