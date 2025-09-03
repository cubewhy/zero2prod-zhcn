# 存储数据的验证

`cargo check` 现在应该返回一个错误:

```text
error[E0308]: mismatched types
  --> src/routes/newsletters.rs:53:17
   |
52 |             .send_email(
   |              ---------- arguments to this method are incorrect
53 |                 subscriber.email,
   |                 ^^^^^^^^^^^^^^^^ expected `SubscriberEmail`, found `String`
   |
```

我们没有对从数据库中检索的数据进行任何验证 - `ConfirmedSubscriber::email` 是字符串类型。

相反, `EmailClient::send_email` 需要一个经过验证的电子邮件地址 - 一个 `SubscriberEmail` 实例。

我们可以先尝试一个简单的解决方案 - 将 `ConfirmedSubscriber::email` 更改为 `SubscriberEmail` 类型。

```rs
//! src/routes/newsletters.rs
// [...]
struct ConfirmedSubscriber {
    email: SubscriberEmail,
}
```

sqlx 似乎不是很喜欢这个类型, 它不知道怎么把 `TEXT` 转换为 `SubscriberEmail`

```text
error[E0277]: the trait bound `SubscriberEmail: From<String>` is not satisfied
  --> src/routes/newsletters.rs:70:16
   |
70 |       let rows = sqlx::query_as!(
   |  ________________^
71 | |         ConfirmedSubscriber,
72 | |         r#"
73 | |         SELECT email
...  |
76 | |         "#,
77 | |     )
   | |_____^ unsatisfied trait bound
   |
```

我们可以浏览 sqlx 的文档，寻找实现自定义类型支持的方法——虽然麻烦不少，但好处却不多。

我们可以采用与 POST /subscriptions 端点类似的方法——
我们使用两个结构体:

- 一个结构体编码了我们期望传输的数据布局 (FormData)；
- 另一个结构体通过使用我们的域类型解析原始表示来构建(`NewSubscriber`)。

对于我们的查询，它看起来像这样:

```rs
//! src/routes/newsletters.rs
// [...]

#[tracing::instrument(name = "Get confirmed subscribers", skip(pool))]
async fn get_confirmed_subscribers(
    pool: &PgPool,
) -> Result<Vec<ConfirmedSubscriber>, anyhow::Error> {
    // We only need `Row` to map the data coming out of this query.
    // Nesting its definition inside the function itself is a simple way
    // to clearly communicate this coupling (and to ensure it doesn't
    // get used elsewhere by mistake).
    struct Row {
        email: String,
    }

    let rows = sqlx::query_as!(
        Row,
        r#"
        SELECT email
        FROM subscriptions
        WHERE status = 'confirmed'
        "#,
    )
    .fetch_all(pool)
    .await?;

    // Map into the domain type
    let confirmed_subscribers = rows
        .into_iter()
        .map(|r| ConfirmedSubscriber {
            email: SubscriberEmail::parse(r.email).unwrap(),
        })
        .collect();

    Ok(confirmed_subscribers)
}
```

`SubscriberEmail::parse(r.email).unwrap()` 是个好主意吗?

所有新订阅者的邮件都会经过 `SubscriberEmail::parse` 中的验证逻辑——这是我们第六章重点讨论的主题。

那么，你可能会争辩说，我们数据库中存储的所有邮件必然都是有效的——这里无需考虑验证失败的情况。直接将它们全部解包就很安全了，因为它永远不会崩溃。

假设我们的软件永远不会更改，这种推理是合理的。但我们正在针对高部署频率进行优化!

存储在 Postgres 实例中的数据会在应用程序的新旧版本之间创建时间耦合。

我们从数据库中检索的邮件已被应用程序的先前版本标记为有效。当前版本可能不同意。

例如，我们可能会发现我们的邮件验证逻辑过于宽松——一些无效的邮件漏掉了，导致在尝试发送新闻通讯时出现问题。我们实施了更严格的验证例程，并部署了修补版本，突然间，电子邮件投递完全失效了!

`get_confirmed_subscribers` 在处理之前被认为有效但现在已经失效的已存储电子邮件时会引发 panic。

那么，我们该怎么办?

从数据库检索数据时，是否应该完全跳过验证?

没有一刀切的答案。

您需要根据域的需求，逐个评估问题。

有时处理无效记录是不可接受的——例程应该失败，并且操作员必须介入以纠正损坏的记录。

有时我们需要处理所有历史记录（例如分析数据），并且应该对数据做出最少的假设——`String` 是我们最安全的选择。

在我们的例子中，我们可以折中一下: 在获取下一期新闻通讯的收件人列表时，我们可以跳过无效的电子邮件。我们将对发现的每个无效地址发出警告，以便操作员识别问题并在未来的某个时间点更正存储的记录。

```rs
//! src/routes/newsletters.rs
// [...]

async fn get_confirmed_subscribers(
    pool: &PgPool,
) -> Result<Vec<ConfirmedSubscriber>, anyhow::Error> {
    // [...]

    // Map into the domain type
    let confirmed_subscribers = rows
        .into_iter()
        .filter_map(|r| match SubscriberEmail::parse(r.email) {
            Ok(email) => Some(ConfirmedSubscriber { email }),
            Err(error) => {
                tracing::warn!(
                    "A confirmed subscriber is using an invalid email address.\n{error}."
                );
                None
            },
        })
        .collect();

    Ok(confirmed_subscribers)
}
```

`filter_map` 是一个方便的组合器——它返回一个新的迭代器，其中只包含我们的闭包返回 `Some` 变量的项。

## 责任边界

我们可以避免这种情况，但值得花点时间思考一下这里谁在做什么。

当遇到无效的电子邮件地址时, `get_confirmed_subscriber` 是否是选择跳过或中止的最合适位置?

这感觉像是一个业务层面的决策，最好放在 `publish_newsletter` 中，它是我们交付工作流的驱动程序。

`get_confirmed_subscriber` 应该简单地充当存储层和领域层之间的适配器。它处理数据库特定的部分（即查询）和映射逻辑，但它将映射或查询失败时的处理决定委托给调用者。

让我们重构一下:

```rs
//! src/routes/newsletters.rs
// [...]

async fn get_confirmed_subscribers(
    pool: &PgPool,
    // We are returning a `Vec` of `Result`s in the happy case.
    // This allows the caller to bubble up errors due to network issues or other
    // transient failures using the `?` operator, while the compiler
    // forces them to handle the subtler mapping error.
    // See http://sled.rs/errors.html for a deep-dive about this technique.
) -> Result<Vec<Result<ConfirmedSubscriber, anyhow::Error>>, anyhow::Error> {
    // We only need `Row` to map the data coming out of this query.
    // Nesting its definition inside the function itself is a simple way
    // to clearly communicate this coupling (and to ensure it doesn't
    // get used elsewhere by mistake).
    struct Row {
        email: String,
    }

    let rows = sqlx::query_as!(
        Row,
        r#"
        SELECT email
        FROM subscriptions
        WHERE status = 'confirmed'
        "#,
    )
    .fetch_all(pool)
    .await?;

    // Map into the domain type
    let confirmed_subscribers = rows
        .into_iter()
        .map(|r| match SubscriberEmail::parse(r.email) {
            Ok(email) => Ok(ConfirmedSubscriber { email }),
            Err(error) => Err(anyhow::anyhow!(error)),
        })
        .collect();

    Ok(confirmed_subscribers)
}
```

我们现在在调用点收到编译器错误:

```text
error[E0609]: no field `email` on type `Result<ConfirmedSubscriber, anyhow::Erro
r>`
  --> src/routes/newsletters.rs:53:28
   |
53 |                 subscriber.email,
   |                            ^^^^^ unknown field
   |
```

我们可以立即修复:

```rs
//! src/routes/newsletters.rs
// [...]

pub async fn publish_newsletter(
    body: web::Json<BodyData>,
    pool: web::Data<PgPool>,
    email_client: web::Data<EmailClient>,
) -> Result<HttpResponse, PublishError> {
    let subscribers = get_confirmed_subscribers(&pool).await?;
    for subscriber in subscribers {
        // The compiler forces us to handle both the happy and unhappy case!
        match subscriber {
            Ok(subscriber) => {
                email_client
                    .send_email(
                        subscriber.email,
                        &body.title,
                        &body.content.html,
                        &body.content.text,
                    )
                    .await
                    .with_context(|| {
                        format!(
                            "Failed to send newsletter issue to {}",
                            subscriber.email
                        )
                    })?;
            }
            Err(error) => {
                tracing::warn!(
                    // We record the error chain as a structured field
                    // on the log record.
                    error.cause_chain = ?error,
                    // Userin `\` to split a long string literal over
                    // two lines, without creating a `\n` character.
                    "Skipping a confirmed subscriber. \
                    Their stored contact details are invalid",
                )
            }
        }
    }
    Ok(HttpResponse::Ok().finish())
}
```

## 关注编译器

编译器几乎可以正常工作:

```text
error[E0277]: `SubscriberEmail` doesn't implement `std::fmt::Display`
  --> src/routes/newsletters.rs:64:29
   |
63 | ...                   "Failed to send newsletter issue to {}",
   |                                                           -- required by th
is formatting parameter
64 | ...                   subscriber.email
   |                       ^^^^^^^^^^^^^^^^ `SubscriberEmail` cannot be formatte
d with the default formatter
   |
```

这是因为我们将 `ConfirmedSubscriber` 中的电子邮件类型从 `String` 更改为了 `SubscriberEmail`。

让我们为新类型实现 `Display`:

```rs
//! src/domain/subscriber_email.rs
// [...]

impl std::fmt::Display for SubscriberEmail {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        // We just forward to the Display implementation of
        // the wrapped String
        self.0.fmt(f)
    }
}
```

进展顺利! 又一个编译器错误，这次是借用检查器的错误!

```text
error[E0382]: borrow of moved value: `subscriber.email`
  --> src/routes/newsletters.rs:61:35
   |
55 |                         subscriber.email,
   |                         ---------------- value moved here
...
61 |                     .with_context(|| {
   |                                   ^^ value borrowed here after move
...
64 |                             subscriber.email
   |                             ---------------- borrow occurs due to use in cl
osure
   |
```

我们可以在第一次使用时直接添加一个 `.clone()` 函数，然后就完事了。

但让我们更复杂一点：我们真的需要在 `Email Client::send_email` 中获取订阅者电子邮件的所有权吗?

```rs
//! src/email_client.rs
// [...]

pub async fn send_email(
    &self,
    recipient: SubscriberEmail,
    subject: &str,
    html_content: &str,
    text_content: &str,
) -> Result<(), reqwest::Error> {
    // [...]
    let request_body = SendEmailRequest {
        from: self.sender.as_ref(),
        to: recipient.as_ref(),
        subject: subject,
        html_body: html_content,
        text_body: text_content,
    };
    // [...]
}
```

我们只需要能够调用 `as_ref` 即可—— `&SubscriberEmail` 就可以了。

让我们相应地更改签名:

```rs
//! src/email_client.rs
// [...]

pub async fn send_email(
    &self,
    recipient: &SubscriberEmail,
    // [...]
) -> Result<(), reqwest::Error> {
    // [...]
}
```

有几个调用点需要更新——编译器会很温柔地指出它们。我会把修复留给读者，作为练习。

完成后, 测试套件应该会通过。

## 移除一些模板代码

在继续之前，让我们最后看一下 `get_confirmed_subscribers`:

```rs
//! src/routes/newsletters.rs
// [...]

async fn get_confirmed_subscribers(
    pool: &PgPool,
) -> Result<Vec<Result<ConfirmedSubscriber, anyhow::Error>>, anyhow::Error> {
    struct Row {
        email: String,
    }

    let rows = sqlx::query_as!(
        Row,
        r#"
        SELECT email
        FROM subscriptions
        WHERE status = 'confirmed'
        "#,
    )
    .fetch_all(pool)
    .await?;

    // Map into the domain type
    let confirmed_subscribers = rows
        .into_iter()
        .map(|r| match SubscriberEmail::parse(r.email) {
            Ok(email) => Ok(ConfirmedSubscriber { email }),
            Err(error) => Err(anyhow::anyhow!(error)),
        })
        .collect();

    Ok(confirmed_subscribers)
}
```

Row 有什么用吗?

其实不然——查询本身就很简单，用一个专门的类型来表示返回的数据并没有什么好处。

我们可以切换回 `query!` 并完全移除 `Row`:

```rs
//! src/routes/newsletters.rs
// [...]

async fn get_confirmed_subscribers(
    pool: &PgPool,
) -> Result<Vec<Result<ConfirmedSubscriber, anyhow::Error>>, anyhow::Error> {
    let rows = sqlx::query!(
        r#"
        SELECT email
        FROM subscriptions
        WHERE status = 'confirmed'
        "#,
    )
    .fetch_all(pool)
    .await?;

    // Map into the domain type
    let confirmed_subscribers = rows
        .into_iter()
        .map(|r| match SubscriberEmail::parse(r.email) {
            Ok(email) => Ok(ConfirmedSubscriber { email }),
            Err(error) => Err(anyhow::anyhow!(error)),
        })
        .collect();

    Ok(confirmed_subscribers)
}
```

我们甚至不需要触及剩余的代码 - 它可以直接编译。
