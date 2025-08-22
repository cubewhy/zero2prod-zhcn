# 日志记录

日志是最常见的遥测数据类型。

即使是从未听说过可观察性的开发人员，也能直观地理解日志的用处：当事情出错时，你会查看日志来了解正在发生的事情，
并祈祷自己能捕获足够的信息来有效地进行故障排除。

那么，日志是什么呢?

它的格式因时代、平台和你使用的技术而异。

如今，日志记录通常是一堆文本数据，并用换行符分隔当前记录和下一个记录。例如

```plaintext
The application is starting on port 8080
Handling a request to /index
Handling a request to /index
Returned a 200 OK
```

对于 Web 服务器来说，这四条日志记录完全有效。

Rust 生态系统在日志记录方面能为我们提供什么?

## `log` crate

Rust 中用于日志记录的首选 crate 是 [log](https://docs.rs/log)。

log 提供了五个宏：`trace`、`debug`、`info`、`warn` 和 `error`。

它们的作用相同——发出一条日志记录——但正如其名称所暗示的那样，它们各自使用不同的日志级别。

trace 是最低级别: trace 级别的日志通常非常冗长，信噪比较低（例如，每次 Web 服务器收到 TCP 数据包时都会发出一条 trace 级别的日志记录）。

然后，我们依次按严重程度递增，依次为 debug、info、warn 和 error。

Error 级别的日志用于报告可能对用户造成影响的严重故障（例如，我们未能处理传入的请求或数据库查询超时）。

让我们看一个简单的使用示例：

```rs
fn fallible_operation() -> Result<String, String> { ... }

pub fn main() {
    match fallible_operation() {
        Ok(success) => {
            log::info!("Operation succeeded: {}", success);
        }
        Err(err) => {
            log::error!("Operation failed: {}", err);
        }
    }
}
```

我们正在尝试执行一个可能会失败的操作。

如果成功，我们将发出一条信息级别的日志记录。

如果失败，我们将发出一条错误级别的日志记录。

另请注意，log 的宏支持与标准库中 println/print 相同的插值语法。

我们可以使用 log 的宏来检测我们的代码库。

选择记录关于特定函数执行的信息通常是一个本地决策：只需查看函数本身即可决定哪些信息值得在日志记录中捕获。

这使得库能够被有效地检测，将遥测的范围扩展到我们亲自编写的代码之外。

### actix-web 的日志中间件

TODO: WIP
