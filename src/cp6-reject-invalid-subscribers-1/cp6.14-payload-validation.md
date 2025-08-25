# Payload 验证

如果您运行 `cargo test`，而不将运行的测试集限制在域内，您将看到我们的集成测试包含无效数据，仍然是红色的。

```plaintext
---- subscribe_returns_a_200_when_fields_are_present_but_invalid stdout ----

thread 'subscribe_returns_a_200_when_fields_are_present_but_invalid' panicked at
 tests/health_check.rs:182:9:
assertion `left == right` failed: The API did not return a 400 OK when the paylo
ad was empty email.
  left: 400
 right: 200
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    subscribe_returns_a_200_when_fields_are_present_but_invalid
```

让我们将我们出色的 `SubscriberEmail` 集成到应用程序中，以便从我们的 `/subscriptions` 端点中的验证中受益。
我们需要从 `NewSubscriber` 开始:

```rs
//! src/domain/new_subscriber.rs
use crate::domain::{SubscriberEmail, SubscriberName};

pub struct NewSubscriber {
    // We are not using `String` anymore!
    pub email: SubscriberEmail,
    pub name: SubscriberName,
}
```

如果你现在尝试编译这个项目，那可就糟了。

我们先从 `cargo check` 报告的第一个错误开始:

```plaintext
  --> src/routes/subscriptions.rs:28:16
   |
28 |         email: form.0.email,
   |                ^^^^^^^^^^^^ expected `SubscriberEmail`, found `String`
   |
```

它指的是我们的请求处理程序中的一行, `subscribe`:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let name = match SubscriberName::parse(form.0.name) {
        Ok(name) => name,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    let new_subscriber = NewSubscriber {
        email: form.0.email,
        name,
    };
    match insert_subscriber(&pool, &new_subscriber).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

我们需要模仿我们已经对 `name` 字段所做的事情: 首先我们解析 `form.0.email`, 然后我们将结果（如果成功）赋值给 `NewSubscriber.email` 。

```rs
//! src/routes/subscriptions.rs

// We added `SubscriberEmail`!
use crate::domain::{NewSubscriber, SubscriberEmail, SubscriberName};
// [...]

pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let name = match SubscriberName::parse(form.0.name) {
        Ok(name) => name,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    let email = match SubscriberEmail::parse(form.0.email) {
        Ok(email) => email,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    let new_subscriber = NewSubscriber {
        email,
        name,
    };
    // [...]
}
```

是时候处理第二个错误了

```plaintext
  --> src/routes/subscriptions.rs:53:9
   |
53 |         new_subscriber.email,
   |         ^^^^^^^^^^^^^^
   |         |
   |         expected `&str`, found `SubscriberEmail`
   |         expected due to the type of this binding

error[E0277]: the trait bound `SubscriberEmail: sqlx::Encode<'_, Postgres>` is not satisfied
```

这是在我们的 `insert_subscriber` 函数中，我们执行 SQL INSERT 查询来存储新订阅者的详细信息:

```rs
//! src/routes/subscriptions.rs

// [...]

pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        Uuid::new_v4(),
        new_subscriber.email,
        new_subscriber.name.as_ref(),
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

解决方案就在下面这行——我们只需要使用 `AsRef<str>` 的实现, 借用 `SubscriberEmail` 的内部字段作为字符串切片。

```rs
#[tracing::instrument(
    name = "Saving new subscriber details in the database",
    skip(new_subscriber, pool)
)]
pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"[...]"#,
        Uuid::new_v4(),
        new_subscriber.email.as_ref(),
        new_subscriber.name.as_ref(),
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

就这样了——现在可以编译了!

我们的集成测试怎么样?

```shell
cargo test
```

```plaintext
running 4 tests
test subscribe_returns_a_400_when_data_is_missing ... ok
test health_check_works ... ok
test subscribe_returns_a_200_when_fields_are_present_but_invalid ... ok
test subscribe_returns_a_200_for_valid_form_data ... ok
```

全部通过! 我们做到了!

## 使用 TryFrom 重构

在继续之前，我们先花几段时间来重构一下刚刚写的代码。我指的是我们的请求处理程序 `subscribe`:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let name = match SubscriberName::parse(form.0.name) {
        Ok(name) => name,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    let email = match SubscriberEmail::parse(form.0.email) {
        Ok(email) => email,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    let new_subscriber = NewSubscriber {
        email,
        name,
    };
    match insert_subscriber(&pool, &new_subscriber).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

我们可以在 `parse_subscriber` 函数中提取前两个语句:

```rs
pub fn parse_subscriber(form: FormData) -> Result<NewSubscriber, String> {
    let name = SubscriberName::parse(form.name)?;
    let email = SubscriberEmail::parse(form.email)?;

    Ok(NewSubscriber { email, name })
}

#[tracing::instrument(/*[...]*/)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let new_subscriber = match parse_subscriber(form.into_inner()) {
        Ok(subscriber) => subscriber,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    // [...]
}
```

重构使我们更加清晰地分离了关注点：

- parse_subscriber 负责将我们的数据格式（从 HTML 表单收集的 URL 解码数据）转换为我们的领域模型（NewSubscriber）
- subscribe 仍然负责生成对传入 HTTP 请求的 HTTP 响应。

Rust 标准库在其 std::convert 子模块中提供了一些处理转换的特性。AsRef 就是从这里来的！

那里有没有什么特性可以捕捉到我们试图用 `parse_subscriber` 做的事情?

`AsRef` 不太适合我们这里要处理的问题: 两种类型之间容易出错的转换, 它会消耗输入值。

我们需要看看 [TryFrom](https://doc.rust-lang.org/std/convert/trait.TryFrom.html):

```rs
pub trait TryFrom<T>: Sized {
    type Error;

    // Required method
    fn try_from(value: T) -> Result<Self, Self::Error>;
}
```

将 `T` 替换为 `FormData`, 将 `Self` 替换为 `NewSubscriber`, 将 `Self::Error` 替换为 `String` ——这就是 `parse_subscriber` 函数的签名！

我们来试试看:

```rs
//! src/routes/subscriptions.rs
// [...]

impl TryFrom<FormData> for NewSubscriber {
    type Error = String;

    fn try_from(value: FormData) -> Result<Self, Self::Error> {
        let name = SubscriberName::parse(value.name)?;
        let email = SubscriberEmail::parse(value.email)?;

        Ok(NewSubscriber { email, name })
    }
}

#[tracing::instrument(/*[...]*/)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let new_subscriber = match form.into_inner().try_into() {
        Ok(subscriber) => subscriber,
        Err(_) => return HttpResponse::BadRequest().finish(),
    };
    match insert_subscriber(&pool, &new_subscriber).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

}
我们实现了 `TryFrom`，但却调用了 `.try_into`? 这到底是怎么回事?

标准库中还有另一个转换 trait，叫做 [`TryInto`](https://doc.rust-lang.org/std/convert/trait.TryInto.html):

```rs
pub trait TryInto<T>: Sized {
    type Error;

    // Required method
    fn try_into(self) -> Result<T, Self::Error>;
}
```

它的签名与 TryFrom 的签名相同——转换方向相反!
如果您提供 TryFrom 实现，您的类型将自动获得相应的 TryInto 实现，无需额外工作。

`try_into` 将 `self` 作为第一个参数，这使得我们可以执行 `form.0.try_into()` 而不是 `NewSubscriber::try_from(form.0)` ——如果您愿意，这完全取决于您的个人喜好。

总的来说，实现 `TryFrom/TryInto` 能给我们带来什么好处?

没什么特别的，也没有什么新功能——我们“只是”让我们的意图更清晰。

我们明确地说明了“这是一个类型转换！”。

这为什么重要？因为它能帮助别人!

当另一位熟悉 Rust 的开发人员进入我们的代码库时，他们会立即发现转换模式，因为我们使用的是他们已经熟悉的特性。
