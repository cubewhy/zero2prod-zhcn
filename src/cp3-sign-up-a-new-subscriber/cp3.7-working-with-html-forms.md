# 使用HTML表单

## 完善我们的要求

为了将访客注册为我们的电子邮件简报，我们应该收集哪些信息?

嗯，我们当然需要他们的电子邮件地址(毕竟这是一封电子邮件简报)。
还有什么?

在典型的业务环境中，这通常会引发团队工程师和产品经理之间的对话。在这种情况下，我们既是技术主管，又是产品负责人，因此我们可以发号施令!

从个人经验来看，人们在订阅新闻通讯时通常会使用一次性或屏蔽电子邮件(或者，至少你们大多数人在订阅“[从零到生产](https://www.lpalmieri.com/subscribe/)”时都是这样做的!)。

因此，收集一个名字会很不错，我们可以用它来作为电子邮件问候语(比如臭名昭著的 **"Hey,{{subscriber.name}}!"** )，也可以在订阅者列表中识别我们认识的人或共同订阅者。

我们不是警察，我们对姓名字段的真实性不感兴趣——我们会让人们在我们的新闻通讯系统中输入他们喜欢的任何身份信息: [DenverCoder9](https://xkcd.com/979/), 我们欢迎你。

那么，事情就解决了：我们需要为所有新订阅者提供电子邮件地址和姓名。

鉴于数据是通过 HTML 表单收集的，它将通过 POST 请求的正文传递给我们的后端 API。正文将如何编码?

使用 HTML 表单时，有几种可用的编码方式: `application/x-www-form-urlencoded`

这最适合我们的用例。

引用 MDN Web 文档，使用 `application/x-www-form-urlencoded`

> 在我们的表单中，键和值被编码为键值元组，并以“&”分隔，键和值之间用“=”分隔。键和值中的非字母数字字符均采用百分号编码。

例如：如果名字是 `Le Guin`, 邮箱是 `ursula_le_guin@gmail.com`，则 POST 请求体应为 `name=le%20guin&email=ursula_le_guin%40gmail.com`(空格替换为 `%20`，而 `@` 替换为 `%40` - 可在[此处](https://www.w3schools.com/tags/ref_urlencode.ASP)找到参考转换表)。

总结:

- 如果使用 `application/x-www-form-urlencoded` 格式提供了有效的姓名和邮箱地址组合，后端应返回 **200 OK**
- 如果姓名或邮箱地址缺失，后端应返回 **400 BAD REQUEST**。

## 将我们的需求作为测试

现在我们更好地理解了需要做什么，让我们将我们的期望编码到几个集成测试中。

让我们将新的测试添加到现有的 `tests/health_check.rs` 文件中——之后我们将重新组织测试套件的文件夹结构。

```rs
//! tests/health_check.rs

// [...]

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // Arrange
    let app_address = spawn_app();
    let client = reqwest::Client::new();
    // Act
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";
    let response = client
        .post(&format!("{}/subscriptions", &app_address))
        .header("Content-Type", "application/x-www-form-urlencoded")
        .body(body)
        .send()
        .await
        .expect("Failed to execute request.");

    // Assert
    assert_eq!(200, response.status().as_u16());
}

#[tokio::test]
async fn subscribe_returns_a_400_when_data_is_missing() {
    // Arrange
    let app_address = spawn_app();
    let client = reqwest::Client::new();
    let test_cases = vec![
        ("name=le%20guin", "missing the email"),
        ("email=ursula_le_guin%40gmail.com", "missing the name"),
        ("", "missing both name and email"),
    ];
    for (invalid_body, error_message) in test_cases {
        // Act
        let response = client
            .post(&format!("{}/subscriptions", &app_address))
            .header("Content-Type", "application/x-www-form-urlencoded")
            .body(invalid_body)
            .send()
            .await
            .expect("Failed to execute request.");

        // Assert
        assert_eq!(
            400,
            response.status().as_u16(),
            // Additional customised error message on test failure
            "The API did not fail with 400 Bad Request when the payload was {}.",
            error_message
        );
    }
}
```

`subscribe_returns_a_400_when_data_is_missing` 是一个表驱动测试的例子，也称为参数化测试。

它在处理错误输入时尤其有用——我们不必多次重复测试逻辑，只需对一组已知无效的主体运行相同的断言即可，这些主体我们预计会以相同的方式失败。

对于参数化测试，在失败时提供清晰的错误消息非常重要：如果您无法确定哪个特定的输入出错，那么在 XYZ 行的断言失败就不太理想！另一方面，该参数化测试涵盖的内容很广泛，因此花更多时间来生成清晰的失败消息是有意义的。

其他语言的测试框架有时原生支持这种测试风格 (例如 [pytest 中的参数化测试](https://docs.pytest.org/en/stable/how-to/parametrize.html)，或 [C# 的 xUnit 中的 InlineData](https://docs.pytest.org/en/stable/parametrize.html))。Rust 生态系统中有一些 crate，它们扩展了基本测试框架，并提供了类似的功能，但遗憾的是，它们与 `#[tokio::test]` 宏的互操作性不佳，而我们需要使用宏来编写惯用的异步测试 (参见 [rstest](https://github.com/la10736/rstest/issues/85) 或 [test-case](https://github.com/frondeus/test-case/issues/36))。

现在让我们运行测试套件：

```plaintext
running 3 tests
test subscribe_returns_a_200_for_valid_form_data ... FAILED
test subscribe_returns_a_400_when_data_is_missing ... FAILED
test health_check_works ... ok

failures:

---- subscribe_returns_a_200_for_valid_form_data stdout ----

thread 'subscribe_returns_a_200_for_valid_form_data' panicked at tests/health_check.rs:39:5:
assertion `left == right` failed
  left: 200
 right: 404
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- subscribe_returns_a_400_when_data_is_missing stdout ----

thread 'subscribe_returns_a_400_when_data_is_missing' panicked at tests/health_check.rs:63:9:
assertion `left == right` failed: The API did not fail with 400 Bad Request when the payload was missing the email.
  left: 400
 right: 404


failures:
    subscribe_returns_a_200_for_valid_form_data
    subscribe_returns_a_400_when_data_is_missing

test result: FAILED. 1 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s

error: test failed, to rerun pass `--test health_check`
```

正如预期的那样，我们所有的新测试都失败了。

你很快就能发现“自行开发”参数化测试的一个局限性: 一旦一个测试用例失败，执行就会停止，我们也无法知道后续测试用例的结果。

让我们开始实现吧。

## 从 POST 请求解析表单数据

所有测试都失败了，因为应用程序在 POST 请求到达 /subscriptions 时返回了 404 NOT FOUND 错误。这是合理的：我们没有为该路径注册处理程序。

让我们通过在 src/lib.rs 中添加一个匹配的路由来解决这个问题:

```rs
//! src/lib.rs

use std::net::TcpListener;

use actix_web::{App, HttpResponse, HttpServer, Responder, dev::Server, web};

// We were returning `impl Responder` at the very beginning.
// We are now spelling out the type explicitly given that we have
// become more familiar with `actix-web`.
// There is no performance difference! Just a stylistic choice :)
async fn health_check() -> impl Responder {
    HttpResponse::Ok()
}

// Let's start simple: we always return a 200 OK
async fn subscribe() -> HttpResponse {
    HttpResponse::Ok().finish()
}

pub fn run(listener: TcpListener) -> Result<Server, std::io::Error> {
    let server = HttpServer::new(|| {
        App::new()
            .route("/health_check", web::get().to(health_check))
            // A new entry in our routing table for POST /subscriptions requests
            .route("/subscriptions", web::post().to(subscribe))
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

再次运行测试

```plaintext
running 3 tests
test health_check_works ... ok
test subscribe_returns_a_200_for_valid_form_data ... ok
test subscribe_returns_a_400_when_data_is_missing ... FAILED

failures:

---- subscribe_returns_a_400_when_data_is_missing stdout ----

thread 'subscribe_returns_a_400_when_data_is_missing' panicked at tests/health_check.rs:63:9:
assertion `left == right` failed: The API did not fail with 400 Bad Request when the payload was missing the email.
  left: 400
 right: 200
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    subscribe_returns_a_400_when_data_is_missing

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
```

`subscribe_returns_a_200_for_valid_form_data` 现在通过了: 好吧，我们的处理程序将所有传入的数据视为有效数据，这并不奇怪。

`subscribe_returns_a_400_when_data_is_missing` 仍然是红色 (未通过)。
是时候对该请求主体进行一些真正的解析了。**actix-web** 为我们提供了什么?

### Extractors

[actix-web 用户指南中](https://actix.rs/docs/), [Extractors 部分](https://actix.rs/docs/extractors/)尤为突出。
顾名思义，Extractors 用于指示框架从传入请求中提取特定信息。
actix-web 提供了几个开箱即用的提取器，以满足最常见的用例:

- Path 用于从请求路径中获取动态路径段
- Query 用于查询参数
- Json 用于解析 JSON 编码的请求正文
- 等等

幸运的是，有一个Extractor正好可以满足我们的用例: [Form](https://docs.rs/actix-web/latest/actix_web/web/struct.Form.html)

阅读它的文档:

> 表单数据助手 (`application/x-www-form-urlencoded`)。
> 可用于从请求正文中提取 URL 编码数据，或将 URL 编码数据作为响应发送。

这真是太棒了!

我们该如何使用它?

查看 actix-web 的用户指南:

> 提取器可以作为处理函数的参数访问。Actix-web 每个处理函数最多支持 10 个提取器。参数位置无关紧要。

例子:

```rs
use actix_web::web;

#[derive(serde::Deserialize)]
struct FormData {
    username: String,
}

/// Extract form data using serde.
/// This handler get called only if content type is *x-www-form-urlencoded*
/// and content of the request could be deserialized to a `FormData` struct
fn index(form: web::Form<FormData>) -> String {
    format!("Welcome {}!", form.username)
}
```

所以，基本上……你只需将它作为处理程序和 actix-web 的参数添加到那里，当请求到达时，它就会以某种方式为你完成繁重的工作。我们现在先了解一下，稍后再回过头来了解底层发生了什么。

我们的 `subscribe` handler目前如下所示:

```rs
async fn subscribe() -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

使用这个例子作为蓝图，我们可能想要一些类似这样的内容:

```rs
//! src/lib.rs
// [...]

#[derive(serde::Deserialize)]
struct FormData {
    email: String,
    name: String
}

async fn subscribe(_form: web::Form<FormData>) -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

`cargo check` 并不是很开心

```plaintext
error[E0433]: failed to resolve: use of unresolved module or unlinked crate `serde`
 --> src/lib.rs:5:10
  |
5 | #[derive(serde::Deserialize)]
  |          ^^^^^ use of unresolved module or unlinked crate `serde`
```

好吧，我们需要将 serde 添加到依赖项中。让我们在 `Cargo.toml` 中添加一行:

```toml
[dependencies]
# We need the optional `derive` feature to use `serde`'s procedural macros:
# `#[derive(Serialize)]` and `#[derive(Deserialize)]`.
# The feature is not enabled by default to avoid pulling in
# unnecessary dependencies for projects that do not need it.
serde = { version = "1", features = ["derive"]}
```

`cargo check` 现在应该可以通过了, 那 `cargo test` 呢

```plaintext
running 3 tests
test subscribe_returns_a_200_for_valid_form_data ... ok
test health_check_works ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
```

全部通过!

但是为什么?

### Form 和 FromRequest

让我们直接进入源码：Form 是什么样子的?

你可以在[这里](https://github.com/actix/actix-web/blob/8c31d137aaf9ce8f536e5a71aa04cf68b556a7cb/actix-web/src/types/form.rs)找到它的源代码

这个定义看起来相当简单:

```rs
#[derive(PartialEq, Eq, PartialOrd, Ord, Debug)]
pub struct Form<T>(pub T);
```

它只不过是一个包装器：它对类型 T 进行泛型，然后用于填充 Form 的唯一字段。

这里没什么可看的。

提取魔法在哪里发生?

提取器是实现 [FromRequest](https://docs.rs/actix-web/latest/actix_web/trait.FromRequest.html) trait 的类型。

FromRequest 的定义有点杂乱，因为 Rust 尚不支持 trait 定义中的 async fn 。

稍微修改一下，它大致可以归结为这样的东西:

```rs
/// Trait implemented by types that can be extracted from request.
///
/// Types that implement this trait can be used with `Route` handlers.
pub trait FromRequest: Sized {
    /// The associated error which can be returned.
    type Error: Into<Error>;

    type Future: Future<Output = Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future;

    /// Omitting some ancillary methods that actix-web implements
    /// out of the box for you and supporting associated types
    /// [...]
}
```

`from_request` 将传入的 HTTP 请求的头部(即 [HttpRequest](https://docs.rs/actix-web/4.0.1/actix_web/struct.HttpRequest.html))及其有效负载(即 [Payload](https://docs.rs/actix-web/4.0.1/actix_web/dev/enum.Payload.html))的字节作为输入。如果提取成功，它将返回 Self；否则，它将返回一个可以转换为 [actix_web::Error](https://docs.rs/actix-web/4.0.1/actix_web/struct.Error.html) 的错误类型。

路由处理程序签名中的所有参数都必须实现 FromRequest trait：actix-web 将[为每个参数调用 `from_request`](https://github.com/actix/actix-web/blob/01cbef700fd9d7ce20f44bed06c649f6b238b9bb/src/handler.rs#L212)，如果所有参数的提取都成功，它将运行实际的处理程序函数。

如果其中一个提取失败，则相应的错误将返回给调用者，并且处理程序永远不会被调用(`actix_web::Error` 可以转换为 `HttpResponse`)。

这非常方便: 您的处理程序无需处理原始的传入请求，而可以直接处理强类型信息，从而大大简化了处理请求所需的代码。

让我们来看看 Form 的 FromRequest 实现：它做了什么?

再次，我稍微修改了[实际代码](https://github.com/actix/actix-web/blob/01cbef700fd9d7ce20f44bed06c649f6b238b9bb/src/types/form.rs#L112)以突出关键元素并忽略具体的实现细节。

```rs
impl<T> FromRequest for Form<T>
where
    T: DeserializeOwned + 'static,
{
    type Error = actix_web::Error;
    
    async fn from_request(
        req: &HttpRequest,
        payload: &mut Payload
    ) -> Result<Self, Self::Error> {
        // Omitted stuff around extractor configuration (e.g. payload size limits)

        match UrlEncoded::new(req, payload).await {
            Ok(item) => Ok(Form(item)),
            // The error handler can be customised.
            // The default one will return a 400, which is what we want.
            Err(e) => Err(error_handler(e))
        }
    }
}
```

所有繁重的工作似乎都发生在 **UrlEncoded** struct中。

**UrlEncoded** 的功能非常丰富: 它透明地处理压缩和未压缩的有效负载，处理请求主体以字节流的形式一次到达一个块的情况，等等。

处理完所有这些事情之后, [关键的部分](https://github.com/actix/actix-web/blob/01cbef700fd9d7ce20f44bed06c649f6b238b9bb/src/types/form.rs#L358)是:

```rs
serde_urlencoded::from_bytes::<T>(&body).map_err(|_| UrlencodedError::Parse)
```

[serde_urlencoded](https://docs.rs/serde_urlencoded/latest/serde_urlencoded/index.html) 为 `application/x-www-form-urlencoded` 数据格式提供（反）序列化支持。

**from_bytes** 接受一个连续的字节切片作为输入，并根据 URL 编码格式的规则将其反序列化为一个类型 T 的实例：键和值被编码为键值对元组，并以 & 分隔，键和值之间用 = 分隔；键和值中的非字母数字字符均采用百分号编码。

它是如何知道对于泛型类型 `T` 执行此操作的?

这是因为 `T` 实现了 **serde** 的 `DeserializedOwned` 特性:

```rs
impl<T> FromRequest for Form<T>
where
    T: DeserializeOwned + 'static,
{
    // [...]
}
```

要了解其内部实际发生的情况，我们需要仔细研究 serde 本身。

> 下一节关于 serde 的内容涉及一些 Rust 的高级主题。
> 如果您第一次阅读时没有完全理解，也没关系！
> 等您熟悉 Rust 之后再回来阅读，并多学习一些 serde，深入了解其中最难的部分。

### Rust 中的序列化: serde

为什么我们需要 **serde**? **serde** 为我们做了什么?

引用来自[它的教程](https://serde.rs/)的话:

> Serde 是一个用于高效且通用地序列化和反序列化 Rust 数据结构的框架。

#### 一般而言

serde 本身并不支持从/反序列化任何特定数据格式: serde 内部没有处理 JSON、Avro 或 MessagePack 细节的代码。如果您需要支持特定数据格式，则需要引入另一个 crate（例如，用于 JSON 的 [serde_json](https://docs.rs/serde_json) 或用于 Avro 的 [avro-rs](https://docs.rs/avro-rs/latest/avro_rs/)）。serde 定义了一组接口，或者用它们自己的话说，一个[数据模型](https://serde.rs/data-model.html)。

如果您想要实现一个库来支持新数据格式的序列化，则必须提供 Serializer trait 的实现。

Serializer trait 中的每个方法都对应于构成 serde 数据模型的 [29 种类型](https://serde.rs/data-model.html)之一——您的 Serializer 实现指定了每种类型如何映射到您的特定数据格式。

例如，如果您要添加对 JSON 序列化的支持，您的 [serialize_seq](https://docs.serde.rs/serde/trait.Serializer.html#tymethod.serialize_seq) 实现
将输出一个左方括号 `[` 并返回一个可用于序列化序列元素的类型。

另一方面，您有 [Serialize](https://docs.serde.rs/serde/ser/trait.Serialize.html) trait：您为 Rust 类型实现 Serialize::serialize 的目的是指定如何根据 serde 的数据模型，使用 **Serializer** trait 中提供的方法对其进行分解。

再次使用序列示例，以下是 Rust 中为 `Vec` 实现 `Serialize` 的方式:

```rs
use serde::ser::{Serialize, Serializer, SerializeSeq};

impl<T> Serialize for Vec<T>
where
    T: Serialize,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(self.len()))?;
        for element in self {
            seq.serialize_element(element)?;
        }
        seq.end()
    }
}
```

这使得 serde 对数据格式保持不可知:一旦你的类型实现了 **Serialize**, 你就可以自由地使用任何具体的 **Serializer** 实现来实际执行序列化步骤 - 也就是说，你可以将你的类型序列化为 crates.io 上可用的 **Serializer** 实现的任何格式（剧透：几乎所有常用的数据格式）。

反序列化也是如此，通过 [Deserialize](https://docs.serde.rs/serde/trait.Deserialize.html) 和 [Deserializer](https://docs.rs/serde/latest/serde/trait.Deserializer.html) 进行，并在生命周期方面有一些额外的细节，以支持零拷贝反序列化。

#### 效率

serde 的速度变慢是否是因为其对底层数据格式具有泛型性?

不是，这要归功于一个称为[单态化](https://doc.rust-lang.org/book/ch10-01-syntax.html)的过程。

每次使用一组具体类型调用泛型函数时，Rust 编译器都会创建函数体的副本，将泛型类型参数替换为具体类型。这使得编译器能够针对所涉及的具体类型优化函数体的每个实例：其结果与我们不使用泛型或特征，为每种类型编写单独的函数所获得的结果并无二致。换句话说，我们不会因为使用泛型而付出任何运行时成本。

这个概念非常强大，通常被称为零成本抽象：使用高级语言结构可以生成与使用更丑陋/更“手工编写”的实现相同的机器码。因此，我们可以编写更易于人类阅读的代码（正如它所期望的那样！），而无需在最终成品的质量上做出妥协。

Serde 在内存使用方面也非常谨慎：我们之前提到的中间数据模型是通过 trait 方法隐式定义的，并没有真正的中间序列化结构体。

如果您想了解更多信息，Josh Mcguigan 写了一篇精彩的深度文章，题为[《理解 Serde》](https://www.joshmcguigan.com/blog/understanding-serde/)。

还值得指出的是，对特定类型进行（反）序列化所需的所有信息在编译时即可获得，没有任何运行时开销。

其他语言中的（反）序列化器通常利用运行时反射来获取要（反）序列化的类型的信息（例如，它们的字段名称列表）。Rust 不提供运行时反射，
因此所有信息都必须预先指定。

#### 便捷

这就是 `#[derive(Serialize)]` 和 `#[derive(Deserialize)]` 发挥作用的地方。

你肯定不想手动详细说明项目中定义的每个类型应该如何执行序列化。这很繁琐，容易出错，而且会浪费你本应专注于特定于应用程序的逻辑的时间。

这两个过程宏与 **derive** feature flag 后面的 **serde** 捆绑在一起，将解析类型的定义并自动为你生成正确的 Serialize/Deserialize 实现。

#### 把所有东西放在一起

结合目前所学的知识，让我们再看一下我们的订阅处理程序:

```rs
#[derive(serde::Deserialize)]
struct FormData {
    email: String,
    name: String,
}

async fn subscribe(_form: web::Form<FormData>) -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

现在，我们对正在发生的事情有了清晰的了解：

- 在调用 subscribe 之前，actix-web 会为所有 subscribe 的输入参数调用          from_request 方法: 在我们的例子中，是 Form::from_request；
- Form::from_request 会尝试根据 URL 编码规则，利用 **serde_urlencoded** 和 **FormData** 的 Deserialize 实现（由 `#[derive(serde::Deserialize)]` 自动生成），将 body 反序列化为 **FormData**
- 如果 Form::from_request 失败，则会向调用者返回 400 BAD REQUEST 错误码。如果成功，则会调用 subscribe 并返回 200 OK 错误码。

稍作思考，您会惊叹不已：它看起来如此简单，但其中却蕴含着如此丰富的内容 ——我们主要依靠 Rust 的强大功能以及其生态系统中一些最完善的 crate。
