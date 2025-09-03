# 获取已确认订阅者列表

我们需要编写一个新的查询来检索所有已确认订阅者的列表。

状态列上的 `WHERE` 子句足以隔离我们关心的行:

```rs
//! src/routes/newsletters.rs
// [...]
struct ConfirmedSubscriber {
    email: String,
}

#[tracing::instrument(name = "Get confirmed subscribers", skip(pool))]
async fn get_confirmed_subscribers(
    pool: &PgPool,
) -> Result<Vec<ConfirmedSubscriber>, anyhow::Error> {
    let rows = sqlx::query_as!(
        ConfirmedSubscriber,
        r#"
        SELECT email
        FROM subscriptions
        WHERE status = 'confirmed'
        "#,
    )
    .fetch_all(pool)
    .await?;

    Ok(rows)
}
```

这里有一些新特性：我们使用 `sqlx::query_as!` 而不是 `sqlx::query!`。

`sqlx::query_as!` 将检索到的行映射到其第一个参数指定的类型, `ConfirmedSubscriber`, 从而省去了我们大量的样板代码。
请注意, `ConfirmedSubscriber` 只有一个字段 - `email`。我们正在最小化从数据库获取的数据量，将查询限制在实际需要发送新闻通讯的列上。数据库的工作量更少，网络上传输的数据也更少。

在这种情况下，这不会带来明显的区别，但在处理数据占用空间更大的大型应用程序时，牢记这一点是一个好习惯。

为了在我们的处理程序中使用 `get_confirmed_subscribers`, 我们需要一个 `PgPool`——我们可以从应用程序状态中提取一个，就像我们在 `POST /subscriptions` 中所做的那样。

```rs
//! src/routes/newsletters.rs
// [...]
pub async fn publish_newsletter(
    _body: web::Json<BodyData>,
    pool: web::Data<PgPool>,
) -> HttpResponse {
    let _subscribers = get_confirmed_subscribers(&pool).await?;
    HttpResponse::Ok().finish()
}
```

这并不能编译通过:

```text
  --> src/routes/newsletters.rs:24:62
   |
23 |   ) -> HttpResponse {
   |  ___________________-
24 | |     let _subscribers = get_confirmed_subscribers(&pool).await?;
   | |                                                              ^ cannot use
 the `?` operator in an async function that returns `HttpResponse`
25 | |     HttpResponse::Ok().finish()
26 | | }
   | |_- this function should return `Result` or `Option` to accept `?`
```

SQL 查询可能会失败, `get_confirmed_subscribers` 也可能会失败——我们需要更改 `publish_newsletter` 的返回类型。

我们需要返回一个带有适当错误类型的 `Result`, 就像我们在上一章中所做的那样:

```rs
//! src/routes/newsletters.rs
// [...]
use actix_web::{http::StatusCode, web, HttpResponse, ResponseError};
use sqlx::PgPool;

use crate::routes::error_chain_fmt;

#[derive(thiserror::Error)]
pub enum PublishError {
    #[error(transparent)]
    UnexpectedError(#[from] anyhow::Error),
}

// Same logic to get the full error chain on `Debug`
impl std::fmt::Debug for PublishError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        error_chain_fmt(self, f)
    }
}

impl ResponseError for PublishError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        match self {
            PublishError::UnexpectedError(_) => StatusCode::INTERNAL_SERVER_ERROR,
        }
    }
}

pub async fn publish_newsletter(
    _body: web::Json<BodyData>,
    pool: web::Data<PgPool>,
) -> Result<HttpResponse, PublishError> {
    let _subscribers = get_confirmed_subscribers(&pool).await?;
    Ok(HttpResponse::Ok().finish())
}
// [...]
```

注: 你还需要自己把 `error_chain_fmt` 移动到 `routes.rs` 并修改访问修饰符为 `pub`

利用我们在第 8 章中学到的知识，推出一个新的错误类型并不需要花费太多精力!

需要说明的是，我们正在对代码进行一些面向未来的设计: 我们将 `PublishError` 建模为枚举, 但目前只有一种变体。结构体 (或 `actix_web::error::InternalError`)暂时就足够了。

`cargo check` 现在应该可以通过了。
