# 我们的第一个Endpoint - 简单的可用性检测

让我们尝试通过实现一个健康检查端点来开始：当我们收到对 /health_check 的 GET 请求时，我们希望返回一个不带正文的 200 OK 响应。

我们可以使用 /health_check 来验证应用程序是否已启动并准备好接受传入请求。

将它与 [pingdom.com](https://www.pingdom.com/) 这样的 SaaS 服务结合使用，您可以在 API 出现故障时[收到警报](https://www.pingdom.com/product/alerting/)——这对于您正在运行的电子邮件简报来说是一个很好的基准。

如果您使用容器编排器 (例如 [Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command) 或 [Nomad](https://www.nomadproject.io/docs/job-specification/service#service-parameters))来协调您的应用程序，那么健康检查端点也会非常方便：编排器可以调用 /health_check 来检测 API 是否无响应并触发重启。

## 编写 Actix-Web

我们的起点是 actix-web 主页上的 Hello World！示例:

```rs
use actix_web::{App, HttpRequest, HttpServer, Responder, web};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &name)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

让我们把示例代码粘贴到 main.rs中

让我们运行 `cargo check`

```plaintext
error[E0432]: unresolved import `actix_web`
 --> src/main.rs:1:5
  |
1 | use actix_web::{App, HttpRequest, HttpServer, Responder, web};
  |     ^^^^^^^^^ use of unresolved module or unlinked crate `actix_web`
  |
  = help: if you wanted to use a crate named `actix_web`, use `cargo add actix_web` to add it to your `Cargo.toml`

error[E0433]: failed to resolve: use of unresolved module or unlinked crate `tokio`
 --> src/main.rs:8:3
  |
8 | #[tokio::main]
  |   ^^^^^ use of unresolved module or unlinked crate `tokio`

Some errors have detailed explanations: E0432, E0433.
For more information about an error, try `rustc --explain E0432`.
```

等..等等...为什么会这样

我们尚未将 actix-web 和 tokio 添加到依赖项列表中，因此编译器无法解析我们导入的内容。

我们可以手动修复此问题，方法是在 **Cargo.toml** 中添加

```toml
#! Cargo.toml

# [...]
[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

或许我们可以执行 `cargo add actix-web` 来快速添加 actix-web 依赖

让我们再次运行 `cargo check`! 现在应该一切正常了!

让我们尝试运行一下应用程序!

```shell
cargo run
```

在你喜欢的终端尝试一下API吧

```shell
curl http://127.0.0.1:8000
```

太好了! 这可以用!

现在, 你可以按下 Ctrl+C来停止web应用程序

## actix-web 应用程序的剖析

现在让我们回过头仔细看看我们刚刚在 main.rs 文件中复制粘贴的内容。

```rs
//! src/main.rs

// [...]

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

### 服务器 - HttpServer

[HttpServer](https://docs.rs/actix-web/4.0.1/actix_web/struct.HttpServer.html) 是支撑我们应用程序的骨干。它负责处理以下事项：

- 应用程序应该在哪里监听传入的请求？TCP socket (例如 127.0.0.1:8000)? Unix 域套接字？
- 我们应该允许的最大并发连接数是多少？单位时间内可以创建多少个新连接？
- 我们应该启用传输层安全性 (TLS) 吗？
- 等等

换句话说，HttpServer 处理所有传输层的问题。

之后会发生什么？当 HttpServer 与我们的 API 客户端建立了新的连接，而我们需要开始处理他们的请求时，它会做什么？

这时，App 就派上用场了！

### 应用程序 - App

[App](https://docs.rs/actix-web/4.0.1/actix_web/struct.App.html) 是所有应用逻辑的存放地：路由、中间件、请求处理程序等。

**App** 组件的作用是接收传入的请求并返回响应。

让我们仔细地看一下如下代码片段：

```rs
App::new()
    .route("/", web::get().to(greet))
    .route("/{name}", web::get().to(greet))
```

App 是builder模式的一个实际示例: `new()` 为我们提供了一个干净的平台，我们可以使用流畅的 API（即链式调用）一点一点地添加新的行为。

我们将在整本书中根据需要了解 App 的大部分 API 接口: 读完本书后，您应该至少接触过一次它的大多数方法。

### 端点 - Route

如何向我们的应用添加新的端点？[`route`](https://docs.rs/actix-web/latest/actix_web/struct.App.html#method.route) 方法可能是最简单的方法 ——毕竟，我们已经在 Hello World! 示例中使用过了！

`route` 方法接受两个参数：

- `path` - 一个字符串，可能为模板(例如`/{name}`), 用于容纳动态路径
- `route` - Route 结构体的实例

[Route](https://docs.rs/actix-web/latest/actix_web/struct.Route.html) 将处理程序与一组守卫组合在一起。

守卫指定请求必须满足的条件才能“匹配”并传递给处理程序。从实现的角度来看，守卫是 [Guard](https://docs.rs/actix-web/4.0.1/actix_web/guard/trait.Guard.html) 特性的实现者：
`Guard::check` 是奇迹发生的地方。

在我们的代码片段中

```rs
.route("/", web::get().to(greet))
```

`"/"` 将匹配所有在基本路径后不带任何段的请求，例如 `http://localhost:8000/`。
`web::get()` 是 `Route::new().guard(guard::Get())` 的快捷方式，也就是说，当且仅当请求的 HTTP 方法是 GET 时，该请求才会传递给处理程序。

您可以想象一下，当一个新请求到来时会发生什么：应用会遍历所有已注册的端点，直到找到一个匹配的端点（路径模板和保护条件都满足），然后将请求对象传递给处理程序。

这并非 100% 准确，但目前来说，这是一个足够好的思维模型。

处理程序应该是什么样的？它的函数签名是什么？

目前我们只有一个示例，`greet`:

```rs
async fn greet(req: HttpRequest) -> impl Responder {
    // [...]
}
```


### 运行时 - Tokio

我们从整个 HttpServer 深入到 Route。让我们再看一下整个 main 函数：

```rs
#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
}
```

`#[tokio::main]` 有什么用? 当然, 让我们删掉它看看会发生什么!

很不幸的, `cargo check` 给出了如下的报错

```plaintext
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:9:1
  |
9 | async fn main() -> std::io::Result<()> {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`

For more information about this error, try `rustc --explain E0752`.
error: could not compile `zero2prod` (bin "zero2prod") due to 1 previous error
```

我们需要 main 函数是异步的，因为 HttpServer::run 是一个异步方法，但 main 函数（我们二进制文件的入口点）不能是异步函数。为什么呢？

Rust 中的异步编程建立在 [Future](https://doc.rust-lang.org/beta/std/future/trait.Future.html) trait 之上: Future 代表一个可能尚未到达的值。所有 Future 都公开一个 [poll](https://doc.rust-lang.org/beta/std/future/trait.Future.html#the-poll-method) 方法，必须调用该方法才能使 Future 继续执行并最终解析出最终值。你可以将 Rust 的 Future 视为惰性的：除非进行轮询，否则无法保证它们会执行完成。与其他语言采用的推送模型相比，这通常被描述为一种拉模型。

Rust 的标准库在设计上不包含异步运行时：你应该将其作为依赖项引入你的项目，在 Cargo.toml 文件的 `[dependencies]` 下添加一个 crate。这种方法非常灵活：您可以自由地实现自己的运行时，并根据用例的特定需求进行优化（参见 [Fuchsia](http://smallcultfollowing.com/babysteps/blog/2019/12/09/async-interview-2-cramertj/#async-interview-2-cramertj) 项目或 [bastion](https://github.com/bastion-rs/bastion) 的 Actor 框架）。

这就解释了为什么 main 不能是异步函数：谁负责调用它的 poll 方法?

没有特殊的配置语法来告诉 Rust 编译器你的依赖项之一是异步运行时（例如，我们对分配器所做的配置），而且，公平地说，甚至没有一个关于运行时的标准化定义（例如，Executor trait）。
因此，你应该在 main 函数的顶部启动异步运行时，

然后用它来驱动你的 Future 完成。

你现在可能已经猜到 #[tokio::main] 的用途了，但仅仅猜测是不够的：

我们想看到它。

但是怎么做呢?

`tokio::main` 是一个过程宏，这是一个引入 `cargo expand` 的绝佳机会，它对于 Rust 开发来说是一个很棒的补充:

```shell
cargo install cargo-expand
```

Rust 宏在 token 级别运行：它们接收一个符号流（例如，在我们的例子中是整个 main 函数），并输出一个新符号流，然后将其传递给编译器。换句话说，Rust 宏的主要用途是**代码生成**。

我们如何调试或检查特定宏的运行情况？您可以检查它输出的 token!

这正是 `cargo expand` 的亮点所在：它会扩展代码中的所有宏，而无需将输出传递给编译器，让您可以单步执行并了解正在发生的事情。

让我们使用 `cargo expand` 来揭开 `#[tokio::main]` 的神秘面纱:

```shell
cargo expand
```

```rs
fn main() -> std::io::Result<()> {
    let body = async { run().await };
    #[allow(
        clippy::expect_used,
        clippy::diverging_sub_expression,
        clippy::needless_return
    )]
    {
        return tokio::runtime::Builder::new_multi_thread()
            .enable_all()
            .build()
            .expect("Failed building the Runtime")
            .block_on(body);
    }
}
```

我们终于可以看看宏扩展后的代码了!

`#[tokio::main]` 扩展后传递给 Rust 编译器的 main 函数
确实是同步 (Sync) 的, 这也解释了为什么它编译时没有任何问题。

关键的一行是：`tokio::runtime::Builder::new_multi_thread().enable_all().build().expect("[...]").block_on(/*[...]*/)`

我们正在启动 tokio 的异步运行时，并使用它来驱动 HttpServer::run 返回的 Future 完成。

换句话说，`#[tokio::main]` 的作用是让我们产生能够定义异步主函数的错觉，而实际上，它只是获取我们的主要异步代码，并编写必要的样板代码，使其在 tokio 的运行时之上运行。

## 实现 Health Check Handler

我们已经回顾了 actix_web 的 Hello World! 示例中所有需要移动的部分：`HttpServer`、`App`、`route` 和 `actix_web::main`。

我们当然已经了解了足够多的内容，可以修改示例，使健康检查能够按预期工作：

在 `/health_check` 收到 GET 请求时，返回一个不带正文的 200 OK 响应。

让我们重新回顾一下我们的起点：

```rs
use actix_web::{App, HttpRequest, HttpServer, Responder, web};

async fn greet(req: HttpRequest) -> impl Responder {
    let name = req.match_info().get("name").unwrap_or("World");
    format!("Hello {}!", &name)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/", web::get().to(greet))
            .route("/{name}", web::get().to(greet))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

首先，我们需要一个请求处理程序。模仿 greet 函数，我们可以从以下签名开始：

```rs
async fn health_check(req: HttpRequest) -> impl Responder {
    todo!()
}
```

我们说过，Responder 只不过是一个转换为 HttpResponse 的 trait。那么直接返回一个 HttpResponse 实例应该就行了！

查看它的[文档](https://docs.rs/actix-web/latest/actix_web/struct.HttpResponse.html#method.Ok)，我们可以使用 HttpResponse::Ok 来获取一个已准备好 200 状态码的 [HttpResponseBuilder](https://docs.rs/actix-web/latest/actix_web/struct.HttpResponseBuilder.html)。HttpResponseBuilder 提供了一个丰富的流畅 API，可以逐步构建 HttpResponse 响应，但我们在这里不需要它: 我们可以通过在构建器上调用 [finish](https://docs.rs/actix-web/latest/actix_web/struct.HttpResponseBuilder.html#method.finish) 来获取一个带有空主体的 HttpResponse。

将所有内容结合在一起：

```rs
use actix_web::{HttpRequest, HttpResponse, Responder};

// [...]

async fn health_check(req: HttpRequest) -> impl Responder {
    HttpResponse::Ok().finish()
}
```

快速运行一下 `cargo check`，确认我们的处理程序没有做任何奇怪的事情。

仔细查看一下 HttpResponseBuilder 的定义, 会发现它也实现了 Responder 接口——因此，我们可以省略对 finish 的调用，并将处理程序简化为:

```rs
// [...]
async fn health_check(req: HttpRequest) -> impl Responder {
    HttpResponse::Ok()
}
```

下一步是处理程序注册 - 我们需要通过 **route** 将其添加到我们的 **App** 中: (别忘了删除示例中的`greet`方法和相关route注册的代码)

```rs
//! src/main.rs

// [...]
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // [...]
    App::new()
        // [...]
        .route("/health_check", web::get().to(health_check))
    // [...]
}
```

现在我们的代码看起来是这样

```rs
use actix_web::{web, App, HttpRequest, HttpResponse, HttpServer, Responder};

async fn health_check(req: HttpRequest) -> impl Responder {
    HttpResponse::Ok()
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/health_check", web::get().to(health_check))
    })
    .bind("127.0.0.1:8000")?
    .run()
    .await
}
```

你可能看到 `cargo check` 在抱怨变量 `req`没有用到

我们的健康检查响应确实是静态的，并且不使用任何与传入 HTTP 请求捆绑的数据（路由除外）。我们可以遵循编译器的建议，在 req 前添加下划线......
或者，我们可以从 health_check 中完全删除该输入参数:

```rs
async fn health_check() -> impl Responder {
    HttpResponse::Ok()
}
```

惊喜啊，它编译通过了！actix-web 在后台运行着一些相当高级的类型处理程序, 并且它接受各种各样的签名作为请求处理程序——稍后会详细介绍。

接下来做什么?

好吧，来个小测试!

```shell
curl -v http://127.0.0.1:8000/health_check
```

```plaintext
$ curl -v http://127.0.0.1:8000/health_check

*   Trying 127.0.0.1:8000...
* Connected to 127.0.0.1 (127.0.0.1) port 8000
* using HTTP/1.x
> GET /health_check HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/8.15.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< content-length: 0
< date: Wed, 20 Aug 2025 06:36:57 GMT
< 
* Connection #0 to host 127.0.0.1 left intact
```

你现在应该可以在响应中看到类似`HTTP/1.1 200 OK`的字样

这太棒了! 我们的Health Check正在工作!

恭喜你实现了第一个 actix-web 端点!

## 我们的第一个集成测试

`/health_check` 是我们的第一个端点，我们通过启动应用程序并通过 curl 手动测试来验证一切是否按预期运行。

然而，手动测试非常耗时: 随着应用程序规模的扩大，每次执行更改时，手动检查我们对其行为的所有假设是否仍然有效，成本会越来越高。

我们希望尽可能地实现自动化: 这些检查应该在每次提交更改时都在我们的持续集成 (CI) 流水线中运行，以防止出现回归问题。

虽然健康检查的行为在整个过程中可能不会有太大变化，但它是正确设置测试框架的良好起点。

### 我应该怎么测试一个API Endpoint

API 是达到目的的手段：一种暴露给外界执行某种任务的工具（例如，存储文档、发布电子邮件等）。

我们在 API 中暴露的端点定义了我们与客户端之间的契约：关于系统输入和输出（即接口）的共同协议。

契约可能会随着时间的推移而演变，我们可以粗略地设想两种情况:

- 向后兼容的变更（例如，添加新的端点）
- 重大变更（例如，移除端点或从其输出的架构中删除字段）。

在第一种情况下，现有的 API 客户端将保持原样运行。在第二种情况下，如果现有的集成依赖于契约中被违反的部分，则可能会中断。

虽然我们可能会故意部署对 API 契约的重大变更，但至关重要的是，我们不要意外地破坏它。

什么是最可靠的方法来检查我们没有引入用户可见的回归？

通过与用户完全一样的方式与 API 交互来测试 API：向 API 执行 HTTP 请求，并验证我们对收到的响应的假设。

这通常被称为黑盒测试：我们通过检查系统的输出来验证系统的行为，而不需要了解其内部实现的细节。

遵循这一原则，我们不会满足于直接调用处理函数的测试——例如:

```rs
#[cfg(test)]
mod tests {
    use crate::health_check;

    #[tokio::test]
    async fn health_check_succeeds() {
        let response = health_check().await;
        // This requires changing the return type of `health_check`
        // from `impl Responder` to `HttpResponse` to compile
        // You also need to import it with `use actix_web::HttpResponse`!
        assert!(response.status().is_success())
    }
}
```

这样做并不是最佳实践

- 我们没有检查处理程序是否在 **GET** 请求时调用
- 我们也没有检查处理程序是否以 **/health_check** 作为路径调用。

更改这两个属性中的任何一个都会破坏我们的 API 契约，但我们的测试仍然会通过——
这还不够好。

actix-web 提供了一些便利，可以在不跳过路由逻辑的情况下与应用进行交互，
但这种方法存在严重的缺陷:

- 迁移到另一个 Web 框架将迫使我们重写整个集成测试套件。
我们希望集成测试尽可能与 API 实现的底层技术高度分离（例如，在进行大规模重写或重构时，进行与框架无关的集成测试可以起到至关重要的作用！）
- 由于 actix-web 的一些限制，我们无法在生产代码和测试代码之间共享应用启动逻辑，因此，由于存在随着时间的推移出现分歧的风险，我们对测试套件提供的保证的信任度会降低。

我们将选择一个完全黑盒解决方案：我们将在每次测试开始时启动我们的应用程序，并使用现成的 HTTP 客户端 (例如 [reqwest](https://docs.rs/reqwest/latest/reqwest/index.html)) 与其交互。

### 我应该把测试放在哪里

Rust 在编写测试时提供了[三种选择](https://doc.rust-lang.org/book/ch11-03-test-organization.html):

- 在嵌入式测试模块中的代码旁边 (使用 `mod tests` )

```rs
// Some code I want to test
#[cfg(test)]
mod tests {
    // Import the code I want to test
    use super::*;
    // My tests
}
```

- 在外部的 `tests/` 文件夹

```plaintext
$ ls

src/
tests/
Cargo.toml
Cargo.lock
```

- 公共文档的一部分 (doc tests)

```rs
/// Check if a number is even.
/// ```rust
/// use zero2prod::is_even;
///
/// assert!(is_even(2));
/// assert!(!is_even(1));
/// ```
pub fn is_even(x: u64) -> bool {
    x % 2 == 0
}
```

有什么区别?

嵌入式测试模块是项目的一部分，只是隐藏在[配置条件检查](https://doc.rust-lang.org/stable/rust-by-example/attribute/cfg.html) `#[cfg(test)]` 后面。而 tests 文件夹下的所有内容以及文档测试则被编译成各自独立的二进制文件。

这会影响*可见性规则*。

嵌入式测试模块对其相邻的代码拥有特权访问权：它可以与未标记为公共的结构体、方法、字段和函数进行交互，而这些内容通常无法被我们代码的用户访问，即使他们将其作为自己项目的依赖项导入也是如此。

嵌入式测试模块对于我所说的“冰山项目”非常有用，即暴露的接口非常有限（例如几个公共函数），但底层机制却非常庞大且相当复杂（例如数十个例程）。通过公开的函数来测试所有可能的边缘情况可能并非易事——您可以利用嵌入式测试模块为私有子组件编写单元测试，从而增强对整个项目正确性的整体信心。

而外部测试文件夹和文档测试对代码的访问级别，与将 crate 添加为另一个项目依赖项时获得的访问级别完全相同。因此，它们主要用于集成测试，即通过与用户完全相同的方式调用代码来测试。

我们的电子邮件简报并非库，因此两者之间的界限有点模糊——我们不会将其作为 Rust crate 公开给世界，而是将其作为可通过网络访问的 API 公开。

尽管如此，我们将使用 tests 文件夹进行 API 集成测试——它更加清晰地划分，并且将测试助手作为外部测试二进制文件的子模块进行管理也更加容易。

### 改变我们的项目结构以便于测试

在真正开始在 `/tests` 下编写第一个测试之前，我们还有一些准备工作要做。

正如我们所说，任何测试代码最终都会被编译成它自己的二进制文件——我们所有测试代码

都以 crate 的形式导入。但目前我们的项目是一个二进制文件：它旨在执行，而不是

共享。因此，我们无法像现在这样在测试中导入 main 函数。

如果您不相信我的话，我们可以做一个快速实验:

```shell
# Create the tests folder
mkdir -p tests
```

创建 `tests/health_check.rs` 然后写入如下代码

```rs
//! tests/health_check.rs
use zero2prod::main;

#[test]
fn dummy_test() {
    main()
}
```

现在执行 `cargo test`, 欸? 好像报错了

```plaintext
error[E0432]: unresolved import `zero2prod`
 --> tests/health_check.rs:1:5
  |
1 | use zero2prod::main;
  |     ^^^^^^^^^ use of unresolved module or unlinked crate `zero2prod`
  |
  = help: if you wanted to use a crate named `zero2prod`, use `cargo add zero2prod` to add it to your `Cargo.toml`

For more information about this error, try `rustc --explain E0432`.
error: could not compile `zero2prod` (test "health_check") due to 1 previous error
```

译者注: 原作此处为修改 Cargo.toml来配置lib.rs, 但在最新的Rust中, lib.rs会自动识别为一个crate, 所以不必那么做了, 这里就没翻译

接下来我们创建文件 `src/lib.rs`

然后我们可以把 `main.rs` 中的逻辑搬过去了

```rs
//! main.rs
use zero2prod::run;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    run().await
}
```

```rs
//! lib.rs
use actix_web::{App, HttpResponse, HttpServer, Responder, web};

async fn health_check() -> impl Responder {
    HttpResponse::Ok()
}

pub async fn run() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/health_check", web::get().to(health_check)))
        .bind("127.0.0.1:8000")?
        .run()
        .await
}
```

好了，我们准备编写一些有趣的集成测试！

## 实现我们的第一个集成测试

我们对健康检查端点的规范是：当我们收到 /health_check 的 GET 请求时，我们会返回没有正文的 200 OK 响应。

这分为这几个部分

- GET 请求
- /health_check 端点
- 响应码 200
- 没有正文的响应

让我们将其转化为测试，并尽可能多地描述特征:

```rs
//! tests/health_check.rs
// `tokio::test` is the testing equivalent of `tokio::main`.
// It also spares you from having to specify the `#[test]` attribute.
//
// You can inspect what code gets generated using
// `cargo expand --test health_check` (<- name of the test file)
#[tokio::test]
async fn health_check_works() {
    // Arrange
    spawn_app().await.expect("Failed to spawn our app.");
    // We need to bring in `reqwest`
    // to perform HTTP requests against our application.
    let client = reqwest::Client::new();
    // Act
    let response = client
        .get("http://127.0.0.1:8000/health_check")
        .send()
        .await
        .expect("Failed to execute request.");

    // Assert
    assert!(response.status().is_success());
    assert_eq!(Some(0), response.content_length());
}

async fn spawn_app() -> std::io::Result<()> {
    todo!()
}
```

当然别忘了添加 `reqwest` 依赖

```shell
cargo add reqwest --dev
```

请花点时间仔细看看这个测试用例。

spawn_app 是唯一一个合理地依赖于我们应用程序代码的部分。

其他所有内容都与底层实现细节完全解耦——如果明天我们决定放弃 Rust，用 Ruby on Rails 重写应用程序，我们仍然可以使用相同的测试套件来检查新堆栈中的回归问题，只要将 spawn_app 替换为合适的触发器（例如，使用 bash 命令启动 Rails 应用程序）。

该测试还涵盖了我们感兴趣的所有属性:

- 健康检查暴露在 /health_check；
- 健康检查使用 GET 方法；
- 健康检查始终返回 200；
- 健康检查的响应没有正文。

如果这个测试通过了，这就完成了。

测试还没来得及做任何有用的事情就崩溃了：我们缺少了 spawn_app，这是集成测试的最后一块拼图。

为什么我们不直接在那里调用 run 呢? 也就是说

```rs
async fn spawn_app() -> std::io::Result<()> {
    zero2prod::run().await
}
```

让我们试试看!

```shell
cargo test
```

无论等待多久，测试执行都不会终止。这是怎么回事?

在 zero2prod::run 中，我们调用（并等待）HttpServer::run。HttpServer::run 返回一个 Server 实例 - 当我们调用 .await 时，它会无限期地监听我们指定的地址：它会处理传入的请求，但永远不会自行关闭或“完成”。

这意味着 spawn_app 永远不会返回，我们的测试逻辑也永远不会执行。

我们需要将应用程序作为后台任务运行。

[tokio::spawn](https://docs.rs/tokio/latest/tokio/fn.spawn.html) 在这里非常方便：tokio::spawn 接受一个 Future 并将其交给运行时进行轮询，

而无需等待其完成；因此，它与下游 Future 和任务（例如我们的测试逻辑）并发运行。

让我们重构 zero2prod::run，使其返回一个 Server 实例而不等待它:

```rs
//! src/lib.rs

// [...]

// Notice the different signature!
// We return `Server` on the happy path and we dropped the `async` keyword
// We have no .await call, so it is not needed anymore.
pub fn run() -> Result<Server, std::io::Error> {
    let server = HttpServer::new(|| App::new().route("/health_check", web::get().to(health_check)))
        .bind("127.0.0.1:8000")?
        .run();

    // No .await here!
    Ok(server)
}
```

我们需要相应地修改我们的 **main.rs**:

```rs
//! src/main.rs
use zero2prod::run;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    run()?.await
}
```

运行一下 `cargo check` 应该能让我们确信一切正常。

现在我们可以实现 `spawn_app` 方法

```rs
// No .await call, therefore no need for `spawn_app` to be async now.
// We are also running tests, so it is not worth it to propagate errors:
// if we fail to perform the required setup we can just panic and crash
// all the things.
fn spawn_app() {
    let server = zero2prod::run().expect("Failed to bind address");
    // Launch the server as a background task
    // tokio::spawn returns a handle to the spawned future,
    // but we have no use for it here, hence the non-binding let
    let _ = tokio::spawn(server);
}
```

快速调整我们的测试以适应 `spawn_app` 方法签名的变化:

```rs
#[tokio::test]
async fn health_check_works() {
    // [...]
    spawn_app();
    // [...]
```

现在是时候运行 `cargo test` 了!

```plaintext
running 1 test
test health_check_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
```

耶！我们的第一次集成测试通过了!

替我给自己鼓个掌，在一个章节内完成了第二个重要的里程碑。

### Polishing

我们已经让它运行起来了，现在我们需要重新审视并改进它，如果需要或可能的话

#### 清理

测试运行结束后，后台运行的应用会发生什么?

它会关闭吗? 它会像僵尸程序一样徘徊在某个地方吗?

嗯，连续多次运行 `cargo test` 总是会成功——这强烈暗示我们的 8000 端口会在每次运行结束时被释放，因此意味着应用已正确关闭。

再次查看 [`tokio::spawn` 的文档](https://docs.rs/tokio/latest/tokio/fn.spawn.html)，这支持了我们的假设：当一个 tokio 运行时关闭时，所有在其上生成的任务都会被丢弃。tokio::test 会在每个测试用例开始时启动一个新的运行时，并在每个测试用例结束时关闭。

换句话说，好消息是——无需实现任何清理逻辑来避免测试运行期间的资源泄漏。

#### 选择随机端口

`spawn_app` 总是会尝试在 8000 端口上运行我们的应用——这并不理想:

- 如果 8000 端口正在被我们机器上的其他程序（例如我们自己的应用！）占用，测试就会失败；
- 如果我们尝试并行运行两个或多个测试，那么只有一个测试能够绑定端口，其他所有测试都会失败。

我们可以做得更好: 测试应该在随机可用的端口上运行它们的后台应用。

首先，我们需要修改 `run` 函数——它应该将应用地址作为参数，而不是依赖于硬编码的值:

让我们修改 **lib.rs**

```rs
//! src/lib.rs

// [...]
pub fn run(address: &str) -> Result<Server, std::io::Error> {
    let server = HttpServer::new(|| App::new().route("/health_check", web::get().to(health_check)))
        .bind(address)?
        .run();

    // No .await here!
    Ok(server)
}
```

然后，所有 `zero2prod::run()` 调用都必须更改为 `zero2prod::run("127.0.0.1:8000")` 才能保留相同的行为并使项目再次编译。
我们如何为测试找到一个随机可用的端口?

操作系统可以帮上忙：我们将使用端口 0。

端口 0 在操作系统层面是特殊情况：尝试绑定端口 0 将触发操作系统扫描可用端口，

然后该端口将被绑定到应用程序。

因此，只需将 `spawn_app` 更改为

```rs
//! tests/health_check.rs

fn spawn_app() {
    let server = zero2prod::run("127.0.0.1:0").expect("Failed to bind address");
    let _ = tokio::spawn(server);
}
```

> 注: 原作这里忘了提到要修改 **main.rs** 了, 请**暂时**将 **main.rs** 中的代码修改为 `run("127.0.0.1:8000")?.await`

这样就好了~ 现在，每次启动 Cargo 测试时，后台应用都会在随机端口上运行! 只有一个小问题...... 我们的测试失败了

```plaintext
running 1 test
test health_check_works ... FAILED

failures:

---- health_check_works stdout ----

thread 'health_check_works' panicked at tests/health_check.rs:19:10:
Failed to execute request.: reqwest::Error { kind: Request, url: "http://127.0.0.1:8000/health_check", source: hyper_util::client::legacy::Error(Connect, ConnectError("tcp connect error", 127.0.0.1:80
00, Os { code: 111, kind: ConnectionRefused, message: "Connection refused" })) }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    health_check_works

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

我们的 HTTP 客户端仍在调用 `127.0.0.1:8000`，我们现在真的不知道该在那里放什么: 应用程序端口是在运行时确定的，我们无法在那里进行硬编码。

我们需要以某种方式找出操作系统分配给我们应用程序的端口，并将其从 `spawn_app` 返回。

有几种方法可以实现这一点——我们将使用 [std::net::TcpListener](https://doc.rust-lang.org/beta/std/net/struct.TcpListener.html)。

我们的 HttpServer 目前承担着双重任务：给定一个地址，它会绑定它，然后启动应用程序。我们可以接手第一步：我们自己用 TcpListener 绑定端口，然后使用 listen 将其交给 HttpServer。

这样做有什么好处呢？

[`TcpListener::local_addr`](https://doc.rust-lang.org/beta/std/net/struct.TcpListener.html#method.local_addr) 返回一个 [`SocketAddr`](https://doc.rust-lang.org/beta/std/net/enum.SocketAddr.html) 对象，它暴露了我们通过 `.port()` 绑定的实际端口。

让我们从 `run` 函数开始:

```rs
//! src/lib.rs

// [...]

pub fn run(listener: TcpListener) -> Result<Server, std::io::Error> {
    let server = HttpServer::new(|| App::new().route("/health_check", web::get().to(health_check)))
        .listen(listener)?
        .run();

    Ok(server)
}
```

这项更改破坏了我们的 `main` 函数和 `spawn_app` 函数。`main` 函数就留给你处理吧，我们来重点处理 `spawn_app` 函数：

```rs
//! tests/health_check.rs

// [...]

fn spawn_app() -> String {
    let listener = TcpListener::bind("127.0.0.1:0").expect("Faield to bind random port");
    // We retrieve the port assigned to us by the OS
    let port = listener.local_addr().unwrap().port();
    let server = zero2prod::run(listener).expect("Failed to bind address");
    let _ = tokio::spawn(server);

    // We return the application address to the caller!
    format!("http://127.0.0.1:{}", port)
}
```

现在我们可以在 `reqwest::Client` 中引用这个地址:

```rs
//! tests/health_check.rs

// [...]

#[tokio::test]
async fn health_check_works() {
    // Arrange
    let address = spawn_app();
    let client = reqwest::Client::new();

    // Act
    let response = client
        // Use the returned application address
        .get(format!("{address}/health_check"))
        .send()
        .await
        .expect("Failed to execute request.");

    // Assert
    assert!(response.status().is_success());
    assert_eq!(Some(0), response.content_length());
}

// [...]
```

## 回顾

让我们稍事休息一下，回顾一下，我们已经完成了相当多的内容！

我们着手实现一个 /health_check 端点，这让我们有机会进一步了解我们的 Web 框架 [actix-web](https://docs.rs/actix-web/latest/actix_web/index.html) 的基础知识，以及 Rust API 的（集成）测试基础知识。

现在是时候利用我们学到的知识，最终完成我们电子邮件通讯项目的第一个用户故事了:

> As a blog visitor,
> I want to subscribe to the newsletter,
> So that I can receive email updates when new content is published on the blog.

我们希望博客访问者在网页嵌入的表单中输入他们的电子邮件地址。

该表单将触发对我们后端 API 的 `POST /subscriptions` 调用，后端 API 将实际处理信息、存储信息并返回响应。

我们将深入研究：

- 如何在 actix-web 中读取 HTML 表单中收集的数据（例如，如何解析 POST 请求体？）；
- 哪些库可以在 Rust 中使用 PostgreSQL 数据库（diesel、sqlx 和 tokio-postgres）；
- 如何设置和管理数据库迁移；
- 如何在 API 请求处理程序中获取数据库连接；
- 如何在集成测试中测试副作用（即存储数据）；
- 如何避免在使用数据库时测试之间出现奇怪的交互。

让我们开始吧！

## 使用 HTML 表单

TODO: WIP
