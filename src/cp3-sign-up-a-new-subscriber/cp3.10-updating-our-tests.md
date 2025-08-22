# 更新我们的测试

错误出在我们的 `spawn_app` 辅助函数中:

```rs
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

我们需要传递一个连接池来运行。

考虑到我们接下来在 `subscribe_returns_a_200_for_valid_form_data` 中需要用到这个连接池来执行 SELECT 查询，因此可以对 `spawn_app` 进行泛化: 我们不会返回原始字符串，而是会给调用者一个结构体 `TestApp`。`TestApp` 将保存测试应用程序实例的地址和连接池的句柄，从而简化测试用例的配置步骤。

```rs
//! tests/health_check.rs
use std::net::TcpListener;

use sqlx::{Connection, PgConnection, PgPool};
use zero2prod::configuration::get_configuration;

pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

async fn spawn_app() -> TestApp {
    let listener = TcpListener::bind("127.0.0.1:0").expect("Faield to bind random port");
    // We retrieve the port assigned to us by the OS
    let port = listener.local_addr().unwrap().port();
    let address = format!("http://127.0.0.1:{}", port);

    let configuration = get_configuration().expect("");
    let connection_pool = PgPool::connect(&configuration.database.connection_string())
        .await
        .expect("Failed to connect to Postgres");
    let server = zero2prod::run(listener, connection_pool.clone()).expect("Failed to bind address");
    let _ = tokio::spawn(server);

    TestApp {
        address,
        db_pool: connection_pool,
    }
}
```

所有测试用例都必须进行相应的更新——这个屏幕外的练习，我留给你了，亲爱的读者。

让我们一起来看看 subscribe_returns_a_200_for_valid_form_data 在完成必要的更改后是什么样子的:

```rs
//! tests/health_check.rs
// [...]

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // Arrange
    let app = spawn_app().await;
    let app_address = app.address.as_str();

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

    let saved = sqlx::query!("SELECT email, name FROM subscriptions",)
        .fetch_one(&app.db_pool)
        .await
        .expect("Failed to fetch saved subscription.");

    assert_eq!(saved.email, "ursula_le_guin@gmail.com");
    assert_eq!(saved.name, "le guin");
}
```

现在, 我们去掉了大部分与建立数据库连接相关的样板代码, 测试意图更加清晰了。

`TestApp` 是我们未来构建的基础, 我们将在此基础上开发出对大多数集成测试都有用的支持功能。

关键时刻终于到来了: 我们更新后的订阅实现是否足以让 `subscribe_returns_a_200_for_valid_form_data` 测试通过?

```plaintext
running 3 tests
test health_check_works ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok
test subscribe_returns_a_200_for_valid_form_data ... ok
```

Yesssssssss!

成功了!

让我们再次奔跑，沐浴在这辉煌时刻的光芒之中!

```shell
cargo test
```

```plaintext
running 3 tests
test health_check_works ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok
test subscribe_returns_a_200_for_valid_form_data ... FAILED

failures:

---- subscribe_returns_a_200_for_valid_form_data stdout ----
Failed to execute query: error returned from database: duplicate key value violates unique constrai
nt "subscriptions_email_key"

thread 'subscribe_returns_a_200_for_valid_form_data' panicked at tests/health_check.rs:45:5:
assertion `left == right` failed
  left: 200
 right: 500
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    subscribe_returns_a_200_for_valid_form_data

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.07s

```

等等，不，搞什么鬼! 别这样对我们!

好吧，我撒谎了——我就知道会发生这种事。

对不起，我让你尝到了胜利的甜蜜滋味，然后又把你扔回了泥潭。

相信我，这里有一个重要的教训需要吸取。

## 测试隔离

您的数据库是一个巨大的全局变量：所有测试都与其交互，并且它们留下的任何数据都将对套件中的其他测试以及后续测试运行可用。

这正是我们刚才发生的事情：我们的第一次测试运行命令我们的应用程序注册一个邮箱地址为 `ursula_le_guin@gmail.com` 的新用户；应用程序执行了命令。

当我们重新运行测试套件时，我们再次尝试使用相同的邮箱地址执行另一个 INSERT 操作, 但是,
我们对邮箱列的 UNIQUE 约束引发了唯一键冲突并拒绝了查询, 导致应用程序返回 500 INTERNAL_SERVER_ERROR 错误。

您真的不希望测试之间有任何形式的交互：这会使您的测试运行变得不确定，并最终导致虚假的测试失败，而这些失败极难排查和修复。

据我所知，在测试中与关系数据库交互时，有两种技术可以确保测试隔离性:

- 将整个测试包装在 SQL 事务中，并在事务结束时回滚；
- 为每个集成测试启动一个全新的逻辑数据库。

第一种方法很巧妙，通常速度更快：回滚 SQL 事务比启动新的逻辑数据库所需的时间更少。在为查询编写单元测试时，这种方法效果很好，但在像我们这样的集成测试中，实现起来比较棘手：我们的应用程序将从 PgPool 借用一个 PgConnection，而我们无法在 SQL 事务上下文中“捕获”该连接。

这就引出了第二种方案：可能速度更慢，但实现起来更容易。

如何实现?

在每次测试运行之前，我们需要:

- 创建一个具有唯一名称的新逻辑数据库；
- 在其上运行数据库迁移。

执行此操作的最佳位置是 spawn_app，在启动我们的 actix-web 测试应用程序之前。

让我们再看一下:

```rs
// [...]
pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

async fn spawn_app() -> TestApp {
    let listener = TcpListener::bind("127.0.0.1:0").expect("Faield to bind random port");

    let port = listener.local_addr().unwrap().port();
    let address = format!("http://127.0.0.1:{}", port);

    let configuration = get_configuration().expect("");
    let connection_pool = PgPool::connect(&configuration.database.connection_string())
        .await
        .expect("Failed to connect to Postgres");
    let server = zero2prod::run(listener, connection_pool.clone()).expect("Failed to bind address");
    let _ = tokio::spawn(server);

    TestApp {
        address,
        db_pool: connection_pool,
    }
}
```

`configuration.database.connection_string()` 使用我们在 `configuration.yaml` 文件中指定的数据库名称 - 所有测试都相同。

让我们将其随机化

```rs
let mut configuration = get_configuration().expect("");
configuration.database.database_name = Uuid::new_v4().to_string();
```

`cargo test` 将会失败: 没有数据库准备好使用我们生成的名称来接受连接。

让我们在 `DatabaseSettings` 中添加一个 `connection_string_without_db` 方法:

```rs
//! src/configuration.rs
// [...]
impl DatabaseSettings {
    pub fn connection_string(&self) -> String {
        format!(
            "postgres://{}:{}@{}:{}/{}",
            self.username, self.password, self.host, self.port, self.database_name
        )
    }

    pub fn connection_string_without_db(&self) -> String {
        format!(
            "postgres://{}:{}@{}:{}",
            self.username, self.password, self.host, self.port
        )
    }
}
```

省略数据库名称，我们连接到 Postgres 实例, 而不是特定的逻辑数据库。

现在我们可以使用该连接创建所需的数据库并在其上运行迁移:

```rs
//! tests/health_check.rs
// [...]
use sqlx::Executor;

// [...]
async fn spawn_app() -> TestApp {
    // [...]
    let mut configuration = get_configuration().expect("");
    configuration.database.database_name = Uuid::new_v4().to_string();

    let connection_pool = configure_database(&configuration.database).await;
    // [...]
}

pub async fn configure_database(config: &DatabaseSettings) -> PgPool {
    // Create database
    let mut connection = PgConnection::connect(&config.connection_string_without_db())
        .await
        .expect("Failed to connect to Postgres");

    connection
        .execute(format!(r#"CREATE DATABASE "{}";"#, config.database_name).as_str())
        .await
        .expect("Failed to create database.");

    // Migrate database
    let connection_pool = PgPool::connect(&config.connection_string())
        .await
        .expect("Failed to connect to Postgres");

    sqlx::migrate!("./migrations")
        .run(&connection_pool)
        .await
        .expect("Failed to migrate the database");

    connection_pool
}
```

`sqlx::migrate!` 与 `sqlx-cli` 在执行 `sqlx migration run` 时使用的宏相同——无需
再添加 Bash 脚本即可达到相同的效果。
让我们再次尝试运行 `cargo test`:

```plaintext
running 3 tests
test subscribe_returns_a_400_when_data_is_missing ... ok
test health_check_works ... ok
test subscribe_returns_a_200_for_valid_form_data ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

这次成功了，而且是永久性的。

您可能已经注意到，我们在测试结束时没有执行任何清理步骤——我们创建的逻辑数据库不会被删除。这是有意为之: 我们可以添加清理步骤，但我们的 Postgres 实例仅用于测试目的，如果在数百次测试运行之后，由于大量残留（几乎为空）数据库而导致性能下降，重启它很容易。
