# 控制流错误

## 分层

我们实现了想要的结果（有用的日志），但我不太喜欢这个解决方案：我们从 Web 框架中实现了一个trait (`ResponseError`), 用于处理由一个完全不了解 REST 或 HTTP 协议的操作（`store_token`）返回的错误类型。我们可以从其他入口点（例如 CLI）调用 `store_token`——它的实现应该没有任何改变。

即使假设我们只会在 REST API 上下文中调用 `store_token`，我们也可能会添加依赖于该例程的其他端点——它们可能不希望在失败时返回 500。

在发生错误时选择合适的 HTTP 状态码是请求处理程序需要考虑的问题，它不应该泄露到其他地方。

让我们删除

```rs
//! src/routes/subscriptions.rs
// [...]

// Nuke it!
impl ResponseError for StoreTokenError {}
```

为了强制执行适当的关注点分离，我们需要引入另一种错误类型: `SubscribeError`。

我们将使用它作为 `subscribe` 的失败变体，并负责 HTTP 相关的逻辑 (`ResponseError` 的实现)。

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(/* */) -> Result<HttpResponse, SubscribeError> {
    // [...]
}

#[derive(Debug)]
struct SubscribeError {}

impl std::fmt::Display for SubscriberError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(
            f,
            "Failed to create a new subscriber."
        )
    }
}

impl std::error::Error for SubscriberError {}

impl ResponseError for SubscriberError {}
```

如果你运行 `cargo check`, 你会看到大量的 '?', 无法将错误转换为 `SubscribeError` ——我们需要实现函数返回的错误类型与 `SubscribeError` 之间的转换。

## 将错误建模为枚举
