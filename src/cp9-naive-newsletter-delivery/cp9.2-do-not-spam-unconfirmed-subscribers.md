# 不要给未订阅用户发垃圾邮件

我们可以先编写一个集成测试，明确哪些情况不应该发生：未经确认的订阅者不应该收到新闻通讯。

在第七章中，我们选择了 Postmark 作为我们的电子邮件传递服务。如果我们没有调用 Postmark，就不会发送电子邮件。

我们可以基于此来设计一个场景，以验证我们的业务规则: 如果所有订阅者都未经确认，那么当我们发布新闻通讯时，就不会向 Postmark 发出任何请求。

让我们将其转化为代码:

```rs
//! tests/api/main.rs
// [...]
mod newsletter;
```

```rs
//! tests/api/newsletter.rs
tchers::{any, method, path}, Mock, ResponseTemplate};

use crate::helpers::{spawn_app, TestApp};

#[tokio::test]
async fn newsletters_are_not_delivered_to_unconfirmed_subscribers() {
    // Arrange
    let app = spawn_app().await;
    create_unconfirmed_subscriber(&app).await;

    Mock::given(any())
        .respond_with(ResponseTemplate::new(200))
        .expect(0)
        .mount(&app.email_server)
        .await;

    // Act
    
    // A sketch of the newsletter payload structure
    // We might change it later on.
    let newsletter_request_body = serde_json::json!({
        "title": "Newsletter title",
        "content": {
            "text": "Newsletter body as plain text",
            "html": "<p>Newsletter body as HTML</p>",
        }
    });
    let response = reqwest::Client::new()
        .post(&format!("{}/newsletters", &app.address))
        .json(&newsletter_request_body)
        .send()
        .await
        .expect("Failed to execute request.");

    // Assert
    assert_eq!(response.status().as_u16(), 200);
    // Mock verifies on Drop that we haven't sent the newsletter email
}

/// Use the public API of the application under test to create
/// an unconfirmed subscriber.
async fn create_unconfirmed_subscriber(app: &TestApp) {
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";
    
    let _mock_guard = Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .named("Create unconfirmed subscriber")
        .expect(1)
        .mount_as_scoped(&app.email_server)
        .await;
    app.post_subscriptions(body.into())
        .await
        .error_for_status()
        .unwrap();
}
```

正如预期的那样，它失败了:

```text
thread 'newsletter::newsletters_are_not_delivered_to_unconfirmed_subscribers' pa
nicked at tests/api/newsletter.rs:36:5:
assertion `left == right` failed
  left: 404
 right: 200
```

我们的 API 中没有用于 `POST /newsletters` 的处理程序: actix-web 返回 404 Not Found，而不是测试预期的 200 OK。

## 使用公共 API 设置状态

让我们花点时间看一下我们刚刚编写的测试的 "Arrange" 部分。

我们的测试场景对应用程序的状态做了一些假设：我们需要一个订阅者，并且该订阅者必须是未确认的。

每个测试都会启动一个全新的应用程序，并在一个空数据库上运行。

```rs
let app = spawn_app().await;
```

我们如何根据测试需求填充它?

我们坚持第三章中描述的黑盒方法: 尽可能通过调用应用程序的公共 API 来驱动应用程序状态。

这就是我们在 `create_unconfirmed_subscriber` 中所做的:

```rs
//! tests/api/newsletter.rs
// [...]

async fn create_unconfirmed_subscriber(app: &TestApp) {
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";
    
    let _mock_guard = Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .named("Create unconfirmed subscriber")
        .expect(1)
        .mount_as_scoped(&app.email_server)
        .await;
    app.post_subscriptions(body.into())
        .await
        .error_for_status()
        .unwrap();
}
```

我们使用在 `TestApp` 中构建的 API 客户端向 `/subscriptions` 端点发出 POST 调用。

## 作用域模拟

我们知道 `POST /subscriptions` 会发送一封确认邮件——我们必须确保我们的 Postmark 测试服务器已准备好处理传入的请求，为此我们需要设置相应的模拟。

匹配逻辑与测试函数体中的逻辑重叠：我们如何确保这两个模拟不会互相干扰?

我们使用一个作用域模拟:

```rs
let _mock_guard = Mock::given(path("/email"))
    .and(method("POST"))
    .respond_with(ResponseTemplate::new(200))
    .named("Create unconfirmed subscriber")
    .expect(1)
    // We are not using `mount`!
    .mount_as_scoped(&app.email_server)
    .await;
```

使用 `mount` 时，只要底层 `MockServer` 正常运行，我们指定的行为就会一直有效。

而使用 `mount_as_scoped` 时，我们会返回一个守护对象——`MockGuard`。

`MockGuard` 有一个自定义的 Drop 实现: 当超出范围时, `wiremock` 会指示底层 `MockServer` 停止执行指定的模拟行为。换句话说，在 `create_unconfirmed_subscriber` 的末尾, 我们会停止向 POST /email 返回 200。

我们的测试助手所需的模拟行为仅对测试助手本身有效。

当 MockGuard 被丢弃时，还会发生另一件事——我们会积极地检查作用域模拟的期望是否已得到验证。

这会创建一个有用的反馈循环，以保持我们的测试辅助函数干净且最新。

我们已经见证了黑盒测试如何促使我们为自己的应用程序编写 API 客户端，以保持测试简洁。

随着时间的推移，您会构建越来越多的辅助函数来驱动应用程序状态——就像我们刚才对 `create_unconfirmed_subscriber` 所做的那样。这些辅助函数依赖于模拟，但随着应用程序的发展，其中一些模拟最终不再需要——例如某个调用被移除，您停止使用某个提供程序等等。

积极地评估作用域模拟的期望有助于我们控制辅助函数代码，并在可能的情况下主动进行清理。

## Green 测试

我们可以通过提供 `POST /newsletters` 的虚拟实现来使测试通过:

```rs
//! src/routes.rs
// [...]
mod newsletters;

pub use newsletters::*;
```

```rs
//! src/routes/newsletters.rs
use actix_web::HttpResponse;

pub async fn publish_newsletter() -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

```rs
//! src/startup.rs
// [...]
pub fn run(
    // [...]
) -> Result<Server, std::io::Error> {
    // [...]

    let server = HttpServer::new(move || {
        App::new()
            // [...]
            .route("/newsletters", web::post().to(publish_newsletter))
            // [...]
    })
    // [...]
}
```

`cargo test` 应该可以通过了。
