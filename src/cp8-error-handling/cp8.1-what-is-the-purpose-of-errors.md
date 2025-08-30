# 错误的目的是什么的

让我们以一个例子开始:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn store_token(
    transaction: &mut PgConnection,
    subscriber_id: Uuid,
    subscription_token: &str,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"INSERT INTO subscription_tokens (subscription_token, subscriber_id)
        VALUES ($1, $2)"#,
        subscription_token,
        subscriber_id
    )
    .execute(transaction)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {e:?}");
    })?;

    Ok(())
}
```

我们正在尝试在 `subscription_tokens` 表中插入一行，以便存储新生成的令牌（根据 `subscription_id` 进行存储）。

执行操作可能存在错误: 我们在与数据库通信时可能会遇到网络问题，我们尝试插入的行可能违反了某些表约束（例如主键的唯一性），等等。

## 内部错误

### 使调用者能够做出反应

如果发生故障, `execute` 的调用者很可能希望得到通知——他们需要做出相应的反应，例如重试查询或使用 `?` 将故障传递到上游，就像我们的例子一样。

Rust 利用类型系统来传达操作可能无法成功: `execute` 的返回类型是 `Result`, 一个枚举。

```rs
pub enum Result<Success, Error> {
  Ok(Success),
  Err(Error)
}
```

不需要通用的 `Error` 类型——我们只需检查 `execute` 是否返回了 `Err` 变量即可, 例如

```rs
let outcome = sqlx::query!(/*[...]*/)
    .execute(transaction)
    .await;
if outcome == ResultSignal::Error {
    // Do something if it failed
}
```

如果只有一种故障模式，这种方法是可行的。事实上，操作可能以多种方式失败，我们可能需要根据具体情况采取不同的应对措施。

让我们看一下 `sqlx::Error` 的框架，它是执行的错误类型:

```rs
//! sqlx-core/src/error.rs
pub enum Error {
    Configuration(#[source] BoxDynError),
    InvalidArgument(String),
    Database(#[source] Box<dyn DatabaseError>),
    Io(#[from] io::Error),
    Tls(#[source] BoxDynError),
    Protocol(String),
    RowNotFound,
    TypeNotFound { type_name: String },
    ColumnIndexOutOfBounds { index: usize, len: usize },
    ColumnNotFound(String),
    ColumnDecode {
        index: String,

        #[source]
        source: BoxDynError,
    },
    Encode(#[source] BoxDynError),
    Decode(#[source] BoxDynError),
    AnyDriverError(#[source] BoxDynError),
    PoolTimedOut,
    PoolClosed,
    WorkerCrashed,
    Migrate(#[source] Box<crate::migrate::MigrateError>),
    InvalidSavePointStatement,
    BeginFailed,
}
```

列表很丰富，不是吗?

`sqlx::Error` 实现为枚举，允许用户匹配返回的错误，并根据底层故障模式采取不同的行为。例如，您可能希望重试 `PoolTimedOut`, 而您可能会放弃 `ColumnNotFound`。

### 帮助操作员排除故障

如果操作只有一种故障模式，我们是否应该只使用 `()` 作为错误类型?

Err(()) 可能足以让调用者决定该做什么——例如，向用户返回 500 内部服务器错误。

但控制流并非应用程序中错误的唯一用途。

我们希望错误能够包含足够的上下文信息，以便为运维人员（例如开发人员）生成包含足够详细信息的报告，以便他们进行故障排除。

我们所说的报告是什么意思?

在像我们这样的后端 API 中，它通常是一个日志事件。
在 CLI 中，它可能是使用 --verbose 标志时显示在终端中的错误消息。

实现细节可能有所不同，但目的保持不变：帮助人们理解哪里出了问题。

这正是我们在初始代码片段中所做的:

```rs
//! src/routes/subscriptions.rs
// [...]

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

    // [...]
}
```

如果查询失败，我们会捕获错误并发出日志事件。然后，我们可以在调查数据库问题时检查错误日志。

## 边缘错误

### 帮助用户排除故障

到目前为止，我们专注于 API 的内部机制——函数调用其他函数，以及操作符在发生问题后试图理清头绪。

那么用户呢?

与操作员一样，用户也希望 API 在遇到故障模式时发出信号。
当 `store_token` 失败时，我们的 API 用户会看到什么?

我们可以通过查看请求处理程序来找到答案:

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
```

他们收到一个没有正文的 HTTP 响应，并带有 500 内部服务器错误状态码。

该状态码的作用与 `store_token` 中的错误类型相同：它是一条机器可解析的信息，调用者（例如浏览器）可以使用它来确定下一步的操作（例如，假设是暂时性故障，则重试请求）。

浏览器背后的人呢? 我们告诉他们什么?

没什么，响应正文是空的。

这实际上是一个很好的实现: 用户不应该关心他们所调用 API 的内部结构——他们没有相关的心理模型，也无法确定失败的原因。

那是操作员的工作范围。

我们特意省略了这些细节。

在其他情况下，我们需要向人类用户传达额外的信息。让我们看看对同一端点的输入验证:

```rs
//! src/routes/subscriptions.rs
#[derive(serde::Deserialize)]
pub struct FormData {
    email: String,
    name: String,
}

impl TryFrom<FormData> for NewSubscriber {
    type Error = String;

    fn try_from(value: FormData) -> Result<Self, Self::Error> {
        let name = SubscriberName::parse(value.name)?;
        let email = SubscriberEmail::parse(value.email)?;

        Ok(NewSubscriber { email, name })
    }
}
```

我们收到了用户提交的表单中附加的电子邮件地址和姓名数据。

这两个字段都需要经过额外的验证——`SubscriberName::parse` 和 `SubscriberEmail::parse` 。这两个方法容易出错——它们会返回一个字符串作为错误类型，来解释出错的原因:

```rs
//! src/domain/subscriber_email.rs
// [...]

impl SubscriberEmail {
    pub fn parse(s: String) -> Result<Self, String> {
        if validator::ValidateEmail::validate_email(&s) {
            Ok(Self(s))
        } else {
            Err(format!("{s} is not a valid subscriber email."))
        }
    }
}
```

我必须承认, 这并不是最有用的错误消息 :我们只是告诉用户他们输入的电子邮件地址有误，
但却没有帮助他们确定错误原因。

说到底, 这根本无关紧要: 我们并没有将任何此类信息作为 API 响应的一部分发送给用户——他们收到的是一个没有正文的 400 Bad Request 错误代码。

```rs
//! src/routes/subscription.rs
// [...]

pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> HttpResponse {
    let new_subscriber = match form.into_inner().try_into() {
        Ok(subscriber) => subscriber,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    // [...]
}
```

这是一个严重的错误: 用户被蒙在鼓里，无法按要求调整自己的行为。

## 小结

让我们总结一下迄今为止的发现。

错误主要有两个用途:

- 控制流（即确定下一步操作）；
- 报告（例如，事后调查哪里出了问题）。我们还可以根据错误的位置来区分错误：
- 内部（即应用程序内某个函数调用另一个函数）；
- 边缘（即我们未能完成的 API 请求）。

控制流是脚本化的：所有决定下一步操作所需的信息都必须能够被机器访问。

我们使用类型（例如枚举变量）、方法和字段来表示内部错误。

我们依靠状态码来处理边缘错误。

而错误报告主要供人类使用。

内容必须根据受众进行调整。

操作员可以访问系统内部——应该为他们提供尽可能多的关于故障模式的上下文信息。

用户位于应用程序的边界之外: 应该只向他们提供调整其行为所需的信息量（例如，在必要时修复格式错误的输入）。

我们可以使用一个2x2的表格来可视化这个心智模型，其中位置为列，目的为行:

|            | Internal                     | At the edge  |
| ---------- | ---------------------------- | ------------ |
| 控制流报告 | 类型, 方法 , 字段, 日志/跟踪 | 状态码和body |

我们将用本章的剩余部分来改进表格中每个单元格的错误处理策略。
