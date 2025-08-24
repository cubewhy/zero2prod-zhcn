# 我们项目的Dockerfile

DigitalOcean 的 App Platform 原生支持容器化应用的部署。

我们的第一项任务，就是编写一个 Dockerfile，以便将应用打包并作为 Docker 容器来构建和运行。

## Dockerfiles

Dockerfile是你项目环境的配方。

它们包含了这些东西：你的基础镜像（通常是一个OS包含了编程语言的工具链）和一个接一个的命令（COPY, RUN, 等）。这样就可以构建一个你需要的环境。

让我们看看一个最简单的可用的Rust项目的Dockerfile

```Dockerfile
# We use the latest Rust stable release as base image
FROM rust:1.89.0

# Let's switch our working directory to `app` (equivalent to `cd app`)
# The `app` folder will be created for us by Docker in case it does not
# exist already.
WORKDIR /app

# Install the required system dependencies for our linking configuration
RUN apt update && apt install lld clang -y

# Copy all files from our working environment to our Docker image
COPY . .

# Let's build our binary!
# We'll use the release profile to make it faaaast
RUN cargo build --release

# When `docker run` is executed, launch the binary!
ENTRYPOINT ["./target/release/zero2prod"]
```

将这个Dockerfile文件保存到项目的根目录：

```sh
zero2prod/
    .github/
    migrations/
    scripts/
    src/
    tests/
    .gitignore
    Cargo.lock
    Cargo.toml
    configuration.yaml
    Dockerfile
```

执行这些命令来生成镜像的过程叫做**构建**。  

使用Docker CLI来操作，我们需要输入以下指令。

```sh
docker build -t zero2prod --file Dockerfile .
```

命令最后的点是干什么的？

## 构建上下文

`docker build`的执行依赖两个要素：一个配方（即`Dockerfile`），以及一个**构建上下文（build context）**  

你可以把正在构建的docker镜像想象成一个完全隔离的环境。  

它和本地机器唯一的接触点就是诸如`COPY`或`ADD`这样的命令。  

而**构建上下文**则决定了：在执行`COPY`等命令时，镜像内部能够“看见”你主机上的那些文件。  

举个例子，当我们使用`.`，就是告诉docker: “请把当前目录作为这次构建镜像时的上下文。”

因此，命令

```Dockerfile
COPY . /app
```

会将当前目录下的所有文件（包括源代码！）复制到镜像 `/app` 目录。

使用 `.` 作为构建上下文同时也意味着：
Docker 不会允许你直接通过 `COPY` 去访问父目录，或任意指定主机上的其他路径。

根据需求，你也可以使用其他路径，甚至是一个 URL (!) 作为构建上下文。

## Sqlx 离线模式

如果你很急，那么你可能已经运行了构建命令……但是你要意识到他是不能正常运行的。

```txt
# [...]
Step 4/5 : RUN cargo build --release
# [...]
error: error communicating with the server:
Cannot assign reguested address (os error 99)
    --> src/routes/subscriptions.rs:35:5
    |
35  | /     sqlx: :query!(
36  | |         r#"
37  | |     INSERT INTO subscriptions (id, email, name, subscribed_at)
38  | |     VALUES ($1，$2，$3，$4)
...   |
43  | |         Utc::now()
44  | |     )
    | |____-
    |
    = note: this error originates in a macro
```

### 发生了什么？

`sqlx` 在构建时访问了我们的数据库来确保所有的查询可以被成功的执行。  

当我们在docker运行 `cargo build` 时，显然的，sqlx无法通过.env文件中确定的 `DATABASE_URL` 环境变量建立到数据库的连接。

### 如何解决？

我们可以通过--network来在构建镜像时允许镜像去和本地主机运行的数据库通信。这是我们在CI管道中遵循的策略，因为我们无论如何都需要数据库来运行集成测试。  

不幸的是，由于在不同的操作系统（例如MACOS）上实施Docker网络会大大损害我们的构建可重现性，这对Docker构建有些麻烦。  

更好的选择是sqlx的离线模式。  

让我们在Cargo.toml加入sqlx的离线模式feature:

注: 在 0.8.6 版本的sqlx 中, offline mode 默认开启, 见 [README](https://github.com/launchbadge/sqlx/blob/bab1b022bd56a64f9a08b46b36b97c5cff19d77e/sqlx-cli/README.md#enable-building-in-offline-mode-with-query)

```toml
#! Cargo.toml
# [...]

[dependencies.sqlx]
version = "0.8.6"
default-features = false
features = [
    "tls-rustls",
    "macros",
    "postgres",
    "uuid",
    "chrono",
    "migrate",
    "runtime-tokio",
]
```

下一步依赖于 sqlx 的命令行界面 (CLI)。我们要找的命令是 `sqlx prepare`。我们来看看它的帮助信息:

```shell
sqlx prepare --help
```

```plaintext
Generate query metadata to support offline compile-time verification.

Saves metadata for all invocations of `query!` and related macros to a `.sqlx` directory in the current directory (or workspace root with `--workspace`), overwriting if needed.

During project compilation, the absence of the `DATABASE_URL` environment variable or the presence of `SQLX_OFFLINE` (with a value of `true` or `1`) will constrain the compile-time verification to
only read from the cached query metadata.

Usage: sqlx prepare [OPTIONS] [-- <ARGS>...]

Arguments:
  [ARGS]...
          Arguments to be passed to `cargo rustc ...`

Options:
      --check
          Run in 'check' mode. Exits with 0 if the query metadata is up-to-date. Exits with 1 if the query metadata needs updating

      --all
          Prepare query macros in dependencies that exist outside the current crate or workspace

      --workspace
          Generate a single workspace-level `.sqlx` folder.
          
          This option is intended for workspaces where multiple crates use SQLx. If there is only one, it is better to run `cargo sqlx prepare` without this option inside that crate.

      --no-dotenv
          Do not automatically load `.env` files

  -D, --database-url <DATABASE_URL>
          Location of the DB, by default will be read from the DATABASE_URL env var or `.env` files
          
          [env: DATABASE_URL=postgres://postgres:password@localhost:5432/newsletter]

      --connect-timeout <CONNECT_TIMEOUT>
          The maximum time, in seconds, to try connecting to the database server before returning an error
          
          [default: 10]

  -h, --help
          Print help (see a summary with '-h')
```

换句话说，prepare 执行的操作与调用 cargo build 时通常执行的操作相同, 但它会将这些查询的结果保存到元数据文件 (sqlx-data.json) 中，该文件稍后可以被 sqlx 自身检测到，并用于完全跳过查询并执行离线构建。

让我们调用它吧!

```shell
# It must be invoked as a cargo subcommand
# All options after `--` are passed to cargo itself
# We need to point it at our library since it contains
# all our SQL queries.
cargo sqlx prepare -- --lib
```

```plaintext
query data written to .sqlx in the current directory; please check this into version control
```

正如命令输出所示，我们确实会将文件提交到版本控制。.

让我们在 Dockerfile 中将 SQLX_OFFLINE 环境变量设置为 true, 以强制 sqlx 查看已保存的元数据，而不是尝试查询实时数据库:

```Dockerfile
FROM rust:1.89.0

WORKDIR /app
RUN apt update && apt install lld clang -y
COPY . .
ENV SQLX_OFFLINE=true
RUN cargo build --release
ENTRYPOINT ["./target/release/zero2prod"]
```

让我们试试构建我们的 docker image!

```shell
docker build --tag zero2prod --file Dockerfile .
```

这次应该不会出错了！
不过，我们有一个问题：如何确保 `sqlx-data.json` 不会不同步（例如，当数据库架构发生变化或添加新查询时）?

我们可以在 CI 流中使用 `--check` 标志来确保它保持最新状态——请参考[本书 GitHub 仓库](https://github.com/LukeMathWalker/zero-to-production)中更新的管道定义。

## 运行映像

在构建映像时，我们为其附加了一个标签 zero2prod:

```shell
docker build --tag zero2prod --file Dockerfile .
```

我们可以在其他命令中使用该标签来引用该图像。具体来说，运行以下命令:

```shell
docker run zero2prod
```

`docker run` 将触发我们在 ENTRYPOINT 语句中指定的命令的执行:

```Dockerfile
ENTRYPOINT ["./target/release/zero2prod"]
```

在我们的例子中，它将执行我们的二进制文件，从而启动我们的 API。

接下来，让我们启动我们的镜像!

你应该会立即看到一个错误:

```plaintext
Failed to connect to Postgres.: PoolTimedOut
```

这是来自我们 `main` 方法中的这一行:

```rs
//! src/main.rs
// [...]
#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let connection_pool = PgPool::connect(&configuration.database.connection_string().expose_secret())
        .await
        .expect("Failed to connect to Postgres.");

    // [...]
}
```

我们可以通过使用 [connect_lazy](https://docs.rs/sqlx/latest/sqlx/struct.Pool.html#method.connect_lazy) 来放宽我们的要求——它只会在第一次使用池时尝试建立连接。

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let connection_pool = PgPool::connect_lazy(&configuration.database.connection_string().expose_secret())
    .expect("Failed to connect to Postgres.");

    // [...]
}
```

现在我们可以重新构建 Docker 镜像并再次运行它: 你应该会立即看到几行日志!

让我们打开另一个终端，尝试向 health_check 端点发送请求:

```shell
curl http://127.0.0.1:8000/health_check
```

```plaintext
curl: (7) Failed to connect to 127.0.0.1 port 8000: Connection refused
```

不是很好...

## Networking

默认情况下，Docker 镜像不会将其端口暴露给底层主机。我们需要使用 -p 标志显式地进行此操作。

让我们终止正在运行的镜像，然后使用以下命令重新启动它:

```shell
docker run -p 8000:8000 zero2prod
```

尝试访问健康检查端点将触发相同的错误消息。

我们需要深入研究 main.rs 文件来了解原因:

我们使用 `127.0.0.1` 作为主机地址 - 指示我们的应用程序仅接受来自同一台机器的连接。

然而，我们从主机向 /health_check 发出了一个 GET 请求，而我们的 Docker 镜像不会将其视为本地主机，因此触发了我们刚刚看到的“连接被拒绝”错误。

我们需要使用 `0.0.0.0` 作为主机地址，以指示我们的应用程序接受来自任何网络接口的连接，而不仅仅是本地接口。

不过，我们需要小心：使用 `0.0.0.0` 会显著增加我们应用程序的“受众”，并带来一些[安全隐患](https://github.com/sinatra/sinatra/issues/1369)。

最好的方法是使地址的主机部分可配置 - 我们将继续使用 `127.0.0.1` 进行本地开发，并在 Docker 镜像中将其设置为 `0.0.0.0`。

```rs
//! src/main.rs

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let address = format!("0.0.0.0:{}", configuration.application_port);
    // [...]
}
```

## 分层配置

我们的设置结构目前如下所示:

```rs
//! src/configuration.rs
// [...]

#[derive(serde::Deserialize)]
pub struct Settings {
    pub database: DatabaseSettings,
    pub application_port: u16,
}

#[derive(serde::Deserialize)]
pub struct DatabaseSettings {
    pub username: String,
    pub password: SecretBox<String>,
    pub port: u16,
    pub host: String,
    pub database_name: String,
}

// [...]
```

让我们引入另一个结构体 `ApplicationSettings`, 将所有与应用程序地址相关的配置值组合在一起:

```rs
//! src/configuration.rs
#[derive(serde::Deserialize)]
pub struct Settings {
    pub database: DatabaseSettings,
    pub application: ApplicationSettings,
}

#[derive(serde::Deserialize)]
pub struct ApplicationSettings {
    pub port: u16,
    pub host: String,
}

// [...]
```

我们需要更新我们的 `configuration.yml` 文件以匹配新的结构:

```yaml
#! configuration.yaml
application:
  port: 8000
  host: 127.0.0.1

database: ...
```

以及我们的 `main.rs`，我们将利用新的可配置 host 参数

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() {
    // [...]
    let address = format!("{}:{}", configuration.application.host, configuration.application.port);

    // [...]
}
```

现在可以从配置中读取主机名了，但是如何针对不同的环境使用不同的值呢?

我们需要将配置分层化。

我们来看看 `get_configuration`, 这个函数负责加载我们的 `Settings` 结构体:

```rs
//! src/configuration.rs
// [...]
pub fn get_configuration() -> Result<Settings, config::ConfigError> {
    let settings = config::Config::builder()
        .add_source(config::File::with_name("configuration"))
        .build()
        .unwrap();

    settings.try_deserialize()
}
```

我们正在从名为 configuration 的文件中读取数据，以填充“设置”的字段。configuration.yaml 中指定的值已无进一步调整的空间。

让我们采用更精细的方法。我们将拥有:

- 一个基础配置文件，用于在本地和生产环境中共享的值（例如数据库名称）；
- 一组特定于环境的配置文件，用于指定需要根据每个环境进行自定义的字段的值（例如主机）；
- 一个环境变量 APP_ENVIRONMENT，用于确定运行环境（例如生产环境或本地环境）。

所有配置文件都将位于同一个顶级目录 configuration 中。
好消息是，我们正在使用的 crate config 开箱即用地支持上述所有功能?

让我们将它们组合起来:

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

    settings.build().unwrap().try_deserialize()
}

pub enum Environment {
    Local,
    Production,
}

impl Environment {
    pub fn as_str(&self) -> &'static str {
        match self {
            Environment::Local => "local",
            Environment::Production => "production",
        }
    }
}

impl TryFrom<String> for Environment {
    type Error = String;

    fn try_from(value: String) -> Result<Self, Self::Error> {
        match value.to_lowercase().as_str() {
            "local" => Ok(Self::Local),
            "production" => Ok(Self::Production),
            other => Err(format!(
                "{other} is not a supported environment. Use either `local` or `production`."
            )),
        }
    }
}
```

让我们重构配置文件以适应新的结构。

我们必须删除 `configuration.yaml` 文件，并创建一个新的配置目录，其中包含 `base.yaml`、`local.yaml` 和 `production.yaml` 文件。

```yaml
#! configuration/base.yaml
application:
  port: 8000

database:
  host: "127.0.0.1"
  port: 5432
  username: "postgres"
  password: "password"
  database_name: "newsletter"

```

```yaml
#! configuration/local.yaml
application:
  host: 127.0.0.1
```

```yaml
#! configuration/production.yaml
application:
  host: 0.0.0.0
```

现在，我们可以通过使用 ENV 指令设置 PP_ENVIRONMENT 环境变量来指示 Docker 镜像中的二进制文件使用生产配置:

```Dockerfile
FROM rust:1.89.0

WORKDIR /app

RUN apt update && apt install lld clang -y
COPY . .
ENV SQLX_OFFLINE=true
RUN cargo build --release
ENV APP_ENVIRONMENT=production
ENTRYPOINT ["./target/release/zero2prod"]
```

让我们重新构建映像并再次启动它:

```shell
docker build --tag zero2prod --file Dockerfile .
docker run -p 8000:8000 zero2prod
```

第一条日志行应该是这样的

```json
{
  "name":"zero2prod",
  "msg":"Starting \"actix-web-service-0.0.0.0:8000\" service on 0.0.0.0:8000",
}
```

如果你的运行结果和这个差不多, 这是个好消息 - 配置文件正在按照我们期待的方式工作!

我们再试试请求 health check 端点

```shell
curl -v http://127.0.0.1:8080/health_check
```

```plaintext
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
< date: Sun, 24 Aug 2025 02:40:04 GMT
< 
* Connection #0 to host 127.0.0.1 left intact
```

太棒了, 这能用!

## 数据库连接

那么 POST /subscriptions 怎么样?

```shell
curl --request POST \
  --data 'name=le%20guin&email=ursula_le_guin%40gmail.com' \
  127.0.0.1:8000/subscriptions --verbose
```

经过长时间的等待, 服务端返回了 500!

我们看看应用程序日志 (很有用的, 难道不是吗?)

```json
{
  "msg": "[SAVING NEW SUBSCRIBER DETAILS IN THE DATABASE - EVENT] \
Failed to execute query: PoolTimedOut",
}
```

这应该不会让你感到意外——我们将 connect 替换成了 connect_lazy，以避免立即与数据库打交道。

我们等了半分钟才看到返回 500 错误——这是因为 sqlx 中从连接池获取连接的默认超时时间为 30 秒。

让我们使用更短的超时时间，让失败速度更快一些:

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let connection_pool = PgPoolOptions::new()
        .acquire_timeout(std::time::Duration::from_secs(2))
        .connect_lazy(&configuration.database.connection_string().expose_secret())
        .expect("Failed to connect to Postgres.");

    // [...]
}
```

使用 Docker 容器获取有效的本地设置有多种方法:

- 使用 --network=host 运行应用程序容器，就像我们目前对 Postgres 容器所做的那样；
- 使用 docker-compose；
- 创建用户自定义网络。

有效的本地设置并不能让我们在部署到 Digital Ocean 时获得有效的数据库连接。因此，我们暂时先这样。

## 优化 Docker 映像

就我们的 Docker 镜像而言，它似乎按预期运行了——是时候部署它了!

嗯，还没有。

我们可以对 Dockerfile 进行两项优化，以简化后续工作:

- 减小镜像大小，提高使用速度
- 使用 Docker 层缓存，提高构建速度。

### Docker 映像大小

我们不会在托管我们应用程序的机器上运行 `docker build`。它们将使用 `docker pull` 下载我们的 Docker 镜像，而无需经历从头构建的过程。

这非常方便：构建镜像可能需要很长时间（Rust 确实如此！），而我们只需支付一次费用。

要实际使用镜像，我们只需支付下载费用，该费用与镜像的大小直接相关。

我们的镜像有多大?

我们可以使用

```shell
docker images zero2prod
```

```plaintext
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
zero2prod    latest    01f54e7e0153   8 minutes ago   16.5GB
```

这是大还是小?

嗯，我们的最终镜像不能小于我们用作基础镜像的 rust。它有多大?

```shell
docker images rust:1.89.0
```

注: 译者这里懒得 docker pull 了, 反正这个映像很大就是了

好的，我们的最终镜像几乎是基础镜像的两倍大。
我们可以做得更好!

我们的第一步是通过排除构建镜像不需要的文件来减小 Docker 构建上下文的大小。

Docker 会在我们的项目中查找一个特定的文件 - `.dockerignore` 来确定哪些文件应该被忽略。

让我们在根目录中创建一个包含以下内容的文件:

```ignore
#! .dockerignore
.env
target/
tests/
Dockerfile
scripts/
migrations/
```

所有符合 `.dockerignore` 中指定模式的文件都不会被 Docker 作为构建上下文的一部分发送到镜像，这意味着它们不在 COPY 指令的范围内。

如果我们能够忽略繁重的目录（例如 Rust 项目的目标文件夹），这将大大加快我们的构建速度（并减小最终镜像的大小）。

而下一个优化则利用了 Rust 的独特优势之一。

Rust 的二进制文件是静态链接的——我们不需要保留源代码或中间编译工件来运行二进制文件，它是完全独立的。

这与 Docker 的一项实用功能——多阶段构建完美兼容。我们可以将构建分为两个阶段:

- `builder` 阶段，用于生成编译后的二进制文件
- `runtime` 阶段，用于运行二进制文件

修改后的 Dockerfile 如下所示:

```Dockerfile
# Builder stage
FROM rust:1.89.0 AS builder

WORKDIR /app

RUN apt update && apt install lld clang -y
COPY . .
ENV SQLX_OFFLINE=true
RUN cargo build --release

# Runtime stage
FROM rust:1.89.0

WORKDIR /app

# Copy the compiled binary from the builder environment
# to your runtime env
COPY --from=builder /app/target/release/zero2prod zero2prod

# We need the configuration file at runtime!
COPY configuration configuration

ENV APP_ENVIRONMENT=production
ENTRYPOINT ["./target/release/zero2prod"]
```

`runtime` 是我们的最终镜像。

`builder` 阶段不会影响其大小——它是一个中间步骤，会在构建结束时被丢弃。最终工件中唯一存在的构建器阶段部分就是我们明确复制的内容——编译后的二进制文件!

使用上述 Dockerfile 生成的镜像大小是多少?

```shell
docker images zero2prod
```

只有1.3GB!

只比基础镜像大 20 MB，好太多了!

我们可以更进一步：在运行时阶段，我们不再使用 `rust:1.89.0`，而是改用 `rust:1.589.0-slim`，这是一个使用相同底层操作系统的较小镜像。

这比我们一开始的体积小了 4 倍——一点也不差!

我们可以通过减少整个 Rust 工具链和机器（例如 rustc、cargo 等）的重量来进一步缩小体积——运行我们的二进制文件时，这些都不再需要。

我们可以使用裸操作系统 (`debian:bullseye-slim`) 作为运行时阶段的基础镜像:

```Dockerfile
# [...]
# Runtime stage
FROM debian:bullseye-slim

WORKDIR /app

RUN apt-get update -y \
  && apt-get install -y
  --no-install-recommends openssl
  ca-certificates \
  # Clean up
  && apt-get autoremove -y \
  && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/*

# Copy the compiled binary from the builder environment
# to your runtime env
COPY --from=builder /app/target/release/zero2prod zero2prod

# We need the configuration file at runtime!
COPY configuration configuration

ENV APP_ENVIRONMENT=production
ENTRYPOINT ["./target/release/zero2prod"]
```

不到 100 MB——比我们最初的尝试小了约 25 倍。

我们可以通过使用 rust:1.89.0-alpine 来进一步减小文件大小，但我们必须交叉编译到 `linux-musl` 目标——目前超出了我们的范围。如果您有兴趣生成微型 Docker 镜像，请查看 rust-musl-builder。

进一步减小二进制文件大小的另一种方法是从中剥离符号——您可以在不到 100 MB——比我们最初的尝试小了约 25 倍 42。
我们可以通过使用 rust:1.59.0-alpine 来进一步减小文件大小，但我们必须交叉编译到
linux-musl 目标——目前超出了我们的范围。如果您有兴趣生成微型 Docker 镜像，请查看 [rust-musl-builder](https://github.com/emk/rust-musl-builder)。
进一步减小二进制文件大小的另一种方法是从中剥离符号——您可以在此处找到
更多信息。找到更多信息。

## 为Rust Docker 构建缓存

Rust 在运行时表现出色，始终提供卓越的性能，但这也带来了一些代价：编译时间。在 Rust 年度调查中，Rust 项目面临的最大挑战或问题中，编译时间一直是热门话题。

优化构建（--release）尤其令人头疼——对于包含多个依赖项的中型项目，编译时间可能长达 15/20 分钟。这在我们这样的 Web 开发项目中相当常见，因为我们会从异步生态系统（tokio、actix-web、sqlx 等）中引入许多基础 crate。

不幸的是，我们在 Dockerfile 中使用 --release 选项来在生产环境中获得最佳性能。如何缓解这个问题？
我们可以利用 Docker 的另一个特性：层缓存。

Dockerfile 中的每个 RUN、COPY 和 ADD 指令都会创建一个层：执行指定命令后，先前状态（上一层）与当前状态之间的差异。

层会被缓存：如果操作的起始点（例如基础镜像）未发生变化，并且命令本身（例如 COPY 复制的文件的校验和）也未发生变化，Docker 不会执行任何计算，而是直接从本地缓存中获取结果副本。

Docker 层缓存速度很快，可以利用它来大幅提升 Docker 构建速度。

诀窍在于优化 Dockerfile 中的操作顺序：任何引用经常更改的文件（例如源代码）的操作都应尽可能晚地出现，从而最大限度地提高前一步保持不变的可能性，并允许 Docker 直接从缓存中获取结果。

开销最大的步骤通常是编译。

大多数编程语言都遵循相同的策略：首先复制某种类型的锁文件，构建依赖项，然后复制其余源代码，最后构建项目。
只要你的依赖关系树在两次构建之间没有发生变化，这就能保证大部分工作都被缓存。

例如，在一个 Python 项目中，你可能会有类似这样的代码:

```Dockerfile
FROM python:3
COPY requirements.txt
RUN pip install -r requirements.txt
COPY src/ /app
WORKDIR /app
ENTRYPOINT ["python", "app"]
```

遗憾的是, `cargo` 没有提供从其 `Cargo.lock` 文件开始构建项目依赖项的机制（例如 `cargo build --only-deps`）。

然而，我们可以依靠社区项目 [`cargo-chef`](https://github.com/LukeMathWalker/cargo-chef) 来扩展 `cargo` 的默认功能。

让我们按照 `cargo-chef` 的 README 中的建议修改 Dockerfile:

注: 不要在构建容器外运行 cargo chef, 防止损坏代码库。 (来自README)

```Dockerfile
FROM lukemathwalker/cargo-chef:latest-rust-1.89.0 AS chef

WORKDIR /app
RUN apt update && apt install lld clang -y

FROM chef AS planner
COPY . .

# Compute a lock-like file for our project
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# Build our project dependencies, not our application!
RUN cargo chef cook --release --recipe-path recipe.json

# Up to this point, if our dependency tree stays the same,
# all layers should be cached.
COPY . .

ENV SQLX_OFFLINE=true
RUN cargo build --release

# Runtime stage
FROM debian:bullseye-slim

WORKDIR /app

RUN apt-get update -y \
  && apt-get install -y --no-install-recommends openssl ca-certificates \
  # Clean up
  && apt-get autoremove -y \
  && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/*

# Copy the compiled binary from the builder environment
# to your runtime env
COPY --from=builder /app/target/release/zero2prod zero2prod

# We need the configuration file at runtime!
COPY configuration configuration

ENV APP_ENVIRONMENT=production
ENTRYPOINT ["./target/release/zero2prod"]
```

我们使用三个阶段：第一阶段计算配方文件，第二阶段缓存依赖项，然后构建二进制文件，第三阶段是运行时环境。只要依赖项保持不变，`recipe.json` 文件就会保持不变，因此 `cargo chef cook --release --recipe-path recipe.json` 的结果会被缓存，从而大大加快构建速度。

我们利用了 Docker 层缓存与多阶段构建的交互方式：`planner` 阶段中的 `COPY . .` 语句会使 `planner` 容器的缓存失效，但只要 `cargo chef prepare` 返回的 `recipe.json` 的校验和保持不变, 它就不会使构建器容器的缓存失效。

您可以将每个阶段视为具有各自缓存的 Docker 镜像 - 它们仅在使用 `COPY --from` 语句时才会相互交互。
这将在下一节中为我们节省大量时间。
