# 数据库迁移

## 状态保存在应用程序之外

负载均衡依赖于一个强有力的假设: 无论使用哪个后端来处理传入的请求，结果都是相同的。

我们在第 3 章中已经讨论过这一点: 为了确保在易出错的环境中实现高可用性，云原生应用程序是无状态的——它们将所有持久化问题委托给外部系统（例如数据库）。

这就是负载均衡有效的原因: 所有后端都与同一个数据库通信，以查询和操作相同的状态。

可以将数据库视为一个巨大的全局变量。我们应用程序的所有副本都会持续访问和修改它。

状态是很难把握的。

## 部署和迁移

在滚动更新部署期间，应用程序的新旧版本同时处理实时流量。
从另一个角度来看：应用程序的新旧版本同时使用同一个数据库。

为了避免停机，我们需要一个两个版本都能理解的数据库模式。
这对于我们的大多数部署来说都不是问题，但当我们需要改进数据库模式时，就会造成严重的制约。

让我们回到我们最初要做的工作：确认邮件。

为了推进我们确定的实施策略，我们需要按如下方式改进数据库模式:

- 添加一个新表 subscription_tokens；
- 在现有的 subscriptions 表中添加一个新的必填列 status。

让我们回顾一下可能的情况，以便确信我们不可能一次性部署所有确认邮件而不会导致停机。

我们可以先迁移数据库，然后再部署新版本。

这意味着当前版本需要在迁移后的数据库上运行一段时间：我们当前的 POST /subscriptions 实现无法获取 `status` 字段，它会尝试在未填充 `status` 字段的情况下向 subscriptions 中插入新行。由于 `status` 字段被限制为 `NOT NULL`（即强制），所有插入操作都会失败——在新版本应用程序部署完成之前，我们将无法接受新的订阅者。

这很糟糕。

我们可以先部署新版本，然后再迁移数据库。

结果却截然相反：新版本的应用程序正在旧数据库架构上运行。

当调用 `POST /subscriptions` 时，它会尝试向 `subscriptions` 中插入一行 `status` 字段不存在的行——所有插入操作都会失败，在数据库迁移完成之前，我们将无法接受新的订阅者。

这又一次很糟糕。

## 多步骤迁移

一次大范围的发布并不能解决问题——我们需要分阶段、分步骤地实现目标。

这种模式与我们在测试驱动开发中看到的有些相似：我们不会同时修改代码和测试——两​​者中需要有一个保持不变，而另一个则进行修改。

这同样适用于数据库迁移和部署：如果我们想要改进数据库架构，就不能同时更改应用程序的行为。

可以将其视为数据库重构：我们奠定基础是为了构建我们以后需要的行为。

## 新的必填 Column

让我们首先查看 `status` column。

### 第一步: 添加为可选

我们首先要确保应用程序代码稳定。

在数据库端，我们生成一个新的迁移脚本:

```shell
sqlx migrate add add_status_to_subscriptions
```

```plaintext
Creating migrations/20250828120711_add_status_to_subscriptions.sql
```

我们现在可以编辑迁移脚本，将状态作为可选列添加到订阅中:

```sql
ALTER TABLE subscriptions ADD COLUMN status TEXT NULL;
```

针对本地数据库运行迁移 (`SKIP_DOCKER=true ./scripts/init_db.sh`): 现在我们可以运行测试套件，以确保代码即使在新的数据库架构下也能正常工作。

测试应该会通过: 继续迁移生产数据库。

### 步骤 2：开始使用新 Column

状态现已存在: 我们可以开始使用它了！

确切地说，我们可以开始写入状态:每次插入新订阅者时，我们都会将状态设置为已确认。

我们只需要将插入查询从

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        // [...]
    )
    // [...]
}
```

改为

```rs
pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at, status)
        VALUES ($1, $2, $3, $4, 'confirmed')
        "#,
        // [...]
    )
    // [...]
}
```

测试应该通过 - 将新版本的应用程序部署到生产中。

### 第三步: 回填并标记为 NOT NULL

最新版本的应用程序确保所有新订阅者的状态信息都会被填充。

为了将状态标记为“非空”，我们只需回填历史记录的值即可：然后我们就可以随意修改该列了。

让我们生成一个新的迁移脚本:

```shell
sqlx migrate add make_status_not_null_in_subscriptions
```

SQL 迁移应该看起来是这样的

```sql
-- We wrap the whole migration in a transaction to make sure
-- it succeeds or fails atomically. We will discuss SQL transactions
-- in more details towards the end of this chapter!
-- `sqlx` does not do it automatically for us.
BEGIN;
    -- Backfill `status` for historical entries
    UPDATE subscriptions
    SET status = 'confirmed'
    WHERE status IS NULL;
    -- Make `status` mandatory
    ALTER TABLE subscriptions ALTER COLUMN status SET NOT NULL;
COMMIT;
```

我们可以迁移本地数据库，运行测试套件，然后部署生产数据库。

我们成功了，我们将 `status` 添加为新的必填列!

## 一个新表

TODO: wip
