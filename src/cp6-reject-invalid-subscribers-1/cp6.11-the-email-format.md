# 电子邮件格式

我们直观地熟悉电子邮件地址的常见结构 - `XXX@YYY.ZZZ` - 但如果你想严谨起见，避免退回实际上有效的电子邮件地址，这个问题很快就会变得更加复杂。

我们如何确定一个电子邮件地址是否“有效”?

互联网工程任务组 (IETF) 发布了几个征求意见稿 (RFC)，概述了电子邮件地址的预期结构 - tools.ietf.org/html/rfc6854、[RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322) 和 [RFC 2822](https://tools.ietf.org/html/rfc2822)。我们必须阅读这些文件，消化其中的内容，然后编写一个符合规范的 `is_valid_email` 函数。

除非你对理解电子邮件地址格式的细微差别有着浓厚的兴趣，否则我建议你先退一步：它相当混乱。混乱到甚至连 [HTML 规范](https://html.spec.whatwg.org/multipage/input.html#valid-e-mail-address)
都故意与我们刚刚链接的 RFC [不兼容](https://html.spec.whatwg.org/multipage/input.html#valid-e-mail-address)。

我们最好的办法是寻找一个长期深入研究过这个问题的现有库，
以便为我们提供即插即用的解决方案。幸运的是, Rust 生态系统中至少有一个这样的库 —— [validator crate](https://crates.io/crates/validator)！
