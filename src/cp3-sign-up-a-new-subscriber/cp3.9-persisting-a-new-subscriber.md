# 持久化新订阅者

就像我们在测试中编写了 `SELECT` 查询来检查哪些订阅已持久保存到数据库中一样，现在我们需要编写 `INSERT` 查询，以便在收到有效的 **POST /subscriptions** 请求时实际存储新订阅者的详细信息。

让我们看看我们的handler:

```rs
use actix_web::{HttpResponse, web};

#[derive(serde::Deserialize)]
pub struct FormData {
    email: String,
    name: String,
}

pub async fn subscribe(_form: web::Form<FormData>) -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

要在 `subscribe` 中执行查询，我们需要获取数据库connection。

让我们来看看如何获​​取。

## actix-web 中的应用程序状态

到目前为止，我们的应用程序完全是无状态的：我们的处理程序仅处理来自传入请求的数据。

actix-web 让我们能够将与单个传入请求的生命周期无关的其他数据附加到应用程序 - 即所谓的应用程序状态。

您可以使用 `App` 上的 `app_data` 方法向应用程序状态添加信息。

让我们尝试使用 `app_data` 将 `PgConnection` 注册为应用程序状态的一部分。我们需要修改 `run` 方法，使其与 `TcpListener` 一起接受 `PgConnection`:

```rs
//! src/startup.rs
use std::net::TcpListener;

use actix_web::{dev::Server, web, App, HttpServer};
use sqlx::PgConnection;

use crate::routes::{health_check, subscribe};

pub fn run(
    listener: TcpListener,
    // New parameter!
    connection: PgConnection,
) -> Result<Server, std::io::Error> {
    let server = HttpServer::new(|| {
        App::new()
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(connection)
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

这不能真正工作, `cargo check` 提示如下信息

```plaintext
error[E0277]: the trait bound `PgConnection: Clone` is not satisfied in `{closure@src/startup.rs:13
:34: 13:36}`
  --> src/startup.rs:13:18
   |
13 |     let server = HttpServer::new(|| {
   |                  ^^^^^^^^^^      -- within this `{closure@src/startup.rs:13:34: 13:36}`
   |                  |
   |                  unsatisfied trait bound
   |
   = help: within `{closure@src/startup.rs:13:34: 13:36}`, the trait `Clone` is not implemented for
 `PgConnection`
note: required because it's used within this closure
  --> src/startup.rs:13:34
   |
13 |     let server = HttpServer::new(|| {
   |                                  ^^
note: required by a bound in `HttpServer`
  --> /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/actix-web-4.11.0/src/serve
r.rs:72:27
   |
70 | pub struct HttpServer<F, I, S, B>
   |            ---------- required by a bound in this struct
71 | where
72 |     F: Fn() -> I + Send + Clone + 'static,
   |                           ^^^^^ required by this bound in `HttpServer`

For more information about this error, try `rustc --explain E0277`.
```

`HttpServer` 期望 `PgConnection` 可克隆, 但不幸的是, 事实并非如此。

为什么它首先需要实现 `Clone` trait呢?

## actix-web Workers

让我们放大对 `HttpServer::new` 的调用:

```rs
let server = HttpServer::new(|| {
    App::new()
        .route("/health_check", web::get().to(health_check))
        .route("/subscriptions", web::post().to(subscribe))
})
```

HttpServer::new 不接受 App 作为参数——它需要一个返回 App 结构体的闭包。

这是为了支持 **actix-web** 的运行时模型：**actix-web** 会为您机器上的每个可用核心启动一个工作进程。

每个工作进程都运行由 HttpServer 构建的应用程序副本，并调用 HttpServer::new 作为参数的同一个闭包。
这就是为什么连接必须是可克隆的——我们需要为每个 App 副本创建一个连接。

但是，正如我们所说，PgConnection 没有实现 Clone 接口，因为它位于一个不可克隆的系统资源之上，即与 Postgres 的 TCP 连接。我们该怎么办?

我们可以使用另一个 **actix-web** 提取器 `web::Data`。

`web::Data` 将我们的连接包装在一个原子引用计数指针 [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) 中：应用程序的每个实例都将获得一个指向 PgConnection 的指针，而不是获取 PgConnection 的原始副本。

`Arc<T>` 始终可克隆，无论 `T` 是什么：克隆 `Arc` 会增加活动引用的数量，并移交包装值内存地址的拷贝。

然后，处理程序可以使用相同的提取器访问应用程序状态。

让我们试一下:

```rs
//! src/startup.rs
use std::net::TcpListener;

use actix_web::{dev::Server, web, App, HttpServer};
use sqlx::PgConnection;

use crate::routes::{health_check, subscribe};

pub fn run(
    listener: TcpListener,
    // New parameter!
    connection: PgConnection,
) -> Result<Server, std::io::Error> {
    let connection = web::Data::new(connection);

    let server = HttpServer::new(move || {
        App::new()
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(connection.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

它还不能编译，但我们只需要做一些整理工作:

```rs
error[E0061]: this function takes 2 arguments but 1 argument was supplied
  --> tests/health_check.rs:96:18
   |
96 |     let server = zero2prod::run(listener).expect("Failed to bind address");
   |                  ^^^^^^^^^^^^^^---------- argument #2 of type `PgConnection` is missing
   |
```

让我们快速修复这个问题:

```rs
//! src/main.rs
use std::net::TcpListener;

use sqlx::{Connection, PgConnection};
use zero2prod::{configuration::get_configuration, run};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let configuration = get_configuration().expect("Failed to read config");
    let connection = PgConnection::connect(&configuration.database.connection_string())
        .await
        .expect("Failed to connect to Postgres.");

    let address = format!("0.0.0.0:{}", configuration.application_port);
    let listener = TcpListener::bind(address)?;

    run(listener, connection)?.await
}
```

完美，编译通过。

## `Data` Extractor

现在, 我们可以在请求处理程序中获取 `Arc<PgConnection>` 了，订阅时可以使用 `web::Data` Extractor:

```rs
//! src/routes/subscriptions.rs
use actix_web::{HttpResponse, web};
use sqlx::PgConnection;

// [...]

pub async fn subscribe(
    _form: web::Form<FormData>,
    _connection: web::Data<PgConnection>,
) -> HttpResponse {
    HttpResponse::Ok().finish()
}
```

我们将 `Data` 称为提取器，但它究竟是从哪里提取 `PgConnection` 的呢?

`actix-web` 使用类型映射来表示其应用程序状态：一个 `HashMap`, 存储任意数据 (使用 `Any` 类型) 及其唯一类型标识符 (通过 `TypeId::of` 获取)。

当新请求到来时, `web::Data` 会计算您在签名中指定的类型 (在我们的例子中是 `PgConnection`) 的 TypeId，并检查类型映射中是否存在与其对应的记录。如果有，它会将检索到的 Any 值转换为您指定的类型(`TypeId` 是唯一的, 无需担心), 并将其传递给您的处理程序。

这是一种有趣的技术，可以执行在其他语言生态系统中可能被称为依赖注入的操作。

## `INSERT` 查询

我们终于在 `subscribe` 中建立了连接: 让我们尝试持久化新订阅者的详细信息。

我们将再次使用在快乐测试中用过的 `query!` 宏。

```rs
//! src/routes/subscriptions.rs
// [...]
use uuid::Uuid;
use chrono::Utc;

// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    connection: web::Data<PgConnection>,
) -> HttpResponse {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        Uuid::new_v4(),
        form.email,
        form.name,
        Utc::now()
    )
    .execute(connection.get_ref())
    .await;
    HttpResponse::Ok().finish()
}
```

让我们来剖析一下发生了什么：

- 我们将动态数据绑定到 INSERT 查询。$1 表示查询本身之后传递给 `query!` 的第一个参数, $2 表示第二个参数，依此类推。`query!` 在编译时会验证
  提供的参数数量是否与查询预期相符，以及它们的
  类型是否兼容 (例如，不能将数字作为 id 传递)
- 我们为 id 生成一个随机的 Uuid
- 我们为 subscribed_at 使用 UTC 时区的当前时间戳

我们还必须在 `Cargo.toml` 中添加两个新的依赖项来修复明显的编译器错误:

```shell
cargo add uuid --features=v4
cargo add chrono
```

如果我们尝试再次编译它会发生什么?

```plaintext
error[E0277]: the trait bound `&PgConnection: Executor<'_>` is not satisfied
   --> src/routes/subscriptions.rs:27:6
    |
27  |     .await;
    |      ^^^^^ the trait `Executor<'_>` is not implemented for `&PgConnection`
    |
    = help: the trait `Executor<'_>` is not implemented for `&PgConnection`
            but it is implemented for `&mut PgConnection`
    = note: `Executor<'_>` is implemented for `&mut PgConnection`, but not for `&PgConnection`
```

`execute` 需要一个实现 **sqlx** 的 [`Executor` trait](https://docs.rs/sqlx/latest/sqlx/trait.Executor.html) 的参数，事实证明, 正如我们在测试中编写的查询中所记得的那样, `&PgConnection` 并没有实现 `Executor` - 只有 `&mut PgConnection` 实现了。

为什么会这样?
sqlx 有一个异步接口，但它不允许你在同一个数据库连接上并发运行多个查询。

要求可变引用允许他们在 API 中强制执行此保证。你可以将可变引用视为唯一引用：编译器保证执行时确实拥有对该 PgConnection 的独占访问权，因为在整个程序中不可能同时存在两个指向相同值的活跃可变引用。相当巧妙。
尽管如此，这看起来像是我们把自己设计成了一个死胡同: `web::Data` 永远不会给我们提供对应用程序状态的可变访问权限。

我们可以利用[内部可变性](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)——例如，将我们的 PgConnection 置于锁（例如 [Mutex](https://docs.rs/tokio/1.17.0/tokio/sync/struct.Mutex.html)）之后，
这将允许我们同步对底层 TCP 套接字的访问，并在获取锁后获得对包装连接的可变引用。

我们可以让它工作，但这并不理想：我们被限制一次最多只能运行一个查询。这不太好。

让我们再看一下 sqlx 的 [Executor trait](https://docs.rs/sqlx/latest/sqlx/trait.Executor.html)的文档: 除了 &mut `PgConnection` 之外，还有什么实现了 Executor?

Bingo: 对 `PgPool` 的共享引用。

`PgPool` 是一个 `Postgres` 数据库连接池。它是如何绕过我们刚刚讨论过的 `PgConnection` 的并发问题的？
内部仍然有可变性，但类型不同：当你对
`&PgPool` 运行查询时，sqlx 会从池中借用一个 `PgConnection` 并用它来执行查询；如果没有可用的连接，它会创建一个新的或等待直到有连接释放。

这增加了我们的应用程序可以运行的并发查询数量，并提高了其弹性: 单个慢查询不会通过在连接锁上创建争用而影响所有传入请求的性能。

让我们重构运行、主函数和订阅函数来使用 `PgPool` 而不是单个 `PgConnection`:

```rs
//! src/main.rs
use std::net::TcpListener;

use sqlx::{Connection, PgPool};
use zero2prod::{configuration::get_configuration, run};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let configuration = get_configuration().expect("Failed to read config");
    // Renamed!
    let connection_pool = PgPool::connect(&configuration.database.connection_string())
        .await
        .expect("Failed to connect to Postgres.");

    let address = format!("0.0.0.0:{}", configuration.application_port);
    let listener = TcpListener::bind(address)?;

    run(listener, connection_pool)?.await
}
```

```rs
//! src/startup.rs
use std::net::TcpListener;

use actix_web::{dev::Server, web, App, HttpServer};
use sqlx::PgPool;

use crate::routes::{health_check, subscribe};

pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);

    let server = HttpServer::new(move || {
        App::new()
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

```rs
//! src/routes/subscriptions.rs
// No longer importing PgConnection!
use sqlx::PgPool;
// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>, // Renamed!
) -> HttpResponse {
    sqlx::query!(
      /* [...] */
    )
    .execute(pool.get_ref())
    .await;
    HttpResponse::Ok().finish()
}
```

编译器几乎很高兴: `cargo check` 向我们发出了警告。

```plaintext
warning: unused `Result` that must be used
  --> src/routes/subscriptions.rs:16:5
   |
16 | /     sqlx::query!(
17 | |         r#"
18 | |         INSERT INTO subscriptions (id, email, name, subscribed_at)
19 | |         VALUES ($1, $2, $3, $4)
...  |
26 | |     .execute(pool.get_ref())
27 | |     .await;
   | |__________^
   |
   = note: this `Result` may be an `Err` variant, which should be handled
   = note: `#[warn(unused_must_use)]` on by default
```

`sqlx::query` 可能会失败——它返回一个 `Result`, 这是 Rust 对易错函数建模的方式。

编译器提醒我们处理错误情况——让我们遵循以下建议:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>, // Renamed!
) -> HttpResponse {
    match sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        Uuid::new_v4(),
        form.email,
        form.name,
        Utc::now()
    )
    .execute(pool.get_ref())
    .await
    {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(e) => {
            println!("Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

`cargo check` 满足要求，但 `cargo test` 却不能满足要求:

```plaintext
error[E0061]: this function takes 2 arguments but 1 argument was supplied
  --> tests/health_check.rs:96:18
   |
96 |     let server = zero2prod::run(listener).expect("Failed to bind address");
   |                  ^^^^^^^^^^^^^^---------- argument #2 of type `Pool<Postgres>` is missing
   |
```
