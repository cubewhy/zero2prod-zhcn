# 可维护测试套件的框架和原则

我们花了不少功夫，现在终于为 Postmark 的 API 打造了一个相当不错的 REST 客户端。

`EmailClient` 只是我们确认邮件流程的第一个组成部分：我们还没有找到一种方法来生成唯一的确认链接，然后我们必须将其嵌入到发出的确认邮件的正文中。

这两项任务都需要再等一段时间。

我们一直采用测试驱动的方法来编写本书中的所有新功能。

虽然这种策略效果很好，但我们并没有投入大量时间重构测试代码。

因此，我们的测试文件夹目前有点混乱。

在继续下一步之前，我们将重构集成测试套件，以应对应用程序复杂性和测试数量的增长。

## 我们为什么要写测试

编写测试是否合理利用了开发人员的时间?

好的测试套件首先是一种风险缓解措施。

自动化测试可以降低现有代码库变更带来的风险——大多数回归测试和错误都会被捕获在持续集成流水线中，而不会影响到用户。因此，团队能够更快地迭代并更频繁地发布。

测试也充当文档的作用。

深入研究未知代码库时，测试套件通常是最佳起点——它向您展示了代码的预期行为，以及哪些场景被认为足够相关，需要进行专门的测试。

如果您想让您的项目更受新贡献者的欢迎，“编写测试套件！”绝对应该列在您的待办事项清单上。

好的测试通常还会带来其他积极的副作用——模块化和解耦。这些副作用很难量化，因为作为一个行业，我们尚未就“好代码”的具体样貌达成一致。

## 为什么我们不写测试

尽管投入时间和精力编写优秀的测试套件的理由令人信服，但现实情况却并非如此。

首先，开发社区并非一直相信测试的价值。

在整个测试驱动开发学科的发展史上，我们都能找到不少例子，但直到1999年《极限编程》(XP) 一书的出版，这项实践才进入主流讨论!

范式转变并非一朝一夕之功——测试驱动方法花了数年时间才在行业内获得认可，成为“最佳实践”。

如果说测试驱动开发已经赢得了开发人员的青睐，那么与管理层的斗争往往仍在继续。

好的测试能够构建技术杠杆，但编写测试需要时间。当截止日期迫在眉睫时，测试往往是第一个被牺牲的。

因此，你找到的大多数资料要么是测试的介绍，要么是如何向利益相关者推介其价值的指南。

关于大规模测试的内容很少——如果你坚持照本宣科，继续编写测试，当代码库增长到数万行，包含数百个测试用例时，会发生什么?

## 测试代码仍然是代码

所有测试套件都以同样的方式开始：一个空文件，一个充满可能性的世界。

你进入文件，添加第一个测试。很简单，搞定。

然后是第二个。轰！

第三个。你只需要从第一个代码中复制几行，就完成了。

第四个……

一段时间后，测试覆盖率开始下降：新代码的测试不如你在项目初期编写的代码那么彻底。你是否开始怀疑测试的价值了?

绝对没有，测试很棒!

然而，随着项目的推进，你编写的测试越来越少。

这是因为摩擦——随着代码库的演变，编写新的测试变得越来越繁琐。

> 测试代码仍然是代码

它必须是模块化的、结构良好的、文档齐全的。它需要维护。

如果我们不积极投入精力维护测试套件的健康状况，它就会随着时间的推移而腐烂。

覆盖率下降，很快我们就会发现应用程序代码中的关键路径从未被自动化测试执行过。

你需要定期回顾一下你的测试套件，从整体上审视它。

是时候审视一下我们的测试套件了，不是吗?

## 我们的测试套件

我们所有的集成测试都位于一个文件 `tests/health_check.rs` 中:

```rs
//! tests/health_check.rs
// [...]
//! tests/health_check.rs

// Ensure that the `tracing` stack is only initialised once using `once_cell`
static TRACING: Lazy<()> = Lazy::new(|| {
    // [...]
});

pub struct TestApp {
    pub address: String,
    pub db_pool: PgPool,
}

async fn spawn_app() -> TestApp {
    // [...]
}

pub async fn configure_database(config: &DatabaseSettings) -> PgPool {
    // [...]
}

#[tokio::test]
async fn health_check_works() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_200_for_valid_form_data() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_400_when_data_is_missing() {
    // [...]
}

#[tokio::test]
async fn subscribe_returns_a_200_when_fields_are_present_but_invalid() {
    // [...]
}
```

## 测试发现

只有一个测试用于处理我们的健康检查端点 - health_check_works。

其他三个测试用于探测我们的 `POST /subscriptions` 端点，而其余代码则用于处理共享的设置步骤（`spawn_app`、`TestApp`、`configure_database` 和 `TRACING`）。

为什么我们将所有内容都放在 `tests/health_check.rs` 中?

因为这样很方便!

设置函数已经存在 - 在同一个文件中添加另一个测试用例比弄清楚如何在多个测试模块之间正确共享代码更容易。

我们此次重构的主要目标是可发现性:

- 给定一个应用程序端点，应该能够在 `tests` 文件夹中轻松找到相应的集成测试
- 编写测试时，应该能够轻松找到相关的测试辅助函数

我们将重点介绍文件夹结构，但这绝对不是测试发现的唯一工具。

测试覆盖率工具通常可以告诉您哪些测试触发了特定应用程序代码行的执行。

您可以依靠覆盖率标记等技术在测试和应用程序代码之间建立明显的联系。

与往常一样，随着测试套件复杂度的增加，多管齐下的方法可能会为您带来最佳效果。

## 一个测试文件, 一个 Crate

在开始移动之前，让我们先了解一些关于 Rust 集成测试的知识。

`tests` 文件夹比较特殊——Cargo 知道去里面查找集成测试。

`tests` 文件夹下的每个文件都会被编译成一个独立的 crate。

我们可以通过运行 `cargo build --tests` 命令，然后在 target/debug/deps 目录下查找来验证这一点:

```shell
# Build test code, without running tests
cargo build --tests
# Find all files with a name starting with `health_check`
ls target/debug/deps | grep health_check
```

```plaintext
health_check-3b9db97bf61f8d77
health_check-3b9db97bf61f8d77.d
health_check-e064e6b2bbe87e3b.d
libhealth_check-e064e6b2bbe87e3b.rmeta
```

您的机器上的尾部哈希值可能会有所不同，但应该有两个以 `health_check-*` 开头的条目。

如果您尝试运行它会发生什么?

```plaintext
running 4 tests
test health_check_works ... ok
test subscribe_returns_a_400_when_fields_are_present_but_invalid ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok
test subscribe_returns_a_200_for_valid_form_data ... ok
test result: ok. 4 passed; finished in 0.44s
```

没错，它运行我们的集成测试!

如果我们测试了五个 `*.rs` 文件，我们会在 `target/debug/deps` 中找到五个可执行文件。

## 共享测试 Helper 方法

如果每个集成测试文件都是独立的可执行文件，那么我们如何共享测试辅助函数呢?

第一个选项是定义一个独立的模块，例如 `tests/helpers/mod.rs` 。

您可以在 `mod.rs` 中添加常用函数（或在其中定义其他子模块），然后在测试文件 (例如 `tests/health_check.rs`) 中引用这些辅助函数，如下所示:

```rs
//! tests/health_check.rs
// [...]
mod helpers;

// [...]
```

`helpers` 被捆绑在 `health_check` 测试可执行文件中，并作为子模块，我们可以访问它在测试用例中公开的函数。
这种方法一开始效果不错，但最终会导致恼人的“函数从未使用”警告。

问题在于, `helpers` 被捆绑为子模块，而不是作为第三方 crate 调用: cargo 会单独编译每个测试可执行文件，如果某个测试文件中的 `helpers` 中的一个或多个公共函数从未被调用，则会发出警告。随着测试套件的增长，这种情况必然会发生——并非所有测试文件都会使用所有辅助方法。

第二种方案充分利用了 tests 下每个文件都是独立可执行文件这一特点——我们可以创建作用域为单个测试可执行文件的子模块！

让我们在 `tests` 下创建一个 `api` 文件夹，其中包含一个 `main.rs` 文件:

```plaintext
tests/
  api/
    main.rs
  health_check.rs
```

首先，我们要明确一点：我们构建 API 的方式与构建二进制 crate 的方式完全相同。它没有那么多魔法——它建立在你编写应用程序代码时构建的模块系统知识之上。

如果你运行 `cargo build --tests`，你应该能够发现

```plaintext
Running target/debug/deps/api-0a1bfb817843fdcf

running 0 tests

test result: ok. 0 passed; finished in 0.00s
```

在输出中 - cargo 将 api 编译为测试可执行文件，并查找测试用例。

无需在 `main.rs` 中定义 `main` 函数 - Rust 测试框架会在后台为我们添加一个。

我们现在可以在 `main.rs` 中添加子模块:

```rs
//! tests/api/main.rs
mod helpers;
mod health_check;
mod subscriptions;
```

添加三个空文件: `tests/api/helpers.rs`、`tests/api/health_check.rs` 和 `tests/api/subscriptions.rs`。
现在是时候删除 `tests/health_check.rs` 并重新分发其内容了:

```rs
//! tests/api/helpers.rs
// TODO: wip
```

TODO: wip
