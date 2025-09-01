# 谁应该记录错误

让我们再次看一下请求失败时发出的日志。

```shell
# sqlx logs are a bit spammy, cutting them out to reduce noise
export RUST_LOG="sqlx=error,info"
export TEST_LOG=enabled
cargo t subscribe_fails_if_there_is_a_fatal_database_error | bunyan
```

错误级别的日志记录有三种:

- 一条由 insert_subscriber 中的代码发出

```rs
//! src/routes/subscriptions.rs
pub async fn insert_subscriber(
    transaction: &mut PgConnection,
    new_subscriber: &NewSubscriber,
) -> Result<Uuid, sqlx::Error> {
    let subscriber_id = Uuid::new_v4();
    sqlx::query!(/* */)
    .execute(transaction)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(subscriber_id)
}
```

- 当将 `SubscribeError` 转换为 `actix_web::Error` 时, `actix_web` 输出的错误
- 一个由我们的遥测中间件 `tracing_actix_web::TracingLogger` 发出的

我们不需要看到三次相同的信息——我们发出了不必要的日志记录，这不仅没有帮助，反而让运维人员更加困惑，难以理解正在发生的事情 (这些日志报告的是同一个错误吗？我处理的是三个不同的错误吗?)。

根据经验,

> 处理错误时应记录错误。

如果您的函数将错误向上传播（例如使用 `?` 运算符），则不应记录该错误。如果合理，可以添加更多上下文。

如果错误一直向上传播到请求处理程序，则将日志记录委托给专用的中间件 - 在本例中为  `tracing_actix_web::TracingLogger`。

actix_web 发出的日志记录将在下一个版本中被移除。我们暂时忽略它。

让我们回顾一下我们自己代码中的 `tracing::error` 语句:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn insert_subscriber(
    transaction: &mut PgConnection,
    new_subscriber: &NewSubscriber,
) -> Result<Uuid, sqlx::Error> {
    let subscriber_id = Uuid::new_v4();
    sqlx::query!(/* */)
    .execute(transaction)
    .await
    .inspect_err(|e| {
        // This needs to go, we are propagating the error via `?`
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(subscriber_id)
}

pub async fn store_token(
    transaction: &mut PgConnection,
    subscriber_id: Uuid,
    subscription_token: &str,
) -> Result<(), StoreTokenError> {
    sqlx::query!(/* */)
    .execute(transaction)
    .await
    .map_err(|e| StoreTokenError(e))?;

    Ok(())
}
```

再次检查日志以确认它们看起来完好无损。
