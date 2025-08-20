# 实现我们的第一个集成测试

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

## Polishing

我们已经让它运行起来了，现在我们需要重新审视并改进它，如果需要或可能的话

### 清理

测试运行结束后，后台运行的应用会发生什么?

它会关闭吗? 它会像僵尸程序一样徘徊在某个地方吗?

嗯，连续多次运行 `cargo test` 总是会成功——这强烈暗示我们的 8000 端口会在每次运行结束时被释放，因此意味着应用已正确关闭。

再次查看 [`tokio::spawn` 的文档](https://docs.rs/tokio/latest/tokio/fn.spawn.html)，这支持了我们的假设：当一个 tokio 运行时关闭时，所有在其上生成的任务都会被丢弃。tokio::test 会在每个测试用例开始时启动一个新的运行时，并在每个测试用例结束时关闭。

换句话说，好消息是——无需实现任何清理逻辑来避免测试运行期间的资源泄漏。

### 选择随机端口

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
