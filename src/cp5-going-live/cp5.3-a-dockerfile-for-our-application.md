# 我们项目的Dockerfile

DigitalOcean 的 App Platform 原生支持容器化应用的部署。

我们的第一项任务，就是编写一个 Dockerfile，以便将应用打包并作为 Docker 容器来构建和运行。

## Dockerfiles

Dockerfile是你项目环境的配方。
他们包含了这些东西：你的基础镜像（通常是一个OS包含了编程语言的工具链）和一个接一个的命令（COPY, RUN, 等）。这样就可以构建一个你需要的环境。

让我们看看一个最简单的可用的Rust项目的Dockerfile

```Dockerfile
# 使用最新的 Rust 稳定版作为基础镜像
FROM rust:1.59.0

# 将工作目录切换到 `app` （相当于执行 `cd app`）
# 如果 `app` 文件夹不存在，Docker 会自动帮我们创建
WORKDIR /app

# 安装构建所需的系统依赖（用于链接配置）
RUN apt update && apt install lld clang -y

# 将当前工作环境中的所有文件复制到 Docker 镜像中
COPY . .

# 构建我们的可执行文件！
# 使用 release 配置以获得更快的运行速度
RUN cargo build --release

# 当执行 `docker run` 时，启动编译好的二进制程序！
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
docker build -t zero2prod .
```

命令最后的点是干什么的？

## 构建上下文

`docker build`的执行依赖两个要素：一个配方（即`Dockerfile`），以及一个**构建上下文（build context）**  
你可以把正在构建的docker镜像想象成一个完全隔离的环境。  
它和本地机器唯一的接触点就是诸如`COPY`或`ADD`这样的命令。  
而**构建上下文**则决定了：在执行`COPY`等命令时，镜像内部能够“看见”你主机上的那些文件。  

举个例子，当我们使用`.`，就是告诉docker：“请把当前目录作为这次构建镜像时的上下文。”
因此，命令

```Dockerfile
COPY . /app
```

会将当前目录下的所有文件（包括源代码！）复制到镜像`/app`目录。

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
当我们在docker运行`cargo build`时，显然的，sqlx无法通过.env文件中确定的DATABASE_URL环境变量建立到数据库的连接。

### 如何解决？

我们可以通过--network来在构建镜像时允许镜像去和本地主机运行的数据库通信。这是我们在CI管道中遵循的策略，因为我们无论如何都需要数据库来运行集成测试。  
不幸的是，由于在不同的操作系统（例如MACOS）上实施Docker网络会大大损害我们的构建可重现性，这对Docker构建有些麻烦。  
更好的选择是sqlx的离线模式。  
让我们在Cargo.toml加入sqlx的离线模式feature：

```toml
#! Cargo.toml
# [...]

# 使用表状的toml语法来避免一些长行！、
[dependencies.sqlx]
version = "0.5.7"
default-features = false
features = [
    "runtime-acitx-rustls",
    "macros",
    "postgres",
    "uuid",
    "chrono",
    "migrate",
    "offline"
]
```
