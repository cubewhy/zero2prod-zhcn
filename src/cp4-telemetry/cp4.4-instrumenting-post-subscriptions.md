# 检测 POST /订阅

让我们利用之前学到的关于日志的知识，来设计一个 POST /subscriptions 请求的处理程序。

目前它看起来像这样:

```rs
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
) -> HttpResponse {
    match sqlx::query!(/* [...] */)
    .execute(pool.get_ref())
    .await
    {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(e) => {
            println!("Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

让我们添加 `log` crate

```shell
cargo add log
```

我们应该在日志记录中捕获什么?

## 与外部系统的交互

让我们从一条久经考验的经验法则开始：任何通过网络与外部系统的交互都应受到密切监控。我们可能会遇到网络问题，数据库可能不可用，查询速度可能会随着订阅者表的变长而变慢，等等。

让我们添加两条日志记录: 一条在查询执行开始之前，一条在查询执行完成后立即记录。

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
) -> HttpResponse {
    log::info!("Saving new subscriber details in the database");
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
    .await
    {
        Ok(_) => {
            log::info!("New subscriber details have been saved");
            HttpResponse::Ok().finish()
        },
        Err(e) => {
            log::error!("Failed to execute query: {e:?}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

目前，我们只会在查询成功时发出日志记录。为了捕获失败，我们需要将 println 语句转换为错误级别的日志:

```rs
log::error!("Failed to execute query: {e:?}");
```

好多了——我们现在已经基本覆盖了那个查询。

请注意一个小而关键的细节：我们使用了 `std::fmt::Debug` 格式的 {:?} 来捕获查询错误。

操作员是日志的主要受众——我们应该尽可能多地提取有关任何故障的信息，以便于故障排除。Debug 提供了原始视图，而 `std::fmt::Display ({})` 会返回更美观的错误消息，更适合直接显示给我们的最终用户。

## 向用户一样思考

我们还应该捕捉什么?

之前我们说过

> 我们很乐意选择一个具有足够可观察性的应用程序，以便我们能够提供我们向用户承诺的服务水平。

这在实践中意味着什么?

我们需要改变我们的参考系统。

暂时忘记我们是这款软件的作者。

设身处地为你的一位用户着想，一个访问你网站、对你发布的内容感兴趣并想订阅你的新闻通讯的人。

对他们来说，失败意味着什么？

故事可能是这样的:

> 嘿！
> 我尝试用我的主邮箱地址
> `thomas_mann@hotmail.com` 订阅你的新闻通讯，但网站出现了一个奇怪的错误。
> 你能帮我看看发生了什么吗？
> 祝好，
> 汤姆
> 附言：继续加油，你的博客太棒了！

Tom 访问了我们的网站，点击“提交”按钮时收到了“一个奇怪的错误”。

如果我们能根据他提供的信息（例如他输入的电子邮件地址）对问题进行分类，那么我们的申请就足够容易被观察到了。

我们能做到吗？

首先，让我们确认一下这个问题：Tom 是否注册为订阅者?

我们可以连接到数据库，并运行一个快速查询，再次确认我们的订阅者表中没有包含 `thomas_mann@hotmail.com` 邮箱地址的记录。

问题已确认。现在怎么办?

我们的日志中没有包含订阅者的邮箱地址，所以我们无法搜索。这完全是死路一条。
我们可以要求 Tom 提供更多信息：我们所有的日志记录都有时间戳，也许如果他记得自己尝试订阅的时间，我们就能找到一些线索?

这清楚地表明我们当前的日志质量不够好。

让我们改进它们:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    log::info!(
        "Adding '{}' '{}' as a new subscriber.",
        form.email,
        form.name,
    );
    log::info!("Saving new subscriber details in the database");
    match sqlx::query!(/* [...] */)
    .execute(pool.get_ref())
    .await
    {
        Ok(_) => {
            log::info!("New subscriber details have been saved");
            HttpResponse::Ok().finish()
        }
        Err(e) => {
            println!("Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

好多了——我们现在有一条日志行，可以同时捕获姓名和电子邮件。

这足以解决 Tom 的问题吗?

我们应该记录姓名和电子邮件吗？如果您在欧洲运营，这些信息通常属于个人身份信息 (PII)，其处理必须遵守《通用数据保护条例》(GDPR) 中规定的原则和规则。我们应该严格控制谁可以访问这些信息、我们计划存储这些信息的时间、如果用户要求被遗忘，应该如何删除这些信息等等。一般来说，有很多类型的信息对于调试目的有用，但不能随意记录（例如密码）——您要么不得不放弃它们，要么依靠混淆技术（例如标记化/假名化）来在安全性、隐私性和实用性之间取得平衡。

## 日志必须易于关联

> 为了让示例简洁，我将在后续的终端输出中省略 sqlx 发出的日志。sqlx 的日志默认使用 INFO 级别——我们将在第 5 章中将其调低为 TRACE 级别。

如果我们的 Web 服务器只有一个副本在运行，并且该副本每次只能处理一个请求，那么我们可能会想象终端中显示的日志大致如下:

```plaintext
# First request
[.. INFO zero2prod] Adding 'thomas_mann@hotmail.com' 'Tom' as a new subscriber
```

```plaintext
[.. INFO zero2prod] Saving new subscriber details in the database
[.. INFO zero2prod] New subscriber details have been saved
[.. INFO actix_web] .. "POST /subscriptions HTTP/1.1" 200 ..
# Second request
[.. INFO zero2prod] Adding 's_erikson@malazan.io' 'Steven' as a new subscriber
[.. ERROR zero2prod] Failed to execute query: connection error with the database
[.. ERROR actix_web] .. "POST /subscriptions HTTP/1.1" 500 ..
```

您可以清楚地看到单个请求从哪里开始，在我们尝试执行该请求时发生了什么，我们返回了什么响应，下一个请求从哪里开始等等。

这很容易理解。

但当您同时处理多个请求时，情况并非如此:

```plaintext
[.. INFO zero2prod] Receiving request for POST /subscriptions
[.. INFO zero2prod] Receiving request for POST /subscriptions
[.. INFO zero2prod] Adding 'thomas_mann@hotmail.com' 'Tom' as a new subscriber
[.. INFO zero2prod] Adding 's_erikson@malazan.io' 'Steven' as a new subscriber
[.. INFO zero2prod] Saving new subscriber details in the database
[.. ERROR zero2prod] Failed to execute query: connection error with the database
[.. ERROR actix_web] .. "POST /subscriptions HTTP/1.1" 500 ..
[.. INFO zero2prod] Saving new subscriber details in the database
[.. INFO zero2prod] New subscriber details have been saved
[.. INFO actix_web] .. "POST /subscriptions HTTP/1.1" 200 ..
```

但是，我们哪些细节没有保存呢? `thomas_mann@hotmail.com` 还是 `s_erikson@malazan.io`?

从日志中无法判断。

我们需要一种方法来关联与同一请求相关的所有日志。

这通常使用请求 ID（也称为关联 ID）来实现：当我们开始处理传入请求时，我们会生成一个随机标识符（例如 UUID），然后将其与所有与该特定请求执行相关的日志关联起来。

让我们在处理程序中添加一个:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let request_id = Uuid::new_v4();
    log::info!(
        "request_id {} - Adding '{}' '{}' as a new subscriber.",
        request_id,
        form.email,
        form.name,
    );
    log::info!("request_id {request_id} - Saving new subscriber details in the database");
    match sqlx::query!(/* [...] */)
    .execute(pool.get_ref())
    .await
    {
        Ok(_) => {
            log::info!("request_id {request_id} - New subscriber details have been saved");
            HttpResponse::Ok().finish()
        }
        Err(e) => {
            println!("request_id {request_id} - Failed to execute query: {e}");
            HttpResponse::InternalServerError().finish()
        }
    }
}
```

传入请求的日志现在看起来像这样:

```shell
curl -i -X POST -d 'email=thomas_mann@hotmail.com&name=Tom' \
    http://127.0.0.1:8000/subscriptions
```

```plaintext
[.. INFO zero2prod] request_id 9ebde7e9-1efe-40b9-ab65-86ab422e6b87 - Adding
'thomas_mann@hotmail.com' 'Tom' as a new subscriber.
[.. INFO zero2prod] request_id 9ebde7e9-1efe-40b9-ab65-86ab422e6b87 - Saving
new subscriber details in the database
[.. INFO zero2prod] request_id 9ebde7e9-1efe-40b9-ab65-86ab422e6b87 - New
subscriber details have been saved
[.. INFO actix_web] .. "POST /subscriptions HTTP/1.1" 200 ..
```

现在，我们可以在日志中搜索 `thomas_mann@hotmail.com`，找到第一条记录，获取 `request_id`，然后拉取与该请求关联的所有其他日志记录。

几乎所有日志的 `request_id` 都是在我们的订阅处理程序中创建的，因此 `actix_web` 的 `Logger` 中间件完全不知道它的存在。

这意味着，当用户尝试订阅我们的新闻通讯时，我们无法知道应用程序返回的状态码是什么。

我们该怎么办?

我们可以硬着头皮移除 actix_web 的 Logger，编写一个中间件为每个传入请求生成一个随机的请求标识符，然后编写我们自己的日志中间件，使其能够识别该标识符并将其包含在所有日志行中。

这样可以吗? 可以。

我们应该这样做吗? 可能不应该。
