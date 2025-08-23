# 日志记录

日志是最常见的遥测数据类型。

即使是从未听说过可观察性的开发人员，也能直观地理解日志的用处：当事情出错时，你会查看日志来了解正在发生的事情，
并祈祷自己能捕获足够的信息来有效地进行故障排除。

那么，日志是什么呢?

它的格式因时代、平台和你使用的技术而异。

如今，日志记录通常是一堆文本数据，并用换行符分隔当前记录和下一个记录。例如

```plaintext
The application is starting on port 8080
Handling a request to /index
Handling a request to /index
Returned a 200 OK
```

对于 Web 服务器来说，这四条日志记录完全有效。

Rust 生态系统在日志记录方面能为我们提供什么?

## `log` crate

Rust 中用于日志记录的首选 crate 是 [log](https://docs.rs/log)。

log 提供了五个宏：`trace`、`debug`、`info`、`warn` 和 `error`。

它们的作用相同——发出一条日志记录——但正如其名称所暗示的那样，它们各自使用不同的日志级别。

trace 是最低级别: trace 级别的日志通常非常冗长，信噪比较低（例如，每次 Web 服务器收到 TCP 数据包时都会发出一条 trace 级别的日志记录）。

然后，我们依次按严重程度递增，依次为 debug、info、warn 和 error。

Error 级别的日志用于报告可能对用户造成影响的严重故障（例如，我们未能处理传入的请求或数据库查询超时）。

让我们看一个简单的使用示例：

```rs
fn fallible_operation() -> Result<String, String> { ... }

pub fn main() {
    match fallible_operation() {
        Ok(success) => {
            log::info!("Operation succeeded: {}", success);
        }
        Err(err) => {
            log::error!("Operation failed: {}", err);
        }
    }
}
```

我们正在尝试执行一个可能会失败的操作。

如果成功，我们将发出一条信息级别的日志记录。

如果失败，我们将发出一条错误级别的日志记录。

另请注意，log 的宏支持与标准库中 println/print 相同的插值语法。

我们可以使用 log 的宏来检测我们的代码库。

选择记录关于特定函数执行的信息通常是一个本地决策：只需查看函数本身即可决定哪些信息值得在日志记录中捕获。

这使得库能够被有效地检测，将遥测的范围扩展到我们亲自编写的代码之外。

### actix-web 的日志中间件

actix_web 提供了一个 [Logger 中间件](https://docs.rs/actix-web/4.0.1/actix_web/middleware/struct.Logger.html)。它会为每个传入的请求发出一条日志记录。

让我们将它添加到我们的应用程序中。

```rs
//! src/startup.rs
use std::net::TcpListener;

use actix_web::{dev::Server, middleware::Logger, web, App, HttpServer};
use sqlx::PgPool;

use crate::routes::{health_check, subscribe};

pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);

    let server = HttpServer::new(move || {
        App::new()
            // Middlewares are added using the `wrap` method on `App`
            .wrap(Logger::default())
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

现在，我们可以使用 cargo run 启动该应用，并使用 curl 快速发送一个请求：`curl http://127.0.0.1:8000/health_check -v`。

请求返回 200，但是......我们用来启动应用的终端上没有任何反应。

没有日志。什么也没有。屏幕一片空白。

## 门面模式

我们说过，检测是一个局部决策。

相反，应用程序需要做出一个全局决策：我们应该如何处理所有这些日志记录?

应该将它们附加到文件中吗? 应该将它们打印到终端吗? 应该通过 HTTP 将它们发送到远程系统（例如 [ElasticSearch](https://www.elastic.co/elasticsearch/)）吗?

log crate 利用[门面模式](https://en.wikipedia.org/wiki/Facade_pattern)来处理这种二元性。

它为您提供了发出日志记录所需的工具，但没有规定应该如何处理这些日志记录。相反，它提供了一个 [Log trait](https://docs.rs/log/0.4.11/log/trait.Log.html):

```rs
//! From `log`'s source code - src/lib.rs
/// A trait encapsulating the operations required of a logger.
pub trait Log: Sync + Send {
    /// Determines if a log message with the specified metadata would be
    /// logged.
    ///
    /// This is used by the `log_enabled!` macro to allow callers to avoid
    /// expensive computation of log message arguments if the message would be
    /// discarded anyway.
    fn enabled(&self, metadata: &Metadata) -> bool;
    /// Logs the `Record`.
    ///
    /// Note that `enabled` is *not* necessarily called before this method.
    /// Implementations of `log` should perform all necessary filtering
    /// internally.
    fn log(&self, record: &Record);
}
```

在主函数的开头, 您可以调用 [set_logger](https://docs.rs/log/latest/log/fn.set_logger.html) 函数并传递 Log trait 的实现: 每次发出日志记录时，都会在您提供的记录器上调用 `Log::log`，从而可以执行您认为必要的任何形式的日志记录处理。

如果不调用 set_logger，所有日志记录都会被丢弃。这正是我们应用程序所发生的事情。

这次让我们初始化我们的日志记录器。

[crates.io](https://docs.rs/env_logger) 上有一些日志实现 - 最常用的选项列在 log 本身的文档中。

我们将使用 [env_logger](https://docs.rs/env_logger) - 如果像我们的例子一样，主要目标是将所有日志记录打印到终端，那么它就非常有效。

让我们将其添加为依赖项

```shell
cargo add env_logger
```

`env_logger::Logger` 将日志记录打印到终端，使用以下格式:

```plaintext
[<timestamp> <level> <module path>] <log message>
```

它会查看 RUST_LOG 环境变量来确定哪些日志应该打印，哪些日志应该被过滤掉。

例如, `RUST_LOG=debug cargo run` 会显示由我们的应用程序或我们正在使用的 crate 发出的所有调试级别或更高级别的日志。而 `RUST_LOG=zero2prod` 则会过滤掉由我们的依赖项发出的所有记录。

让我们根据需要修改 `main.rs` 文件：

```rs
//! src/main.rs
// [...]
use env_logger::Env;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // `init` does call `get_logger`, so this is all we need to do.
    // We are falling back to printing all logs at into-level or above
    // if the RUST_LOG envirionment variable has not been set.
    env_logger::Builder::from_env(Env::default().default_filter_or("info")).init();

    // [...]
}
```

让我们尝试使用 cargo run 再次启动该应用程序（根据我们的默认逻辑，这相当于 `RUST_LOG=info cargo run`）。终端上应该会显示两条日志记录（使用换行符，并缩进以使其适合页边距）。

```plaintext
[2025-08-23T01:23:50Z INFO  actix_server::builder] starting 20 workers
[2025-08-23T01:23:50Z INFO  actix_server::server] Tokio runtime found; starting in existing Tokio r
untime
[2025-08-23T01:23:50Z INFO  actix_server::server] starting service: "actix-web-service-0.0.0.0:8000
", workers: 20, listening on: 0.0.0.0:8000
```

如果我们使用 curl 发送 `http://127.0.0.1:8000/health_check` 请求，你应该会看到另一条日志记录，
这条记录是由我们在前几段中添加的 Logger 中间件发出的。

```plaintext
[2025-08-23T01:24:24Z INFO  actix_web::middleware::logger] 127.0.0.1 "GET /health_check HTTP/1.1" 200 0 "-" "curl/8.15.0" 0.000094
```

日志也是探索我们正在使用的软件如何工作的绝佳工具。

尝试将 RUST_LOG 设置为 trace 并重新启动应用程序。

您应该会看到一堆来自 mio（一个用于非阻塞 IO 的底层库）的轮询器注册日志记录，以及由 actix-web 生成的每个工作进程的几个启动日志记录（每个工作进程对应您机器上每个可用的物理核心！）。

通过研究 trace 级别的日志，您可以学到很多有深度的东西。
