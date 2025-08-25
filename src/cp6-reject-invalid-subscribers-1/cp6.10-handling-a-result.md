# 处理 Result

`SubscriberName::parse` 现在返回的是 `Result`, 但 `subscribe` 调用了 `expect` 方法，因此
如果返回 Err 变量，就会触发 `panic`。
整个应用程序的行为没有任何变化。

我们如何修改 subscribe，使其在验证错误时返回 400 Bad Request 错误? 我们可以看看我们在 insert_subscriber 调用中已经做了什么!

## match

我们如何处理调用方发生故障的可能性?

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn insert_subscriber(
    pool: &PgPool,
    new_subscriber: &NewSubscriber,
) -> Result<(), sqlx::Error> {
    // [...]
}
```

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
) -> HttpResponse {
    // [...]
    match insert_subscriber(&pool, &new_subscriber).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}
```

`insert_subscriber` 返回 `Result<(), sqlx::Error>` , 而 `subscribe` 使用的是 REST API 的语言——它的输出必须是 `HttpResponse` 类型。为了在错误情况下向调用者返回 `HttpResponse`, 我们需要将 sqlx::Error 转换为 REST API 技术领域内合理的表示形式——在我们的例子中，是 500 Internal Server Error。

这时 `match` 就派上用场了：我们告诉编译器在 `Ok` 和 `Err` 两种情况下该做什么。

## `?` 操作符

说到错误处理，让我们再看一下 `insert_subscriber`:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(/*[...]*/)
    .execute(pool)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(())
}
```

您是否注意到了 `Ok(())` 之前的 `?` ?

它是问号操作符 `?`。

`?` 在 Rust 1.13 中引入 - 它是语法糖。

当你使用易错函数并希望 “冒泡” 失败 (例如，类似于重新抛出已捕获的异常) 时, 它可以减少视觉噪音。

此代码块中的 `?`

```rs
insert_subscriber(&pool, &new_subscriber)
.await
.map_err(|_| HttpResponse::InternalServerError().finish())?;
```

等于如下代码块中控制流

```rs
if let Err(error) = insert_subscriber(&pool, &new_subscriber)
    .await
    .map_err(|_| HttpResponse::InternalServerError().finish())
{
    return Err(error);
}
```

它允许我们在出现故障时使用单个字符（而不是多行代码块）提前返回。

鉴于 `?` 会使用 Err 变量触发提前返回, 因此它只能在返回 Result 的函数中使用。subscribe (暂时) 不符合条件。

### 400 Bad Request

现在让我们处理 `SubscriberName::parse` 返回的错误:

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

`cargo test` 尚未通过，但我们收到了不同的错误:

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

test result: FAILED. 3 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; 
finished in 0.20s
```

使用空名称的测试用例现在可以通过了，但当提供空的电子邮件地址时，我们无法返回 400 Bad Request。

这并不出乎意料——我们还没有实现任何类型的电子邮件验证!
