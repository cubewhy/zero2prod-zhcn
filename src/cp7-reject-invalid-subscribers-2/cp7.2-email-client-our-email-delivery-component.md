# EmailClient - 我们的的电子邮件传递组件

## 怎么发一封邮件

你究竟是如何发送电子邮件的?

它是如何工作的?

你必须了解一下 [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol)。

它自互联网早期就已存在——[第一个 RFC](https://tools.ietf.org/html/rfc821) 可以追溯到 1982 年。

SMTP 之于电子邮件的作用就如同 HTTP 之于网页：它是一个应用级协议，确保不同的电子邮件服务器和客户端实现能够相互理解并交换消息。

现在，让我们明确一点——我们不会构建自己的私人电子邮件服务器，这会耗费太长时间，

而且我们不会从中获益太多。我们将利用第三方服务。

如今的电子邮件递送服务需要什么？我们需要通过 SMTP 来与它们沟通吗?

不一定。

SMTP 是一种专用协议：除非你以前使用过电子邮件，否则你不太可能有直接使用它的经验。学习新的协议需要时间，而且过程中难免会犯错——这就是为什么大多数提供商会提供两个接口: SMTP 和 REST API。

如果您熟悉电子邮件协议，或者需要一些非常规的配置，那么
建议您使用 SMTP 接口。否则，大多数开发人员使用 REST API 会更快（也更可靠）地上手。

您可能已经猜到了，这也是我们的目标——我们将编写一个 REST 客户端。

### 选择一个电子邮件 API

市面上的电子邮件 API 提供商不胜枚举，你很可能知道一些主流提供商的名字——AWS SES、SendGrid、MailGun、Mailchimp 和 Postmark。

我正在寻找一个足够简单的 API（例如，发送一封电子邮件有多简单？）、一个流畅的入门流程和一个免费计划，无需输入信用卡信息即可测试服务。

就这样，我选择了 [Postmark](https://postmarkapp.com/)。

注: 译者认为使用 SaaS 平台来支持自己的服务不是最佳实践, 他自己会用自建的 SMTP 服务来发邮件, crates.io 上有很多 crate 可以帮助我们解决 SMTP 问题

要完成接下来的部分，您必须注册 `Postmark`, 并在登录其门户后，授权单个发件人电子邮件。

![Postmark Signup page](images/cp7.2-postmark-signup-page-screenshot.png)

完成后，我们就可以继续了!

> 免责声明：Postmark 没有付费让我在这里推广他们的服务。

### 电子邮件客户端接口

开发一个新功能通常有两种方法: 一种是自下而上，从实现细节开始，慢慢地向上推进；另一种是自上而下，先设计接口，然后再（在一定程度上）确定实现的工作原理。

在这种情况下，我们将选择第二种方法。

我们希望我们的电子邮件客户端拥有什么样的接口?

我们希望有某种 `send_email` 方法。目前我们只需要一次发送一封电子邮件——当我们开始处理新闻通讯问题时，我们会处理批量发送电子邮件的复杂性。

`send_email` 应该接受哪些参数?

我们肯定需要收件人的电子邮件地址、邮件主题和邮件内容。我们会要求提供 HTML 和纯文本版本的电子邮件内容——有些邮件客户端无法渲染 HTML，有些用户也明确禁用了 HTML 邮件。为了安全起见，我们发送两个版本。

那么发件人的电子邮件地址呢?

我们假设客户端实例发送的所有邮件都来自同一个地址——
因此我们不需要将其作为 `send_email` 的参数，它会作为客户端本身构造函数的参数之一。

我们还希望 `send_email` 是一个异步函数，因为我们将执行 I/O 操作来与远程服务器通信。

将所有内容组合在一起，我们得到的内容大致如下:

```rs
//! src/email_client.rs
use crate::domain::SubscriberEmail;

pub struct EmailClient {
    sender: SubscriberEmail,
}

impl EmailClient {
    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        todo!()
    }
}
```

```rs
//! src/lib.rs

// New entry!
pub mod email_client;
// [...]
```

还有一个未解决的问题——返回类型。我们草拟了一个 `Result<(), String>` 类，这相当于表达了“我稍后再考虑错误处理”的意思。

还有很多工作要做，但这只是一个开始——我们说过要从接口开始，而不是一次性搞定！

## 怎么使用 reqwest 写 REST 客户端

要与 REST API 通信，我们需要一个 HTTP 客户端。

Rust 生态系统中有一些不同的选择：同步 vs 异步、纯 Rust vs 绑定到底层原生库、与 tokio 或 async-std 绑定、固定 vs 高度可定制等等。

我们将选择 crates.io 上最受欢迎的选项：[reqwest](https://crates.io/crates/reqwest)。
关于 reqwest，您有什么想说的吗？

- 它已经过广泛的测试（下载量约 850 万次）
- 它提供了一个主要的异步接口，并可以通过阻止功能标志启用同步接口
- 它依赖于 tokio 作为其异步执行器，与我们已经在使用的 actix-web 兼容
- 如果您选择使用 rustls 来支持 TLS 实现（使用 rustls-tls 功能标志而不是 default-tls），它不依赖于任何系统库，因此具有极高的可移植性。

如果您仔细观察，就会发现我们已经在使用 reqwest！

它是我们在集成测试中用来向 API 发起请求的 HTTP 客户端。让我们将它从开发依赖项提升为运行时依赖项：

```toml
#! Cargo.toml
[dependencies]
# [...]
# We need the `json` feature flag to serialize/deserialize JSON paylaods
reqwest = { version = "0.12.23", default-features = false, features = ["json", "rustls-tls"] }

[dev-dependencies]
# Remove `reqwest`'s entry from this table
```

### reqwest::Client

使用 `reqwest` 时，主要处理的类型是 `reqwest::Client` - 它公开了我们向 REST API 执行请求所需的所有方法。

我们可以通过调用 `Client::new` 获取一个新的客户端实例，或者，如果需要调整默认配置，也可以使用 `Client::builder`。

我们暂时使用 `Client::new`。

让我们向 `EmailClient` 添加两个字段:

- `http_client`, 用于存储 `Client` 实例
- `base_url`, 用于存储我们将要发送请求的 API 的 URL

```rs
//! src/email_client.rs
use reqwest::Client;

use crate::domain::SubscriberEmail;

pub struct EmailClient {
    sender: SubscriberEmail,
    base_url: String,
    http_client: Client,
}

impl EmailClient {
    pub fn new(base_url: String, sender: SubscriberEmail) -> Self {
        Self {
            http_client: Client::new(),
            base_url,
            sender,
        }
    }

    // [...]
}
```

### 连接池

在对远程服务器上托管的 API 执行 HTTP 请求之前，我们需要建立连接。

事实证明，连接是一项相当昂贵的操作，如果使用 HTTPS 连接则更是如此：每次需要发起请求时都创建一个全新的连接会影响应用程序的性能，并可能导致所谓的“负载下套接字耗尽”问题。

为了缓解这个问题，大多数 HTTP 客户端都提供了连接池：在对远程服务器的第一个请求完成后，它们会保持连接打开（一段时间），并在我们需要向同一服务器发起另一个请求时重新使用它，从而避免了重新建立连接。

reqwest 也不例外——每次创建 `Client` 实例时，reqwest 都会在底层初始化一个连接池。

为了利用这个连接池，我们需要在多个请求中重用同一个 `Client`。

还需要指出的是, `Client::clone` 不会创建新的连接池——我们只是克隆一个指向底层连接池的指针。

### 怎么在 actix-web 中复用相同的 reqwest::Client

为了在 actix-web 中的多个请求中重复使用同一个 HTTP 客户端，我们需要在应用上下文中存储它的副本。 ——这样我们就可以使用提取器（例如 `actix_web::web::Data`）在请求处理程序中检索对客户端的引用。

如何实现呢？让我们看一下构建 HttpServer 的代码:

```rs
//! src/startup.rs
// [...]
pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);

    let server = HttpServer::new(move || {
        App::new()
            // Middlewares are added using the `wrap` method on `App`
            .wrap(TracingLogger::default())
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

我们有两个选择:

- 为 `EmailClient` 派生 `Clone` trait，构建一次它的实例，然后在每次需要构建应用时将一个克隆传递给 app_data:

```rs
//! src/email_client.rs
#[derive(Debug)]
pub struct EmailClient {
    sender: SubscriberEmail,
    base_url: String,
    http_client: Client,
}

// [...]
```

```rs
//! src/startup.rs
use crate::email_client::EmailClient;
// [...]

pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
    email_client: EmailClient,
) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);

    let server = HttpServer::new(move || {
        App::new()
            // Middlewares are added using the `wrap` method on `App`
            .wrap(TracingLogger::default())
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
            .app_data(email_client.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

- 将 `EmailClient` 包装在 `actix_web::web::Data` (一个 `Arc` 指针) 中, 并在每次需要构建应用程序时将指针传递给 app_data——就像我们在 PgPool 中所做的那样:

```rs
//! src/startup.rs
use crate::email_client::EmailClient;



pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
    email_client: EmailClient,
) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);
    let email_client = web::Data::new(email_client);

    let server = HttpServer::new(move || {
        App::new()
            // Middlewares are added using the `wrap` method on `App`
            .wrap(TracingLogger::default())
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
            .app_data(email_client.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

哪种方法最好?

如果 EmailClient 只是 Client 实例的包装器，那么第一种方案会更可取——我们避免使用 Arc 两次包装连接池。

但事实并非如此: `EmailClient` 附加了两个数据字段 (`base_url` 和 `sender`)。

第一种实现会在每次创建 App 实例时分配新的内存来保存这些数据的副本，而第二种实现则会在所有 App 实例之间共享这些数据。

这就是我们使用第二种策略的原因。

但请注意: 我们为每个线程创建一个 App 实例——从全局来看，字符串分配 (或指针克隆) 的成本可以忽略不计。

我们在这里将这个决策过程作为一个练习，以了解其中的利弊——将来您可能需要做出类似的决策，届时两种方案的成本可能会有显著差异。

### 配置我们的 EmailClient

如果你运行 `cargo check` 你将会得到如下错误

```plaintext
error[E0061]: this function takes 3 arguments but 2 arguments were supplied
  --> src/main.rs:24:5
   |
24 |     run(listener, connection_pool)?.await
   |     ^^^--------------------------- argument #3 of type `EmailClient` is mis
sing
   |
```

让我们修复它!

`main` 函数现在有什么?

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]

    let configuration = get_configuration().expect("Failed to read config");
    let connection_pool = PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy_with(configuration.database.with_db());

    let address = format!(
        "{}:{}",
        configuration.application.host, configuration.application.port
    );
    let listener = TcpListener::bind(address)?;

    run(listener, connection_pool)?.await
}
```

我们正在使用通过 `get_configuration` 获取的配置中指定的值来构建应用程序的依赖项。

要构建 `EmailClient` 实例，我们需要获取要向其发送请求的 API 的基本 URL 以及发件人的电子邮件地址 - 让我们将它们添加到 `Settings` 结构体中:

```rs
//! src/configuration.rs
// [...]

use crate::domain::SubscriberEmail;

#[derive(serde::Deserialize)]
pub struct Settings {
    pub database: DatabaseSettings,
    pub application: ApplicationSettings,
    pub email_client: EmailClientSettings,
}

#[derive(serde::Deserialize)]
pub struct EmailClientSettings {
    pub base_url: String,
    pub sender_email: String,
}

impl EmailClientSettings {
    pub fn sender(&self) -> Result<SubscriberEmail, String> {
        SubscriberEmail::parse(self.sender_email.clone())
    }
}
```

然后我们需要在配置文件中为它们设置值:

```yaml
#! configuration/base.yaml
application:
  # [...]

database:
  # [...]

email_client:
  base_url: "localhost"
  sender_email: "test@gmail.com"
```

```yaml
#! configuration/production.yaml
application:
  host: 0.0.0.0
database:
  require_ssl: true

email_client:
  # Value retrieved from Postmark's API documentation
  base_url: "https://api.postmarkapp.com"
  sender_email: "something@gmail.com"
```

我们现在可以在 `main` 中构建一个 `EmailClient` 实例并将其传递给 `run` 函数:

```rs
//! src/main.rs
// [...]
use zero2prod::email_client::EmailClient;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let subscriber = get_subscriber("zero2prod", "info", std::io::stdout);
    init_subscriber(subscriber);

    let configuration = get_configuration().expect("Failed to read config");
    let connection_pool = PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy_with(configuration.database.with_db());

    // Build an `EmailClient` using `configuration`
    let sender_email = configuration.email_client.sender()
        .expect("Invalid sender email address");
    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email,
    );

    let address = format!(
        "{}:{}",
        configuration.application.host, configuration.application.port
    );
    let listener = TcpListener::bind(address)?;

    run(listener, connection_pool, email_client)?.await
}
```

`cargo check` 现在应该可以通过了，尽管有一些关于未使用变量的警告——我们很快就会处理它们。
我们的测试怎么样?

`cargo check --all-targets` 返回的错误与我们之前使用 `cargo check` 时看到的类似:

```plaintext
error[E0061]: this function takes 3 arguments but 2 arguments were supplied
  --> tests/health_check.rs:48:18
   |
48 | ...rver = zero2prod::run(listener, connection_pool.clone()).expect("Fail...
   |           ^^^^^^^^^^^^^^----------------------------------- argument #3 of 
type `EmailClient` is missing
   |
```

你说得对——这是代码重复的症状。我们会重构集成测试的初始化逻辑，但目前还不行。

让我们快速打个补丁，让它能编译通过:

```rs
//! tests/health_check.rs
// [...]
use zero2prod::email_client::EmailClient;

async fn spawn_app() -> TestApp {
    // [...]

    // Build a new email client
    let sender_email = configuration.email_client.sender()
        .expect("Invalid sender email address.");
    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email
    );

    let server = zero2prod::run(listener, connection_pool.clone(), email_client)
        .expect("Failed to bind address");
    let _ = tokio::spawn(server);

    TestApp {
        address,
        db_pool: connection_pool,
    }
}
```

`cargo test` 现在应该可以通过了

## 怎么测试一个 REST 客户端

我们已经完成了大部分设置步骤：我们绘制了 EmailClient 的接口，并使用新的配置类型 EmailClientSettings 将其与应用程序连接起来。

为了坚持我们的测试驱动开发方法，现在是时候编写测试了!

我们可以从集成测试开始: 修改 POST /订阅的测试，以确保端点符合我们的新要求。

不过，要让它们通过测试需要很长时间：除了发送电子邮件之外，我们还需要添加逻辑来生成并存储唯一的令牌。
让我们从小处着手：我们将单独测试 EmailClient 组件。

这将增强我们对它在作为一个单元进行测试时是否符合预期的信心，从而减少将其集成到更大的确认电子邮件流程中可能遇到的问题。

这也让我们有机会看看最终的界面是否符合人体工程学且易于测试。

我们实际上应该测试什么?

`EmailClient::send_email` 的主要目的是执行 HTTP 调用：: 我们如何知道
它是否发生了？我们如何检查正文和标头是否按预期填充?

我们需要拦截该 HTTP 请求——是时候启动一个模拟服务器了!

## 使用 wiremock 进行 HTTP 模拟

让我们在 `src/email_client.rs` 的底部添加一个新的测试模块，其中包含一个新测试的框架:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        todo!()
    }
}
```

这不会立即编译 - 我们需要在 Cargo.toml 中向 tokio 添加两个 feature flags:

```toml
[dev-dependencies]
# [...]
tokio = { version = "1", features = ["rt", "macros"] }
```

我们对 `Postmark` 的了解还不够，无法断言我们应该在传出的 `HTTP` 请求中看到什么。

不过，正如测试名称所示，可以合理地预期会向服务器发送一个请求，地址是 `EmailClient::base_url`!

让我们将 `wiremock` 添加到我们的开发依赖项中:

```shell
cargo add wiremock --dev
```

使用 `wiremock`, 我们可以将 `send_email_fires_a_request_to_base_url` 写成如下形式:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    use fake::{faker::{internet::en::SafeEmail, lorem::en::Sentence}, Fake};
    use wiremock::{matchers::any, Mock, MockServer, ResponseTemplate};

    use crate::{domain::SubscriberEmail, email_client::EmailClient};

    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        // Arrange
        let mock_server = MockServer::start().await;
        let sender = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let email_client = EmailClient::new(mock_server.uri(), sender);

        Mock::given(any())
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;

        let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();

        let subject: String = Sentence(1..2).fake();
        let content: String = Sentence(1..10).fake();

        // Act
        let _ = email_client
            .send_email(subscriber_email, &subject, &content, &content)
            .await;

        // Assert
    }
}
```

让我们一步一步分析一下正在发生的事情。

```rs
let mock_server = MockServer::start().await;
```

### wiremock::MockServer

[wiremock::MockServer](https://docs.rs/wiremock/latest/wiremock/struct.MockServer.html) 是一个功能齐全的 HTTP 服务器。

[MockServer::start](https://docs.rs/wiremock/latest/wiremock/struct.MockServer.html#method.start) 会向操作系统请求一个随机可用端口，并在后台线程中启动服务器，准备监听传入的请求。

如何将电子邮件客户端指向模拟服务器？我们可以使用 [`MockServer::uri`](https://docs.rs/wiremock/latest/wiremock/struct.MockServer.html#method.uri) 方法获取模拟服务器的地址；然后将其作为 base_url 传递给 `EmailClient::new`: `let email_client = EmailClient::new(mock_server.uri(), sender);`

### wiremock::Mock

开箱即用，`wiremock::MockServer` 会向所有传入请求返回 404 Not Found。

我们可以通过挂载 Mock 来指示模拟服务器执行不同的操作。

```rs
Mock::given(any())
    .respond_with(ResponseTemplate::new(200))
    .expect(1)
    .mount(&mock_server)
    .await;
```

当 `wiremock::MockServer` 收到请求时，它会遍历所有已挂载的模拟，检查请求是否符合它们的条件。

模拟的匹配条件使用 [`Mock::given`](https://docs.rs/wiremock/latest/wiremock/struct.Mock.html#method.given) 指定。

我们将 `any()` 传递给 `Mock::Given`，根据 wiremock 的文档，

> 匹配所有传入请求，无论其方法、路径、标头或正文如何。您可以使用它来验证请求是否已向服务器发出，而无需对其进行任何其他断言。

基本上，无论请求是什么，它总是匹配的——这正是我们想要的!

当传入的请求符​​合已挂载模拟的条件时，`wiremock::MockServer` 将按照 `respond_with` 中指定的内容返回响应。

我们传递了 `ResponseTemplate::new(200)` - 一个没有正文的 **200 OK** 响应。

`wiremock::Mock` 只有在挂载到 `wiremock::Mockserver` 后才会生效——这就是我们调用 `Mock::mount` 的原因。

### 测试的目的应该明确

然后我们实际调用了 `EmailClient::send_email`:

```rs
let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
let subject: String = Sentence(1..2).fake();
let content: String = Paragraph(1..10).fake();

// Act
let _ = email_client
    .send_email(subscriber_email, &subject, &content, &content)
    .await;
```

你会注意到，我们在这里严重依赖 fake: 我们为 send_email（以及上一节中的 sender）的所有输入生成随机数据。

我们本来可以直接硬编码一堆值，为什么我们要一路硬编码，把它们都变成随机的呢?

读者只需浏览一下测试代码，就应该能够轻松识别我们要测试的属性。

使用随机数据传达了一个特定的信息：不要关注这些输入，它们的值不会影响测试结果，这就是它们随机的原因!

相反，硬编码值应该总是让你犹豫：将 `subscriber_email` 设置为 `marco@gmail.com` 是否重要？如果我将其设置为其他值，测试应该通过吗?

在我们这样的测试中，答案显而易见。但在更复杂的设置中，答案通常并非如此。

### 模拟期望

测试的结尾看起来有点神秘：有一个 // Assert
注释……但后面没有断言。

让我们回到 Mock 设置那一行:

```rs
Mock::given(any())
    .respond_with(ResponseTemplate::new(200))
    .expect(1)
    .mount(&mock_server)
    .await;
```

`.expect(1)` 的作用是什么?
它为我们的模拟设置了一个期望: 我们告诉模拟服务器，在本次测试中，它应该接收恰好一个符合此模拟设置条件的请求。

我们也可以使用范围来设置期望——例如，expect(1..) 表示我们希望至少接收一个请求；expect(1..=3) 表示我们希望接收至少一个请求，但不超过三个请求，等等。

当 MockServer 超出范围时，会验证期望——确实，是在测试函数的末尾!

在关闭之前，MockServer 会迭代所有已挂载的模拟，并检查它们的期望是否已验证。如果验证步骤失败，则会触发 panic（并导致测试失败）。

让我们运行 `cargo test`:

```plaintext
---- email_client::tests::send_email_fires_a_request_to_base_url stdout ----

thread 'email_client::tests::send_email_fires_a_request_to_base_url' panicked at
 src/email_client.rs:27:9:
not yet implemented
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

好的，我们甚至还没有结束测试，因为我们有一个占位符 `todo!()` 作为 send_e`mail 的主体。

让我们用一个虚拟的 `Ok` 来替换它:

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    // [...]

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        // No matter the input
        Ok(())
    }
}
```

如果我们再次运行 `cargo test`, 我们将看到 wiremock 的运行情况:

```plaintext
---- email_client::tests::send_email_fires_a_request_to_base_url stdout ----

thread 'email_client::tests::send_email_fires_a_request_to_base_url' panicked at
 /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/wiremock-0.6
.5/src/mock_server/exposed_server.rs:367:17:
Verifications failed:
- Mock #0.
        Expected range of matching incoming requests: == 1
        Number of matched incoming requests: 0

The server did not receive any request.
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    email_client::tests::send_email_fires_a_request_to_base_url
```

服务器预期收到一个请求，但实际上没有收到，因此测试失败。

现在是时候完善 `EmailClient::send_email` 了。

## EmailClient::send_email 的第一个草图

要实现 `EmailClient::send_email`, 我们需要查看 Postmark 的 [API 文档](https://postmarkapp.com/developer/user-guide/send-email-with-api#send-a-single-email)。

让我们从他们的[“发送单封邮件”用户指南](https://postmarkapp.com/developer/user-guide/send-email-with-api#send-a-single-email)开始。

他们的邮件发送示例如下:

```shell
curl "https://api.postmarkapp.com/email" \
  -X POST \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -H "X-Postmark-Server-Token: server token" \
  -d '{
  "From": "sender@example.com",
  "To": "receiver@example.com",
  "Subject": "Postmark test",
  "TextBody": "Hello dear Postmark user.",
  "HtmlBody": "<html><body><strong>Hello</strong> dear Postmark user.</body></html>",
  "MessageStream": "outbound"
}'
```

让我们分解一下 - 要发送电子邮件，我们需要:

- 向 /email 端点发送一个 POST 请求
- 一个 JSON 主体，其中包含与 `send_email` 参数紧密映射的字段。我们需要谨慎使用字段名称，必须使用 Pascal-case
- 一个授权标头 `X-Postmark-Server-Token`，其值设置为一个我们可以从其门户网站检索的秘密令牌

如果请求成功，我们会收到类似如下的返回信息:

```plaintext
HTTP/1.1 200 OK
Content-Type: application/json

{
    "To": "receiver@example.com",
    "SubmittedAt": "2021-01-12T07:25:01.4178645-05:00",
    "MessageID": "0a129aee-e1cd-480d-b08d-4f48548ff48d",
    "ErrorCode": 0,
    "Message": "OK"
}
```

我们有足够的资源来实现happy path！

### reqwest::Client::post

`reqwest::Client` 公开了一个 `post` 方法 - 它接受我们想要通过 POST 请求调用的 URL 作为参数，并返回一个 [RequestBuilder](https://docs.rs/reqwest/latest/reqwest/struct.RequestBuilder.html)。

[RequestBuilder](https://docs.rs/reqwest/latest/reqwest/struct.RequestBuilder.html) 为我们提供了一个流畅的 API，让我们可以逐步构建我们想要发送的其余请求。

让我们尝试一下:

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    // [...]

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        // You can do better using `reqwest::Url::join` if you change
        // `base_url`'s type from `String' to `reqwest::Url`.
        // I'll leave it as an exercise for the reader!
        let url = format!("{}/email", self.base_url);
        let builder = self.http_client.post(&url);
        Ok(())
    }
}
```

### JSON Body

我们可以将请求 body 编码为结构体:

```rs
//! src/email_client.rs
// [...]


impl EmailClient {
    // [...]

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        let url = format!("{}/email", self.base_url);
        let request_body = SendEmailRequest {
            from: self.sender.as_ref().to_owned(),
            to: recipient.as_ref().to_owned(),
            subject: subject.to_owned(),
            html_body: html_content.to_owned(),
            text_body: text_content.to_owned(),
        };
        let builder = self.http_client.post(&url);
        Ok(())
    }
}

struct SendEmailRequest {
    from: String,
    to: String,
    subject: String,
    html_body: String,
    text_body: String,
}
```

如果启用了 `reqwest` 的 `json` 功能标志（就像我们所做的那样）, `builder` 将公开一个 `json` 方法，我们可以利用该方法将 `request_body` 设置为请求的 JSON 主体:

```rs
impl EmailClient {
    // [...]

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        let url = format!("{}/email", self.base_url);
        let request_body = SendEmailRequest {
            from: self.sender.as_ref().to_owned(),
            to: recipient.as_ref().to_owned(),
            subject: subject.to_owned(),
            html_body: html_content.to_owned(),
            text_body: text_content.to_owned(),
        };
        let builder = self.http_client.post(&url).json(&request_body);
        Ok(())
    }
}
```

就差一点了!

```plaintext
error[E0277]: the trait bound `SendEmailRequest: configuration::_::_serde::Seria
lize` is not satisfied
   --> src/email_client.rs:38:56
    |
38  |         let builder = self.http_client.post(&url).json(&request_body);
    |                                                   ---- ^^^^^^^^^^^^^ unsatisfied trait bound
```

让我们为 `SendEmailRequest` 派生 `serde::Serialize` 以使其可序列化:

```rs
//! src/email_client.rs
// [...]

#[derive(serde::Serialize)]
struct SendEmailRequest {
    from: String,
    to: String,
    subject: String,
    html_body: String,
    text_body: String,
}
```

太棒了，编译通过了!
json 方法比简单的序列化更进一步：它还会将 `Content-Type` 标头设置为 `application/json` —— 与我们在示例中看到的一致!

### 鉴权令牌

快完成了——我们需要在请求中添加一个授权标头, `X-Postmark-Server-Token`。
就像发件人邮箱地址一样, 我们希望将令牌值作为字段存储在 `EmailClient` 中。

让我们修改 `EmailClient::new` 和 `EmailClientSettings`:

```rs
//! src/email_client.rs
use secrecy::SecretBox;

// [...]

pub struct EmailClient {
    // [...]
    // We don't want to log this by accident
    authorization_token: SecretBox<String>,
}

impl EmailClient {
    pub fn new(
        base_url: String,
        sender: SubscriberEmail,
        authorization_token: SecretBox<String>,
    ) -> Self {
        Self {
            // [...]
            authorization_token,
        }
    }
}
```

```rs
//! src/configuration.rs
// [...]
#[derive(serde::Deserialize)]
pub struct EmailClientSettings {
    // [...]
    // New (secret) configuration value!
    pub authorization_token: SecretBox<String>,
}
```

然后我们可以让编译器告诉我们还需要修改什么:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    use fake::{
        faker::{internet::en::SafeEmail, lorem::en::Sentence}, Fake, Faker
    };
    use secrecy::SecretBox;
    use wiremock::{Mock, MockServer, ResponseTemplate, matchers::any};

    use crate::{domain::SubscriberEmail, email_client::EmailClient};

    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        // Arrange
        let mock_server = MockServer::start().await;
        let sender = SubscriberEmail::parse(SafeEmail().fake()).unwrap();

        // New argument!
        let email_client = EmailClient::new(
            mock_server.uri(),
            sender,
            SecretBox::new(Faker.fake()),
        );

        // [...]
    }
}
```

```rs
//! src/main.rs
// [...]
#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]

    let configuration = get_configuration().expect("Failed to read config");

    // [...]
    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email,
        configuration.email_client.authorization_token,
    );

    // [...]
}
```

```yaml
#! configuration/base.yml
# [...]
email_client:
  base_url: "localhost"
  sender_email: "test@gmail.com"
  # New value!
  # We are only setting the development value,
  # we'll deal with the production token outside of version control
  # (given that it's a sensitive secret!)
  authorization_token: "my-secret-token"
```

我们现在可以在 send_email 中使用授权令牌:

```rs
//! src/email_client.rs
use secrecy::{ExposeSecret, SecretBox};

// [...]

impl EmailClient {
    // [...]

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        // [...]
        let builder = self
            .http_client
            .post(&url)
            .header(
                "X-Postmark-Server-Token",
                self.authorization_token.expose_secret(),
            )
            .json(&request_body);
        Ok(())
    }
}
```

它立即编译通过。

### 执行请求

我们已准备好所有材料 - 我们只需要立即发出请求!

我们可以使用 `send` 方法:

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), String> {
        // [...]
        self
            .http_client
            .post(&url)
            .header(
                "X-Postmark-Server-Token",
                self.authorization_token.expose_secret(),
            )
            .json(&request_body)
            .send()
            .await?;
        Ok(())
    }
}
```

`send` 是异步的，因此我们需要等待它返回的 `Future`。

send 也是一个容易出错的操作——例如，我们可能无法与服务器建立连接。我们希望在 `send` 失败时返回一个错误——这就是为什么我们使用 `?` 运算符。

然而，编译器并不满意:

```plaintext
error[E0277]: `?` couldn't convert the error to `std::string::String`
  --> src/email_client.rs:52:19
   |
43 | /         self
44 | |             .http_client
45 | |             .post(&url)
46 | |             .header(
...  |
51 | |             .send()
52 | |             .await?;
   | |                  -^ unsatisfied trait bound
   | |__________________|
```

`send` 返回的错误变量类型为 `reqwest::Error`,而我们的 `send_email` 使用 `String` 作为错误类型。编译器查找了转换 (`From trait` 的实现), 但找不到，因此报错。

还记得吗，我们使用 `String` 作为错误变量主要是为了占位符——让我们将 `send_email` 的签名改为返回 `Result<(), reqwest::Error>`。

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    // [...]

    pub async fn send_email(
        // [...]
    ) -> Result<(), reqwest::Error> {
        // [...]
    }
}
```

现在错误应该消失了!

`cargo test` 也应该通过: 恭喜!

注: 测试中的方法签名错误需要你自己修复, 本教程没有提及, 我相信你可以做到的

## 加强 happy path 测试

让我们再次看一下我们的“happy path”测试:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    use fake::{
        Fake, Faker,
        faker::{internet::en::SafeEmail, lorem::en::Sentence},
    };
    use secrecy::SecretBox;
    use wiremock::{Mock, MockServer, ResponseTemplate, matchers::any};

    use crate::{domain::SubscriberEmail, email_client::EmailClient};

    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        // Arrange
        let mock_server = MockServer::start().await;
        let sender = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let email_client =
            EmailClient::new(mock_server.uri(), sender, SecretBox::new(Faker.fake()));

        Mock::given(any())
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;

        let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();

        let subject: String = Sentence(1..2).fake();
        let content: String = Sentence(1..10).fake();

        // Act
        let _ = email_client
            .send_email(subscriber_email, &subject, &content, &content)
            .await;

        // Assert
    }
}
```

为了轻松上手 `wiremock`，我们从一些非常基础的操作开始——我们只是断言模拟服务器被调用了一次。接下来，让我们进一步完善它，检查发出的请求是否确实符合我们的预期。

### Headers, Path 和 Method

`any` 并非 `wiremock` `提供的唯一开箱即用的匹配器：wiremock` 的 `matchers` 模块中还有许多其他可用的匹配器。

我们可以使用 `header_exists` 来验证 `X-Postmark-Server-Token` 是否已在发送至服务器的请求中设置:

```rs
//! src/email_client.rs
// [...]
#[cfg(test)]
mod tests {
    use fake::{
        Fake, Faker,
        faker::{internet::en::SafeEmail, lorem::en::Sentence},
    };
    use secrecy::SecretBox;
    use wiremock::{matchers::{any, header_exists}, Mock, MockServer, ResponseTemplate};

    use crate::{domain::SubscriberEmail, email_client::EmailClient};

    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        Mock::given(header_exists("X-Postmark-Server-Token"))
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;
    }
}
```

我们可以使用 `and` 方法将多个匹配器连接在一起。

让我们添加 [`header`](https://docs.rs/wiremock/latest/wiremock/matchers/fn.header.html) 来检查 `Content-Type` 是否设置为正确的值，添加 [`path`](https://docs.rs/wiremock/latest/wiremock/matchers/fn.path.html) 来在被调用的端点上进行断言，以及添加 [`method`](https://docs.rs/wiremock/latest/wiremock/matchers/fn.method.html) 来验证 HTTP 方法:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    use wiremock::matchers::{header, header_exists, method, path}

    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        // [...]

        Mock::given(header_exists("X-Postmark-Server-Token"))
            .and(header("Content-Type", "application/json"))
            .and(path("/email"))
            .and(method("POST"))
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;

        // [...]
    }
}
```

### Body

到目前为止，一切顺利: `cargo test` 仍然通过。

那么请求体呢?

我们可以使用 body_json 来精确匹配请求体。

我们可能不需要那么深入——只要检查请求体是否为有效的 JSON 格式，并且包含 Postmark 示例中所示的字段名称集合就足够了。

目前没有现成的匹配器能满足我们的需求——我们需要自己实现!

`wiremock` 暴露了一个 `Match trait`——所有实现该 `trait` 的程序都可以在给定的匹配器中用作匹配器。

让我们把它存根掉:

```rs
//! src/email_client.rs
// [...]
#[cfg(test)]
mod tests {
    // [...]

    struct SendEmailBodyMatcher;

    impl wiremock::Match for SendEmailBodyMatcher {
        fn matches(&self, request: &wiremock::Request) -> bool {
            unimplemented!();
        }
    }

    // [...]
}
```

我们将传入的请求作为输入 `request`, 并需要返回一个布尔值作为输出：
如果模拟匹配，则返回 `true`; 否则返回 `false`。

我们需要将请求主体反序列化为 JSON——让我们将 `serde_json` 添加到我们的开发依赖项列表中:

```shell
cargo add serde_json --dev
```

现在我们可以编写 `matches` 的实现:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    struct SendEmailBodyMatcher;

    impl wiremock::Match for SendEmailBodyMatcher {
        fn matches(&self, request: &wiremock::Request) -> bool {
            // Try to parse the body as a JSON value
            let result: Result<serde_json::Value, _> = 
                serde_json::from_slice(&request.body);
            if let Ok(body) = result {
                // Check that all the mandatory fields are populated
                // without inspecting the field values
                body.get("From").is_some()
                    && body.get("To").is_some()
                    && body.get("Subject").is_some()
                    && body.get("HtmlBody").is_some()
                    && body.get("TextBody").is_some()
            } else {
                // If parsing failed, do not match the request
                false
            }
        }
    }

    #[tokio::test]
    async fn send_email_fires_a_request_to_base_url() {
        // [...]

        Mock::given(header_exists("X-Postmark-Server-Token"))
            .and(header("Content-Type", "application/json"))
            .and(path("/email"))
            .and(method("POST"))
            // Use our custom matcher!
            .and(SendEmailBodyMatcher)
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;

        // [...]
    }
}
```

编译成功了！

但是我们的测试现在失败了...

```plaintext
---- email_client::tests::send_email_fires_a_request_to_base_url stdout ----

thread 'email_client::tests::send_email_fires_a_request_to_base_url' panicked at
 /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/wiremock-0.6
.5/src/mock_server/exposed_server.rs:367:17:
Verifications failed:
- Mock #0.
        Expected range of matching incoming requests: == 1
        Number of matched incoming requests: 0

Received requests:
- Request #1
        POST http://localhost/email
x-postmark-server-token: HjhDDhi05zX5gNM
content-type: application/json
accept: */*
host: 127.0.0.1:43849
content-length: 138
{"from":"joaquin@example.net","to":"weldon@example.com","subject":"fuga.","html_
body":"nam ut esse amet.","text_body":"nam ut esse amet."}
```

为什么会这样?

我们好像忘了大小写要求——字段名称必须使用 Pascal 大小写!

我们可以通过在 `SendEmailRequest` 上添加注解来轻松解决这个问题!

```rs
//! src/email_client.rs
// [...]

#[derive(serde::Serialize)]
#[serde(rename_all = "PascalCase")]
struct SendEmailRequest {
    from: String,
    to: String,
    subject: String,
    html_body: String,
    text_body: String,
}
```

测试现在应该可以通过了。

在继续下一步之前，让我们将测试重命名为 `send_email_sends_the_expected_request` —— 这样可以更好地体现测试目的。

### 重构：避免不必要的内存分配

我们专注于让 `send_email` 正常工作——现在我们可以再次检查一下, 看看是否还有改进的空间。

让我们放大查看请求正文:

```rs
//! src/email_client.rs
// [...]
impl EmailClient {
    pub fn new(
        base_url: String,
        sender: SubscriberEmail,
        authorization_token: SecretBox<String>,
    ) -> Self {
        Self {
            http_client: Client::new(),
            base_url,
            sender,
            authorization_token,
        }
    }

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), reqwest::Error> {
        // [...]
        let request_body = SendEmailRequest {
            from: self.sender.as_ref().to_owned(),
            to: recipient.as_ref().to_owned(),
            subject: subject.to_owned(),
            html_body: html_content.to_owned(),
            text_body: text_content.to_owned(),
        };
        // [...]
    }
}

#[derive(serde::Serialize)]
#[serde(rename_all = "PascalCase")]
struct SendEmailRequest {
    from: String,
    to: String,
    subject: String,
    html_body: String,
    text_body: String,
}
```

对于每个字段，我们都会分配一大堆新内存来存储克隆的字符串——这很浪费。更高效的方法是引用现有数据而不执行任何额外的分配。

我们可以通过重构 `SendEmailRequest` 来实现这一点：所有字段的类型都改为字符串切片 (&str)，而不是字符串。

字符串切片只是一个指向他人拥有的内存缓冲区的指针。为了将引用存储在结构体中，我们需要添加一个生命周期参数: 它会跟踪这些引用的有效期——编译器的职责是确保引用的保留时间不超过它们指向的内存缓冲区!

我们开始吧!

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    pub fn new(
        base_url: String,
        sender: SubscriberEmail,
        authorization_token: SecretBox<String>,
    ) -> Self {
        Self {
            http_client: Client::new(),
            base_url,
            sender,
            authorization_token,
        }
    }

    pub async fn send_email(
        &self,
        recipient: SubscriberEmail,
        subject: &str,
        html_content: &str,
        text_content: &str,
    ) -> Result<(), reqwest::Error> {
        // [...]
        let request_body = SendEmailRequest {
            from: self.sender.as_ref(),
            to: recipient.as_ref(),
            subject: subject,
            html_body: html_content,
            text_body: text_content,
        };
        // [...]
    }
}

#[derive(serde::Serialize)]
#[serde(rename_all = "PascalCase")]
// Lifetime parameters always start with an apostrophe, `'`
struct SendEmailRequest<'a> {
    from: &'a str,
    to: &'a str,
    subject: &'a str,
    html_body: &'a str,
    text_body: &'a str,
}
```

就是这样，快速又轻松——Serde 为我们完成了所有繁重的工作，我们得到了更高性能的代码！

## 处理失败

我们已经很好地掌握了最佳路径——如果事情没有按预期进行，会发生什么？

我们将讨论两种情况:

- 不成功状态代码（例如 4xx、5xx 等）
- 响应缓慢

### 错误状态码

我们当前的 happy path 测试仅对 `send_email` 执行的副作用进行断言——我们实际上并没有检查它的返回值!

让我们确保如果服务器返回 200 OK，它就是 `Ok(())`:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    // New happy-path test!
    #[tokio::test]
    async fn send_email_succeeds_if_the_server_returns_200() {
        // Arrange
        let mock_server = MockServer::start().await;
        let sender = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let email_client = EmailClient::new(
            mock_server.uri(),
            sender,
            SecretBox::new(Box::new(Faker.fake()))
        );

        let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let subject: String = Sentence(1..2).fake();
        let content: String = Paragraph(1..10).fake();

        // We do not copy in all the matchers we have in the other test.
        // The purpose of this test is not to assert on the request we
        // are sending out!
        // We add the bare minimum needed to trigger the path we want
        // to test in `send_email`.
        Mock::given(any())
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;

        // Act
        let outcome = email_client
            .send_email(subscriber_email, &subject, &content, &content)
            .await;

        // Assert
        assert_ok!(outcome);
    }
}
```

不出所料，测试通过了。

现在让我们看看相反的情况——如果服务器返回 `500 Internal Server Error`, 我们预期会出现 Err。

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    #[tokio::test]
    async fn send_email_fails_of_the_server_returns_500() {
        // Arrange
        let mock_server = MockServer::start().await;
        let sender = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let email_client = EmailClient::new(
            mock_server.uri(),
            sender,
            SecretBox::new(Box::new(Faker.fake())),
        );
        
        let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let subject: String = Sentence(1..2).fake();
        let content: String = Paragraph(1..10).fake();
        
        Mock::given(any())
            // Not a 200 anymore!
            .respond_with(ResponseTemplate::new(500))
            .expect(1)
            .mount(&mock_server)
            .await;

        // Act
        let outcome = email_client
            .send_email(subscriber_email, &subject, &content, &content)
            .await;

        // Assert
        assert_err!(outcome);
    }
}
```

我们这里还有一些工作要做:

```plaintext
test email_client::tests::send_email_fails_of_the_server_returns_500 ... FAILED

failures:

---- email_client::tests::send_email_fails_of_the_server_returns_500 stdout ----

thread 'email_client::tests::send_email_fails_of_the_server_returns_500' panicke
d at src/email_client.rs:195:9:
assertion failed, expected Err(..), got Ok(())
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

让我们再看看 `send_email`:

```rs
//! src/email_client.rs
// [...]
impl EmailClient {
    // [...]

    pub async fn send_email(
        // [...]
    ) -> Result<(), reqwest::Error> {
        let url = format!("{}/email", self.base_url);
        let request_body = SendEmailRequest {
            from: self.sender.as_ref(),
            to: recipient.as_ref(),
            subject: subject,
            html_body: html_content,
            text_body: text_content,
        };
        self.http_client
            .post(&url)
            .header(
                "X-Postmark-Server-Token",
                self.authorization_token.expose_secret(),
            )
            .json(&request_body)
            .send()
            .await?;
        Ok(())
    }
}
```

唯一可能返回错误的步骤是 `send` - 让我们检查 reqwest 的文档!

> 如果发送请求时出现错误、检测到重定向循环或重定向限制已用尽，则此方法失败。

基本上，只要从服务器收到有效响应, `send` 就会返回 `Ok` ——无论状态码是什么!

为了获得我们想要的行为，我们需要查看 `reqwest::Response` 中可用的方法——特别是 `error_for_status`:

> 如果服务器返回错误，则将响应转变为错误。

它似乎符合我们的需要，让我们尝试一下。

```rs
//! src/email_client.rs
// [...]
impl EmailClient {
    // [...]

    pub async fn send_email(
        // [...]
    ) -> Result<(), reqwest::Error> {
        // [...]
        self.http_client
            .post(&url)
            .header(
                "X-Postmark-Server-Token",
                self.authorization_token.expose_secret(),
            )
            .json(&request_body)
            .send()
            .await?
            .error_for_status()?;
        Ok(())
    }
}
```

太棒了，测试通过了!

### 超时

如果服务器返回 200 OK，但需要很长时间才能返回，会发生什么情况?

我们可以指示我们的模拟服务器等待一段可配置的时间后再发送响应。

让我们用一个新的集成测试稍微试验一下——如果服务器需要 3 分钟才能响应怎么办!?

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    // [...]

    #[tokio::test]
    async fn send_email_times_out_if_the_server_takes_too_long() {
        // Arrange
        let mock_server = MockServer::start().await;
        let sender = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let email_client = EmailClient::new(
            mock_server.uri(),
            sender,
            SecretBox::new(Box::new(Faker.fake())),
        );

        let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();
        let subject: String = Sentence(1..2).fake();
        let content: String = Paragraph(1..10).fake();

        let response = ResponseTemplate::new(200)
            // 3 minutes!
            .set_delay(std::time::Duration::from_secs(180));
        Mock::given(any())
            .respond_with(response)
            .expect(1)
            .mount(&mock_server)
            .await;

        // Act
        let outcome = email_client
            .send_email(subscriber_email, &subject, &content, &content)
            .await;

        // Assert
        assert_err!(outcome);
    }
}
```

过了一会儿，你应该会看到类似这样的内容:

```plaintext
test email_client::tests::send_email_times_out_if_the_server_takes_too_long has 
been running for over 60 seconds
```

这远非理想情况：如果服务器开始出现问题，我们可能会开始积累多个“挂起”的请求。

我们并没有挂起服务器，所以连接处于繁忙状态：每次我们需要发送电子邮件时, 我们都必须打开一个新的连接。如果服务器恢复速度不够快，而我们又没有关闭任何打开的连接，最终可能会导致套接字耗尽/性能下降。

经验法则：每次执行 IO 操作时，务必设置超时!

如果服务器响应时间超过超时时间，我们就应该失败并返回错误。

选择合适的超时值通常更像是一门艺术而非科学，尤其是在涉及重试的情况下: 设置得太低，可能会使服务器不堪重负; 设置得太高，则可能会再次面临客户端性能下降的风险。

尽管如此，设置一个保守的超时阈值总比没有好。

`reqwest` 提供了两种选择: 我们可以在客户端本身添加一个默认超时时间，该时间适用于所有发出的请求；或者，我们可以为每个请求指定一个超时时间。

我们来设置一个客户端范围的超时时间: 我们将在 `EmailClient::new` 中设置它。

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    pub fn new(
        base_url: String,
        sender: SubscriberEmail,
        authorization_token: SecretBox<String>,
    ) -> Self {
        let http_client = Client::builder()
            .timeout(std::time::Duration::from_secs(10))
            .build()
            .unwrap();
        Self {
            http_client,
            base_url,
            sender,
            authorization_token,
        }
    }
}

// [...]
```

如果我们再次运行测试，它应该会通过（10秒后）。

### 重构：测试辅助函数

四个 EmailClient 测试中有很多重复的代码，让我们从一组测试助手中提取出其中的共同点。

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    // [...]

    /// Generate a ranodm email subject
    fn subject() -> String {
        Sentence(1..2).fake()
    }

    /// Generate a random email content
    fn content() -> String {
        Paragraph(1..10).fake()
    }

    /// Generate a ranadom subscriber email
    fn email() -> SubscriberEmail {
        SubscriberEmail::parse(SafeEmail().fake()).unwrap()
    }

    /// Get a test instance of `EmailClient`.
    fn email_client(base_url: String) -> EmailClient {
        EmailClient::new(base_url, email(), SecretBox::new(Box::new(Faker.fake())))
    }

    // [...]
}
```

让我们在 `send_email_sends_the_expected_request` 中使用它们:

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    // [...]

    #[tokio::test]
    async fn send_email_sends_the_expected_request() {
        // Arrange
        let mock_server = MockServer::start().await;
        let email_client = email_client(mock_server.uri());

        Mock::given(header_exists("X-Postmark-Server-Token"))
            .and(header("Content-Type", "application/json"))
            .and(path("/email"))
            .and(method("POST"))
            .and(SendEmailBodyMatcher)
            .respond_with(ResponseTemplate::new(200))
            .expect(1)
            .mount(&mock_server)
            .await;

        let subscriber_email = SubscriberEmail::parse(SafeEmail().fake()).unwrap();

        let subject: String = Sentence(1..2).fake();
        let content: String = Sentence(1..10).fake();

        // Act
        let _ = email_client
            .send_email(subscriber_email, &subject, &content, &content)
            .await;

        // Assert
    }

}
```

视觉干扰更少——测试的目的才是核心。

继续重构其他三个!

注: 我相信你知道怎么重构另外三个测试了, 所以这里没有列出详细的代码。

### 重构: 让测试失败的快一点

我们的 HTTP 客户端的超时时间目前硬编码为 10 秒

```rs
//! src/email_client.rs
// [...]

impl EmailClient {
    pub fn new(
        base_url: String,
        sender: SubscriberEmail,
        authorization_token: SecretBox<String>,
    ) -> Self {
        let http_client = Client::builder()
            .timeout(std::time::Duration::from_secs(10))
        // [...]
    }

    // [...]
}
```

这意味着我们的超时测试大约需要 10 秒才会失败——这是一个很长的时间，尤其是如果你在每次小改动后都要运行测试的话。

让我们将超时阈值设置为可配置，以保持测试套件的响应速度。

```rs
//! src/email_client.rs
// [...]
impl EmailClient {
    pub fn new(
        base_url: String,
        sender: SubscriberEmail,
        authorization_token: SecretBox<String>,
        // New argument!
        timeout: std::time::Duration,
    ) -> Self {
        let http_client = Client::builder()
            .timeout(timeout)
        // [...]
    }
}
```

```rs
//! src/configuration.rs
// [...]

#[derive(serde::Deserialize)]
pub struct EmailClientSettings {
    // [...]
    // New configuration value!
    pub timeout_milliseconds: u64,
}

impl EmailClientSettings {
    // [...]

    pub fn timeout(&self) -> std::time::Duration {
        std::time::Duration::from_millis(self.timeout_milliseconds)
    }
}
```

```rs
//! src/main.rs
// [...]
#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let sender_email = configuration.email_client.sender()
        .expect("Invalid sender email address");
    let timeout = configuration.email_client.timeout();

    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email,
        configuration.email_client.authorization_token,
        timeout,
    );

    // [...]
}
```

```yaml
#! configuration/base.yaml
# [...]
email_client:
  # [...]
  timeout_milliseconds: 10000
```

项目应该可以编译了。

不过我们仍然需要编辑测试!

```rs
//! src/email_client.rs
// [...]

#[cfg(test)]
mod tests {
    // [...]
    fn email_client(base_url: String) -> EmailClient {
        EmailClient::new(
            base_url,
            email(),
            SecretBox::new(Box::new(Faker.fake())),
            // Much lower than 10s!
            std::time::Duration::from_millis(200),
        )
    }
}
```

```rs
//! tests/health_check.rs
// [...]
async fn spawn_app() -> TestApp {
    // [...]
    let sender_email = configuration
        .email_client
        .sender()
        .expect("Invalid sender email address.");
    let timeout = configuration.email_client.timeout();
    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email,
        SecretBox::new(Box::new(Faker.fake())),
        timeout,
    );

    // [...]
}
```

所有测试都应该成功——并且整个测试套件的总执行时间应该缩短到一秒以内。
