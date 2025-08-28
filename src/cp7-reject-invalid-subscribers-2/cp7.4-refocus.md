# 短暂回顾

现在是时候回到我们在本章开头起草的计划了;

- 编写一个模块来发送电子邮件
- 调整现有 POST /subscriptions 请求处理程序的逻辑以适应新的需求
- 从头编写一个 GET /subscriptions/confirm 请求处理程序

第一项已完成，接下来该处理剩下的两项了。

我们之前已经画好了这两个处理程序的工作原理图:

POST /subscriptions 将:

- • 将订阅者详细信息添加到数据库的 subscriptions 表中，状态等于 pending_confirmation
- • 生成一个（唯一的）subscription_token
- • 将 subscription_token 存储在数据库中，并与 subscription_tokens 表中的订阅者 ID 对应
- • 向新订阅者发送一封电子邮件，其中包含结构为 `https://<our-api-domain>/subscriptions/confirm?token=<subscription_token>` 的链接
- • 返回 200 OK

一旦订阅者点击该链接，浏览器标签页将打开，并向我们的 `GET /subscriptions/confirm` 端点发送 GET 请求。请求处理程序将:

- 从查询参数中检索 subscription_token
- 从 subscription_tokens 表中检索与 subscription_token 关联的订阅者 ID
- 在 subscriptions 表中将订阅者状态从 pending_confirmation 更新为 active
- 返回 200 OK

这让我们对应用程序在实现完成后的工作方式有了相当精确的了解。

但这对我们弄清楚如何实现目标并没有多大帮助。

我们应该从哪里开始?

我们应该立即处理 `/subscriptions` 的变更吗?

我们应该先处理 `/subscriptions/confirm` 吗?

我们需要找到一条可以零停机时间上线的实现路线。
