# 可维护测试套件的框架和原则

我们花了不少功夫，现在终于为 Postmark 的 API 打造了一个相当不错的 REST 客户端。

`EmailClient` 只是我们确认邮件流程的第一个组成部分：我们还没有找到一种方法来生成唯一的确认链接，然后我们必须将其嵌入到发出的确认邮件的正文中。

这两项任务都需要再等一段时间。

我们一直采用测试驱动的方法来编写本书中的所有新功能。

虽然这种策略效果很好，但我们并没有投入大量时间重构测试代码。

因此，我们的测试文件夹目前有点混乱。

在继续下一步之前，我们将重构集成测试套件，以应对应用程序复杂性和测试数量的增长。

## 我们为什么要写测试

编写测试是否合理利用了开发人员的时间?

好的测试套件首先是一种风险缓解措施。

自动化测试可以降低现有代码库变更带来的风险——大多数回归测试和错误都会被捕获在持续集成流水线中，而不会影响到用户。因此，团队能够更快地迭代并更频繁地发布。

测试也充当文档的作用。

深入研究未知代码库时，测试套件通常是最佳起点——它向您展示了代码的预期行为，以及哪些场景被认为足够相关，需要进行专门的测试。

如果您想让您的项目更受新贡献者的欢迎，“编写测试套件！”绝对应该列在您的待办事项清单上。

好的测试通常还会带来其他积极的副作用——模块化和解耦。这些副作用很难量化，因为作为一个行业，我们尚未就“好代码”的具体样貌达成一致。

## 为什么我们不写测试

尽管投入时间和精力编写优秀的测试套件的理由令人信服，但现实情况却并非如此。

首先，开发社区并非一直相信测试的价值。

在整个测试驱动开发学科的发展史上，我们都能找到不少例子，但直到1999年《极限编程》(XP) 一书的出版，这项实践才进入主流讨论!

范式转变并非一朝一夕之功——测试驱动方法花了数年时间才在行业内获得认可，成为“最佳实践”。

如果说测试驱动开发已经赢得了开发人员的青睐，那么与管理层的斗争往往仍在继续。

好的测试能够构建技术杠杆，但编写测试需要时间。当截止日期迫在眉睫时，测试往往是第一个被牺牲的。

因此，你找到的大多数资料要么是测试的介绍，要么是如何向利益相关者推介其价值的指南。

关于大规模测试的内容很少——如果你坚持照本宣科，继续编写测试，当代码库增长到数万行，包含数百个测试用例时，会发生什么?

## 测试代码仍然是代码

所有测试套件都以同样的方式开始：一个空文件，一个充满可能性的世界。

你进入文件，添加第一个测试。很简单，搞定。

然后是第二个。轰！

第三个。你只需要从第一个代码中复制几行，就完成了。

第四个……

一段时间后，测试覆盖率开始下降：新代码的测试不如你在项目初期编写的代码那么彻底。你是否开始怀疑测试的价值了?

绝对没有，测试很棒!

然而，随着项目的推进，你编写的测试越来越少。

这是因为摩擦——随着代码库的演变，编写新的测试变得越来越繁琐。

> 测试代码仍然是代码

它必须是模块化的、结构良好的、文档齐全的。它需要维护。

如果我们不积极投入精力维护测试套件的健康状况，它就会随着时间的推移而腐烂。

覆盖率下降，很快我们就会发现应用程序代码中的关键路径从未被自动化测试执行过。

你需要定期回顾一下你的测试套件，从整体上审视它。

是时候审视一下我们的测试套件了，不是吗?

## 我们的测试套件

我们所有的集成测试都位于一个文件 `tests/health_check.rs` 中:

```rs
//! tests/health_check.rs
// [...]
//! tests/health_check.rs

// Ensure that the `tracing` stack is only initialised once using `once_cell`
static TRACING: Lazy<()> = Lazy::new(|| {
    // [...]
});

pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

async fn spawn_app() -> TestApp {
    // [...]
}

pub async fn configure_database(config: &DatabaseSettings) -> PgPool {
    // [...]
}

#[tokio::test]
async fn health_check_works() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_400_when_data_is_missing() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_200_when_fields_are_present_but_invalid() {
    // [...]
}
```

## 测试发现

只有一个测试用于处理我们的健康检查端点 - health_check_works。

其他三个测试用于探测我们的 `POST /subscriptions` 端点，而其余代码则用于处理共享的设置步骤（`spawn_app`、`TestApp`、`configure_database` 和 `TRACING`）。

为什么我们将所有内容都放在 `tests/health_check.rs` 中?

因为这样很方便!

设置函数已经存在 - 在同一个文件中添加另一个测试用例比弄清楚如何在多个测试模块之间正确共享代码更容易。

我们此次重构的主要目标是可发现性:

- 给定一个应用程序端点，应该能够在 `tests` 文件夹中轻松找到相应的集成测试
- 编写测试时，应该能够轻松找到相关的测试辅助函数

我们将重点介绍文件夹结构，但这绝对不是测试发现的唯一工具。

测试覆盖率工具通常可以告诉您哪些测试触发了特定应用程序代码行的执行。

您可以依靠覆盖率标记等技术在测试和应用程序代码之间建立明显的联系。

与往常一样，随着测试套件复杂度的增加，多管齐下的方法可能会为您带来最佳效果。

## 一个测试文件, 一个 Crate

在开始移动之前，让我们先了解一些关于 Rust 集成测试的知识。

`tests` 文件夹比较特殊——Cargo 知道去里面查找集成测试。

`tests` 文件夹下的每个文件都会被编译成一个独立的 crate。

我们可以通过运行 `cargo build --tests` 命令，然后在 target/debug/deps 目录下查找来验证这一点:

```shell
# Build test code, without running tests
cargo build --tests
# Find all files with a name starting with `health_check`
ls target/debug/deps | grep health_check
```

```plaintext
health_check-3b9db97bf61f8d77
health_check-3b9db97bf61f8d77.d
health_check-e064e6b2bbe87e3b.d
libhealth_check-e064e6b2bbe87e3b.rmeta
```

您的机器上的尾部哈希值可能会有所不同，但应该有两个以 `health_check-*` 开头的条目。

如果您尝试运行它会发生什么?

```plaintext
running 4 tests
test health_check_works ... ok
test subscribe_returns_a_400_when_fields_are_present_but_invalid ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok
test subscribe_returns_a_200_for_valid_form_data ... ok
test result: ok. 4 passed; finished in 0.44s
```

没错，它运行我们的集成测试!

如果我们测试了五个 `*.rs` 文件，我们会在 `target/debug/deps` 中找到五个可执行文件。

## 共享测试 Helper 方法

如果每个集成测试文件都是独立的可执行文件，那么我们如何共享测试辅助函数呢?

第一个选项是定义一个独立的模块，例如 `tests/helpers.rs` 。

您可以在 `helper.rs` 中添加常用函数（或在其中定义其他子模块），然后在测试文件 (例如 `tests/health_check.rs`) 中引用这些辅助函数，如下所示:

```rs
//! tests/health_check.rs
// [...]
mod helpers;

// [...]
```

`helpers` 被捆绑在 `health_check` 测试可执行文件中，并作为子模块，我们可以访问它在测试用例中公开的函数。
这种方法一开始效果不错，但最终会导致恼人的“函数从未使用”警告。

问题在于, `helpers` 被捆绑为子模块，而不是作为第三方 crate 调用: cargo 会单独编译每个测试可执行文件，如果某个测试文件中的 `helpers` 中的一个或多个公共函数从未被调用，则会发出警告。随着测试套件的增长，这种情况必然会发生——并非所有测试文件都会使用所有辅助方法。

第二种方案充分利用了 tests 下每个文件都是独立可执行文件这一特点——我们可以创建作用域为单个测试可执行文件的子模块！

让我们在 `tests` 下创建一个 `api` 文件夹，其中包含一个 `main.rs` 文件:

```plaintext
tests/
  api/
    main.rs
  health_check.rs
```

首先，我们要明确一点：我们构建 API 的方式与构建二进制 crate 的方式完全相同。它没有那么多魔法——它建立在你编写应用程序代码时构建的模块系统知识之上。

如果你运行 `cargo build --tests`，你应该能够发现

```plaintext
Running target/debug/deps/api-0a1bfb817843fdcf

running 0 tests

test result: ok. 0 passed; finished in 0.00s
```

在输出中 - cargo 将 api 编译为测试可执行文件，并查找测试用例。

无需在 `main.rs` 中定义 `main` 函数 - Rust 测试框架会在后台为我们添加一个。

我们现在可以在 `main.rs` 中添加子模块:

```rs
//! tests/api/main.rs
mod helpers;
mod health_check;
mod subscriptions;
```

添加三个空文件: `tests/api/helpers.rs`、`tests/api/health_check.rs` 和 `tests/api/subscriptions.rs`。
现在是时候删除 `tests/health_check.rs` 并重新分发其内容了:

```rs
//! tests/api/helpers.rs
use fake::{Fake, Faker};
use once_cell::sync::Lazy;
use secrecy::SecretBox;
use sqlx::Executor;
use std::net::TcpListener;

use sqlx::{Connection, PgConnection, PgPool};
use uuid::Uuid;
use zero2prod::{
    configuration::{DatabaseSettings, get_configuration},
    email_client::EmailClient,
    telemetry::{get_subscriber, init_subscriber},
};

// Ensure that the `tracing` stack is only initialised once using `once_cell`
static TRACING: Lazy<()> = Lazy::new(|| {
    /// [...]
});

pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

pub async fn spawn_app() -> TestApp {
    // [...]
}

pub async fn configure_database(config: &DatabaseSettings) -> PgPool {
    // [...]
}

```

```rs
//! tests/api/health_check.rs
use crate::helpers::spawn_app;

#[tokio::test]
async fn health_check_works() {
    // [...]
}
```

```rs
//! tests/api/subscriptions.rs
use crate::helpers::spawn_app;

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_400_when_data_is_missing() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_200_when_fields_are_present_but_invalid() {
    // [...]
}
```

`cargo test` 应该会成功，并且不会出现任何警告。

恭喜，您已将测试套件分解为更小、更易于管理的模块!

新的结构有一些积极的副作用: 它是递归的。

如果 `tests/api/subscriptions.rs` 变得过于庞大，我们可以将其转换为一个模块，其中 `tests/api/subscriptions/helpers.rs` 包含特定于订阅的测试帮助程序以及一个或多个专注于特定流程或关注点的测试文件; - 我们的帮助程序函数的实现细节被封装了。

事实证明，我们的测试只需要了解 `spawn_app` 和 `TestApp` - 无需暴露 `configure_database` 或 TRACING，我们可以将这些复杂性隐藏在帮助程序模块中 - 我们只有一个测试二进制文件。

如果您有一个采用扁平文件结构的大型测试套件，那么每次运行 `cargo test` 时，您很快就会构建数十个可执行文件。虽然每个可执行文件都是并行编译的，但[链接](https://en.wikipedia.org/wiki/Linker_(computing))阶段却是完全顺序执行的！将所有测试用例打包成一个可执行文件可以减少在 CI 中编译测试套件的时间。

如果您正在运行 Linux，您可能会看到类似这样的错误

```plaintext
thread 'actix-rt:worker' panicked at
'Can not create Runtime: Os { code: 24, kind: Other, message: "Too many open files" }',
```

重构后运行 cargo test 时。
这是由于操作系统对每个进程打开的文件描述符（包括套接字）的最大数量进行了限制——考虑到我们现在将所有测试都作为单个二进制文件的一部分运行，我们可能会超出这个限制。该限制通常设置为 1024，但您可以使用 `ulimit -n X` (例如 `ulimit -n 10000`) 来提高该限制以解决此问题。

## 共享 Startup 逻辑

现在我们已经重新设计了测试套件的布局，是时候深入研究测试逻辑本身了。

我们将从 spawn_app 开始:

```rs
//! tests/api/helpers.rs
// [...]


pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

pub async fn spawn_app() -> TestApp {
    // The first time `initialize` is invoked the code in `TRACING` is executed.
    // All other invocations will instead skip execution.
    Lazy::force(&TRACING);

    let listener = TcpListener::bind("127.0.0.1:0").expect("Faield to bind random port");
    // We retrieve the port assigned to us by the OS
    let port = listener.local_addr().unwrap().port();
    let address = format!("http://127.0.0.1:{}", port);

    let mut configuration = get_configuration().expect("");

    configuration.database.database_name = Uuid::new_v4().to_string();

    let connection_pool = configure_database(&configuration.database).await;

    // Build a new email client
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

    let server = zero2prod::run(listener, connection_pool.clone(), email_client)
        .expect("Failed to bind address");
    let _ = tokio::spawn(server);

    TestApp {
        address,
        db_pool: connection_pool,
    }
}

// [...]
```

这里的大部分代码与我们在 `main` 入口点中发现的代码极为相似:

```rs
//! src/main.rs

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let subscriber = get_subscriber("zero2prod", "info", std::io::stdout);
    init_subscriber(subscriber);

    let configuration = get_configuration().expect("Failed to read config");
    let connection_pool = PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy_with(configuration.database.with_db());

    let sender_email = configuration.email_client.sender()
        .expect("Invalid sender email address");
    let timeout = configuration.email_client.timeout();

    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email,
        configuration.email_client.authorization_token,
        timeout,
    );

    let address = format!(
        "{}:{}",
        configuration.application.host, configuration.application.port
    );
    let listener = TcpListener::bind(address)?;

    run(listener, connection_pool, email_client)?.await
}
```

每次添加依赖项或修改服务器构造函数时，我们至少有两个地方需要修改——最近我们只是敷衍地修改了 `EmailClient`。这有点烦人。

更重要的是，我们应用程序代码中的启动逻辑从未经过测试。

随着代码库的演变，它们可能会开始出现细微的差异，导致测试代码与生产环境中的行为有所不同。

我们将首先从主函数中提取逻辑，然后找出在测试代码中利用相同代码路径所需的钩子。

### 提取 Startup 代码

从结构角度来看，我们的启动逻辑是一个函数，
它以“设置”作为输入，并返回一个应用程序实例作为输出。

因此，我们的主函数应该如下所示:

```rs
//! src/main.rs

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let subscriber = get_subscriber("zero2prod", "info", std::io::stdout);
    init_subscriber(subscriber);

    let configuration = get_configuration().expect("Failed to read config");

    let server = build(configuration).await?;
    server.await?;
    Ok(())
}
```

我们首先执行一些二进制特定的逻辑（例如遥测初始化），然后从支持的源（文件 + 环境变量）构建一组配置值，并使用它来启动一个应用程序。线性的。

然后我们定义这个 `build` 函数:

```rs
//! src/startup.rs
// [...]
use crate::configuration::Settings;
use sqlx::postgres::PgPoolOptions;

pub async fn build(configuration: Settings) -> Result<Server, std::io::Error> {
    let connection_pool = PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy_with(configuration.database.with_db());

    // Build an `EmailClient` using `configuration`
    let sender_email = configuration
        .email_client
        .sender()
        .expect("Invalid sender email address");
    let timeout = configuration.email_client.timeout();

    let email_client = EmailClient::new(
        configuration.email_client.base_url,
        sender_email,
        configuration.email_client.authorization_token,
        timeout,
    );

    let address = format!(
        "{}:{}",
        configuration.application.host, configuration.application.port
    );
    let listener = TcpListener::bind(address)?;

    run(listener, connection_pool, email_client)
}
```

没什么特别的——我们只是移动了之前主函数里的代码。现在就让它更易于测试吧!

### 在我们的启动逻辑中测试钩子

让我们再次看一下 spawn_app 函数:

```rs
//! tests/api/helpers.rs
// [...]
use zero2prod::startup::build;
// [...]

pub async fn spawn_app() -> TestApp {
    // The first time `initialize` is invoked the code in `TRACING` is executed.
    // All other invocations will instead skip execution.
    Lazy::force(&TRACING);

    let listener = TcpListener::bind("127.0.0.1:0").expect("Faield to bind random port");
    // We retrieve the port assigned to us by the OS
    let port = listener.local_addr().unwrap().port();
    let address = format!("http://127.0.0.1:{}", port);

    let mut configuration = get_configuration().expect("");

    configuration.database.database_name = Uuid::new_v4().to_string();

    let connection_pool = configure_database(&configuration.database).await;

    // Build a new email client
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

    let server = zero2prod::run(listener, connection_pool.clone(), email_client)
        .expect("Failed to bind address");
    let _ = tokio::spawn(server);

    TestApp {
        address,
        db_pool: connection_pool,
    }
}
```

概括来说，我们有以下几个阶段:

- 执行特定于测试的设置（例如，初始化跟踪订阅者）；
- 随机化配置以确保测试不会相互干扰（例如，为每个测试用例使用不同的逻辑数据库）；
- 初始化外部资源（例如，创建和迁移数据库！）；
- 构建应用程序；
- 将应用程序作为后台任务启动，并返回一组与其交互的资源。

我们可以直接把构建过程放在那里就完事了吗?

当然不行，但让我们尝试看看它有哪些不足之处:

```rs
//! tests/api/helpers.rs
// [...]

pub async fn spawn_app() -> TestApp {
    Lazy::force(&TRACING);

    // Randomise configuration to ensure test isolation
    let mut configuration = {
        let mut c = get_configuration().expect("Failed to read configuration.");
        c.database.database_name = Uuid::new_v4().to_string();
        c.application_port = 0;
        c
    };

    // Create and migrate the database
    let connection_pool = configure_database(&configuration.database).await;

    // Launch the application as a background task
    let server = build(configuration).await.expect("Failed to build application.");
    let _ = tokio::spawn(server);

    TestApp {
        // How do we get these?
        address: todo!(),
        db_pool: todo!()
    }
}
```

它几乎成功了——但最终却出了问题: 我们无法检索操作系统分配给应用程序的随机地址，而且我们也不知道如何构建一个连接到数据库的连接池，而这个连接池需要对影响持久状态的副作用执行断言。

我们先来处理连接池: 我们可以将构建过程中的初始化逻辑提取到一个独立的函数中，并调用它两次。

```rs
//! src/startup.rs
// [...]

// We are taking a reference now!
pub async fn build(configuration: &Settings) -> Result<Server, std::io::Error> {
    let connection_pool = get_connection_pool(&configuration.database);

    // [...]
}


pub fn get_connection_pool(configuration: &DatabaseSettings) -> PgPool {
    PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy_with(configuration.with_db())
}
```

```rs
//! tests/api/helpers.rs
// [...]

pub async fn spawn_app() -> TestApp {
    Lazy::force(&TRACING);

    // Randomise configuration to ensure test isolation
    let configuration = {
        let mut c = get_configuration().expect("Failed to read configuration")   ;
        c.database.database_name = Uuid::new_v4().to_string();
        c.application.port = 0;
        c
    };

    // Create and migrate the database
    configure_database(&configuration.database).await;

    // Launch the application as a background task
    let server = build(&configuration).await.expect("Failed to build application") ;
    let _ = tokio::spawn(server);

    TestApp {
        address: todo!(),
        db_pool: get_connection_pool(&configuration.database),
    }
}
```

您必须在 `src/configuration.rs` 中的所有结构体中添加 `#[derive(Clone)]` 才能使编译器正常运行，但数据库连接池已经完成了。

我们如何获取应用程序地址呢?

`actix_web::dev::Server` 是 `build` 返回的类型，它不允许我们检索应用程序端口。

注: 你还需要将 `SecretBox` 修改为 `SecretString` 来满足 Clone trait 的要求, 使用 `SecretString::from(String)` 创建 `SecretString`

我们需要在您的应用程序代码中做一些准备工作——我们将把 `actix` `web::dev::Server` 包装成一个新的类型，以保存我们想要的信息。

```rs
//! src/startup.rs
// [...]

// A new type to hold the newly built server and its port
pub struct Application {
    port: u16,
    server: Server,
}

impl Application {
    // We have converted the `build` function into a constructor for
    // `Application`.
    pub async fn build(configuration: Settings) -> Result<Self, std::io::Error> {
        let connection_pool = get_connection_pool(&configuration.database);

        let sender_email = configuration
            .email_client
            .sender()
            .expect("Invalid sender email address.");
        let email_client = EmailClient::new(
            configuration.email_client.base_url,
            sender_email,
            configuration.email_client.authorization_token,
            std::time::Duration::from_secs(10),
        );

        let address = format!(
            "{}:{}",
            configuration.application.host,
            configuration.application.port
        );
        let listener = TcpListener::bind(&address)?;
        let port = listener.local_addr()?.port();
        let server = run(listener, connection_pool, email_client)?;

        Ok(Self {
            port,
            server
        })
    }


    pub fn port(&self) -> u16 {
        self.port
    }

    // A more expressive name that makes it clear that
    // this function only returns when the application is stopped.
    pub async fn run_until_stopped(self) -> Result<(), std::io::Error> {
        self.server.await
    }
}
```

```rs
//! tests/api/helpers.rs
// [...]
// New import!
use zero2prod::startup::Application;

pub async fn spawn_app() -> TestApp {
    Lazy::force(&TRACING);

    // Randomise configuration to ensure test isolation
    let configuration = {
        let mut c = get_configuration().expect("Failed to read configuration")   ;
        c.database.database_name = Uuid::new_v4().to_string();
        c.application.port = 0;
        c
    };

    // Create and migrate the database
    configure_database(&configuration.database).await;

    let application = Application::build(configuration.clone())
        .await
        .expect("Failed to build application.");

    // Get the port before spawning the application
    let address = format!("http://127.0.0.1:{}", application.port());
    let _ = tokio::spawn(application.run_until_stopped());

    TestApp {
        address,
        db_pool: get_connection_pool(&configuration.database),
    }
}
```

```rs
//! src/main.rs
// New import!
use zero2prod::startup::Application;

async fn main() -> std::io::Result<()> {
    // [...]

    let application = Application::build(configuration).await?;
    application.run_until_stopped().await?;
    Ok(())
}
```

完成了 - 如果您想再次检查，请运行 `cargo test` !
