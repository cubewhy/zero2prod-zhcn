# 发送新闻邮件

是时候把邮件发出去了!

我们可以利用前几章写的 `EmailClient` —— 就像 `PgPool` 一样，它已经是应用程序状态的一部分了, 我们可以使用 `web::Data` 来提取它。

```rs
//! src/routes/newsletters.rs
// [...]
pub async fn publish_newsletter(
    body: web::Json<BodyData>,
    pool: web::Data<PgPool>,
    // New argument!
    email_client: web::Data<EmailClient>,
) -> Result<HttpResponse, PublishError> {
    let subscribers = get_confirmed_subscribers(&pool).await?;
    for subscriber in subscribers {
        email_client
            .send_email(
                subscriber.email,
                &body.title,
                &body.content.html,
                &body.content.text,
            )
            .await?;
    }
    Ok(HttpResponse::Ok().finish())
}
```

差一点就能工作:

```text
error[E0308]: mismatched types
  --> src/routes/newsletters.rs:52:17
   |
51 |             .send_email(
   |              ---------- arguments to this method are incorrect
52 |                 subscriber.email,
   |                 ^^^^^^^^^^^^^^^^ expected `SubscriberEmail`, found `String`
   |


error[E0277]: `?` couldn't convert the error to `PublishError`
  --> src/routes/newsletters.rs:57:19
   |
50 | /         email_client
51 | |             .send_email(
52 | |                 subscriber.email,
53 | |                 &body.title,
...  |
57 | |             .await?;
   | |                  -^ unsatisfied trait bound
   | |__________________|
   |                    this can't be annotated with `?` because it has type `Re
sult<_, reqwest::Error>`
```

## context Vs with_context

我们可以快速修复第二个错误

```rs
//! src/routes/newsletters.rs
// [...]
// Bring anyhow's extension trait into scope!
use anyhow::Context;

pub async fn publish_newsletter(
    body: web::Json<BodyData>,
    pool: web::Data<PgPool>,
    email_client: web::Data<EmailClient>,
) -> Result<HttpResponse, PublishError> {
    let subscribers = get_confirmed_subscribers(&pool).await?;
    for subscriber in subscribers {
        email_client
            .send_email(/* */)
            .await
            .with_context(|| {
                format!("Failed to send newsletter issue to {}", subscriber.email)
            })?;
    }
    Ok(HttpResponse::Ok().finish())
}

// [...]
```

我们正在使用一个新方法, `with_context`。
它与 context 密切相关，后者是我们在第 8 章中广泛使用的方法，用于将 `Result` 的错误版本转换为 `anyhow::Error`, 同时用上下文信息丰富它。

两者之间有一个关键区别: `with_context` 是惰性的。
它接受一个闭包作为参数，并且只有在发生错误时才会调用该闭包。

如果您添加的上下文是静态的 - 例如 `context("Oh no!")` - 它们等效。

如果您添加的上下文有运行时开销，请使用 `with_context` - 这样可以避免在易出错的操作成功时为错误路径付出代价。
让我们以我们的情况为例: `format!` 在堆上分配内存来存储其输出字符串。使用 `context`, 我们每次发送电子邮件时都会分配该字符串。

而使用 `with_context`, 我们只有在电子邮件发送失败时才会调用 `format!`。
