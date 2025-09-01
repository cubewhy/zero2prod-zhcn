# 错误处理

为了发送确认邮件，我们必须将多个操作拼凑在一起：验证用户输入、发送电子邮件、各种数据库查询。

它们都有一个共同点：它们可能会失败。

在第六章中，我们讨论了 Rust 中错误处理的构建块——Result 和 ? 运算符。

我们留下了许多未解答的问题: 错误如何融入我们应用程序的更广泛架构中? 一个好的错误是什么样的? 错误是为谁设计的？我们应该使用库吗? 哪一个?

本章将重点深入分析 Rust 中的错误处理模式。

- [错误的目的是什么](cp8.1-what-is-the-purpose-of-errors.md)
- [操作员错误报告](cp8.2-error-reporting-for-operators.md)
- [控制流错误](cp8.3-errors-for-control-flow.md)
- [避免 "泥球" 错误枚举](cp8.4-avoid-ball-of-mud-error-enums.md)
- [谁应该记录错误](cp8.5-who-should-log-errors.md)
- [小结](cp8.6-summary.md)
