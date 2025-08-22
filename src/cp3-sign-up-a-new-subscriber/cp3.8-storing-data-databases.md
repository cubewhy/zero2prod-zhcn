# 存储数据: 数据库

我们的 POST /订阅端点通过了测试，但其实用性相当有限：我们没有在任何地方存储有效的电子邮件和姓名。

我们从 HTML 表单收集的信息没有永久记录。

该如何解决这个问题?

在定义云原生时，我们列出了我们期望在系统中看到的一些新兴行为: 特别是，我们希望在易出错的环境中运行时实现高可用性。

因此，我们的应用程序被迫分布式——应该在多台机器上运行多个实例，以应对硬件故障。

这会对数据持久性产生影响：我们不能依赖主机的文件系统作为传入数据的存储层。

我们保存在磁盘上的任何内容都只能供应用程序的众多副本之一使用。

此外，如果宿主机崩溃，这些数据很可能会消失。

这解释了为什么云原生应用程序通常是无状态的: 它们的持久性需求被委托给专门的外部系统——数据库。

## 选择一个数据库

我们的newsletter项目应该使用什么数据库?

我先给出我的个人经验法则，听起来可能有点争议:

> 如果您不确定持久性需求，请使用关系数据库。
>
> 如果您不需要大规模部署，请使用 PostgreSQL。

在过去的二十年里，数据库产品种类繁多。

从数据模型的角度来看, **NoSQL** 运动为我们带来了文档存储 (例如 [MongoDB](https://www.mongodb.com/)) c、键值存储 (例如 [AWS DynamoDB](https://aws.amazon.com/dynamodb/))、图形数据库(例如 [Neo4J](https://neo4j.com/)) 等。

我们有一些数据库使用 RAM 作为主存储（例如 [Redis](https://redis.io/)）。

我们有一些数据库通过列式存储针对分析查询进行了优化（例如 [AWS RedShift](https://aws.amazon.com/redshift/)）。

系统设计中存在着无限可能，您绝对应该充分利用这些可能性。

然而，如果您对应用程序所使用的数据访问模式还不甚了解，那么使用专门的数据存储解决方案很容易让您陷入困境。

关系数据库可以说是万能的：在构建应用程序的第一个版本时，它们通常是一个不错的选择，可以在您探索领域约束的过程中为您提供支持。

即使是关系数据库，也有很多选择。

除了 [PostgreSQL](https://www.postgresql.org/) 和 [MySQL](https://www.mysql.com/) 等经典数据库外，您还会发现一些令人兴奋的新数据库，例如 [AWS Aurora](https://aws.amazon.com/rds/aurora/)、[Google Spanner](https://cloud.google.com/spanner) 和 [CockroachDB](https://www.cockroachlabs.com/)。

它们有什么共同点?

它们都是为了扩展而构建的。远远超出了传统 SQL 数据库的处理能力。

如果您担心扩展性问题，请务必考虑一下。如果不是，则无需考虑额外的复杂性。

这就是我们最终选择 [PostgreSQL](https://www.postgresql.org/) 的原因：它是一项久经考验的技术，如果您需要托管服务，它受到所有云提供商的广泛支持，开源，拥有详尽的文档，易于在本地运行以及通过 Docker 在 CI 中运行，并且在 Rust 生态系统中得到良好支持。

## 选择一个数据库 Crate

截至 2020 年 8 月，在 Rust 项目中与 PostgreSQL 交互时，有三个最常用的选项:

- [tokio-postgres](https://docs.rs/tokio-postgres/)
- [sqlx](https://docs.rs/sqlx/latest/sqlx/)
- [diesel](https://docs.rs/diesel/latest/diesel/)

译者注: 在2025还有一个比较欢迎的选项叫做 [sea-ql](https://www.sea-ql.org/SeaORM/), 基于sqlx 的一个数据库crate

这三个项目都非常受欢迎，并且被广泛采用，在生产环境中也占有相当的份额。您该如何选择呢？
这取决于您对以下三个主题的看法：

- 编译时安全性
- SQL 优先 vs. DSL 用于查询构建
- 异步 vs. 同步接口

### 编译期安全

与关系数据库交互时，很容易犯错——例如，我们可能会：

- 查询中提到的列名或表名出现拼写错误；
- 尝试执行数据库引擎拒绝的操作（例如，将字符串和数字相加，或在错误的列上连接两个表）；
- 期望返回的数据中包含实际上不存在的某个字段。

关键问题是：我们何时意识到自己犯了错误?

在大多数编程语言中，错误发生在运行时: 当我们尝试执行查询时，

数据库会拒绝它，然后我们会收到错误或异常。使用 tokio-postgres 时就会发生这种情况。

diesel 和 sqlx 试图通过在编译时检测大多数此类错误来加快反馈周期。

diesel 利用其 [CLI](https://github.com/diesel-rs/diesel/tree/master/diesel_cli) [将数据库schema生成为 Rust 代码的表示](https://docs.rs/diesel/latest/diesel/macro.table.html)，然后用于检查所有查询的假设。

相反, **sqlx** 使用过程宏在编译时连接到数据库，并检查提供的查询是否确实合理

### 查询接口

tokio-postgres 和 sqlx 都要求您直接使用 SQL 编写查询。

而 diesel 则提供了自己的查询构建器：查询以 Rust 类型表示，您可以通过调用其方法来添加过滤器、执行连接和类似的操作。这通常被称为领域特定语言 (DSL)。

哪一个更好?

一如既往，这取决于具体情况。

SQL 具有极高的可移植性——您可以在任何需要与关系数据库交互的项目中使用它，
无论应用程序使用哪种编程语言或框架编写。

而 diesel 的 DSL 仅在使用 diesel 时才有意义：您需要预先支付学习成本才能熟练掌握它，而且只有在您当前和未来的项目中坚持使用 diesel 时，这才值得。还值得指出的是，使用 diesel 的 DSL 表达复杂查询可能很困难，
您最终可能还是需要编写原始 SQL。

另一方面，Diesel 的 DSL 使得编写可重用组件变得更加容易：你可以将复杂的查询拆分成更小的单元，并在多个地方使用它们，就像使用普通的 Rust 函数一样。

### 异步支持

我记得在某处读过一篇关于异步IO的精彩解释，
大致如下:

> 线程用于并行工作，异步用于并行等待。

您的数据库并不与您的应用程序位于同一物理主机上：要运行查询，您必须执行网络调用。

异步数据库驱动程序不会减少处理单个查询所需的时间，但它可以让您的应用程序在等待数据库返回结果的同时，充分利用所有 CPU 核心来执行其他有意义的工作（例如，处理另一个 HTTP 请求）。

这是否足以让您接受异步代码带来的额外复杂性?

这取决于您应用程序的性能要求。

一般来说，对于大多数用例来说，在单独的线程池上运行查询应该已经足够了。同时，如果您的 Web 框架已经是异步的，那么使用异步数据库驱动程序实际上会减少您的麻烦28。

sqlx 和 tokio-postgres 都提供了异步接口，而 diesel 是同步的，并且
不打算在不久的将来推出异步支持。

还值得一提的是，tokio-postgres 是目前唯一支持[查询流水线](https://docs.rs/tokio-postgres/latest/tokio_postgres/index.html#pipelining)的 crate。该功能在 sqlx 中仍处于[设计阶段](https://github.com/launchbadge/sqlx/issues/408)，而我在 diesel 的文档或问题跟踪器中找不到任何提及。

### 总结

让我们总结一下比较矩阵中涵盖的所有内容:

| Crate          | 编译期安全 | 查询接口 | 异步 |
| -------------- | ---------- | -------- | ---- |
| tokio-postgres | No         | SQL      | Yes  |
| sqlx           | Yes        | SQL      | Yes  |
| diesel         | Yes        | DSL      | No   |

### 我们的选择: sqlx

对于“从零到生产”阶段，我们将使用 sqlx：它的异步支持简化了与 actix-web 的集成，而无需我们在编译时保证上做出妥协。

由于它使用原始 SQL 进行查询，因此它还限制了我们必须覆盖和精通的 API 范围。

## 有副作用的集成测试

我们想要实现什么目标?

让我们再回顾一下“满意案例”测试:

```rs
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
```

我们现有的断言是不够的。

我们无法仅通过查看 API 响应来判断预期的业务结果是否已实现——我们想知道是否发生了副作用，例如数据存储。

我们想检查新订阅者的详细信息是否已实际保存。

该怎么做呢?

我们有两个选择：

1. 利用公共 API 的另一个端点来检查应用程序状态；
2. 在测试用例中直接查询数据库。

如果可能，您应该选择选项 1：您的测试不会考虑 API 的实现细节（例如底层数据库技术及其模式），因此不太可能受到未来重构的干扰。

遗憾的是，我们的 API 上没有任何公共端点可以让我们验证订阅者是否存在。

我们可以添加一个 GET /subscriptions 端点来获取现有订阅者列表，但这样一来，我们就必须担心它的安全性：我们不希望在没有任何形式的身份验证的情况下，将订阅者的姓名和电子邮件暴露在公共互联网上。

我们最终可能会编写一个 GET /subscriptions 端点（也就是说，我们不想登录生产数据库来查看订阅者列表），但我们不应该仅仅为了测试正在开发的功能而开始编写新功能。

让我们咬紧牙关，在测试中编写一个小查询。当出现更好的测试策略时，我们会将其删除。

## 设置数据库

为了在测试套件中运行查询，我们需要:

- 一个正在运行的 Postgres 实例
- 一个用于存储订阅者数据的表

### Docker

我们将使用 Docker 来运行 Postgres。在启动测试套件之前，我们将使用 Postgres 的官方 Docker 镜像启动一个新的 Docker 容器。

您可以按照 Docker 网站上的说明将其安装到您的机器上。

让我们为它创建一个小型 Bash 脚本 scripts/init_db.sh，并添加一些自定义 Postgres 默认设置的功能:

```shell
#!/usr/bin/env bash
set -x
set -eo pipefail

# Check if a custom user has been set, otherwise default to 'postgres'
DB_USER=${POSTGRES_USER:=postgres}
# Check if a custom password has been set, otherwise default to 'password'
DB_PASSWORD="${POSTGRES_PASSWORD:=password}"
# Check if a custom database name has been set, otherwise default to 'newsletter'
DB_NAME="${POSTGRES_DB:=newsletter}"
# Check if a custom port has been set, otherwise default to '5432'
DB_PORT="${POSTGRES_PORT:=5432}"
# Launch postgres using Docker
docker run \
  -e POSTGRES_USER=${DB_USER} \
  -e POSTGRES_PASSWORD=${DB_PASSWORD} \
  -e POSTGRES_DB=${DB_NAME} \
  -p "${DB_PORT}":5432 \
  -d postgres \
  postgres -N 1000
# ^ Increased maximum number of connections for testing purposes
```

执行如下命令让这个文件可以被执行

```shell
chmod +x scripts/init_db.sh
```

然后我们可以启动 PostgreSQL

```shell
./scripts/init_db.sh
```

如果你运行 `docker ps` 你应该会看到类似这样的内容

```plaintext
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                         NAMES
06b8e8a7252d   postgres   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   zen_herschel
```

注意 - 如果您没有在用 Linux，端口映射位可能会略有不同!

### 数据库迁移

为了存储订阅者的详细信息，我们需要创建第一张表。

要向数据库添加新表，我们需要更改其[架构](https://www.postgresql.org/docs/9.5/ddl-schemas.html)——这通常称为数据库迁移。

#### sqlx-cli

sqlx 提供了一个命令行界面 sqlx-cli 来管理数据库迁移。

我们可以使用以下命令安装 CLI:

```shell
cargo install --version=0.8.6 sqlx-cli --no-default-features --features postgres
```

运行 `sqlx --help` 检查一切是否按预期工作。

### 数据库创建

我们通常要运行的第一个命令是 `sqlx database create`。根据帮助文档:

```plaintext
Creates the database specified in your DATABASE_URL

Usage: sqlx database create [OPTIONS]

Options:
      --no-dotenv                          Do not automatically load `.env` files
  -D, --database-url <DATABASE_URL>        Location of the DB, by default will be read from the DATABASE_URL env var or `.env`
                                           files [env: DATABASE_URL=]
      --connect-timeout <CONNECT_TIMEOUT>  The maximum time, in seconds, to try connecting to the database server before
                                           returning an error [default: 10]
  -h, --help                               Print help
```

在我们的例子中，这并非绝对必要：我们的 Postgres Docker 实例已经自带了一个名为 newsletter 的默认数据库，这要归功于我们在启动它时使用环境变量指定的设置。尽管如此，您仍需要在 CI 管道和生产环境中执行创建步骤，所以无论如何都值得介绍一下。

正如帮助文档所示，`sqlx database create` 依赖于 `DATABASE_URL` 环境变量来
确定要执行的操作。

`DATABASE_URL` 应为有效的 Postgres 连接字符串 - 格式如下:

```plaintext
postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}
```

因此，我们可以在 scripts/init_db.sh 脚本中添加几行

```shell
# [...]
export DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@localhost:${DB_PORT}/${DB_NAME}
sqlx database create
```

您可能偶尔会遇到一个恼人的问题：当我们尝试运行 `sqlx database create` 命令时，Postgres 容器尚未准备好接受连接。

我经常遇到这种情况，因此想找个解决办法：我们需要等待 Postgres 恢复正常，
然后才能开始对其运行命令。让我们将脚本更新为:

```shell
#!/usr/bin/env bash
set -x
set -eo pipefail

# Check if a custom user has been set, otherwise default to 'postgres'
DB_USER=${POSTGRES_USER:=postgres}
# Check if a custom password has been set, otherwise default to 'password'
DB_PASSWORD="${POSTGRES_PASSWORD:=password}"
# Check if a custom database name has been set, otherwise default to 'newsletter'
DB_NAME="${POSTGRES_DB:=newsletter}"
# Check if a custom port has been set, otherwise default to '5432'
DB_PORT="${POSTGRES_PORT:=5432}"
# Launch postgres using Docker
docker run \
  -e POSTGRES_USER=${DB_USER} \
  -e POSTGRES_PASSWORD=${DB_PASSWORD} \
  -e POSTGRES_DB=${DB_NAME} \
  -p "${DB_PORT}":5432 \
  -d postgres \
  postgres -N 1000
# ^ Increased maximum number of connections for testing purposes

# Keep pinging Postgres until it's ready to accept commands
export PGPASSWORD="${DB_PASSWORD}"
until psql -h "localhost" -U "${DB_USER}" -p "${DB_PORT}" -d "postgres" -c '\q'; do
  >&2 echo "Postgres is still unavailable - sleeping"
  sleep 1
done

>&2 echo "Postgres is up and running on port ${DB_PORT}!"

export DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@localhost:${DB_PORT}/${DB_NAME}
sqlx database create
```

问题解决了!

健康检查使用 Postgres 的命令行客户端 psql。请查看以下说明，了解如何在您的操作系统上安装它。

脚本本身没有附带清单文件来声明其依赖项：很遗憾，在没有安装所有先决条件的情况下启动脚本的情况非常常见。这通常会导致脚本在执行过程中崩溃，有时会导致系统处于半崩溃状态。

我们可以在初始化脚本中做得更好：让我们在一开始就检查 **psql** 和 **sqlx-cli** 是否都已安装。

```shell
set -x
set -eo pipefail

if ! [ -x "$(command -v psql)" ]; then
  echo >&2 "Error: psql is not installed."
  exit 1
fi

if ! [ -x "$(command -v sqlx)" ]; then
  echo >&2 "Error: sqlx is not installed."
  echo >&2 "Use:"
  echo >&2 "
 cargo install --version=0.5.7 sqlx-cli --no-default-features --features postgres"
echo >&2 "to install it."
exit 1
fi

# [...]
```

### 添加迁移

现在让我们创建第一个迁移

```shell
# Assuming you used the default parameters to launch Postgres in Docker!
export DATABASE_URL=postgres://postgres:password@127.0.0.1:5432/newsletter
sqlx migrate add create_subscriptions_table
```

您的项目中现在应该会出现一个新的顶级目录 - migrations。sqlx 的 CLI 将把我们项目的所有迁移文件存储在这里。

在 migrations 下，您应该已经有一个名为 {timestamp}_create_subscriptions_table.sql 的文件。

我们需要在这里编写第一个迁移文件的 SQL 代码。

让我们快速勾勒出我们需要的查询：

```sql
-- migrations/{timestamp}_create_subscriptions_table.sql
-- Create Subscriptions Table
CREATE TABLE subscriptions(
  id uuid NOT NULL,
  PRIMARY KEY (id),
  email TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  subscribed_at timestamptz NOT NULL
);
```

关于[主键](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-PRIMARY-KEYS)的[争论](https://www.mssqltips.com/sqlservertip/5431/surrogate-key-vs-natural-key-differences-and-when-to-use-in-sql-server/)一直存在：有些人喜欢使用具有业务含义的列（例如，电子邮件、自然键），而另一些人则觉得使用没有任何业务含义的合成键更安全（例如，ID、随机生成的 UUID、代理键）。

除非有非常充分的理由，否则我通常默认使用合成标识符——如果您对此有不同意见，请随意。

另外需要注意以下几点:

- 我们使用 subscribed_at 来跟踪订阅的创建时间（timestamptz 是一种支持时区的日期和时间类型）
- 我们使用 UNIQUE 约束在数据库级别强制确保电子邮件的唯一性
- 我们强制所有字段的每一列都应使用 NOT NULL 约束进行填充
- 我们使用 TEXT 格式来表示电子邮件和姓名，因为我们对它们的最大长度没有任何限制

数据库约束是防止应用程序错误的最后一道防线，但它是有代价的——数据库必须确保所有检查都通过后才能将新数据写入表。因此，约束会影响我们的写入吞吐量，即单位时间内我们可以在表中插入/更新的行数。

特别是 **UNIQUE** 约束，它会在我们的电子邮件列上引入一个额外的 BTree 索引：该索引必须在每次执行 **INSERT/UPDATE/DELETE** 查询时更新，而且会占用磁盘空间。

就我们的具体情况而言，我不会太担心: 我们的邮件列表必须*非常受欢迎*，才会遇到写入吞吐量的问题。如果真的遇到这个问题，那绝对是个好事。

### 运行迁移

我们可以使用以下方法对数据库进行迁移:

```shell
sqlx migrate run
```

它的行为与 `sqlx database create` 相同——它会查看 `DATABASE_URL` 环境变量，以了解需要迁移的数据库。
让我们将它添加到 `scripts/init_db.sh` 脚本中:

```shell
#!/usr/bin/env bash
set -x
set -eo pipefail

if ! [ -x "$(command -v psql)" ]; then
  echo >&2 "Error: psql is not installed."
  exit 1
fi

if ! [ -x "$(command -v sqlx)" ]; then
  echo >&2 "Error: sqlx is not installed."
  echo >&2 "Use:"
  echo >&2 "
 cargo install --version=0.5.7 sqlx-cli --no-default-features --features postgres"
echo >&2 "to install it."
exit 1
fi

# Check if a custom user has been set, otherwise default to 'postgres'
DB_USER=${POSTGRES_USER:=postgres}
# Check if a custom password has been set, otherwise default to 'password'
DB_PASSWORD="${POSTGRES_PASSWORD:=password}"
# Check if a custom database name has been set, otherwise default to 'newsletter'
DB_NAME="${POSTGRES_DB:=newsletter}"
# Check if a custom port has been set, otherwise default to '5432'
DB_PORT="${POSTGRES_PORT:=5432}"

if [[ -z "${SKIP_DOCKER}" ]]
then
  # Launuh postgrecs sing Docker
  docker run \
    -e POSTGRES_USER=${DB_USER} \
    -e POSTGRES_PASSWORD=${DB_PASSWORD} \
    -e POSTGRES_DB=${DB_NAME} \
    -p "${DB_PORT}":5432 \
    -d postgres \
    postgres -N 1000 
  # ^ Increased maximum number of connections for testing purposes
fi

# Keep pinging Postgres until it's ready to accept commands
export PGPASSWORD="${DB_PASSWORD}"
until psql -h "localhost" -U "${DB_USER}" -p "${DB_PORT}" -d "postgres" -c '\q'; do
  >&2 echo "Postgres is still unavailable - sleeping"
  sleep 1
done

>&2 echo "Postgres is up and running on port ${DB_PORT}!"

export DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@localhost:${DB_PORT}/${DB_NAME}
sqlx database create
sqlx migrate run
```

注意在这里我们进行了两处修改, 一处是文件末尾的 `sqlx migrate run` 另外一处是对 `SKIP_DOCKER` 环境变量的判断

我们将 docker run 命令置于 SKIP_DOCKER 标志之后，以便轻松针对现有 Postgres 实例运行迁移，而无需手动将其关闭并使用 scripts/init_db.sh 重新创建。如果我们的脚本未启动 Postgres，此命令在 CI 中也非常有用。

现在，我们可以使用以下命令迁移数据库

```shell
SKIP_DOCKER=true ./scripts/init_db.sh
```

您应该能够在输出中发现类似这样的内容

```plaintext
+ sqlx migrate run
```

如果您使用您[最喜欢的 Postgres GUI](https://www.pgadmin.org/) 检查数据库，您现在会看到一个 `subscriptions` 表，旁边还有一个全新的 `_sqlx_migrations` 表: 这是 sqlx 跟踪针对您的数据库运行了哪些迁移的地方——现在它应该包含一行，用于记录我们的 `create_subscriptions_table` 迁移。

## 编写我们的第一个查询

我们已迁移并运行数据库。该如何与它通信?

### Sqlx Feature Flags

我们安装了 `sqlx-cli`，但实际上还没有将 **sqlx** 本身添加为我们应用程序的依赖项。

我们在 `Cargo.toml` 中添加一行新代码:

```toml
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
    "runtime-tokio"
]
```

是的，有很多功能开关。让我们逐一介绍一下：

- `tls-rustls` 指示 sqlx 使用 actix 运行时作为其 Future，并使用 rustls 作为 TLS 后端；
- `macros` 允许我们访问 sqlx::query! 和 sqlx::query_as!，我们将会广泛使用它们；
- `postgres` 解锁 Postgres 特有的功能（例如非标准 SQL 类型）；
- `uuid` 增加了将 SQL UUID 映射到 [uuid crate](https://docs.rs/uuid/) 中的 Uuid 类型的支持。我们需要它来处理我们的 id 列；
- `chrono` 增加了将 SQL timestamptz 映射到 [chrono cratev](https://docs.rs/chrono/latest/chrono/) 中的 `DateTime<T>` 类型的支持。我们需要它来处理我们的 subscribed_at 列；
- `migrate` 允许我们访问 **sqlx-cli** 后台用来管理迁移的相同函数。事实证明，它对我们的测试套件很有用。
这些应该足够我们完成本章所需的工作了。

### 配置管理

连接到 Postgres 数据库最简单的入口点是 PgConnection。

[PgConnection](https://docs.rs/sqlx/latest/sqlx/struct.PgConnection.html) 实现了 [Connection](https://docs.rs/sqlx/latest/sqlx/prelude/trait.Connection.html) trait，它为我们提供了一个 [connect](https://docs.rs/sqlx/latest/sqlx/prelude/trait.Connection.html#method.connect) 方法:
它接受连接字符串作为输入，并异步返回一个 `Result<PostgresConnection,sqlx::Error>` 。

我们从哪里获取连接字符串?

我们可以在应用程序中硬编码一个连接字符串，然后将其用于测试。

或者，我们可以选择立即引入一些基本的配置管理机制。

这比听起来简单，而且可以节省我们在整个应用程序中追踪一堆硬编码值的成本。

[config](https://docs.rs/config/latest/config/) crate 是 Rust 配置方面的“瑞士军刀”：它支持多种文件格式，并允许您分层组合不同的源（例如环境变量、配置文件等），从而轻松地根据每个部署环境定制应用程序的行为。

我们暂时不需要任何花哨的东西: 一个配置文件就可以了。

### 腾出空间

目前，我们所有的应用程序代码都位于一个文件 lib.rs 中。

为了避免在添加新功能时造成混乱，我们需要快速将其拆分成多个子模块。我们希望采用以下文件夹结构:

```plaintext
src/
├── configuration.rs
├── lib.rs
├── main.rs
├── routes
│   ├── health_check.rs
│   └── subscriptions.rs
├── routes.rs
└── startup.rs
```

`lib.rs` 看起来是这样

```rs
//! src/lib.rs
pub mod configuration;
pub mod routes;
pub mod startup;
```

`startup.rs` 将包含我们的运行函数, `health_check` 函数存放在 `routes/health_check.rs` 中, `subscribe` 和 `FormData` 函数存放在 `routes/subscriptions.rs` 中,
`configuration.rs` 初始为空。这两个处理程序都重新导出到 `routes.rs` 中：

```rs
//! /src/routes.rs
mod health_check;
mod subscriptions;

pub use health_check::*;
pub use subscriptions::*;
```

您可能需要添加一些 pub 可见性修饰符, 以及对 `main.rs` 和 `tests/health_check.rs` 中的 `use` 语句进行一些修正。

请确保 `cargo test` 通过, 然后再继续下一步。

### 读取配置文件

要使用 `config` 管理配置，我们必须将应用程序设置表示为实现 `serde` 的 `Deserialize` trait 的 Rust 类型。

让我们创建一个新的 `Settings` struct:

```rs
//! src/configuration.rs
#[derive(serde::Deserialize)]
pub struct Settings {}
```

目前我们有两组配置值:

- 应用程序端口，actix-web 监听传入请求(目前在 main.rs 中硬编码为 8000)
- 数据库连接参数

让我们在“设置”中为每个配置值添加一个字段:

我们需要在 `DatabaseSettings` 之上添加 `#[derive(serde::Deserialize)]`, 否则编译器会报错

```plaintext
error[E0277]: the trait bound `DatabaseSettings: configuration::_::_serde::Deserialize<'_>` is not satisfied
    --> src/configuration.rs:3:19
     |
3    |     pub database: DatabaseSettings,
     |                   ^^^^^^^^^^^^^^^^ the trait `configuration::_::_serde::Deserialize<'_>` is not implemented for `DatabaseSettings`
     |
```

这是有道理的: 为了使整个类型可反序列化，类型中的所有字段都必须可反序列化。

我们有配置类型了，接下来做什么?

首先, 让我们使用以下命令将配置添加到依赖项中

```shell
cargo add config
```

我们想从名为 `configuration` 的配置文件中读取我们的应用程序设置:

```rs
pub fn get_configuration() -> Result<Settings, config::ConfigError> {
    let settings = config::Config::builder()
        // Add configuration values from a file named `configuration`.
        // It will look for any top-level file with an extension
        // that `config` knows how to parse: yaml, json, etc.
        .add_source(config::File::with_name("configuration"))
        .build()
        .unwrap();

    // Try to convert the configuration values it read into
    // our Settings type
    settings.try_deserialize()
}
```

让我们修改 `main` 方法 以读取配置作为第一步:

```rs
//! src/main.rs
use std::net::TcpListener;

use zero2prod::{configuration::get_configuration, run};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let configuration = get_configuration().expect("Failed to read config");
    let address = format!("0.0.0.0:{}", configuration.application_port);
    let listener = TcpListener::bind(address)?;

    run(listener)?.await
}
```

如果这时候尝试运行 `cargo run` 会崩溃

```plaintext
thread 'main' panicked at src/configuration.rs:20:10:
called `Result::unwrap()` on an `Err` value: configuration file "configuration" not found
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

让我们写一个配置文件来修复这个问题

我们可以使用任何文件格式，只要 config crate 知道如何处理它: 我们将选择 YAML。

```yaml
# configuration.yaml
application_port: 8000

database:
  host: "127.0.0.1"
  port: 5432
  username: "postgres"
  password: "password"
  database_name: "newsletter"
```

`cargo run` 现在应该可以顺利执行了。

### 连接到 Postgres

`PgConnection::connect` 需要单个连接字符串作为输入, 而 `DatabaseSettings` 则提供了对所有连接参数的精细访问。
让我们添加一个便捷的 connection_string 方法来做到这一点:

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
}
```

我们终于可以连接了!

让我们调整一下正常情况下的测试:

```rs
use sqlx::{Connection, PgConnection};
use zero2prod::configuration::get_configuration;

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // Arrange
    let app_address = spawn_app();
    let configuration = get_configuration().expect("Failed to read configuration");
    let connection_string = configuration.database.connection_string();
    // The `Connection` trait MUST be in scope for us to invoke
    // `PgConnection::connect` - it is not an inherent method of the struct!
    let connection = PgConnection::connect(&connection_string)
            .await
            .expect("Failed to connect to Postgres.");

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
```

而且... `cargo test` 通过了!

我们刚刚确认, 测试结果显示我们能够成功连接到 Postgres!

对于世界来说，这是一小步，而对于我们来说，却是一大步。

### 我们的测试断言

现在我们已经连接上了，终于可以编写测试断言了。

我们之前 10 页一直梦想着这些断言。

我们将使用 sqlx 的 query! 宏:

```rs
#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // [...]
    // The connection has to be marked as mutable!
    let mut connection = PgConnection::connect(&connection_string)
            .await
            .expect("Failed to connect to Postgres.");

    // [...]

    // Assert
    assert_eq!(200, response.status().as_u16());

    let saved = sqlx::query!("SELECT email, name FROM subscriptions",)
        .fetch_one(&mut connection)
        .await
        .expect("Failed to fetch saved subscription.");

    assert_eq!(saved.email, "ursula_le_guin@gmail.com");
    assert_eq!(saved.name, "le guin");
}
```

`saved` 的类型是什么? `query!` 宏返回一个匿名记录类型: 在编译时验证查询有效后，会生成一个结构体定义，其结果中的每个列都有一个成员 (例如, `email` 列对应的是 `saved.email`)。

如果我们尝试运行 `cargo test`, 将会报错:

```plaintext
error: set `DATABASE_URL` to use query macros online, or run `cargo sqlx prepare` to update the query cache
  --> tests/health_check.rs:52:17
   |
52 |     let saved = sqlx::query!("SELECT email, name FROM subscriptions",)
   |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: this error originates in the macro `$crate::sqlx_macros::expand_query` which comes from the expansion of the macro `sqlx::query` (in Nightly builds, run with -Z macro-backtrace for more inf
o)

error: could not compile `zero2prod` (test "health_check") due to 1 previous error
```

正如我们之前讨论过的，sqlx 在编译时会联系 Postgres 来检查查询是否格式正确。就像 `sqlx-cli` 命令一样，它依赖 `DATABASE_URL` 环境变量来获取数据库的地址。

我们可以手动导出 `DATABASE_URL`, 但每次启动机器并开始处理这个项目时，都会遇到同样的问题。我们不妨参考 [sqlx 作者的建议](https://github.com/launchbadge/sqlx#compile-time-verification)——添加一个顶层 `.env` 文件。

```properties
DATABASE_URL="postgres://postgres:password@localhost:5432/newsletter"
```

**sqlx** 会从中读取 `DATABASE_URL`, 省去了我们每次都重新导出环境变量的麻烦。

数据库连接参数放在两个地方 (`.env` 和
`configuration.yaml`) 感觉有点麻烦，但这不是什么大问题: **configuration.yaml** 可以用来
在应用程序编译后更改其运行时行为，而 **.env** 只与我们的开发流程、构建和测试步骤相关。

将 .env 文件提交到版本控制——我们很快就会在持续集成 (CI) 中用到它!

让我们再次尝试运行 `cargo test`:

```plaintext
running 3 tests
test health_check_works ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok
test subscribe_returns_a_200_for_valid_form_data ... FAILED

failures:

---- subscribe_returns_a_200_for_valid_form_data stdout ----

thread 'subscribe_returns_a_200_for_valid_form_data' panicked at tests/health_check.rs:55:10:
Failed to fetch saved subscription.: RowNotFound
```

它失败了，这正是我们想要的!

现在我们可以专注于修补应用程序，让它恢复正常。

### 更新 CI 流

如果你检查一下，你会发现你的 CI 流现在无法执行我们在开始时引入的大多数检查。

我们的测试现在依赖于正在运行的 Postgres 数据库才能正确执行。由于 `sqlx` 的编译时检查，我们所有的构建命令 (`cargo check`、`cargo lint`、`cargo build` )都需要一个正常运行的数据库!

我们不想再冒险使用一个损坏的 CI。

您可以在[这里](https://github.com/LukeMathWalker/zero-to-production/blob/root-chapter-03-part1/.github/workflows/general.yml)找到 GitHub Actions 设置的更新版本。只需更新 general.yml 文件即可。
