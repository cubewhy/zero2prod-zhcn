# 发送确认邮件

虽然花了一段时间，但基础工作已经完成：我们的生产数据库已经准备好支持我们想要构建的新功能——确认邮件。

现在该专注于应用程序代码了。

我们将以适当的测试驱动方式构建整个功能: 在紧密的“红-绿-重构”循环中，循序渐进地推进。

做好准备!

## 静态电子邮件

我们将从简单的开始：测试 POST /subscriptions 是否正在发送电子邮件。

在此阶段，我们不会查看电子邮件的正文，特别是，我们不会检查其中是否包含确认链接。

### Red 测试

要编写此测试，我们需要增强 `TestApp`。

它目前包含我们的应用程序以及数据库连接池的句柄:

```rs
//! tests/api/helpers.rs
// [...]

pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}
```

我们需要启动一个模拟服务器来代替 Postmark 的 API 并拦截外发请求，就像我们构建电子邮件客户端时所做的那样。

让我们相应地编辑 `spawn_app` :

```rs
//! tests/api/helpers.rs

pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
    // New field!
    pub email_server: MockServer,
}

pub async fn spawn_app() -> TestApp {
    // [...]

    // Launch a mock server to stand in for Postmark's API
    let email_server = MockServer::start().await;

    // Randomise configuration to ensure test isolation
    let configuration = {
        let mut c = get_configuration().expect("Failed to read configuration")   ;
        c.database.database_name = Uuid::new_v4().to_string();
        c.application.port = 0;
        c
    };

    // [...]

    TestApp {
        address,
        db_pool: get_connection_pool(&configuration.database),
        email_server
    }
}
```

现在我们可以编写新的测试用例:

```rs
//! tests/api/subscriptions.rs
// New imports
use wiremock::matchers::{method, path};
use wiremock::{Mock, ResponseTemplate};

#[tokio::test]
async fn subscribe_sends_a_confirmation_email_for_valid_data() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .expect(1)
        .mount(&app.email_server)
        .await;

    // Act
    app.post_subscriptions(body.into()).await;

    // Assert
    // Mock asserts on drop
}
```

正如预期的那样，测试失败:

```plaintext
failures:

---- subscriptions::subscribe_sends_a_confirmation_email_for_valid_data stdout -
---

thread 'subscriptions::subscribe_sends_a_confirmation_email_for_valid_data' pani
cked at /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/wirem
ock-0.6.5/src/mock_server/exposed_server.rs:367:17:
Verifications failed:
- Mock #0.
        Expected range of matching incoming requests: == 1
        Number of matched incoming requests: 0

The server did not receive any request.
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

请注意，如果发生故障, `wiremock` 会提供详细的故障原因分析：我们预期会收到一个请求，但实际上什么也没收到。

让我们来解决这个问题。

### Green 测试

TODO: wip
