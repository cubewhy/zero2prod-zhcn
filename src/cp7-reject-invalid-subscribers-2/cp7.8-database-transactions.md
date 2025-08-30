# 数据库事务

## 全部还是什么都没有

但现在就宣布胜利还为时过早。

我们的 `POST /subscriptions` 处理程序变得越来越复杂——我们现在对 Postgres 数据库执行两个 `INSERT` 查询：一个用于存储新订阅者的详细信息，另一个用于存储
新生成的订阅令牌。

如果应用程序在这两个操作之间崩溃会发生什么?

第一个查询可能成功完成，但第二个查询可能永远不会执行。

调用 POST /subscriptions 后，我们的数据库可能处于三种状态:

- 已持久化新订阅者及其令牌
- 已持久化新订阅者，但没有令牌
- 未持久化任何内容。

查询越多，就越难以推断数据库可能的最终状态。

关系数据库（以及其他一些数据库）提供了一种缓解此问题的机制：事务。

事务是一种将相关操作组合成单个工作单元的方法。

数据库保证事务中的所有操作都会同时成功或失败: 数据库永远不会处于只有事务中部分查询的影响可见的状态。

回到我们的例子，如果我们将两个 INSERT 查询包装在一个事务中，现在有两种可能的最终状态:

- 新的订阅者及其令牌已被持久化
- 没有任何内容被持久化

处理起来更容易。

## Postgres 中的事务

要在 Postgres 中启动事务，请使用 [`BEGIN` 语句](https://www.postgresql.org/docs/current/sql-begin.html)。BEGIN 之后的所有查询都属于该事务。
然后，使用 [COMMIT 语句](https://www.postgresql.org/docs/current/sql-commit.html)完成事务。

实际上，我们已经在一个迁移脚本中使用了事务!

```sql
BEGIN;
    -- Backfill `status` for historical entries
    UPDATE subscriptions
    SET status = 'confirmed'
    WHERE status IS NULL;
    -- Make `status` mandatory
    ALTER TABLE subscriptions ALTER COLUMN status SET NOT NULL;
COMMIT;
```

如果事务中的任何查询失败，数据库就会回滚：所有先前查询执行的更改都将被还原，操作将中止。

您也可以使用 [ROLLBACK 语句](https://www.postgresql.org/docs/current/sql-rollback.html)显式触发回滚。

事务是一个深奥的主题:它们不仅提供了一种将多个语句转换为全有或全无操作的方法，还能隐藏未提交更改对可能同时针对同一张表运行的其他查询的影响。

随着需求的变化，您通常需要显式选择事务的[隔离级别](https://en.wikipedia.org/wiki/Isolation_(database_systems))，

以便微调数据库为您的操作提供的并发保证。随着系统规模和复杂性的增长，掌握各种与并发相关的问题（例如[脏读](https://en.wikipedia.org/wiki/Isolation_(database_systems))、[幻读](https://en.wikipedia.org/wiki/Isolation_(database_systems))等）变得越来越重要。

如果您想了解更多关于这些主题的信息，我强烈推荐[《设计数据密集型应用程序》](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/)。

## sqlx 中的事务

回到代码：我们如何在 sqlx 中利用事务?

您无需手动编写 BEGIN 语句：事务对于关系数据库的使用至关重要, 因此 sqlx 提供了专用 API。

通过在连接池中调用 `begin`, 我们可以从连接池中获取一个连接并启动一个事务:

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
    let new_subscriber = /*[...]*/;
    let mut transaction = match pool.begin().await {
        Ok(transaction) => transaction,
        Err(_) => return HttpResponse::InternalServerError().finish(),
    };
}
```

如果成功, `begin` 将返回一个 [`Transaction` 结构体](https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html)。
对 `Transaction` 的可变引用实现了 `sqlx` 的 [`Executor` 特性](https://docs.rs/sqlx/latest/sqlx/struct.Transaction.html)，因此可以用于
运行查询。所有使用 `Transaction` 作为执行器运行的查询都会成为该事务的一部分。

让我们将事务传递给 `insert_subscriber` 和 `store_token` 而不是 `pool`:

```rs
//! src/routes/subscriptions.rs
pub async fn subscribe(
    form: web::Form<FormData>,
    pool: web::Data<PgPool>,
    // Get the email client from the app context
    email_client: web::Data<EmailClient>,
    base_url: web::Data<ApplicationBaseUrl>,
) -> HttpResponse {
    // [...]
    let mut transaction = match pool.begin().await {
        Ok(transaction) => transaction,
        Err(_) => return HttpResponse::InternalServerError().finish(),
    };

    let Ok(subscriber_id) = insert_subscriber(&mut *transaction, &new_subscriber).await else {
        return HttpResponse::InternalServerError().finish();
    };
    let subscription_token = generate_subscription_token();
    if store_token(&mut *transaction, subscriber_id, &subscription_token)
        .await
        .is_err()
    {
        return HttpResponse::InternalServerError().finish();
    }
    // [...]
}

#[tracing::instrument(
    name = "Store subscription toke in the database",
    skip(subscription_token, transaction)
)]
pub async fn store_token(
    transaction: &mut PgConnection,
    subscriber_id: Uuid,
    subscription_token: &str,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"[...]"#,
        subscription_token,
        subscriber_id
    )
    .execute(transaction)
    .await
    // [...]
}

#[tracing::instrument(
    name = "Saving new subscriber details in the database",
    skip(new_subscriber, transaction)
)]
pub async fn insert_subscriber(
    transaction: &mut PgConnection,
    new_subscriber: &NewSubscriber,
) -> Result<Uuid, sqlx::Error> {
    // [...]
    sqlx::query!(
        r#"[...]"#,
        subscriber_id,
        new_subscriber.email.as_ref(),
        new_subscriber.name.as_ref(),
        Utc::now()
    )
    .execute(transaction)
    .await
    // [...]
}
```

如果你现在运行 cargo test，你会发现一些有趣的事情：我们的一些测试失败了!

为什么会这样?

正如我们讨论过的，事务要么提交，要么回滚。
Transaction 公开了两个专用方法：`Transaction::commit` 用于持久化更改，以及 `Transaction::rollback` 用于中止整个操作。

我们并没有调用这两个方法——在这种情况下会发生什么?

我们可以查看 sqlx 的源代码来更好地理解。

特别是 `Transaction` 的 `Drop` 实现:

```rs
impl<'c, DB> Drop for Transaction<'c, DB>
where
    DB: Database,
{
    fn drop(&mut self) {
        if self.open {
            // starts a rollback operation

            // what this does depends on the database but generally this means we queue a rollback
            // operation that will happen on the next asynchronous invocation of the underlying
            // connection (including if the connection is returned to a pool)

            DB::TransactionManager::start_rollback(&mut self.connection);
        }
    }
}
```

self.open 是一个内部布尔值，附加到用于启动事务并运行附加查询的连接上。

使用 `begin` 创建事务时，该值会被设置为 `true`, 直到调用 `rollback` 或 `commit` 为止:

```rs

impl<'c, DB> Transaction<'c, DB>
where
    DB: Database,
{
    #[doc(hidden)]
    pub fn begin(
        conn: impl Into<MaybePoolConnection<'c, DB>>,
        statement: Option<Cow<'static, str>>,
    ) -> BoxFuture<'c, Result<Self, Error>> {
        let mut conn = conn.into();

        Box::pin(async move {
            DB::TransactionManager::begin(&mut conn, statement).await?;

            Ok(Self {
                connection: conn,
                open: true,
            })
        })
    }

    /// Commits this transaction or savepoint.
    pub async fn commit(mut self) -> Result<(), Error> {
        DB::TransactionManager::commit(&mut self.connection).await?;
        self.open = false;

        Ok(())
    }

    /// Aborts this transaction or savepoint.
    pub async fn rollback(mut self) -> Result<(), Error> {
        DB::TransactionManager::rollback(&mut self.connection).await?;
        self.open = false;

        Ok(())
    }
}
```

换句话说：如果在 `Transaction` 对象超出范围（即调用 `Drop` 函数）之前未调用提交或回滚函数，则回滚命令会排队等待，一旦有机会就会执行。

这就是我们的测试失败的原因: 我们使用了事务，但并未显式提交更改。当连接返回到池中时，在请求处理程序的末尾，所有更改都会回滚，这不符合我们的测试预期。

我们可以通过在 `subscribe` 中添加一行代码来解决这个问题:

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
    let mut transaction = match pool.begin().await {
        Ok(transaction) => transaction,
        Err(_) => return HttpResponse::InternalServerError().finish(),
    };

    let Ok(subscriber_id) = insert_subscriber(&mut *transaction, &new_subscriber).await else {
        return HttpResponse::InternalServerError().finish();
    };
    let subscription_token = generate_subscription_token();
    if store_token(&mut *transaction, subscriber_id, &subscription_token)
        .await
        .is_err()
    {
        return HttpResponse::InternalServerError().finish();
    }
    if transaction.commit().await.is_err() {
        return HttpResponse::InternalServerError().finish();
    }

    // [...]
}
```

测试套件应该再次通过。

继续部署应用程序: 看到功能在实时环境中运行，会带来全新的满足感!
