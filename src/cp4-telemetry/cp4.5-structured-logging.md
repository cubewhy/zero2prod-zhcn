# 结构化日志

为了确保所有日志记录都包含 `request_id`, 我们必须：

- 重写请求处理管道中的所有上游组件 (例如 **actix-web** 的 **Logger**)
- 更改我们从订阅处理程序调用的所有下游函数的签名；如果它们要发出日志语句，则需要包含 `request_id`, 因此需要将其作为参数传递下去。

那么我们导入到项目中的 crate 发出的日志记录呢？我们也应该重写这些 crate 吗?

显然，**这种方法无法扩展**。

让我们退一步思考：我们的代码是什么样的?

我们有一个总体任务（HTTP 请求），它被分解为一系列子任务（例如，解析输入、进行查询等），这些子任务又可以递归地分解为更小的子例程。

每个工作单元都有一个持续时间（即开始和结束）。

每个工作单元都有一个与其关联的上下文（例如，新订阅者的姓名和电子邮件地址、request_id），这些上下文自然会被其所有子工作单元共享。

毫无疑问，我们正面临困境:日志语句是在特定时间点发生的孤立事件，而我们却固执地试图用它来表示树状处理流水线。
日志是一种错误的抽象。

那么，我们应该使用什么呢?

## `tracing` Crate

[tracing crate](https://docs.rs/tracing) 可以帮助我们

> tracing 扩展了日志式诊断，允许库和应用程序记录结构化事件，并附加有关时间性和因果关系的信息——与日志消息不同，跟踪中的跨度具有开始和结束时间，可以通过执行流进入和退出，并且可以存在于类似跨度的嵌套树中。

这真是天籁之音。

实际效果如何?

## 从 log 迁移到 tracing

只有一种方法可以找到答案——让我们将订阅处理程序转换为使用 `tracing` 而不是 `log` 进行检测。

让我们将 `tracing` 添加到依赖项中:

```shell
cargo add tracing --features=log
```

迁移的第一步非常简单：搜索函数主体中所有出现的 `log::` 字符串，并将其替换为 tracing。

```rs
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let request_id = Uuid::new_v4();
    tracing::info!(
        "request_id {} - Adding '{}' '{}' as a new subscriber.",
        request_id,
        form.email,
        form.name,
    );
    tracing::info!("request_id {request_id} - Saving new subscriber details in the database");
    match sqlx::query!(/* [...] */)
    .execute(pool.get_ref())
    .await
    {
        Ok(_) => {
            tracing::info!("request_id {request_id} - New subscriber details have been saved");
            HttpResponse::Ok().finish()
        }
        Err(e) => {
            println!("request_id {request_id} - Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

这样就好了。

如果您运行该应用程序并发出 POST /subscriptions 请求,

您将在控制台中看到完全相同的日志。完全相同。

很酷，不是吗?

这得益于我们在 Cargo.toml 中启用的 tracing [log 功能标志](https://docs.rs/tracing/latest/tracing/index.html#log-compatibility)。它确保每次使用 tracing 的宏创建事件或跨度时，都会发出相应的日志事件,

以便日志的记录器（在本例中为 **env_logger**）能够捕获它。

## tracing 的 Span

现在，我们可以开始利用跟踪的 Span 来更好地捕获程序的结构。

我们需要创建一个代表整个 HTTP 请求的 span:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let request_id = Uuid::new_v4();
    // Spans, like logs, have an associated level
    // `into_span` creates a span at the into-level
    let request_span = tracing::info_span!(
        "Adding a new subscriber.",
        %request_id,
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    );

    // Using `enter` in an async function is a recipe for disaster!
    // Bear with me for now, but don't do this at home.
    // See the floowing secion on `Instrumenting Futures`
    let _request_span_guard = request_span.enter();

    // [...]
    // `_request_span_guard` is dropped at the end of `subscribe`
    // That's when we "exit" the span
}
```

这里有很多事情要做——让我们分解一下。

我们使用 `info_span!` 宏创建一个新的 span，并将一些值附加到其上下文中: `request_id`、`form.email` 和 `form.name`。

我们不再使用字符串插值：跟踪允许我们将结构化信息关联到 span，作为键值对的集合32。我们可以显式命名它们（例如，将 `form.email` 命名为`subscriber_email`）, 也可以隐式使用变量名作为键（例如，单独的 `request_id` 等同于 `request_id = request_id`）。

请注意，我们在所有 span 前面都添加了 % 符号: 我们告诉 tracing 使用它们的 `Display` 实现进行日志记录。您可以在[它们的文档](https://docs.rs/tracing/latest/tracing/index.html#recording-fields)中找到有关其他可用选项的更多详细信息。

`info_span` 返回新创建的 span，但我们必须使用 `.enter()` 方法显式进入其中才能激活它。

.enter() 返回 [Entered](https://docs.rs/tracing/latest/tracing/span/struct.Entered.html) 的一个实例，它是一个守卫：只要守卫变量未被丢弃，所有下游 span 和日志事件都将被注册为已进入 span 的子级。这是一种典型的 [Rust 模式](https://doc.rust-lang.org/stable/rust-by-example/scope/raii.html)，通常被称为资源获取即初始化 (RAII)：编译器会跟踪所有变量的生命周期，当它们超出作用域时，会插入对其析构函数的调用，`Drop::drop`。

Drop trait 的默认实现只负责释放该变量所拥有的资源。不过，我们可以指定一个自定义的 Drop 实现，以便在 drop 时执行其他清理操作——例如，当 Entered 守卫被 drop 时退出 span:

```rs
//! `tracing`'s source code
impl<'a> Drop for Entered<'a> {
    #[inline]
    fn drop(&mut self) {
        // Dropping the guard exits the span.
        //
        // Running this behaviour on drop rather than with an explicit function
        // call means that spans may still be exited when unwinding.
        if let Some(inner) = self.span.inner.as_ref() {
            inner.subscriber.exit(&inner.id);
        }
    }
}
if_log_enabled! {{
    if let Some(ref meta) = self.span.meta {
        self.span.log(
            ACTIVITY_LOG_TARGET,
            log::Level::Trace,
            format_args!("<- {}", meta.name())
        );
    }
}}
```

检查依赖项的源代码通常可以发现一些有用信息——我们刚刚发现，如果启用了日志功能标志，当 span 退出时，跟踪将会发出跟踪级别的日志。

让我们立即尝试一下:

```shell
RUST_LOG=trace cargo run
```

```plaintext
[.. INFO zero2prod] Adding a new subscriber.; request_id=f349b0fe..
subscriber_email=ursulale_guin@gmail.com subscriber_name=le guin
[.. TRACE zero2prod] -> Adding a new subscriber.
[.. INFO zero2prod] request_id f349b0fe.. - Saving new subscriber details
in the database
[.. INFO zero2prod] request_id f349b0fe.. - New subscriber details have
been saved
[.. TRACE zero2prod] <- Adding a new subscriber.
[.. TRACE zero2prod] -- Adding a new subscriber.
[.. INFO actix_web] .. "POST /subscriptions HTTP/1.1" 200 ..
```

注意，我们在 span 上下文中捕获的所有信息是如何在发出的日志行中报告的。

我们可以使用发出的日志密切跟踪 span 的生命周期:

- 创建 span 时记录添加新订阅者的操作；
- 进入 span (->)；
- 执行 INSERT 查询；
- 退出 span (<-)；
- 最终关闭 span (--)。

等等，退出 span 和关闭 span 有什么区别?

很高兴你问这个问题!

你可以多次进入（和退出）一个 span。而关闭 span 是最终操作: 它发生在 span 本身被丢弃时。

当你有一个可以暂停然后恢复的工作单元时，这非常方便——例如一个异步任务！

## 检测 Futures

让我们以数据库查询为例。

执行器可能需要多次[轮询其 Future](https://doc.rust-lang.org/beta/std/future/trait.Future.html#tymethod.poll) 才能使其完成——当该 Future 处于空闲状态时，我们将处理其他 Future。
这显然会引发问题：我们如何确保不混淆它们各自的跨度?

最好的方法是紧密模拟 Future 的生命周期：每次执行器轮询 Future 时，我们都应该进入与其关联的跨度，
并在其每次暂停时退出。

这就是 [Instrument](https://docs.rs/tracing/latest/tracing/trait.Instrument.html) 的用武之地。它是 Future 的一个扩展特性。`Instrument::instrument` 正是我们想要的：每次轮询 self（Future）时，进入我们作为参数传递的跨度；并在 Future 每次暂停时退出跨度。

让我们在查询中尝试一下:

```rs
//! src/routes/subscriptions.rs
use tracing::Instrument;
// [...]

pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let request_id = Uuid::new_v4();
    let request_span = tracing::info_span!(
        "Adding a new subscriber.",
        %request_id,
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    );
    let _request_span_guard = request_span.enter();

    // We do not call `.enter` on query_span!
    // `.instrument` takes care of it at the right moments
    // in the query future lifetime
    let query_span = tracing::info_span!("Saving new subscriber details in the database");

    match sqlx::query!(/* [...] */)
    .execute(pool.get_ref())
    .instrument(query_span)
    .await
    {
        Ok(_) => {
            tracing::info!("request_id {request_id} - New subscriber details have been saved");
            HttpResponse::Ok().finish()
        }
        Err(e) => {
            println!("request_id {request_id} - Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

如果我们使用 `RUST_LOG=trace` 再次启动应用程序并尝试 POST /subscriptions 请求，我们将看到类似以下的日志:

```plaintext
[.. INFO zero2prod] Adding a new subscriber.; request_id=f349b0fe..
subscriber_email=ursulale_guin@gmail.com subscriber_name=le guin
[.. TRACE zero2prod] -> Adding a new subscriber.
[.. INFO zero2prod] Saving new subscriber details in the database
[.. TRACE zero2prod] -> Saving new subscriber details in the database
[.. TRACE zero2prod] <- Saving new subscriber details in the database
[.. TRACE zero2prod] -> Saving new subscriber details in the database
[.. TRACE zero2prod] <- Saving new subscriber details in the database
[.. TRACE zero2prod] -> Saving new subscriber details in the database
[.. TRACE zero2prod] <- Saving new subscriber details in the database
[.. TRACE zero2prod] -> Saving new subscriber details in the database
[.. TRACE zero2prod] -> Saving new subscriber details in the database
[.. TRACE zero2prod] <- Saving new subscriber details in the database
[.. TRACE zero2prod] -- Saving new subscriber details in the database
[.. TRACE zero2prod] <- Adding a new subscriber.
[.. TRACE zero2prod] -- Adding a new subscriber.
[.. INFO actix_web] .. "POST /subscriptions HTTP/1.1" 200 ..
```

我们可以清楚地看到查询 Future 在完成之前被执行器轮询了多少次。

太酷了!?

## tracing 的 Subscriber

我们着手从日志迁移到跟踪，是因为我们需要一个更好的抽象来有效地检测我们的代码。我们特别希望将 request_id 附加到与同一传入 HTTP 请求相关的所有日志中。

虽然我保证跟踪会解决我们的问题，但看看那些日志：request_id 只打印在第一个日志语句中，我们把它明确地附加到 span 上下文中。

为什么呢?

嗯，我们还没有完成**迁移**。

虽然我们已经将所有检测代码从 log 迁移到了 tracing，但我们仍然使用 env_logger 来处理所有事情!

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    env_logger::Builder::from_env(Env::default().default_filter_or("info")).init();

    // [...]
}
```

env_logger 的日志记录器实现了 log 的 Log 特性——它对 tracing 的 Span 所暴露的丰富结构一无所知！
tracing 与 log 的兼容性非常出色，现在是时候用 tracing 原生解决方案替换 env_logger 了。

tracing crate 遵循 log 使用的相同外观模式——您可以自由地使用它的宏来
检测您的代码，但应用程序负责明确如何处理该 Span 遥测数据。

[Subscriber](https://docs.rs/tracing/latest/tracing/trait.Subscriber.html) 是 log 的 Log 的 tracing 对应物：Subscriber 特性的实现
暴露了各种方法来管理 Span 生命周期的每个阶段——创建、进入/退出、闭包等等。

```rs
//! `tracing`'s source code
pub trait Subscriber: 'static {
    fn new_span(&self, span: &span::Attributes<'_>) -> span::Id;
    fn event(&self, event: &Event<'_>);
    fn enter(&self, span: &span::Id);
    fn exit(&self, span: &span::Id);
    fn clone_span(&self, id: &span::Id) -> span::Id;
    // [...]
}
```

跟踪文档的质量令人叹为观止——我强烈建议您亲自查看 [Subscriber 的文档](https://docs.rs/tracing/latest/tracing/trait.Subscriber.html)，以正确理解每个方法的作用。

## tracing-subscriber

tracing 不提供任何开箱即用的 subscriber。

我们需要研究 `tracing-subscriber`（这是 tracing 项目内部维护的另一个 crate）, 以便找到一些基本的订阅器来启动它。让我们将它添加到我们的依赖项中:

```shell
cargo add tracing-subscriber --features=registry,env-filter
```

tracing-subscriber 的功能远不止提供一些便捷的订阅器。

它引入了另一个关键特性：[Layer](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/layer/trait.Layer.html)。

Layer 使得构建跨度数据的处理管道成为可能：我们不必提供一个包罗万象的订阅器来完成我们想要的一切；相反，我们可以组合多个较小的层来获得所需的处理管道。

这大大减少了整个追踪生态系统中的重复工作：人们专注于通过大量创建新的层来添加新功能，而不是试图构建功能最齐全的订阅器。

分层方法的基石是 [Registry](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/?search=Registry)。

Registry 实现了 Subscriber 特性，并处理了所有棘手的问题:

> Registry 实际上并不记录自身 trace: 相反，它收集并存储暴露给任何包裹它的层的 span 数据 [...]。Registry 负责存储 span 元数据，记录 span 之间的关系，并跟踪哪些 span 处于活动状态，哪些 span 已关闭。

下游层可以搭载 Registry 的功能并专注于其目的: 过滤需要处理的 span、格式化 span 数据、将 span 数据发送到远程系统等。

## tracing-bunyan-formatter

我们希望创建一个与旧版 env_logger 功能相同的订阅器。

我们将通过组合三个层来实现此目标:

- [tracing_subscriber::filter::EnvFilter](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/?search=EnvFilter) 根据日志级别和来源丢弃 span，就像我们在 env_logger 中通过 RUST_LOG 环境变量所做的那样；
- [tracing_bunyan_formatter::JsonStorageLayer](https://docs.rs/tracing-bunyan-formatter/latest/tracing_bunyan_formatter/struct.JsonStorageLayer.html) 处理 span 数据，并将相关元数据以易于理解的 JSON 格式存储，以供下游层使用。它尤其会将上下文从父 span 传播到其子 span；
- [tracing_bunyan_formatter::BunyanFormatterLayer](https://docs.rs/tracing-bunyan-formatter/latest/tracing_bunyan_formatter/struct.BunyanFormattingLayer.html) 构建于 JsonStorageLayer 之上，并以兼容 [bunyan](https://github.com/trentm/node-bunyan) 的 JSON 格式输出日志记录。

让我们将 [tracing_bunyan_formatter](https://docs.rs/tracing-bunyan-formatter/latest/tracing_bunyan_formatter/index.html) 添加到我们的依赖项中:

```shell
cargo add tracing_bunyan_formatter
```

现在我们可以将所有内容整合到我们的 `main` 方法中:

```rs
//! src/main.rs
use tracing::subscriber::set_global_default;
use tracing_bunyan_formatter::{BunyanFormattingLayer, JsonStorageLayer};
use tracing_subscriber::{layer::SubscriberExt, EnvFilter, Registry};

// [...]

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // We removed the `env_logger` line we had before!
    
    // We are falling back to printing all spans at into-level or above
    // if the RUST_LOG environment variable has not been set.
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));
    let formatting_layer = BunyanFormattingLayer::new(
        "zero2prod".into(),
        std::io::stdout
    );

    // The `with` method is provided by `SubscriberExt`, an extension
    // trait for `Subscriber` exposed by `tracing_subscriber`
    let subscriber = Registry::default()
        .with(env_filter)
        .with(JsonStorageLayer)
        .with(formatting_layer);

    // `set_global_default` can be used by applications to specify
    // what subscriber should be used to process spans
    set_global_default(subscriber).expect("Failed to set subscriber");
    // [...]
}
```

如果你使用 `cargo run` 启动应用程序并发出请求，你会看到这些日志（为了更容易阅读，这里格式化后打印）:

```plaintext
{
  "msg": "[ADDING A NEW SUBSCRIBER - START]",
  "subscriber_name": "le guin",
  "request_id": "30f8cce1-f587-4104-92f2-5448e1cc21f6",
  "subscriber_email": "ursula_le_guin@gmail.com"
  ...
}
{
  "msg": "[SAVING NEW SUBSCRIBER DETAILS IN THE DATABASE - START]",
  "subscriber_name": "le guin",
  "request_id": "30f8cce1-f587-4104-92f2-5448e1cc21f6",
  "subscriber_email": "ursula_le_guin@gmail.com"
  ...
}
{
  "msg": "[SAVING NEW SUBSCRIBER DETAILS IN THE DATABASE - END]",
  "elapsed_milliseconds": 4,
  "subscriber_name": "le guin",
  "request_id": "30f8cce1-f587-4104-92f2-5448e1cc21f6",
  "subscriber_email": "ursula_le_guin@gmail.com"
  ...
}
{
  "msg": "[ADDING A NEW SUBSCRIBER - END]",
  "elapsed_milliseconds": 5
  "subscriber_name": "le guin",
  "request_id": "30f8cce1-f587-4104-92f2-5448e1cc21f6",
  "subscriber_email": "ursula_le_guin@gmail.com",
  ...
}
```

我们成功了：所有附加到原始上下文的内容都已传播到其所有子跨度。

tracing-bunyan-formatter 还提供了开箱即用的持续时间：每次关闭跨度时，都会在控制台上打印一条 JSON 消息，并附加 elapsed_millisecond 属性。

JSON 格式在搜索方面非常友好：像 ElasticSearch 这样的引擎可以轻松提取所有这些记录，推断出模式并索引 request_id、name 和 email 字段。它释放了查询引擎的全部功能来筛选我们的日志!

这比我们以前的方法好得多：为了执行复杂的搜索，我们必须使用自定义的正则表达式，因此大大限制了我们可以轻松向日志提出的问题范围。

## tracing-log

如果你仔细观察，就会发现我们遗漏了一些东西：我们的终端只显示由应用程序直接发出的日志。actix-web 的日志记录怎么了?

tracing 的日志功能标志确保每次发生跟踪事件时都会发出一条日志记录，从而允许 log 的记录器获取它们。
反之则不然：log 本身并不提供跟踪事件的发送功能，也没有提供启用此功能的功能标志。

如果需要，我们需要显式注册一个记录器实现，将日志重定向到我们的跟踪订阅者进行处理。

我们可以使用 [tracing-log crate](https://docs.rs/tracing-log) 提供的 [LogTracer](https://docs.rs/tracing-log/latest/tracing_log/struct.LogTracer.html)。

```shell
cargo add tracing-log
```

让我们按需求修改 `main.rs`

```rs
//! src/main.rs
// [...]
#[tokio::main]
async fn main() -> std::io::Result<()> {
    // Redirect all `log`'s events to our subscriber
    LogTracer::init().expect("Failed to set logger");

    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));
    let formatting_layer = BunyanFormattingLayer::new(
        "zero2prod".into(),
        std::io::stdout
    );

    let subscriber = Registry::default()
        .with(env_filter)
        .with(JsonStorageLayer)
        .with(formatting_layer);

    set_global_default(subscriber).expect("Failed to set subscriber");
    // [...]
}
```

所有 actix-web 的日志应该再次在我们的控制台中输出。

## 删除没有用到的依赖

如果你快速浏览一下我们所有的文件，你会发现我们目前还没有在任何地方使用 log 或 env_logger。我们应该将它们从 Cargo.toml 文件中删除。

在大型项目中，重构后很难发现某个依赖项已不再使用。

幸运的是，工具再次派上用场——让我们安装 `cargo-udeps` (未使用的依赖项):

```shell
cargo install cargo-udeps
```

`cargo-udeps` 会扫描你的 Cargo.toml 文件，并检查 [dependencies] 下列出的所有 crate 是否已在项目中实际使用。查看 `cargo-deps` 的[“战利品陈列柜”](https://github.com/est31/cargo-udeps#trophy-case)，了解一系列热门 Rust 项目，这些项目都曾使用 `cargo-udeps` 识别未使用的依赖项并缩短构建时间。

现在就在我们的项目上运行它吧!

```shell
# cargo-udeps requires the nightly compiler.
# We add +nightly to our cargo invocation
# to tell cargo explicitly what toolchain we want to use.
cargo +nightly udeps
```

输出应该是

```plaintext
zero2prod
  dependencies
    "env-logger"
```

不幸的是，它没有识别到 `log`。
让我们从 `Cargo.toml` 文件中删除这两项。

## 清理初始化

我们坚持不懈地努力改进应用程序的可观察性。

现在，让我们回顾一下我们编写的代码，看看是否有任何有意义的改进空间。

让我们从 main 函数开始:

```rs
#[tokio::main]
async fn main() -> std::io::Result<()> {
    LogTracer::init().expect("Failed to set logger");

    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));
    let formatting_layer = BunyanFormattingLayer::new(
        "zero2prod".into(),
        std::io::stdout
    );

    let subscriber = Registry::default()
        .with(env_filter)
        .with(JsonStorageLayer)
        .with(formatting_layer);

    set_global_default(subscriber).expect("Failed to set subscriber");

    let configuration = get_configuration().expect("Failed to read config");
    let connection_pool = PgPool::connect(&configuration.database.connection_string())
        .await
        .expect("Failed to connect to Postgres.");

    let address = format!("0.0.0.0:{}", configuration.application_port);
    let listener = TcpListener::bind(address)?;

    run(listener, connection_pool)?.await
}
```

现在主函数中有很多事情要做。

我们来分解一下:

```rs
//! src/main.rs
use std::net::TcpListener;

use sqlx::PgPool;
use tracing::{subscriber::set_global_default, Subscriber};
use tracing_bunyan_formatter::{BunyanFormattingLayer, JsonStorageLayer};
use tracing_log::LogTracer;
use tracing_subscriber::{layer::SubscriberExt, EnvFilter, Registry};
use zero2prod::{configuration::get_configuration, run};

/// Compose multiple layers into a `tracing`'s subscriber.
///
/// # Implementation Notes
///
/// We are using `impl Subscriber` as return type to avoid having to
/// spell out the actual type of the returned subscriber, which is
/// indeed quite complex.
/// We need to explicitly call out that the returned subscriber is
/// `Send` and `Sync` to make it possible to pass it to `init_subscriber`
/// later on.
pub fn get_subscriber(
    name: impl Into<String>,
    env_filter: impl Into<String>,
) -> impl Subscriber + Send + Sync {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(env_filter.into()));
    let formatting_layer = BunyanFormattingLayer::new(
        name.into(),
        std::io::stdout
    );

    Registry::default()
        .with(env_filter)
        .with(JsonStorageLayer)
        .with(formatting_layer)
}

/// Register a subscriber as global default to process span data.
///
/// It should only be called once!
pub fn init_subscriber(subscriber: impl Subscriber + Send + Sync) {
    LogTracer::init().expect("Failed to set logger");

    set_global_default(subscriber).expect("Failed to set subscriber");
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let subscriber = get_subscriber("zero2prod", "into");
    init_subscriber(subscriber);

    // [...]
}
```

我们现在可以将 get_subscriber 和 init_subscriber 移到 zero2prod 库中的一个模块中，叫做 `telemetry`。

```rs
//! src/lib.rs
pub mod configuration;
pub mod routes;
pub mod startup;
pub mod telemetry;
```

```rs
//! src/telemetry.rs
use tracing::{subscriber::set_global_default, Subscriber};
use tracing_bunyan_formatter::{BunyanFormattingLayer, JsonStorageLayer};
use tracing_log::LogTracer;
use tracing_subscriber::{layer::SubscriberExt, EnvFilter, Registry};

pub fn get_subscriber(
    name: impl Into<String>,
    env_filter: impl Into<String>,
) -> impl Subscriber + Send + Sync {
    // [...]
}

pub fn init_subscriber(subscriber: impl Subscriber + Send + Sync) {
    // [...]
}
```

然后在 `main.rs` 中删除 `get_subscriber` 和 `init_subscriber`, 随后导入我们在 `telemetry` 模块的两个方法

```rs
//! src/main.rs
use zero2prod::telemetry::{get_subscriber, init_subscriber};

// [...]
```

太棒了!

## 集成测试的日志

我们不仅仅是为了美观/可读性而进行清理——我们还将这两个函数移到了 zero2prod 库中，以便我们的测试套件可以使用它们!

根据经验，我们在应用程序中使用的所有内容都应该反映在我们的集成测试中。

尤其是结构化日志记录，当集成测试失败时，它可以显著加快我们的调试速度: 我们可能不需要附加调试器，日志通常可以告诉我们哪里出了问题。这也是一个很好的基准：如果你无法从日志中调试，想象一下在生产环境中调试会有多么困难!

让我们修改我们的 `spawn_app` 辅助函数，让它负责初始化我们的 tracing 堆栈:

```rs
//! tests/health_check.rs
use zero2prod::telemetry::{get_subscriber, init_subscriber};

async fn spawn_app() -> TestApp {
  let subscriber = get_subscriber("test", "debug");
    init_subscriber(subscriber);
    // [...]
}

// [...]
```

如果您尝试运行 `cargo test`，您将会看到一次成功和一系列的测试失败:

```plaintext
test subscribe_returns_a_400_when_data_is_missing ... ok

failures:

---- subscribe_returns_a_200_for_valid_form_data stdout ----

thread 'subscribe_returns_a_200_for_valid_form_data' panicked at zero
2prod/src/telemetry.rs:27:23:
Failed to set logger: SetLoggerError(())
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- health_check_works stdout ----

thread 'health_check_works' panicked at zero2prod/src/telemetry.rs:27
:23:
Failed to set logger: SetLoggerError(())

failures:
    health_check_works
    subscribe_returns_a_200_for_valid_form_data

```

init_subscriber 应该只调用一次，但我们所有的测试都在调用它。

我们可以使用 once_cell 来解决这个问题

```shell
cargo add once_cell
```

```rs
use once_cell::sync::Lazy;

//! tests/health_check.rs
// Ensure that the `tracing` stack is only initialised once using `once_cell`
static TRACING: Lazy<()> = Lazy::new(|| {
    let subscriber = get_subscriber("test", "debug");
    init_subscriber(subscriber);
});


async fn spawn_app() -> TestApp {
    // The first time `initialize` is invoked the code in `TRACING` is executed.
    // All other invocations will instead skip execution.
    Lazy::force(&TRACING);
    
    // [...]
}
```

`cargo test` 现在通过了

然而，输出非常嘈杂：每个测试用例都会输出多行日志。

我们希望跟踪工具在每个测试中都能运行，但我们不想每次运行测试套件时都查看这些日志。

`cargo test` 解决了 println/print 语句的相同问题。默认情况下，它会吞掉所有打印到控制台的内容。您可以使用 `cargo test ---nocapture` 明确选择查看这些打印语句。

我们需要一个与跟踪工具等效的策略。

让我们为 get_subscriber 添加一个新参数，以便自定义日志应该写入到哪个接收器:

```rs
pub fn get_subscriber<Sink>(
    name: impl Into<String>,
    env_filter: impl Into<String>,
    sink: Sink
) -> impl Subscriber + Send + Sync 
    // This "weird" syntax is a higher-ranked trait bound (HRTB)
    // It basically means that Sink implements the `MakeWriter`
    // trait for all choices of the lifetime parameter `'a`
    // Check out https://doc.rust-lang.org/nomicon/hrtb.html
    // for more details.
    where Sink: for<'a> MakeWriter<'a> + Send + Sync + 'static
{
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new(env_filter.into()));
    let formatting_layer = BunyanFormattingLayer::new(
        name.into(),
        sink
    );

    Registry::default()
        .with(env_filter)
        .with(JsonStorageLayer)
        .with(formatting_layer)
}
```

我们可以调整 `main` 方法使用 `stdout`:

```rs
//! src/main.rs
// [...]

#[tokio::main]
async fn main() {
    let subscriber = get_subscriber("zero2prod", "into", std::io::stdout);
    // [...]
}
```

在我们的测试套件中，我们将根据环境变量 TEST_LOG 动态选择接收器。

- 如果设置了 TEST_LOG，我们将使用 std::io::stdout
- 如果未设置 TEST_LOG，我们将使用 std::io::sink 将所有日志发送到 void

我们自己编写的 --nocapture flag 版本

```rs
//! tests/health_check.rs
// [...]
static TRACING: Lazy<()> = Lazy::new(|| {
    let default_filter_level = "info";
    let subscriber_name = "test";c

    // We cannot assign the output of `get_subscriber` to a variable based on the value of `TEST_LOG`
    // because the sink is part of the type returned by `get_subscriber`, therefore they are not the
    // same type. We could work around it, but this is the most straight-forward way of moving forward.
    if std::env::var("TEST_LOG").is_ok() {
        let subscriber = get_subscriber(subscriber_name, default_filter_level, std::io::stdout);
        init_subscriber(subscriber);
    } else {
        let subscriber = get_subscriber(subscriber_name, default_filter_level, std::io::sink);
        init_subscriber(subscriber);
    }
});
```

当你想查看某个测试用例的所有日志来调试它时，你可以运行

```shell
# We are using the `bunyan` CLI to prettify the outputted logs
# The original `bunyan` requires NPM, but you can install a Rust-port with
# `cargo install bunyan`
TEST_LOG=true cargo test health_check_works | bunyan
```

并仔细检查输出以了解发生了什么。

是不是很棒?

## 清理仪表代码 - tracing::instrument

我们重构了初始化逻辑。现在来看看我们的插桩代码。

是时候再次回归 subscribe 了。

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let request_id = Uuid::new_v4();
    let request_span = tracing::info_span!(
        "Adding a new subscriber.",
        %request_id,
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    );
    let _request_span_guard = request_span.enter();

    // We do not call `.enter` on query_span!
    // `.instrument` takes care of it at the right moments
    // in the query future lifetime
    let query_span = tracing::info_span!("Saving new subscriber details in the database");

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
    // First we attach the instrumentation, then we `.await` it
    .instrument(query_span)
    .await
    {
        Ok(_) => {
            tracing::info!("request_id {request_id} - New subscriber details have been saved");
            HttpResponse::Ok().finish()
        }
        Err(e) => {
            println!("request_id {request_id} - Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

公平地说，日志记录给我们的订阅函数带来了一些噪音。

让我们看看能否稍微减少一下。

我们将从 `request_span` 开始: 我们希望订阅函数中的所有操作都在 `request_span` 的上下文中发生。
换句话说，我们希望将订阅函数包装在一个 `span` 中。

这种需求相当普遍: 将每个子任务提取到其各自的函数中是构建例程的常用方法，可以提高可读性并简化测试的编写；因此，我们经常会希望将 span 附加到函数声明中。

`tracing` 通过其 `tracing::instrument` 过程宏来满足这种特定的用例。让我们看看它的实际效果:

```rs
//! src/rotues/subscriptions.rs
// [...]
#[tracing::instrument(
    name = "Adding a new subscriber",
    skip(form, pool),
    fields(
        request_id = %Uuid::new_v4(),
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    )
)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let query_span = tracing::info_span!("Saving new subscriber details in the database");

    match sqlx::query!(/* [...] */)
    .execute(pool.get_ref())
    .instrument(query_span)
    .await
    {
        Ok(_) => {
            HttpResponse::Ok().finish()
        }
        Err(e) => {
            tracing::error!("Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

`#[tracing::instrument]` 在函数调用开始时创建一个 span，并自动将传递给函数的所有参数附加到 span 的上下文中——在我们的例子中是 `form` 和 `pool`。函数参数通常不会显示在日志记录中（例如 `pool`），或者我们希望更明确地指定应该捕获哪些参数/如何捕获它们（例如，命名 form 的每个字段）——我们可以使用 `skip` 指令明确地告诉跟踪忽略它们。

`name` 可用于指定与函数 `span` 关联的消息 - 如果省略，则默认为函数名称。

我们还可以使用 fields 指令来丰富 span 的上下文。它利用了我们之前在 info_span! 宏中见过的相同语法。
结果相当不错：所有插桩关注点在视觉上都被执行关注点分隔开来，

前者由一个过程宏来处理，该宏“修饰”函数声明，而函数体则专注于实际的业务逻辑。

需要指出的是，如果将 ``tracing::instrument`` 应用于异步函数，它也会小心地使用 `Instrument::instrument`。

让我们将查询提取到其自己的函数中，并使用 `tracing::instrument` 来摆脱 query_span
以及对 `.instrument` 方法的调用:

```rs
//! src/routes/subscription.rs
// [...]

#[tracing::instrument(
    name = "Adding a new subscriber",
    skip(form, pool),
    fields(
        request_id = %Uuid::new_v4(),
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    )
)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    match insert_subscriber(&pool, &form).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}

#[tracing::instrument(
    name = "Saving new subscriber details in the database",
    skip(form, pool)
)]
pub async fn insert_subscriber(pool: &PgPool, form: &FormData) -> Result<(), sqlx::Error> {
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
    .execute(pool)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(())
}
```

错误事件现在确实落在查询范围内，并且我们实现了更好的关注点分离：

- insert_subscriber 负责数据库逻辑，它不感知周围的 Web 框架 - 也就是说，我们不会将 `web::Form` 或 `web::Data` 包装器作为输入类型传递
- subscribe 通过调用所需的例程来协调要完成的工作，并根据 HTTP 协议的规则和约定将其结果转换为正确的响应

我必须承认我对 `tracing::instrument` 的无限热爱: 它显著降低了检测代码所需的工作量。

它会将你推向成功的深渊: 做正确的事是最容易的事。

## 保护你的秘密 - secrecy

`#[tracing::instrument]` 中其实有一个我不太喜欢的元素：它会自动将传递给函数的所有参数附加到 span 的上下文中——你必须选择不记录函数输入（通过 skip 选项），而不是选择加入。

你肯定不希望日志中包含机密信息（例如密码）或个人身份信息（例如最终用户的账单地址）。

选择退出是一个危险的默认设置——每次使用 `#[tracing::instrument]` 向函数添加新输入时，你都需要问自己: 记录这段输入安全吗? 我应该跳过它吗?

如果时间过长，别人就会忘记——你现在要处理一个安全事件。

你可以通过引入一个包装器类型来避免这种情况，该包装器类型明确标记哪些字段被视为敏感字段——`secrecy::Secret`。

```shell
cargo add secrecy --features=serde
```

我们来看看它的定义:

```rs
/// Wrapper type for values that contains secrets, which attempts to limit
/// accidental exposure and ensure secrets are wiped from memory when dropped.
/// (e.g. passwords, cryptographic keys, access tokens or other credentials)
///
/// Access to the secret inner value occurs through the [...]
/// `expose_secret()` method [...]
pub struct Secret<S>
where
    S: Zeroize,
{
    /// Inner secret value
    inner_secret: S,
}
```

Zeroize trait 提供的内存擦除功能非常实用。

我们正在寻找的关键属性是 `SecretBox` 的屏蔽 `Debug` 实现: `println!("{:?}", my_secret_string)` 输出的是 Secret([REDACTED String]) 而不是实际的 secret 值。这正是我们防止敏感信息通过 `#[tracing::instrument]` 或其他日志语句意外泄露所需要的。

显式包装器类型还有一个额外的好处: 它可以作为新开发人员的文档，帮助他们熟悉代码库。它明确了在你的领域/根据相关法规，哪些内容被视为敏感信息。

现在我们唯一需要担心的秘密值是数据库密码。让我们写一下:

```rs
//! src/configuration.rs
use secrecy::SecretBox;
// [...]

#[derive(serde::Deserialize)]
pub struct DatabaseSettings {
    // [...]
    pub password: SecretBox<String>,
}
```

`SecretBox` 不会干扰反序列化 - `SecretBox` 通过委托给包装类型的反序列化逻辑来实现 `serde::Deserialize`（如果您像我们一样启用了 serde 功能标志）。

编译器不满意:

```plaintext
error[E0277]: `SecretBox<std::string::String>` doesn't implement `std::fmt::Display`
  --> src/configuration.rs:22:28
   |
21 |             "postgres://{}:{}@{}:{}/{}",
   |                            -- required by this formatting parameter
22 |             self.username, self.password, self.host, self.port, self.database_name
   |                            ^^^^^^^^^^^^^ `SecretBox<std::string::String>` cannot be formatted 
with the default formatter
   |
   = help: the trait `std::fmt::Display` is not implemented for `SecretBox<std::string::String>`
   = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
   = note: this error originates in the macro `$crate::__export::format_args` which comes from the 
expansion of the macro `format` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: `SecretBox<std::string::String>` doesn't implement `std::fmt::Display`
  --> src/configuration.rs:29:28
   |
28 |             "postgres://{}:{}@{}:{}",
   |                            -- required by this formatting parameter
29 |             self.username, self.password, self.host, self.port
   |                            ^^^^^^^^^^^^^ `SecretBox<std::string::String>` cannot be formatted 
with the default formatter
   |
   = help: the trait `std::fmt::Display` is not implemented for `SecretBox<std::string::String>`
   = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
   = note: this error originates in the macro `$crate::__export::format_args` which comes from the 
expansion of the macro `format` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
error: could not compile `zero2prod` (lib) due to 2 previous errors
```

这是一项功能，而非 bug —— `secret::SecretBox` 没有实现 `Display` 接口，因此我们需要
明确允许暴露已包装的 secret。编译器错误提示我们，
由于整个数据库连接字符串嵌入了数据库密码，因此也应该将其标记为 `SecretBox`:

```rs
//! src/configuration.rs
use secrecy::{ExposeSecret, SecretBox};
// [...]

impl DatabaseSettings {
    pub fn connection_string(&self) -> SecretBox<String> {
        SecretBox::new(Box::new(format!(
            "postgres://{}:{}@{}:{}/{}",
            self.username, self.password.expose_secret(), self.host, self.port, self.database_name
        )))
    }

    pub fn connection_string_without_db(&self) -> SecretBox<String> {
        SecretBox::new(Box::new(format!(
            "postgres://{}:{}@{}:{}",
            self.username, self.password.expose_secret(), self.host, self.port
        )))
    }
}
```

```rs
//! src/main.rs
use secrecy::ExposeSecret;
use sqlx::PgPool;
use zero2prod::{configuration::get_configuration, run, telemetry::{get_subscriber, init_subscriber}};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // [...]
    let connection_pool = PgPool::connect(&configuration.database.connection_string().expose_secret())
        .await
        .expect("Failed to connect to Postgres.");

    // [...]
}
```

```rs
//! tests/health_check.rs
use secrecy::ExposeSecret;
// [...]

pub async fn configure_database(config: &DatabaseSettings) -> PgPool {
    // Create database
    let mut connection = PgConnection::connect(&config.connection_string_without_db().expose_secret())
        .await
        .expect("Failed to connect to Postgres");

    connection
        .execute(format!(r#"CREATE DATABASE "{}";"#, config.database_name).as_str())
        .await
        .expect("Failed to create database.");

    // Migrate database
    let connection_pool = PgPool::connect(&config.connection_string().expose_secret())
        .await
        .expect("Failed to connect to Postgres");

    sqlx::migrate!("./migrations")
        .run(&connection_pool)
        .await
        .expect("Failed to migrate the database");

    connection_pool
}
```

暂时就是这样——以后，一旦引入敏感值，我们将确保将其包装到 `SecretBox` 中。

## 请求Id

我们还有最后一项工作要做：确保特定请求的所有日志，特别是包含返回状态码的记录，都添加了 request_id 属性。怎么做呢?

如果我们的目标是避免接触 actix_web::Logger，最简单的解决方案是添加另一个中间件,`RequestIdMiddleware`, 它负责:

- 生成唯一的请求标识符
- 创建一个新的 span，并将请求标识符作为上下文附加
- 将其余的中间件链包装到新创建的 span 中

不过，这样会留下很多问题: `actix_web::Logger` 无法像其他日志那样以相同的结构化 JSON 格式让我们访问其丰富的信息（状态码、处理时间、调用者 IP 等）——我们必须从其消息字符串中解析出所有这些信息。

在这种情况下，我们最好引入一个支持跟踪的解决方案。

让我们将 `tracing-actix-web` 添加为依赖项之一

```shell
cargo add tracing-actix-web
```

```rs
//! src/startup.rs
use std::net::TcpListener;

use actix_web::{dev::Server, web, App, HttpServer};
use sqlx::PgPool;
use tracing_actix_web::TracingLogger;

use crate::routes::{health_check, subscribe};

pub fn run(
    listener: TcpListener,
    db_pool: PgPool,
) -> Result<Server, std::io::Error> {
    let db_pool = web::Data::new(db_pool);

    let server = HttpServer::new(move || {
        App::new()
            // Instead of `Logger::default`
            .wrap(TracingLogger::default())
            .route("/health_check", web::get().to(health_check))
            .route("/subscriptions", web::post().to(subscribe))
            .app_data(db_pool.clone())
    })
    .listen(listener)?
    .run();

    Ok(server)
}
```

如果您启动应用程序并发出请求，您应该会在所有日志中看到 `request_id` 以及 `request_path` 和其他一些有用的信息。

我们快完成了——还有一个未解决的问题需要解决。

让我们仔细看看 POST /subscriptions 请求发出的日志记录:

```plaintext
{
  "msg": "[REQUEST - START]",
  "request_id": "21fec996-ace2-4000-b301-263e319a04c5",
  ...
}
{
  "msg": "[ADDING A NEW SUBSCRIBER - START]",
  "request_id":"aaccef45-5a13-4693-9a69-5",
  ...
}
```

同一个请求却有两个不同的 request_id!

这个 bug 可以追溯到我们 `subscribe` 函数中的 `#[tracing::instrument]` 注解:

```rs
//! src/routes/subscriptions.rs
// [...]

#[tracing::instrument(
    name = "Adding a new subscriber",
    skip(form, pool),
    fields(
        request_id = %Uuid::new_v4(),
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    )
)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    // [...]
}
```

我们仍在函数级别生成一个 request_id，它会覆盖来自 TracingLogger 的 request_id。

让我们摆脱它来解决这个问题:

```rs
//! src/routes/subscriptions.rs
// [...]

#[tracing::instrument(
    name = "Adding a new subscriber",
    skip(form, pool),
    fields(
        subscriber_email = %form.email,
        subscriber_name = %form.name,
    )
)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    // [...]
}
```

现在一切都很好 - 我们应用程序的每个端点都有一个一致的 request_id。

## 利用 tracing 生态系统

我们介绍了 tracing 的诸多功能——它显著提升了我们收集的遥测数据的质量，并提高了插桩代码的清晰度。

与此同时，我们几乎没有触及整个 tracing 生态系统在订阅层方面的丰富性。

以下列举一些现成的组件:

- tracing-actix-web 与 OpenTelemetry 兼容。如果您插入 [tracing-opentelemetry](https://docs.rs/tracing-opentelemetry)，则可以将 span 发送到与 [OpenTelemetry](https://opentelemetry.io/) 兼容的服务（例如 Jaeger 或 Honeycomb.io）进行进一步分析；
- [tracing-error](https://docs.rs/tracing-error/) 使用 [SpanTrace](https://docs.rs/tracing-error/latest/tracing_error/struct.SpanTrace.html) 丰富了我们的错误类型，从而简化了故障排除。

毫不夸张地说，tracing 是 Rust 生态系统的基础 crate。虽然日志是最小公分母，但 tracing 现已成为整个诊断和插桩生态系统的现代支柱。
