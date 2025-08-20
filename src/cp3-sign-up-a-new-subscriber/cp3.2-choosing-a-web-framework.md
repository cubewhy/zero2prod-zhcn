# 选择一个Web框架

我们应该使用哪个 Web 框架来编写 Rust API？

这部分原本应该讨论当前可用的 Rust Web 框架的优缺点。

但最终篇幅过长，实在没必要在这里写，所以我把它作为一篇衍生文章发布：请参阅 [《选择 Rust Web 框架，2020 版》 (英语)](https://www.lpalmieri.com/posts/2020-07-04-choosing-a-rust-web-framework-2020-edition/)，深入了解 [actix-web](https://actix.rs/)、[rocket](https://rocket.rs/)、[tide](https://github.com/http-rs/tide) 和 [warp](https://docs.rs/warp/latest/warp/)

简而言之：截至 2022 年 3 月，actix-web 应该是您在生产环境中使用 Rust API 时的首选 Web 框架——它在过去几年中得到了广泛的使用，拥有庞大而健康的社区，并且运行在 tokio 上，因此最大限度地减少了处理不同异步运行时之间不兼容/互操作的可能性。

因此，它将成为我们从零到生产环境的首选。

尽管如此，tide、rocket和wrap都拥有巨大的潜力，我们最终可能会在2022年晚些时候做出不同的决定——如果你正在使用不同的框架进行“从零到生产”的实践，我很乐意看看你的代码！请给原作者发邮件至 [contact@lpalmieri.com](mailto:contact@lpalmieri.com)

在本章及以后的内容中，我建议您打开几个额外的浏览器标签页：[actix-web 的网站](https://actix.rs/)、[actix-web 的文档](https://docs.rs/actix-web/latest/actix_web/index.html)和 [actix-web 的示例集](https://github.com/actix/examples)。
