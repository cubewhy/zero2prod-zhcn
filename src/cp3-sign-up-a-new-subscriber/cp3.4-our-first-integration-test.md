# 我们的第一个集成测试

`/health_check` 是我们的第一个端点，我们通过启动应用程序并通过 curl 手动测试来验证一切是否按预期运行。

然而，手动测试非常耗时: 随着应用程序规模的扩大，每次执行更改时，手动检查我们对其行为的所有假设是否仍然有效，成本会越来越高。

我们希望尽可能地实现自动化: 这些检查应该在每次提交更改时都在我们的持续集成 (CI) 流水线中运行，以防止出现回归问题。

虽然健康检查的行为在整个过程中可能不会有太大变化，但它是正确设置测试框架的良好起点。

## 我应该怎么测试一个API Endpoint

API 是达到目的的手段：一种暴露给外界执行某种任务的工具（例如，存储文档、发布电子邮件等）。

我们在 API 中暴露的端点定义了我们与客户端之间的契约：关于系统输入和输出（即接口）的共同协议。

契约可能会随着时间的推移而演变，我们可以粗略地设想两种情况:

- 向后兼容的变更（例如，添加新的端点）
- 重大变更（例如，移除端点或从其输出的架构中删除字段）。

在第一种情况下，现有的 API 客户端将保持原样运行。在第二种情况下，如果现有的集成依赖于契约中被违反的部分，则可能会中断。

虽然我们可能会故意部署对 API 契约的重大变更，但至关重要的是，我们不要意外地破坏它。

什么是最可靠的方法来检查我们没有引入用户可见的回归？

通过与用户完全一样的方式与 API 交互来测试 API：向 API 执行 HTTP 请求，并验证我们对收到的响应的假设。

这通常被称为黑盒测试：我们通过检查系统的输出来验证系统的行为，而不需要了解其内部实现的细节。

遵循这一原则，我们不会满足于直接调用处理函数的测试——例如:

```rs
#[cfg(test)]
mod tests {
    use crate::health_check;

    #[tokio::test]
    async fn health_check_succeeds() {
        let response = health_check().await;
        // This requires changing the return type of `health_check`
        // from `impl Responder` to `HttpResponse` to compile
        // You also need to import it with `use actix_web::HttpResponse`!
        assert!(response.status().is_success())
    }
}
```

这样做并不是最佳实践

- 我们没有检查处理程序是否在 **GET** 请求时调用
- 我们也没有检查处理程序是否以 **/health_check** 作为路径调用。

更改这两个属性中的任何一个都会破坏我们的 API 契约，但我们的测试仍然会通过——
这还不够好。

actix-web 提供了一些便利，可以在不跳过路由逻辑的情况下与应用进行交互，
但这种方法存在严重的缺陷:

- 迁移到另一个 Web 框架将迫使我们重写整个集成测试套件。
我们希望集成测试尽可能与 API 实现的底层技术高度分离（例如，在进行大规模重写或重构时，进行与框架无关的集成测试可以起到至关重要的作用！）
- 由于 actix-web 的一些限制，我们无法在生产代码和测试代码之间共享应用启动逻辑，因此，由于存在随着时间的推移出现分歧的风险，我们对测试套件提供的保证的信任度会降低。

我们将选择一个完全黑盒解决方案：我们将在每次测试开始时启动我们的应用程序，并使用现成的 HTTP 客户端 (例如 [reqwest](https://docs.rs/reqwest/latest/reqwest/index.html)) 与其交互。

## 我应该把测试放在哪里

Rust 在编写测试时提供了[三种选择](https://doc.rust-lang.org/book/ch11-03-test-organization.html):

- 在嵌入式测试模块中的代码旁边 (使用 `mod tests` )

```rs
// Some code I want to test
#[cfg(test)]
mod tests {
    // Import the code I want to test
    use super::*;
    // My tests
}
```

- 在外部的 `tests/` 文件夹

```plaintext
$ ls

src/
tests/
Cargo.toml
Cargo.lock
```

- 公共文档的一部分 (doc tests)

```rs
/// Check if a number is even.
/// ```rust
/// use zero2prod::is_even;
///
/// assert!(is_even(2));
/// assert!(!is_even(1));
/// ```
pub fn is_even(x: u64) -> bool {
    x % 2 == 0
}
```

有什么区别?

嵌入式测试模块是项目的一部分，只是隐藏在[配置条件检查](https://doc.rust-lang.org/stable/rust-by-example/attribute/cfg.html) `#[cfg(test)]` 后面。而 tests 文件夹下的所有内容以及文档测试则被编译成各自独立的二进制文件。

这会影响*可见性规则*。

嵌入式测试模块对其相邻的代码拥有特权访问权：它可以与未标记为公共的结构体、方法、字段和函数进行交互，而这些内容通常无法被我们代码的用户访问，即使他们将其作为自己项目的依赖项导入也是如此。

嵌入式测试模块对于我所说的“冰山项目”非常有用，即暴露的接口非常有限（例如几个公共函数），但底层机制却非常庞大且相当复杂（例如数十个例程）。通过公开的函数来测试所有可能的边缘情况可能并非易事——您可以利用嵌入式测试模块为私有子组件编写单元测试，从而增强对整个项目正确性的整体信心。

而外部测试文件夹和文档测试对代码的访问级别，与将 crate 添加为另一个项目依赖项时获得的访问级别完全相同。因此，它们主要用于集成测试，即通过与用户完全相同的方式调用代码来测试。

我们的电子邮件简报并非库，因此两者之间的界限有点模糊——我们不会将其作为 Rust crate 公开给世界，而是将其作为可通过网络访问的 API 公开。

尽管如此，我们将使用 tests 文件夹进行 API 集成测试——它更加清晰地划分，并且将测试助手作为外部测试二进制文件的子模块进行管理也更加容易。

## 改变我们的项目结构以便于测试

在真正开始在 `/tests` 下编写第一个测试之前，我们还有一些准备工作要做。

正如我们所说，任何测试代码最终都会被编译成它自己的二进制文件——我们所有测试代码

都以 crate 的形式导入。但目前我们的项目是一个二进制文件：它旨在执行，而不是

共享。因此，我们无法像现在这样在测试中导入 main 函数。

如果您不相信我的话，我们可以做一个快速实验:

```shell
# Create the tests folder
mkdir -p tests
```

创建 `tests/health_check.rs` 然后写入如下代码

```rs
//! tests/health_check.rs
use zero2prod::main;

#[test]
fn dummy_test() {
    main()
}
```

现在执行 `cargo test`, 欸? 好像报错了

```plaintext
error[E0432]: unresolved import `zero2prod`
 --> tests/health_check.rs:1:5
  |
1 | use zero2prod::main;
  |     ^^^^^^^^^ use of unresolved module or unlinked crate `zero2prod`
  |
  = help: if you wanted to use a crate named `zero2prod`, use `cargo add zero2prod` to add it to your `Cargo.toml`

For more information about this error, try `rustc --explain E0432`.
error: could not compile `zero2prod` (test "health_check") due to 1 previous error
```

译者注: 原作此处为修改 Cargo.toml来配置lib.rs, 但在最新的Rust中, lib.rs会自动识别为一个crate, 所以不必那么做了, 这里就没翻译

接下来我们创建文件 `src/lib.rs`

然后我们可以把 `main.rs` 中的逻辑搬过去了

```rs
//! main.rs
use zero2prod::run;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    run().await
}
```

```rs
//! lib.rs
use actix_web::{App, HttpResponse, HttpServer, Responder, web};

async fn health_check() -> impl Responder {
    HttpResponse::Ok()
}

pub async fn run() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/health_check", web::get().to(health_check)))
        .bind("127.0.0.1:8000")?
        .run()
        .await
}
```

好了，我们准备编写一些有趣的集成测试!
