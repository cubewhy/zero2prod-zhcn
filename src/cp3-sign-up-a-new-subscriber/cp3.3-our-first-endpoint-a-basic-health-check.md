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

#[tokio::main]
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

#[tokio::main]
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

#[tokio::main]
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
#[tokio::main]
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

#[tokio::main]
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
