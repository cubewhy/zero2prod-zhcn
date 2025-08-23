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
// [...]
// TODO: wip
```

TODO: WIP
