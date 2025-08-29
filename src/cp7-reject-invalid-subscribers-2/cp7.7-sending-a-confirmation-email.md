# 发送确认邮件

虽然花了一段时间，但基础工作已经完成：我们的生产数据库已经准备好支持我们想要构建的新功能——确认邮件。

现在该专注于应用程序代码了。

我们将以适当的测试驱动方式构建整个功能: 在紧密的“红-绿-重构”循环中，循序渐进地推进。

做好准备!

## 静态电子邮件

我们将从简单的开始：测试 POST /subscriptions 是否正在发送电子邮件。

在此阶段，我们不会查看电子邮件的正文，特别是，我们不会检查其中是否包含确认链接。

### 静态电子邮件 - Red 测试

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
        c.email_client.base_url = email_server.uri();

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

### 静态电子邮件 - Green 测试

我们的处理程序现在看起来像这样：

```rs
//! src/routes/subscriptions.rs
// [...]

#[tracing::instrument(/*[...]*/)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let new_subscriber = match form.into_inner().try_into() {
        Ok(subscriber) => subscriber,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    match insert_subscriber(&pool, &new_subscriber).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

要发送电子邮件，我们需要获取 `EmailClient` 的实例。

作为编写模块时所做的工作之一，我们还将其注册到了应用程序上下文中:

```rs
//! src/startup.rs
// [...]
pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
    email_client: EmailClient,
) -> Result<Server, std::io::Error> {
    // [...]

    let server = HttpServer::new(move || {
        App::new()
            // Middlewares are added using the `wrap` method on `App`
            .wrap(TracingLogger::default())
            // [...]
            // here!
            .app_data(email_client.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

因此，我们可以使用 `web::Data` 在我们的处理程序中访问它，就像我们对 `pool` 所做的那样:

```rs
//! src/routes/subscriptions.rs
// New import!
use crate::email_client::EmailClient;
// [...]

#[tracing::instrument(
    name = "Adding a new subscriber",
    skip(form, pool, email_client),
    fields(
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    )
)]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
) -> HttpResponse {
    let new_subscriber = match form.into_inner().try_into() {
        Ok(subscriber) => subscriber,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    if insert_subscriber(&pool, &new_subscriber).await.is_err() {
        return HttpResponse::InternalServerError().finish();
    }
    // Send a (useless) email to the new subscriber
    // We are ignoring email delivery errors for now.
    if email_client
        .send_email(
            new_subscriber.email,
            "Welcome",
            "Welcome to our newsletter!",
            "Welcome to our newsletter!",
        )
        .await
        .is_err()
    {
        return HttpResponse::InternalServerError().finish();
    }
    HttpResponse::Ok().finish()
}
```

`subscribe_sends_a_confirmation_email_for_valid_data` 现已通过，但 `subscribe_returns_a_200_for_valid_f` 失败:

```plaintext
thread 'subscriptions::subscribe_returns_a_200_for_valid_form_data' panicked at tests/api/subscriptions.rs:15:5:
assertion `left == right` failed
  left: 200
 right: 500
```

它正在尝试发送电子邮件，但由于我们没有在该测试中设置模拟，因此失败了。让我们修复它:

```rs
//! tests/api/subscriptions.rs
// [...]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    // New section!
    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .mount(&app.email_server)
        .await;


    // Act
    let response = app.post_subscriptions(body.into()).await;

    // Assert
    assert_eq!(200, response.status().as_u16());

    // [...]
}
```

一切顺利，测试通过了。

目前没有太多需要重构的地方，我们继续吧。

## 静态确认链接

让我们稍微提高一点标准 --我们将扫描电子邮件的正文以检索确认链接。

### 静态确认链接 - Red 测试

我们（目前）并不关心链接是动态的还是实际有意义的——我们
只想确保正文中有一些看起来像链接的内容。

我们还应该在纯文本和 HTML 版本的邮件正文中使用相同的链接。

如何获取 `wiremock::MockServer` 拦截的请求正文?

我们可以使用它的 `received_requests` 方法——只要启用了请求记录（默认设置），它就会返回一个包含服务器拦截的所有请求的向量。

```rs
//! tests/api/subscriptions.rs
// [...]

#[tokio::test]
async fn subscribe_sends_a_confirmation_email_with_a_link() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        // We are not setting an expectation here anymore
        // The test is focused on another aspect of the app
        // behaviour.
        .mount(&app.email_server)
        .await;

    // Act
    app.post_subscriptions(body.into()).await;

    // Assert
    // Get the first intercepted request
    let email_request = &app.email_server.received_requests().await.unwrap()[0];
    // Parse the body as JSON, start from raw bytes
    let body: serde_json::Value =  serde_json::from_slice(&email_request.body).unwrap();
}
```

现在我们需要从中提取链接。

最明显的方法是使用正则表达式。不过，我们必须面对现实：正则表达式本身就很复杂，而且需要一段时间才能正确使用。

再次，我们可以利用 Rust 生态系统的成果——让我们将 linkify 添加为开发依赖项:

```shell
cargo add linkify --dev
```

我们可以使用 `linkify` 扫描文本并返回提取的链接的迭代器。

```rs
//! tests/api/subscriptions.rs
// [...]
async fn subscribe_sends_a_confirmation_email_with_a_link() {
    // [...]
    let body: serde_json::Value =  serde_json::from_slice(&email_request.body).unwrap();

    // Extract the link from one of the request fields.
    let get_link = |s: &str| {
        let links: Vec<_> = linkify::LinkFinder::new()
            .links(s)
            .filter(|l| *l.kind() == linkify::LinkKind::Url)
            .collect();
        assert_eq!(links.len(), 1);
        links[0].as_str().to_owned()
    };

    let html_link = get_link(&body["HtmlBody"].as_str().unwrap());
    let text_link = get_link(&body["TextBody"].as_str().unwrap());
    // The two links should be identical
    assert_eq!(html_link, text_link);
}
```

如果我们运行测试套件，我们应该看到新的测试用例失败:

```plaintext
thread 'subscriptions::subscribe_sends_a_confirmation_email_with_a_link' panicke
d at tests/api/subscriptions.rs:133:9:
assertion `left == right` failed
  left: 0
 right: 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### 静态确认链接 - Green 测试

我们需要再次调整请求处理程序以满足新的测试用例:

```rs
//! src/route/subscriptions.rs
// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
) -> HttpResponse {
    // [...]
    let confirmation_link = "https://my-api.com/subscriptions/confirm";
    // Send a (useless) email to the new subscriber
    // We are ignoring email delivery errors for now.
    if email_client
        .send_email(
            new_subscriber.email,
            "Welcome",
            &format!("Welcome to our newsletter!<br />\
            Click <a href=\"{confirmation_link}\">here</a> to confirm your subscription."),
            &format!("Welcome to our newsletter!\nVisit {confirmation_link} to confirm your subscription."),
        )
        .await
        .is_err()
    {
        return HttpResponse::InternalServerError().finish();
    }
    HttpResponse::Ok().finish()
}
```

测试应该立即通过。

### 静态确认链接 - 重构

我们的请求处理程序有点忙——现在有很多代码在处理我们的确认电子邮件。

让我们将其提取到一个单独的函数中:

```rs
//! src/routes/subscriptions.rs
// [...]
#[tracing::instrument(/*[...]*/)]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
) -> HttpResponse {
    let new_subscriber = match form.into_inner().try_into() {
        Ok(subscriber) => subscriber,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    if insert_subscriber(&pool, &new_subscriber).await.is_err() {
        return HttpResponse::InternalServerError().finish();
    }
    if send_confirmation_email(&email_client, new_subscriber)
        .await
        .is_err()
    {
        return HttpResponse::InternalServerError().finish();
    }
    HttpResponse::Ok().finish()
}

#[tracing::instrument(
    name = "Send a confirmation email to a new subscriber",
    skip(email_client, new_subscriber)
)]
pub async fn send_confirmation_email(
    email_client: &EmailClient,
    new_subscriber: NewSubscriber,
) -> Result<(), reqwest::Error> {
    let confirmation_link = "https://my-api.com/subscriptions/confirm";
    let html_body = format!(
        "Welcome to our newsletter!<br />\
            Click <a href=\"{confirmation_link}\">here</a> to confirm your subscription."
    );
    let plain_body = format!(
        "Welcome to our newsletter!\nVisit {confirmation_link} to confirm your subscription."
    );
    email_client
        .send_email(new_subscriber.email, "Welcome", &html_body, &plain_body)
        .await
}
```

`subscribe` 再次关注整体流程，而不必担心任何步骤的细节。

## 待确认

现在让我们来看看新订阅者的状态。

我们目前在 `POST /subscriptions` 中将其状态设置为“已确认”，但在他们点击确认链接之前，它应该是“待确认”。

是时候修复这个问题了。

### 待确认 - Red 测试

我们可以先重新看一下我们的第一个“快乐路径”测试:

```rs
//! tests/api/subscriptions.rs
// [...]

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    // New section!
    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .mount(&app.email_server)
        .await;

    // Act
    let response = app.post_subscriptions(body.into()).await;

    // Assert
    assert_eq!(200, response.status().as_u16());

    let saved = sqlx::query!("SELECT email, name FROM subscriptions",)
        .fetch_one(&app.db_pool)
        .await
        .expect("Failed to fetch saved subscription.");

    assert_eq!(saved.email, "ursula_le_guin@gmail.com");
    assert_eq!(saved.name, "le guin");
}
```

这个名字有点夸张——它的作用是检查状态码，并根据数据库中存储的状态执行一些断言。

让我们把它拆分成两个独立的测试用例:

```rs
//! tests/api/subscriptions.rs
// [...]

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    // New section!
    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .mount(&app.email_server)
        .await;

    // Act
    let response = app.post_subscriptions(body.into()).await;

    // Assert
    assert_eq!(200, response.status().as_u16());
}

#[tokio::test]
async fn subscribe_persists_the_new_subscriber() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    // New section!
    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .mount(&app.email_server)
        .await;

    // Act
    app.post_subscriptions(body.into()).await;

    // Assert
    let saved = sqlx::query!("SELECT email, name FROM subscriptions",)
        .fetch_one(&app.db_pool)
        .await
        .expect("Failed to fetch saved subscription.");

    assert_eq!(saved.email, "ursula_le_guin@gmail.com");
    assert_eq!(saved.name, "le guin");
}
```

我们现在可以修改第二个测试用例来检查状态。

```rs
//! tests/api/subscriptions.rs
// [...]

#[tokio::test]
async fn subscribe_persists_the_new_subscriber() {
    // [...]
    // Assert
    let saved = sqlx::query!("SELECT email, name, status FROM subscriptions",)
        .fetch_one(&app.db_pool)
        .await
        .expect("Failed to fetch saved subscription.");

    assert_eq!(saved.email, "ursula_le_guin@gmail.com");
    assert_eq!(saved.name, "le guin");
    assert_eq!(saved.status, "pending_confirmation");
}
```

注: 如果你找不到 `status` 字段, 请仔细检查 `query!` 语句中输入的查询语句是否与示例中的相同。

测试如预期失败:

```plaintext
thread 'subscriptions::subscribe_persists_the_new_subscriber' panicked at tests/api/subscriptions.rs:52:5:
assertion `left == right` failed
  left: "confirmed"
 right: "pending_confirmation"
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### 待确认 - Green 测试

我们可以通过再次通过插入查询将其变为绿色:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn insert_subscriber(
    pool: &PgPool,
    new_subscriber: &NewSubscriber,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at, status)
        VALUES ($1, $2, $3, $4, 'confirmed')
        "#,
        // [...]
    )
    // [...]
}
```

我们需要将 `confirmed` 修改为 `pending_confirmation`

```rs
//! src/routes/subscriptions.rs


pub async fn insert_subscriber(
    pool: &PgPool,
    new_subscriber: &NewSubscriber,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at, status)
        VALUES ($1, $2, $3, $4, 'pending_confirmation')
        "#,
        Uuid::new_v4(),
        new_subscriber.email.as_ref(),
        new_subscriber.name.as_ref(),
        Utc::now()
    )
    // [...]
}
```

现在测试应该通过了

## GET /subscriptions/confirm 的骨架

我们已经完成了 `POST /subscriptions` 的大部分基础工作——是时候将注意力转移到旅程的另一半，`GET /subscriptions/confirm`。

我们想要构建端点的框架——我们需要在 `src/startup.rs` 中注册针对路径的处理程序，并拒绝没有必需查询参数 (`subscription_token`) 的传入请求。

这将使我们能够构建令人满意的路径，而无需一次性编写大量代码——循序渐进!

### confirm 的骨架 - Red 测试

让我们在测试项目中添加一个新模块，用于托管所有处理确认回调的测试用例。

```rs
//! tests/api/main.rs
// [...]
mod subscriptions_confirm;
```

```rs
//! tests/api/subscriptions_confirm.rs
use crate::helpers::spawn_app;

#[tokio::test]
async fn confirmations_without_token_are_rejected_with_a_400() {
    // Arrange
    let app = spawn_app().await;

    // Act
    let response = reqwest::get(&format!("{}/subscriptions/confirm", app.address))
        .await
        .unwrap();

    // Assert
    assert_eq!(response.status().as_u16(), 400);
}
```

由于我们还没有处理程序，因此正如预期的那样失败了:

```plaintext
thread 'subscriptions_confirm::confirmations_without_token_are_rejected_with_a_400' panicked at tests/api/subscriptions_confirm.rs:14:5:
assertion `left == right` failed
  left: 404
 right: 400
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### confirm 的骨架 - Green 测试

让我们从一个虚拟处理程序开始，无论传入的请求是什么，它都会返回 200 OK:

```rs
//! src/routes.rs
// [...]
mod subscriptions_confirm;

pub use subscriptions_confirm::*;
```

```rs
//! src/routes/subscriptions_confirm.rs
use actix_web::HttpResponse;

#[tracing::instrument(
    name = "Confirm a pending subscriber"
)]
pub async fn confirm() -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

```rs
//! src/startup.rs
// [...]
use crate::routes::confirm;

pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
    email_client: EmailClient,
) -> Result<Server, std::io::Error> {
    // [...]

    let server = HttpServer::new(move || {
        App::new()
            // [...]
            .route("/subscriptions/confirm", web::get().to(confirm))
            .app_data(db_pool.clone())
            // [...]
    })
    // [...]
}
```

现在运行 `cargo test` 时我们应该会得到不同的错误:

```plaintext
thread 'subscriptions_confirm::confirmations_without_token_are_rejected_with_a_400' panicked at tests/api/subscriptions_confirm.rs:14:5:
assertion `left == right` failed
  left: 200
 right: 400
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

成功了！

是时候把 200 OK 变成 400 Bad Request 了。

我们要确保有一个 `subscription_token` 查询参数:我们可以依赖另一个

`actix-web` 的提取器——`Query`。

```rs
//! src/routes/subscriptions_confirm.rs
use actix_web::{web, HttpResponse};

#[derive(serde::Deserialize)]
pub struct Parameters {
    subscription_token: String,
}

#[tracing::instrument(
    name = "Confirm a pending subscriber",
    skip(_parameters)
)]
pub async fn confirm(_parameters: web::Query<Parameters>) -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

参数结构体定义了我们期望在传入请求中看到的所有查询参数。

它需要实现 `serde::Deserialize` 接口，以便 actix-web 能够根据传入的请求路径构建它。只需添加一个 `web::Query<Parameter>` 类型的函数参数来确认，指示 actix-web 仅在提取成功时调用处理程序。如果提取失败，则会自动向调用者返回 400 Bad Request 错误。

我们的测试现在应该可以通过了。

## 连接到点

现在我们有了 `GET /subscriptions/confirm` 处理程序，我们可以尝试执行完整的旅程！

### 连接到点 - Red 测试

我们将像用户一样操作：我们将调用 `POST /subscriptions` 方法，从发出的电子邮件请求中提取
确认链接（使用我们已经构建的 `linkify` 机制），然后调用该方法确认订阅，并期望返回 200 OK。
我们暂时不会从数据库中检查状态，因为这将是我们最后的收尾工作。

我们来记录一下:

```rs
//! tests/api/subscriptions_confirm.rs
// [...]
use reqwest::Url;
use wiremock::{matchers::{method, path}, Mock, ResponseTemplate};

#[tokio::test]
async fn the_link_returned_by_subscribe_returns_a_200_if_called() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";

    Mock::given(path("/email"))
        .and(method("POST"))
        .respond_with(ResponseTemplate::new(200))
        .mount(&app.email_server)
        .await;
    
    app.post_subscriptions(body.into()).await;
    let email_request = &app.email_server.received_requests().await.unwrap()[0];
    let body: serde_json::Value = serde_json::from_slice(&email_request.body).unwrap();
    
    // Extract the link from one of the request fields
    let get_link = |s: &str| {
        let links: Vec<_> = linkify::LinkFinder::new()
            .links(s)
            .filter(|l| *l.kind() == linkify::LinkKind::Url)
            .collect();
        assert_eq!(links.len(), 1);
        links[0].as_str().to_owned()
    };

    let raw_confirmation_link = get_link(&body["HtmlBody"].as_str().unwrap());
    let confirmation_link = Url::parse(&raw_confirmation_link).unwrap();
    
    // Let's make sure we don't call random APIs on the web
    assert_eq!(confirmation_link.host_str().unwrap(), "127.0.0.1");

    // Act
    let response = reqwest::get(confirmation_link)
        .await
        .unwrap();

    // Assert
    assert_eq!(response.status().as_u16(), 200);
}
```

这里存在相当多的代码重复，但我们会适时处理。

我们现在的首要任务是确保测试顺利通过。

### 连接到点 - Green 测试

我们先来处理一下 URL 问题。

目前，它被硬编码在

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn send_confirmation_email(
    email_client: &EmailClient,
    new_subscriber: NewSubscriber,
) -> Result<(), reqwest::Error> {
    let confirmation_link = "https://my-api.com/subscriptions/confirm";
    // [...]
}
```

域名和协议会根据应用程序运行的环境而有所不同：测试环境为 `http://127.0.0.1`, 而生产环境中运行的应用程序则需要正确的 DNS 记录，并且使用 HTTPS 协议。

最简单的正确方法是将域名作为配置值传入。

让我们在 `ApplicationSettings` 中添加一个新字段:

```rs
//! src/configurations.rs
// [...]
#[derive(serde::Deserialize, Clone)]
pub struct ApplicationSettings {
    #[serde(deserialize_with = "deserialize_number_from_string")]
    pub port: u16,
    pub host: String,
    pub base_url: String,
}
```

```yaml
# configuration/local.yaml
application:
  base_url: "http://127.0.0.1"
```

```yaml
#! spec.yaml
# [...]
services:
  - name: zero2prod
    # [...]
    envs:
    # We use DO's APP_URL to inject the dynamically
    # provisioned base url as an environment variable
      - key: APP_APPLICATION__BASE_URL
        scope: RUN_TIME
        value: ${APP_URL}
      # [...]
# [...]
```

每次修改 spec.yaml 时，请务必将更改应用到 DigitalOcean; 通过 `doctl apps list --format ID` 获取您的应用标识符，然后运行 `​​doctl apps update $APP_ID --spec spec.yaml` .

现在我们需要在应用上下文中注册该值——你应该已经熟悉这个过程了:

```rs
//! src/startup.rs
// [...]
impl Application {
    // We have converted the `build` function into a constructor for
    // `Application`.
    pub async fn build(configuration: Settings) -> Result<Self, std::io::Error> {
        // [...]
        let server = run(
            listener,
            connection_pool,
            email_client,
            // New parameter!
            configuration.application.base_url,
        )?;

        Ok(Self { port, server })
    }

    // [...]
}

// We need to define a wrapper type in order to retrieve the URL
// in the `subscribe` handler.
// Retrieval from the context, in actix-web, is type-based: using
// a raw `String` would expose us to conflicts.
#[derive(Clone)]
pub struct ApplicationBaseUrl(pub String);

pub fn run(
    // [...]
    // New parameter!
    base_url: String,
) -> Result<Server, std::io::Error> {
    // [...]
    let base_url = ApplicationBaseUrl(base_url);

    let server = HttpServer::new(move || {
        App::new()
            // [...]
            .app_data(base_url.clone())
    })
    // [...]
}
```

我们现在可以在请求处理程序中访问它:

```rs
//! src/routres/subscriptions.rs
// [...]
// TODO: WIP
```
