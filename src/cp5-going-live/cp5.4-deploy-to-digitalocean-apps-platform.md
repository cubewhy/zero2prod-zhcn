# 部署到 DigitalOcean Apps Platform

注: 本章翻译过程中没有核实内容, 因为译者本人不用这个 Digital Ocean (也没有兴趣为其付费), 他本人认为使用SaaS平台托管自己的服务是不好的. 不要完全跳过本章, 建议还是看一下

我们已经构建了一个（非常棒的）容器化应用程序版本。现在就部署它吧!

## 设置

您必须在 Digital Ocean 的网站上[注册](https://cloud.digitalocean.com/registrations/new)。

注册账户后，请安装 `doctl` (Digital Ocean 的 CLI)——您可以在[此处找到说明](https://docs.digitalocean.com/reference/doctl/how-to/install/)。

在 Digital Ocean 的应用平台上托管并非免费——维护我们的应用及其相关数据库的正常运行大约需要每月 20 美元。

我建议您在每次会话结束时销毁该应用——这样可以将您的支出保持在 1 美元以下。我在撰写本章的过程中只花了 0.2 美元!

## 应用程序规范

Digital Ocean 的应用平台使用声明式配置文件来指定应用程序部署的样子——他们称之为 [App Spec](https://www.digitalocean.com/docs/app-platform/concepts/app-spec/)。
通过查看[参考文档](https://www.digitalocean.com/docs/app-platform/references/app-specification-reference/)以及一些示例，我们可以拼凑出 App Spec 的初稿。
让我们将这个清单 `spec.yaml` 放在项目目录的根目录下。

```yaml
#! spec.yaml
name: zero2prod
# See https://www.digitalocean.com/docs/app-platform/#regional-availability for the available options
# You can get region slugs from https://www.digitalocean.com/docs/platform/availability-matrix/
# `fra` stands for Frankfurt (Germany - EU)
region: fra
services:
  - name: zero2prod
    # Relative to the repository root
    dockerfile_path: Dockerfile
    source_dir: .
    github:
      branch: main
      deploy_on_push: true
      repo: LukeMathWalker/zero-to-production
    # Active probe used by DigitalOcean's to ensure our application is healthy
    health_check:
      # The path to our health check endpoint! It turned out to be useful in the end!
      http_path: /health_check
    # The port the application will be listening on for incoming requests
    # It should match what we specify in our configuration.yaml file!
    http_port: 8000
    # For production workloads we'd go for at least two!
    instance_count: 1
    # Let's keep the bill lean for now...
    instance_size_slug: basic-xxs
    # All incoming requests should be routed to our app
    routes:
      - path: /
```

请花点时间仔细检查所有指定的值，并了解它们的用途。

我们可以使用它们的命令行界面 (CLI) doctl 来首次创建应用程序:

```shell
doctl apps create --spec spec.yaml
```

```plaintext
Error: Unable to initialize DigitalOcean API client: access token is required.
(hint: run 'doctl auth init')
```

嗯，我们必须先进行身份验证。

我们按照他们的建议来吧:

```shell
doctl auth init
```

```plaintext
Please authenticate doctl for use with your DigitalOcean account.
You can generate a token in the control panel at
https://cloud.digitalocean.com/account/api/tokens
```

一旦您提供了令牌，我们可以再试一次:

```shell
doctl apps create --spec spec.yaml
```

好的，按照他们的[指示](https://www.digitalocean.com/docs/app-platform/how-to/troubleshoot-app/#review-github-permissions)关联你的 GitHub 帐户。

第三次就成功了，我们再试一次!

```shell
doctl apps create --spec spec.yaml
```

成功了!

您可以使用以下命令检查应用状态

```shell
doctl apps list
```

或者查看 [DigitalOcean 的仪表盘](https://cloud.digitalocean.com/apps/)。

虽然应用已成功创建，但尚未运行!

查看他们仪表盘上的“部署”选项卡 - 它可能正在构建 Docker 镜像。

查看他们 [bug 跟踪器](https://www.digitalocean.com/community/questions/docker-build-too-slow-in-app-platform-does-the-speed-depend-on-the-app-resources-used)上最近的一些问题，发现可能需要一段时间 - 不少人都报告了构建速度缓慢的问题。Digital Ocean 的支持工程师建议利用 Docker 层缓存来缓解这个问题 - 我们已经涵盖了所有基础内容!

> 如果您在 DigitalOcean 上构建 Docker 镜像时遇到内存不足错误，请查看此 [GitHub issue](https://github.com/LukeMathWalker/zero-to-production/issues/71)。

等待这些行显示在其仪表板构建日志中:

```plaintext
zero2prod | 00:00:20 => Uploaded the built image to the container registry
zero2prod | 00:00:20 => Build complete
```

部署成功!

您应该能够看到每十秒左右一次的健康检查日志，此时Digital Ocean 平台会 ping 我们的应用程序以确保其正常运行。

```shell
doctl apps list
```

你可以检索应用程序的公开 URI。类似于 `https://zero2prod-aaaaa.ondigitalocean.app`

现在尝试发送健康检查请求，它应该会返回 200 OK!

请注意，DigitalOcean 通过配置证书并将 HTTPS 流量重定向到我们在应用程序规范中指定的端口，帮我们设置了 HTTPS。这样就省了一件需要担心的事情。

`POST /subscriptions` 端点仍然失败，就像它在本地失败一样：我们的生产环境中没有支持我们应用程序的实时数据库。

让我们配置一个。

将此段添加到您的 spec.yaml 文件中:

```yaml
#! spec.yaml
# [...]
databases:
  # PG = Postgres
  - engine: PG
    # Database name
    name: newsletter
    # Again, let's keep the bill lean
    num_nodes: 1
    size: db-s-dev-database
    # Postgres version - using the latest here
    version: "14"
```

然后更新您的应用规范:

```shell
# You can retrieve your app id using `doctl apps list`
doctl apps update YOUR-APP-ID --spec=spec.yaml
```

DigitalOcean 需要一些时间来配置 Postgres 实例。

与此同时，我们需要弄清楚如何在生产环境中将应用程序指向数据库。

如何使用环境变量注入 secret

连接字符串将包含我们不想提交到版本控制的值 - 例如，数据库根用户的用户名和密码。

我们最好的选择是使用环境变量在运行时将机密信息注入应用程序环境。例如，DigitalOcean 的应用程序可以引用 DATABASE_URL 环境变量（或其他一些[更精细的视图](https://www.digitalocean.com/docs/app-platform/how-to/use-environment-variables/)）来在运行时获取数据库连接字符串。

我们需要（再次）升级 `get_configuration` 函数以满足我们的新需求。

```rs
//! src/configuration.rs
// [...]

pub fn get_configuration() -> Result<Settings, config::ConfigError> {
    let mut settings = config::Config::builder();

    let base_path = std::env::current_dir().expect("Failed to determine the current directory");
    let configuration_directory = base_path.join("configuration");

    // Read the "default" configuration file
    settings = settings
        .add_source(config::File::from(configuration_directory.join("base")).required(true));

    // Detect the running environment.
    // Default to `local` if unspecified.
    let environment: Environment = std::env::var("APP_ENVIRONMENT")
        .unwrap_or_else(|_| "local".into())
        .try_into()
        .expect("Failed to parse APP_ENVIRONMENT.");

    // Layer on the environment-specific values
    settings = settings.add_source(
        config::File::from(configuration_directory.join(environment.as_str())).required(true),
    );

    // Add in settings from environment variables (with a prefix of APP and '__' as separator)
    // E.g. `APP_APPLICATION__PORT=5001` would set `Settings.application.port`
    settings = settings.add_source(config::Environment::with_prefix("app").separator("__"));

    settings.build().unwrap().try_deserialize()
}
```

这使我们能够使用环境变量自定义 Settings 结构体中的任何值，从而覆盖配置文件中指定的值。

为什么这很方便? 它使得注入过于动态（即无法预先知道）或过于敏感而无法存储在版本控制中的值成为可能。

它还可以快速更改应用程序的行为: 如果我们想要调整其中一个值（例如数据库端口），则无需进行完全重建。

对于像 Rust 这样的语言来说，全新构建可能需要十分钟或更长时间，这可能会导致短暂的中断和对客户造成可见影响的严重服务质量下降。

在继续之前，我们先处理一个烦人的细节: 环境变量对于配置包来说，是字符串，如果使用 serde 的标准反序列化例程，它将无法获取整数。

幸运的是，我们可以指定一个自定义反序列化函数。

让我们添加一个新的依赖项，serde-aux（serde 辅助函数）:

```shell
cargo add serde-aux
```

然后修改 `ApplicationSettings` 和 `DatabaseSettings`

```rs
//! src/configuration.rs
// [...]
use serde_aux::prelude::deserialize_number_from_string;

#[derive(serde::Deserialize)]
pub struct DatabaseSettings {
    #[serde(deserialize_with = "deserialize_number_from_string")]
    pub port: u16,
    // [...]
}

#[derive(serde::Deserialize)]
pub struct ApplicationSettings {
    #[serde(deserialize_with = "deserialize_number_from_string")]
    pub port: u16,
    // [...]
}

// [...]
```

## 连接到 Digital Ocean 的 Postgres 实例

让我们使用 DigitalOcean 仪表板（Compo-
nents -> Database）查看数据库的连接字符串:

```plaintext
postgresql://newsletter:<PASSWORD>@<HOST>:<PORT>/newsletter?sslmode=require
```

我们当前的 `DatabaseSettings` 不支持 SSL 模式——这在本地开发中并不适用，但在生产环境中，为客户端/数据库通信提供传输级加密是非常必要的。

在尝试添加新功能之前, 让我们先重构 `DatabaseSettings` 来腾出空间。

当前版本如下所示:

```rs
#[derive(serde::Deserialize)]
pub struct DatabaseSettings {
    pub username: String,
    pub password: SecretBox<String>,
    #[serde(deserialize_with = "deserialize_number_from_string")]
    pub port: u16,
    pub host: String,
    pub database_name: String,
}

impl DatabaseSettings {
    pub fn connection_string(&self) -> SecretBox<String> {
        // [...]
    }

    pub fn connection_string_without_db(&self) -> SecretBox<String> {
        // [...]
    }
}
```

我们将修改它的两个方法, 使其返回 `PgConnectOptions` 而不是连接字符串：这将使管理所有这些参数变得更加容易。

```rs
//! src/configuration.rs
use sqlx::postgres::PgConnectOptions;

// [...]
impl DatabaseSettings {
    pub fn without_db(&self) -> PgConnectOptions {
        PgConnectOptions::new()
            .host(&self.host)
            .username(&self.username)
            .password(&self.password.expose_secret())
            .port(self.port)
    }

    pub fn with_db(&self) -> PgConnectOptions {
        self.without_db().database(&self.database_name)
    }
}
```

当然我们还得修改 `src/main.rs` 和 `tests/health_check.rs`

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let connection_pool = PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy_with(configuration.database.with_db());
   
    // [...]
}
```

```rs
//! tests/health_check.rs
// [...]
pub async fn configure_database(config: &DatabaseSettings) -> PgPool {
    // Create database
    let mut connection = PgConnection::connect_with(&config.without_db())
        .await
        .expect("Failed to connect to Postgres");

    connection
        .execute(format!(r#"CREATE DATABASE "{}";"#, config.database_name).as_str())
        .await
        .expect("Failed to create database.");

    // Migrate database
    let connection_pool = PgPool::connect_with(config.with_db())
        .await
        .expect("Failed to connect to Postgres");

    sqlx::migrate!("./migrations")
        .run(&connection_pool)
        .await
        .expect("Failed to migrate the database");

    connection_pool
}
```

运行 `cargo test` 以确保一切正常

我们现在需要在 `DatabaseSettings` 里增加字段 `require_ssl`:

```rs
//! src/configuration.rs
// [...]

#[derive(serde::Deserialize)]
pub struct DatabaseSettings {
    // [...]
    pub require_ssl: bool,
}

impl DatabaseSettings {
    pub fn without_db(&self) -> PgConnectOptions {
        let ssl_mode = if self.require_ssl {
            PgSslMode::Require
        } else {
            // Try an encrypted connection, fallback to unencrypted if it fails
            PgSslMode::Prefer
        };
        PgConnectOptions::new()
            .host(&self.host)
            .username(&self.username)
            .password(&self.password.expose_secret())
            .port(self.port)
            .ssl_mode(ssl_mode)
    }

    // [...]
}
```

我们希望在本地运行应用程序（以及测试套件）时, `require_ssl` 为 `false`, 但在生产环境中则为 `true`。

让我们相应地修改配置文件:

```yaml
#! configuration/local.yaml
application:
  host: 127.0.0.1
database:
  # New entry!
  require_ssl: false
```

```yaml
#! configuration/production.yaml
application:
  host: 127.0.0.1
database:
  # New entry!
  require_ssl: true
```

注: 调整Sqlx log level 部分已过时, 暂时未找到替代方法

## 应用程序规范中的环境变量

最后一步：我们需要修改 spec.yaml 清单，以注入所需的环境变量。

```yaml
#! spec.yaml
name: zero2prod
region: fra
services:
  - name: zero2prod
    # [...]
    envs:
      - key: APP_APPLICATION__BASE_URL
        scope: RUN_TIME
        value: ${APP_URL}
      - key: APP_DATABASE__USERNAME
        scope: RUN_TIME
        value: ${newsletter.USERNAME}
      - key: APP_DATABASE__PASSWORD
        scope: RUN_TIME
        value: ${newsletter.PASSWORD}
      - key: APP_DATABASE__HOST
        scope: RUN_TIME
        value: ${newsletter.HOSTNAME}
      - key: APP_DATABASE__PORT
        scope: RUN_TIME
        value: ${newsletter.PORT}
      - key: APP_DATABASE__DATABASE_NAME
        scope: RUN_TIME
        value: ${newsletter.DATABASE}
databases:
  # PG = Postgres
  - engine: PG
    # Database name
    name: newsletter
    # [...]
```

范围设置为 `RUN_TIME`, 以区分 Docker 构建过程中所需的环境变量和 Docker 镜像启动时所需的环境变量。
我们通过插入 Digital Ocean 平台公开的内容（例如 `${newsletter.PORT}`）来填充环境变量的值 - 更多详情请参阅其[文档](https://www.digitalocean.com/docs/app-platform/how-to/use-environment-variables/)。

## 最后一次 Push

让我们应用新规范

```shell
# You can retrieve your app id using `doctl apps list`
doctl apps update YOUR-APP-ID --spec=spec.yaml
```

并将我们的更改推送到 GitHub 以触发新的部署。

我们现在需要迁移数据库:

```shell
DATABASE_URL=YOUR-DIGITAL-OCEAN-DB-CONNECTION-STRING sqlx migrate run
```

一切准备就绪！

让我们向 /subscriptions 发送一个 POST 请求:

```shell
curl --request POST \
    --data 'name=le%20guin&email=ursula_le_guin%40gmail.com' \
    https://zero2prod-adqrw.ondigitalocean.app/subscriptions \
    --verbose
```

服务器应该会返回 200 OK。

恭喜，您刚刚部署了您的第一个 Rust 应用程序！

据说，[Ursula Le Guin](https://en.wikipedia.org/wiki/Ursula_K._Le_Guin) 刚刚订阅了您的电子邮件简报！

如果您已经看到这里，我很想获取一张您的 Digital Ocean 仪表板的屏幕截图，以展示正在运行的应用程序!

请将其发送至 [rust@lpalmieri.com](mailto:rust@lpalmieri.com), 或在 X 上分享，并标记 "Zero To Production In Rust"帐户 [zero2prod](https://x.com/zero2prod)。
