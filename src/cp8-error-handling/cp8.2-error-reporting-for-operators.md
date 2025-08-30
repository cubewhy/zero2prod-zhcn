# 操作员错误报告

让我们从操作符的错误报告开始。

我们现在的错误日志记录做得好吗?

让我们编写一个快速测试来找出答案:

```rs
//! tests/api/subscriptions.rs
// [...]
#[tokio::test]
async fn subscribe_fails_if_there_is_a_fatal_database_error() {
    // Arrange
    let app = spawn_app().await;
    let body = "name=le%20guin&email=ursula_le_guin%40gmail.com";
    // Sabotage the database
    sqlx::query!("ALTER TABLE subscription_tokens DROP COLUMN subscription_token;",)
        .execute(&app.db_pool)
        .await
        .unwrap();

    // Act
    let response = app.post_subscriptions(body.into()).await;
    
    // Assert
    assert_eq!(response.status().as_u16(), 500);
}
```

测试立即通过 - 让我们看看应用程序发出的日志

```shell
export RUST_LOG="sqlx=error,info"
export TEST_LOG=enabled
cargo t subscribe_fails_if_there_is_a_fatal_database_error | bunyan
```

```log
[2025-08-30T09:18:10.900Z]  INFO: test/75405 on qby-workspace: starting 20 worke
rs (line=310,target=actix_server::builder)
    file: /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/act
ix-server-2.6.0/src/builder.rs
[2025-08-30T09:18:10.900Z]  INFO: test/75405 on qby-workspace: Tokio runtime fou
nd; starting in existing Tokio runtime (line=192,target=actix_server::server)
    file: /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/act
ix-server-2.6.0/src/server.rs
[2025-08-30T09:18:10.900Z]  INFO: test/75405 on qby-workspace: starting service:
 "actix-web-service-127.0.0.1:37105", workers: 20, listening on: 127.0.0.1:37105
 (line=197,target=actix_server::server)
    file: /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/act
ix-server-2.6.0/src/server.rs
[2025-08-30T09:18:10.945Z]  INFO: test/75405 on qby-workspace: [HTTP REQUEST - S
TART] (http.client_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.m
ethod=POST,http.route=/subscriptions,http.scheme=http,http.target=/subscriptions
,http.user_agent="",line=41,otel.kind=server,otel.name="POST /subscriptions",req
uest_id=7fca956f-0fe2-46b2-8128-905edb39b79f,target=tracing_actix_web::root_span
_builder)
    file: /home/cubewhy/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tra
cing-actix-web-0.7.19/src/root_span_builder.rs
[2025-08-30T09:18:10.945Z]  INFO: test/75405 on qby-workspace: [ADDING A NEW SUB
SCRIBER - START] (file=src/routes/subscriptions.rs,http.client_ip=127.0.0.1,http
.flavor=1.1,http.host=127.0.0.1:37105,http.method=POST,http.route=/subscriptions
,http.scheme=http,http.target=/subscriptions,http.user_agent="",line=40,otel.kin
d=server,otel.name="POST /subscriptions",request_id=7fca956f-0fe2-46b2-8128-905e
db39b79f,subscriber_email=ursula_le_guin@gmail.com,subscriber_name="le guin",tar
get=zero2prod::routes::subscriptions)
[2025-08-30T09:18:10.983Z]  INFO: test/75405 on qby-workspace: [SAVING NEW SUBSC
RIBER DETAILS IN THE DATABASE - START] (file=src/routes/subscriptions.rs,http.cl
ient_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.method=POST,htt
p.route=/subscriptions,http.scheme=http,http.target=/subscriptions,http.user_age
nt="",line=142,otel.kind=server,otel.name="POST /subscriptions",request_id=7fca9
56f-0fe2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_guin@gmail.com,subscr
iber_name="le guin",target=zero2prod::routes::subscriptions)
[2025-08-30T09:18:10.984Z]  INFO: test/75405 on qby-workspace: [SAVING NEW SUBSC
RIBER DETAILS IN THE DATABASE - END] (elapsed_milliseconds=0,file=src/routes/sub
scriptions.rs,http.client_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105
,http.method=POST,http.route=/subscriptions,http.scheme=http,http.target=/subscr
iptions,http.user_agent="",line=142,otel.kind=server,otel.name="POST /subscripti
ons",request_id=7fca956f-0fe2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_
guin@gmail.com,subscriber_name="le guin",target=zero2prod::routes::subscriptions
)
[2025-08-30T09:18:10.984Z]  INFO: test/75405 on qby-workspace: [STORE SUBSCRIPTI
ON TOKE IN THE DATABASE - START] (file=src/routes/subscriptions.rs,http.client_i
p=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.method=POST,http.rout
e=/subscriptions,http.scheme=http,http.target=/subscriptions,http.user_agent="",
line=93,otel.kind=server,otel.name="POST /subscriptions",request_id=7fca956f-0fe
2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_guin@gmail.com,subscriber_id
=cc96b7d3-40cf-4355-b856-13415718c380,subscriber_name="le guin",target=zero2prod
::routes::subscriptions)
[2025-08-30T09:18:10.985Z] ERROR: test/75405 on qby-workspace: [STORE SUBSCRIPTI
ON TOKE IN THE DATABASE - EVENT] Failed to execute query: Database(PgDatabaseErr
or { severity: Error, code: "42703", message: "column \"subscription_token\" of 
relation \"subscription_tokens\" does not exist", detail: None, hint: None, posi
tion: Some(Original(34)), where: None, schema: None, table: None, column: None, 
data_type: None, constraint: None, file: Some("parse_target.c"), line: Some(1065
), routine: Some("checkInsertTargets") }) (file=src/routes/subscriptions.rs,http
.client_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.method=POST,
http.route=/subscriptions,http.scheme=http,http.target=/subscriptions,http.user_
agent="",line=111,otel.kind=server,otel.name="POST /subscriptions",request_id=7f
ca956f-0fe2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_guin@gmail.com,sub
scriber_id=cc96b7d3-40cf-4355-b856-13415718c380,subscriber_name="le guin",target
=zero2prod::routes::subscriptions)
[2025-08-30T09:18:10.985Z]  INFO: test/75405 on qby-workspace: [STORE SUBSCRIPTI
ON TOKE IN THE DATABASE - END] (elapsed_milliseconds=0,file=src/routes/subscript
ions.rs,http.client_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.
method=POST,http.route=/subscriptions,http.scheme=http,http.target=/subscription
s,http.user_agent="",line=93,otel.kind=server,otel.name="POST /subscriptions",re
quest_id=7fca956f-0fe2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_guin@gm
ail.com,subscriber_id=cc96b7d3-40cf-4355-b856-13415718c380,subscriber_name="le g
uin",target=zero2prod::routes::subscriptions)
[2025-08-30T09:18:10.985Z]  INFO: test/75405 on qby-workspace: [ADDING A NEW SUB
SCRIBER - END] (elapsed_milliseconds=39,file=src/routes/subscriptions.rs,http.cl
ient_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.method=POST,htt
p.route=/subscriptions,http.scheme=http,http.target=/subscriptions,http.user_age
nt="",line=40,otel.kind=server,otel.name="POST /subscriptions",request_id=7fca95
6f-0fe2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_guin@gmail.com,subscri
ber_name="le guin",target=zero2prod::routes::subscriptions)
[2025-08-30T09:18:10.985Z]  INFO: test/75405 on qby-workspace: [HTTP REQUEST - E
ND] (elapsed_milliseconds=40,http.client_ip=127.0.0.1,http.flavor=1.1,http.host=
127.0.0.1:37105,http.method=POST,http.route=/subscriptions,http.scheme=http,http
.status_code=500,http.target=/subscriptions,http.user_agent="",line=41,otel.kind
=server,otel.name="POST /subscriptions",otel.status_code=OK,request_id=7fca956f-
0fe2-46b2-8128-905edb39b79f,target=tracing_actix_web::root_span_builder)
```

没有任何可操作的信息。记录 "Oops! Something went wrong!" 也同样有用。

我们需要继续查找，直到找到最后剩下的错误日志:

```log
[2025-08-30T09:18:10.985Z] ERROR: test/75405 on qby-workspace: [STORE SUBSCRIPTI
ON TOKE IN THE DATABASE - EVENT] Failed to execute query: Database(PgDatabaseErr
or { severity: Error, code: "42703", message: "column \"subscription_token\" of 
relation \"subscription_tokens\" does not exist", detail: None, hint: None, posi
tion: Some(Original(34)), where: None, schema: None, table: None, column: None, 
data_type: None, constraint: None, file: Some("parse_target.c"), line: Some(1065
), routine: Some("checkInsertTargets") }) (file=src/routes/subscriptions.rs,http
.client_ip=127.0.0.1,http.flavor=1.1,http.host=127.0.0.1:37105,http.method=POST,
http.route=/subscriptions,http.scheme=http,http.target=/subscriptions,http.user_
agent="",line=111,otel.kind=server,otel.name="POST /subscriptions",request_id=7f
ca956f-0fe2-46b2-8128-905edb39b79f,subscriber_email=ursula_le_guin@gmail.com,sub
scriber_id=cc96b7d3-40cf-4355-b856-13415718c380,subscriber_name="le guin",target
=zero2prod::routes::subscriptions)
```

当我们尝试与数据库通信时出现了问题——我们原本期望在 `subscription_tokens` 表中看到 `subscription_token` 列, 但不知何故, 它并没有出现。

这其实很有用!

但这是否是导致 500 错误的原因呢?

仅凭查看日志很难判断——开发人员必须克隆代码库，检查日志行的来源，并确保它确实是问题的原因。

这可以做到，但需要时间: 如果 [HTTP REQUEST - END] 日志记录在 `exception.details` 和 `exception.message` 中报告一些关于根本原因的有用信息，那就容易多了。

## 跟踪错误根本原因

要理解为什么 `tracing_actix_web` 的日志记录如此糟糕，我们需要（再次）检查我们的请求处理程序和 `store_token`:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> HttpResponse {
    // [...]
    if store_token(&mut *transaction, subscriber_id, &subscription_token)
        .await
        .is_err()
    {
        return HttpResponse::InternalServerError().finish();
    }
    // [...]
}

pub async fn store_token(
    transaction: &mut PgConnection,
    subscriber_id: Uuid,
    subscription_token: &str,
) -> Result<(), sqlx::Error> {
    sqlx::query!(/*[...]*/)
    .execute(transaction)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {e:?}");
    })?;

    Ok(())
}
```

我们发现的有用错误日志确实是由 `tracing::error` 调用发出的——错误消息包含由 `execute` 返回的 `sqlx::Error` 。

我们使用 `?` 运算符向上传播错误，但错误链在 `subscribe` 中中断——我们丢弃了从 `store_token` 收到的错误，并构建了一个裸露的 500 响应。

`HttpResponse::InternalServerError().finish()` 是 **actix_web** 和
`tracing_actix_web::TracingLogger` 在即将发出各自的日志记录时唯一能够访问的对象。

该错误不包含任何关于根本原因的上下文，因此日志记录同样毫无用处。

该如何修复它?

我们需要开始利用 actix_web 公开的错误处理机制，特别是
`actix_web::Error`。根据文档:

> `actix_web::Error` 用于以方便的方式通过 actix_web 传输来自 `std::error` 的错误。

这听起来正是我们想要的。那么，我们如何构建 actix_web::Error 的实例呢?

文档中提到:

> 可以通过使用 `into()` 将错误转换为创建 `actix_web::Error`。

有点间接，但我们可以弄清楚。

浏览文档中列出的实现，我们唯一可以使用的 From/Into 实现, 似乎是这个:

```rs
/// Build an `actix_web::Error` from any error that implements `ResponseError`
impl<T: ResponseError + 'static> From<T> for Error {
    fn from(err: T) -> Error {
        Error {
            cause: Box::new(err),
        }
    }
}
```

[ResponseError](https://docs.rs/actix-web/latest/actix_web/error/trait.ResponseError.html) 是 actix_web 暴露的 trait

```rs
pub trait ResponseError: Debug + Display {
    // Provided methods
    fn status_code(&self) -> StatusCode { ... }
    fn error_response(&self) -> HttpResponse<BoxBody> { ... }
}
```

我们只需要针对错误代码实现它!

actix_web 为这两个方法提供了默认实现, 返回 500 内部服务器错误——这正是我们需要的。因此，只需编写:

```rs
//! src/routes/subscriptions.rs
use actix_webL::ResponseError;
// [...]

impl ResponseError for sqlx::Error {}
```

编译器报错了

```text
error[E0117]: only traits defined in the current crate can be implemented for ty
pes defined outside of the crate
```

我们刚刚碰到了 Rust 的[孤儿规则](https://doc.rust-lang.org/book/ch10-02-traits.html#implementing-a-trait-on-a-type)：禁止为外部类型实现外部特征，其中“foreign”代表“来自另一个 crate”。

此限制旨在保持一致性：想象一下，如果你添加了一个依赖项，它定义了自己的 `sqlx::Error` 的 `ResponseError` 实现——当调用特征方法时，编译器应该使用哪一个?

抛开孤儿规则不谈，为 `sqlx::Error` 实现 `ResponseError` 仍然是一个错误。

我们希望在尝试持久化订阅者令牌时遇到 `sqlx::Error` 时返回 500 内部服务器错误。

在其他情况下，我们可能希望以不同的方式处理 `sqlx::Error`。

我们应该遵循编译器的建议：定义一个新类型来包装 `sqlx::Error`。

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn store_token(/*[...]*/) -> /*Using the new error type! */ Result<(), StoreTokenError> {
    sqlx::query!(
        r#"INSERT INTO subscription_tokens (subscription_token, subscriber_id)
        VALUES ($1, $2)"#,
        subscription_token,
        subscriber_id
    )
    .execute(transaction)
    .await
    .map_err(|e| {
        // [...]
        // Wrapping the underlying error
        StoreTokenError(e)
    })?;

    Ok(())
}

// A new error type, wrapping a sqlx::Error
pub struct StoreTokenError(sqlx::Error);

impl ResponseError for StoreTokenError {}
```

它不起作用，但原因不同:

```text
error[E0277]: `StoreTokenError` doesn't implement `std::fmt::Display`
   --> src/routes/subscriptions.rs:118:24
    |
118 | impl ResponseError for StoreTokenError {}
    |                        ^^^^^^^^^^^^^^^ the trait `std::fmt::Display` is no
t implemented for `StoreTokenError`
    |
```

`StoreTokenError` 缺少两个 trait 实现: `Debug` 和 `Display`。

这两个 trait 都与格式化有关，但它们的用途不同。

Debug 应该返回面向程序员的表示，尽可能忠实于底层类型结构，以便于调试（顾名思义）。几乎所有公共类型都应该实现
Debug。

而 Display 应该返回面向用户的底层类型表示。大多数类型没有实现 Display，并且无法通过 `#[derive(Display)]` 属性自动实现。

处理错误时，我们可以这样理解这两个 trait：`Debug` 返回尽可能多的信息，而 `Display` 则提供我们遇到的失败的简要描述，并提供必要的上下文。

让我们试试 `StoreTokenError`:

```rs
//! src/routes/subscriptions.rs
// [...]
#[derive(Debug)]
pub struct StoreTokenError(sqlx::Error);

impl ResponseError for StoreTokenError {}

impl std::fmt::Display for StoreTokenError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "A database error was encountered while trying to store a subscription token."
        )
    }
}
```

编译通过了！

现在我们可以在请求处理程序中利用它了：

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> Result<HttpResponse, actix_web::Error> {
    // [...]

    // You will have to wrap (early) returns in `Ok(...)` as well!
    // The `?` operator transarently invokes the `Into` trait
    // on your behalf - we don't need an explicit `map_err` anymore.
    store_token(/*[...]*/).await?;
    // [...]
}
```

让我们再次查看日志:

```shell
# sqlx logs are a bit spammy, cutting them out to reduce noise
export RUST_LOG="sqlx=error,info"
export TEST_LOG=enabled
cargo t subscribe_fails_if_there_is_a_fatal_database_error | bunyan
```

```text
    exception.details: StoreTokenError(Database(PgDatabaseError { severity: Erro
r, code: "42703", message: "column \"subscription_token\" of relation \"subscrip
tion_tokens\" does not exist", detail: None, hint: None, position: Some(Original
(34)), where: None, schema: None, table: None, column: None, data_type: None, co
nstraint: None, file: Some("parse_target.c"), line: Some(1065), routine: Some("c
heckInsertTargets") }))
```

好多了!

请求处理结束时发出的日志记录现在包含导致应用程序向用户返回 500 内部服务器错误的详细和简要描述。

查看此日志记录足以准确了解与此请求相关的所有信息。

## Error Trait

到目前为止，我们遵循了编译器的建议，并尝试满足 actix-web 在错误处理方面施加的限制。

让我们回过头来看看更大的图景: 在 Rust 中，错误应该是什么样的（不考虑 actix-web 的具体细节）?

Rust 的标准库有一个专用的 trait, Error。

```rs
pub trait Error: Debug + Display {
    /// Thw lower-level source of this error, if any.
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

它需要实现 `Debug` 和 `Display` 接口，就像 ResponseError 一样。

它还允许我们实现一个 `source` 方法，该方法返回错误的根本原因（如果有）。

为我们的错误类型实现 `Error` trait 的意义何在?

`Result` 不需要它——任何类型都可以用作错误变体。

```rs
pub enum Result<T, E> {
    /// Contains the success value
    Ok(T),

    /// Contains the error value
    Err(E),
}
```

`Error` trait 首先是一种在语义上将我们的类型标记为错误的方法。它可以帮助代码库的读者立即发现其用途。

它也是 Rust 社区标准化良好错误的最低要求的一种方式:

- 它应该提供不同的表示形式（调试和显示），以适应不同的受众
- 应该能够查看错误的根本原因（如果有）（来源）

此列表仍在不断更新 - 例如，有一个不稳定的回溯方法。

错误处理是 Rust 社区中一个活跃的研究领域 - 如果您有兴趣了解接下来的发展，我强烈建议您关注 Rust 错误处理工作组。

通过提供所有可选方法的良好实现，我们可以充分利用错误处理生态系统 - 这些函数已被设计为通用地处理错误。我们将在接下来的几个部分中编写一个!

### Trait 对象

在开始实现 `source` 之前，让我们仔细看看它的
返回值 - `Option<&(dyn Error + 'static)>`。
`dyn Error` 是一个 trait 对象 - 除了它实现了 `Error` trait 之外，我们对这个类型一无所知。

trait 对象，就像泛型类型参数一样，是 Rust 中实现多态性的一种方法：调用同一接口的不同实现。泛型类型在编译时解析（静态调度），而 trait 对象会产生运行时开销（动态调度）。

为什么标准库会返回 trait 对象?

它为开发人员提供了一种访问当前错误的根本原因的方法，同时保持其不透明。

它不会泄露任何关于根本原因类型的信息 - 您只能访问`Error` trait 公开的方法: 不同的表示形式（`Debug`、`Display`），以及使用 `source` 在错误链中更深入一层的机会。

### Error::source

让我们为 `StoreTokenError` 实现 `Error` :

```rs
//! src/routes/subscriptions.rs
// [...]

// TODO: wip
```
