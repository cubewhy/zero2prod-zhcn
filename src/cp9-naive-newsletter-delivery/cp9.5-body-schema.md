# Body 结构

为了发送新闻通讯，我们需要了解哪些信息?

如果我们力求使其尽可能简洁:

- 标题，用作电子邮件主题
- 内容，以 HTML 和纯文本形式呈现，以满足所有电子邮件客户端的需求。

我们可以使用派生自 `serde::Deserialize` 的结构体来编码我们的需求，就像我们在 `POST /subscriptions` 中使用 `FormData` 进行编码一样。

```rs
//! src/routes/newsletters.rs
// [...]
#[derive(serde::Deserialize)]
pub struct BodyData {
    title: String,
    content: Content,
}

#[derive(serde::Deserialize)]
pub struct Content {
    html: String,
    text: String,
}
```

由于 `BodyData` 中的所有字段类型都实现了 `serde::Deserialize`, 因此 serde 对我们的嵌套布局没有任何问题。然后，我们可以使用 actix-web 提取器从传入的请求正文中解析出 `BodyData`。只有一个问题需要回答：我们使用什么序列化格式?

对于 `POST /subscriptions`, 由于我们处理的是 HTML 表单，我们使用 `application/x-www-form-urlencode`
作为 `Content-Type`。
对于 `POST /newsletters`, 我们不受网页中嵌入表单的约束: 我们将使用 JSON，这是构建 REST API 时的常见选择。

相应的提取器是 `actix_web::web::Json`:

```rs
//! src/routes/newsletters.rs
// [...]
use actix_web::web;

// We are prefixing `body` with a `_` to avoid
// a compiler warning about unused arguments
pub async fn publish_newsletter(_body: web::Json<BodyData>) -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

## 测试不合法输入

信任但要验证: 让我们添加一个新的测试用例, 在 `POST /newsletters` 端点抛出无效数据。

```rs
//! tests/api/newsletter.rs
// [...]

#[tokio::test]
async fn newsletters_returns_400_for_invalid_data() {
    // Arrange
    let app = spawn_app().await;
    let test_cases = vec![
        (
            serde_json::json!({
            "content": {
            "text": "Newsletter body as plain text",
            "html": "<p>Newsletter body as HTML</p>",
            }
            }),
            "missing title",
        ),
        (
            serde_json::json!({"title": "Newsletter!"}),
            "missing content",
        ),
    ];

    for (invalid_body, error_message) in test_cases {
        let response = reqwest::Client::new()
            .post(&format!("{}/newsletters", &app.address))
            .json(&invalid_body)
            .send()
            .await
            .expect("Failed to execute request.");

        // Assert
        assert_eq!(
            400,
            response.status().as_u16(),
            "The API did not fail with 400 Bad Request when the payload was {}.",
            error_message
        )
    }
}
```

新的测试通过了——如果你愿意，还可以添加一些用例。

让我们抓住机会稍微重构一下，删除一些重复的代码——我们可以将触发 `POST /newsletters` 请求的逻辑提取到 `TestApp` 上的一个共享辅助方法中，就像我们对 `POST /subscriptions` 所做的那样:

```rs
//! tests/api/helpers.rs
// [...]

impl TestApp {
    // [...]
    pub async fn post_newsletters(&self, body: serde_json::Value) -> reqwest::Response {
        reqwest::Client::new()
            .post(&format!("{}/newsletters", &self.address))
            .json(&body)
            .send()
            .await
            .expect("Failed to execute request.")
    }
}
```

```rs
//! tests/api/newsletter.rs
// [...]

async fn newsletters_are_delivered_to_confirmed_subscribers() {
    // [...]
    let response = app.post_newsletters(newsletter_request_body).await;

    // [...]
}

async fn newsletters_returns_400_for_invalid_data() {
    // Arrange
    // [...]

    for (invalid_body, error_message) in test_cases {
        let response = app.post_newsletters(invalid_body).await;

        // Assert
        // [...]
    }
}

async fn newsletters_are_not_delivered_to_unconfirmed_subscribers() {
    // Arrange
    // [...]
    
    let response = app.post_newsletters(newsletter_request_body).await;

    // Assert
    // [...]
}
```
