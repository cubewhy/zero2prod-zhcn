# 内部开发循环

在项目开发过程中，我们会反复执行相同的步骤：

- 进行修改
- 编译应用程序
- 运行测试
- 运行应用程序

这也称为内部开发循环。

内部开发循环的速度是单位时间内可以完成的迭代次数的上限。

如果编译和运行应用程序需要 5 分钟，那么每小时最多可以完成 12 次迭代。如果将其缩短到 2 分钟，那么每小时就可以完成 30 次迭代！

Rust 在这方面帮不上忙——编译速度可能会成为大型项目的痛点。在继续下一步之前，让我们看看可以采取哪些措施来缓解这个问题。

## 让链接 (Linking) 更快点

在考察内部开发循环时，我们主要关注增量编译的性能——在对源代码进行微小更改后,Cargo 需要多长时间来构建二进制文件。

[链接阶段](https://en.wikipedia.org/wiki/Linker_(computing)) 会花费相当多的时间——根据早期编译阶段的输出来组装实际的二进制文件。

默认链接器的性能不错，但根据您使用的操作系统，还有其他更快的替代方案：

- Windows 和Linux上的 **lld**, LLVM 项目开发的链接器
- MacOS上的 **zld**

为了加快链接阶段，您必须在计算机上安装备用链接器，并将此配置文件添加到项目中：(别忘了安装链接器!)

```toml
# .cargo/config.toml
# On Windows
# ```
# cargo install -f cargo-binutils
# rustup component add llvm-tools-preview
# ```
[target.x86_64-pc-windows-msvc]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
[target.x86_64-pc-windows-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]
# On Linux:
# - Ubuntu, `sudo apt-get install lld clang`
# - Arch, `sudo pacman -S lld clang`
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "linker=clang", "-C", "link-arg=-fuse-ld=lld"]
# On MacOS, `brew install michaeleisel/zld/zld`
[target.x86_64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]
[target.aarch64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]
```

Rust 编译器目前正在努力尽可能使用 lld 作为默认链接器——很快，这种自定义配置将不再是实现更高编译性能的必需品!

## cargo-watch

我们还可以通过减少感知编译时间(即您花在终端上等待 cargo check 或 cargo run 完成的时间)来减轻对生产力的影响。

有工具可以帮我们! 我们可以用这个命令来安装 `cargo-watch`

```shell
cargo install cargo-watch
```

`cargo-watch` 会监控你的源代码，并在文件每次更改时自动执行命令。例如：

```shell
cargo watch -x check
```

这个命令将在每次源代码发生变化的时候自动运行 `cargo check`

这会减少你的编译时间

- 您仍在 IDE 中，重新阅读刚刚所做的代码更改
- 与此同时，cargo-watch 已经启动了编译过程
- 切换到终端后，编译器已经运行了一半

cargo-watch 也支持命令链:

```shell
cargo watch -x check -x test -x run
```

- 它会先运行 `cargo check`
- 如果成功，它会启动 `cargo test`
- 如果测试通过，它会使用 `cargo run` 启动应用程序。

我们的内部开发循环，就在这里！

## 持续集成

工具链已安装。
项目框架已完成。
IDE 已准备就绪。

在我们深入了解构建细节之前，最后要考虑的是我们的 **持续集成 (CI) 流水线**

在基于主干的开发中，我们应该能够在任何时间点部署主分支。

团队的每个成员都可以从主分支分支开发一个小功能或修复一个错误，然后合并回主分支并发布给用户。

> 持续集成使团队的每个成员能够每天多次将他们的更改集成到主分支中。

这会产生强大的连锁反应。

有些是显而易见且易于察觉的：它减少了由于长期分支而不得不处理混乱的合并冲突的机会。没有人喜欢合并冲突。

有些则更为微妙：**持续集成可以缩短反馈循环。** 你不太可能独自开发几天或几周，却发现你选择的方法并未得到团队其他成员的认可，或者无法与项目的其他部分很好地集成。

它迫使你尽早与同事沟通，并在必要时在仍然容易进行(并且不会冒犯任何人)的情况下进行纠正。

我们怎么让这些事情变得可能呢?

我们的 CI 流水线会在每次提交时运行一系列自动化检查。

如果其中一项检查失败，您将无法合并到主代码库 - 就这么简单。

CI 流水线通常不仅仅是确保代码健康：它们还是执行一系列其他重要检查的理想场所 - 例如，扫描依赖关系树以查找已知漏洞、进行 linting、格式化等等。

我们将逐一介绍您可能希望在 Rust 项目的 CI 流水线中运行的各种检查，并逐步介绍相关工具。
然后，我们将为一些主要的 CI 提供商提供一套现成的 CI 流水线。

## CI 步骤

### 测试

如果您的 CI 流水线只有一个步骤，那么它应该是测试。

测试是 Rust 生态系统中的一流概念，您可以利用 cargo 来运行单元测试和集成测试：

```shell
cargo test
```

`cargo test` 还会在运行测试之前构建项目，因此您无需
事先运行 `cargo build` (尽管大多数流水线会在运行测试之前调用 `cargo build` 来缓存依赖项)。

## 代码覆盖率

关于测量代码覆盖率的利弊，已经有很多文章进行了探讨。

虽然使用[代码覆盖率作为质量检查有一些缺点](http://www.exampler.com/testing-com/writings/coverage.pdf)，但我确实认为它是一种快速收集信息并发现代码库中某些部分是否长期被忽视以及
测试是否不足的方法。
测量 Rust 项目代码覆盖率最简单的方法是使用 cargo tarpaulin，这是 [xd009642](https://github.com/xd009642) 开发的 cargo 子命令。您可以使用以下命令安装 [tarpaulin](https://github.com/xd009642/tarpaulin)

```shell
# At the time of writing tarpaulin only supports
# x86_64 CPU architectures running Linux.
cargo install cargo-tarpaulin
```

当运行如下命令的时候, 将计算应用程序代码的代码覆盖率，忽略测试函数。
tarpaulin 可用于将代码覆盖率指标上传到 [Codecov](https://codecov.io/) 或 [Coveralls](https://coveralls.io/) 等热门服务。

请参阅 tarpaulin 的 [README](https://github.com/xd009642/tarpaulin#travis-ci-and-coverage-sites) 文件，了解如何上传代码覆盖率指标。

```shell
cargo tarpaulin --ignore-tests
```

## 代码检查 (Linting)

用任何编程语言编写符合风格的代码都需要时间和练习。

在学习初期，很容易遇到一些可以用更简单方法解决的问题，最终却得到相当复杂的解决方案。

静态分析可以提供帮助：就像编译器单步执行代码以确保其符合语言规则和约束一样，**linter** 会尝试识别不符合风格的代码、过于复杂的结构以及常见的错误/低效之处。

Rust 团队维护着 [Clippy](https://github.com/rust-lang/rust-clippy), 官方的 Rust Linter

如果您使用默认配置文件，clippy 会包含在 rustup 安装的组件集中。

CI 环境通常使用 rustup 的最小配置文件，其中不包含 clippy。

但是您可以使用以下命令来安装它

```shell
rustup component add clippy
```

如果你已经安装过 clippy 了, 执行这个命令什么也不会发生

你可以执行如下的命令来为你的项目运行 clippy

```shell
cargo clippy
```

在我们的 CI 管道中，如果 clippy 发出任何警告，我们希望 Linter 检查失败。

我们可以通过以下方式实现：

```shell
cargo clippy -- -D warnings
```

静态分析并非万无一失：有时 clippy 可能会建议一些你认为不正确或不理想的更改。

你可以在受影响的代码块上使用 `#[allow(clippy::lint_name)]` 属性来关闭特定的警告，

或者在 clippy.toml 中使用一行配置语句，为整个项目禁用干扰性的 lint 检查。

或使用项目级 `#![allow(clippy::lint_name)]` 宏。

有关可用的 lint 以及如何根据您的特定目的进行调整的详细信息，请参阅 [clippy 的 README 文件](https://github.com/rust-lang/rust-clippy#configuration)。

## 代码格式化

大多数组织对主分支都有不止一道防线：

- 一道是 **持续集成 (CI) 流水线检查**
- 另一道通常是*拉取请求 (PR) 审查**

关于有价值的 PR 审查流程与枯燥乏味 PR 审查流程的区别，有很多说法——无需在此重新展开争论。

我确信好的 PR 审查不应该关注以下几点：格式上的小瑕疵——例如，"你能在这里加个换行符吗?"、"我觉得那里尾部有个空格!" 等等。

让机器处理格式，而审查人员则专注于架构、测试的完整性、可靠性和可观察性。自动格式化可以消除 PR 审查流程中复杂的干扰。你可能不喜欢这样或那样的格式选择，但彻底消除格式上的繁琐，
值得承受些许不适。

[`rustfmt`](https://github.com/rust-lang/rustfmt) 是 Rust 官方的格式化程序。

与 clippy 一样，rustfmt 也包含在 rustup 安装的默认组件中

您可以使用以下命令安装它

```shell
rustup component add rustfmt
```

你可以使用如下命令来为你的整个项目格式化代码

```shell
cargo fmt
```

在我们的 CI 流中，我们将添加格式化步骤

```shell
cargo fmt -- --check
```

如果提交包含未格式化的代码，它将失败，并将差异打印到控制台。

您可以使用配置文件 rustfmt.toml 针对项目调整 rustfmt。详细信息请参阅 [rustfmt 的 README 文件](https://github.com/rust-lang/rustfmt#configuring-rustfmt)。

## 安全漏洞

cargo 使得利用生态系统中现有的 crate 来解决当前问题变得非常容易。

另一方面，每个 crate 都可能隐藏可利用的漏洞，从而危及软件的安全状况。

[Rust 安全代码工作组 (Rust Secure Code working group)](https://github.com/RustSec) 维护着一个 [数据库](https://github.com/RustSec/advisory-db)，其中包含 crates.io 上发布的 crate 的最新漏洞报告。

他们还提供了 `cargo-audit`，这是一个便捷的 cargo 子命令，用于检查项目依赖关系树中是否有任何 crate 存在漏洞。

您可以使用以下命令安装它

```shell
cargo install cargo-audit
```

当你安装之后, 你可以执行如下命令来扫描你的依赖树

```shell
cargo audit
```

我们将在每次提交时运行 cargo-audit，作为 CI 流水线的一部分。

我们还将每天运行它，以随时掌握那些我们目前可能没有积极开发但仍在生产环境中运行的项目依赖项的新漏洞！

## 开箱即用的 CI 流

> 授人以鱼，不如授人以渔

希望我教给你的知识足以让你为你的 Rust 项目构建一个可靠的 CI 流水线。

我们也应该坦诚地承认，学习如何使用 CI 提供商所使用的特定配置语言可能需要花费数小时的反复尝试，而且调试过程通常非常痛苦，反馈周期也很长。

因此，我决定编写一套适用于最常用现成配置文件，
—— 就是我们刚才描述的那些步骤，可以直接放到你的项目仓库中：

- [GitHub Actions](https://gist.github.com/LukeMathWalker/5ae1107432ce283310c3e601fac915f3)
- [CircleCI](https://gist.github.com/LukeMathWalker/6153b07c4528ca1db416f24b09038fca)
- [GitLab CI](https://gist.github.com/LukeMathWalker/d98fa8d0fc5394b347adf734ef0e85ec)
- [Travis](https://gist.github.com/LukeMathWalker/41c57a57a61c75cc8a9d137a8d41ec10)

调整现有设置以满足您的特定需求通常比从头开始编写新设置要容易得多
