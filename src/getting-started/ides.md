# 选择一个集成开发环境  (IDE)

项目框架已准备就绪，现在是时候启动你最喜欢的编辑器，开始摆弄它了。

每个人的偏好各不相同，但我认为，你至少应该拥有一个支持语法高亮、代码导航和代码补全的设置，尤其是在你刚开始学习一门新的编程语言时。

语法高亮可以立即反馈明显的语法错误，而代码导航和代码补全则支持“探索性”编程：你可以快速跳转至依赖项的源代码，快速访问从 crate 导入的结构体或枚举的可用方法，而无需在编辑器和 docs.rs 之间不断切换。

IDE 设置主要有两个选项：rust-analyzer 和 IntelliJ Rust (现在叫做RustRover)

> 译者注: 当然我本人喜欢用Neovim + rust-analyzer, VS Code看起来也很不错的说

## Rust-analyzer

[rust-analyzer](https://rust-analyzer.github.io/) 是 Rust LSP ([Language Server Protocol](https://microsoft.github.io/language-server-protocol/)) 的一个实现。

LSP 使得在各种编辑器中轻松使用 rust-analyzer 变得容易

这意味着你可以在 VS Code、Emacs、Vim/NeoVim 和 Sublime Text 3获得相同的Rust开发体验

特定编辑器的设置说明可在[此处](https://rust-analyzer.github.io/book/)找到

## IntelliJ Rust/RustRover

IntelliJ Rust 为 JetBrains 开发的编辑器套件提供 Rust 支持。

如果您没有 JetBrains 许可证，可以免费使用支持 In从telliJ Rust 的 IntelliJ IDEA 社区版本

如果您拥有 JetBrains 许可证，那么 CLion 是您首选的 Rust 编辑器。 (译者注: 现在用[RustRover](https://www.jetbrains.com/rust/)的话会更合适的)

## 我该用哪个IDE

> 译者注: 本段疑似过时, 现在 rust-analyzer 看起来很稳定, 如果你介意专有软件的话, 建议使用 rust-analyzer + 任何一个支持LSP的编辑器

截至 2022 年 3 月，IntelliJ Rust 应为首选。

尽管 rust-analyzer 前景光明，并且在过去一年中取得了令人难以置信的进步，但它距离提供与 IntelliJ Rust 目前提供的 IDE 体验相当的体验还相去甚远。

另一方面，IntelliJ Rust 会强制您使用 JetBrains 的 IDE，而您可能愿意，也可能不愿意。如果您想继续使用您选择的编辑器，请寻找其 rust-analyzer 集成/插件。

值得一提的是，rust-analyzer 是 Rust 编译器内部正在进行的更大规模库化工作的一部分：rust-analyzer 和 rustc 之间存在重叠，存在大量重复工作。

将编译器的代码库演化为一组可重用的模块，将使 rust-analyzer 能够利用编译器代码库中越来越大的子集，从而释放提供一流 IDE 体验所需的按需分析功能。

这是一个值得未来关注的有趣空间
