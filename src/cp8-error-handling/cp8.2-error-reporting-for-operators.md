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
// TODO: wip
```
