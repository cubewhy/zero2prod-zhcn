# 基于属性的测试

我们可以使用另一种方法来测试我们的解析逻辑：与其验证特定输入集是否被正确解析，不如构建一个随机生成器来生成有效值，并检查我们的解析器是否不会拒绝这些值。

换句话说，我们验证我们的实现是否具有某个属性——“不会拒绝任何有效的电子邮件地址”。

这种方法通常被称为基于属性的测试。

例如，如果我们使用时间，我们可以反复采样三个随机整数:

- H，介于 0 到 23 之间（含）
- M，介于 0 到 59 之间（含）
- S，介于 0 到 59 之间（含）

并验证 H:M:S 始终能够被正确解析。

基于属性的测试显著扩大了我们验证的输入范围，从而增强了我们对代码正确性的信心，但它并不能证明我们的解析器是正确的——它并没有彻底探索输入空间（除了微小的输入空间）。

让我们看看我们的 `SubscriberEmail` 的属性测试是什么样的。

## 如何使用 `fake` 生成随机测试数据

首先，我们需要一个有效邮箱地址的随机生成器。
我们可以自己写一个，但这是一个引入 [fake](https://crates.io/crates/fake) crate 的好机会。

fake 提供了原始数据类型（整数、浮点数、字符串）和高级对象（IP 地址、国家/地区代码等）的生成逻辑，特别是电子邮件！让我们将 fake 添加为我们项目的开发依赖项:

```shell
cargo add fake
```

让我们在新的测试中使用它

```rs
//! src/domain/subscriber_email.rs
#[cfg(test)]
mod tests {
    // We are importing the `SafeEmail` faker!
    // We also need the `Fake` trait to get access to the
    // `.fake` method on `SafeEmail`
    use claim::{assert_err, assert_ok};
    use fake::{faker::internet::en::SafeEmail, Fake};

    use crate::domain::SubscriberEmail;
    
    // [...]

    #[test]
    fn valid_emails_are_parsed_successfully() {
        let email = SafeEmail().fake();
        assert_ok!(SubscriberEmail::parse(email));
    }
}
```

每次运行测试套件时，SafeEmail().fake() 都会生成一个新的随机有效电子邮件地址，然后我们用它来测试我们的解析逻辑。

与硬编码的有效电子邮件地址相比，这已经是一个重大改进，但为了捕捉到边缘情况的问题，我们不得不多次运行测试套件。一个快速而粗略的解决方案是在测试中添加一个 for 循环，但同样，我们可以借此机会深入研究并探索一个围绕基于属性的测试而设计的测试crate。

## quickcheck Vs proptest

Rust 生态系统中有两种主流的基于属性的测试方案: [quickcheck](https://crates.io/crates/quickcheck) 和 [proptest](https://crates.io/crates/proptest)。
它们的领域有所重叠，但各自在各自的领域中都独树一帜——请查看它们的 README 文件，了解所有细节。

对于我们的项目，我们将使用 **quickcheck** ——它入门相当简单，而且不使用太多宏，从而带来愉悦的 IDE 体验。

## quickcheck 入门

让我们看一下其中一个例子来了解它的工作原理:

```rs
/// This function we want to test
fn reverse<T: Clone>(xs: &[T]) -> Vec<T> {
    let mut rev = vec![];
    for x in xs.iter() {
        rev.insert(0, x.clone());
    }
    rev
}

#[cfg(test)]
mod tests {
    #[quickcheck_macros::quickcheck]
    fn prop(xs: Vec<u32>) -> bool {
        /// A property that is always true, regardless
        /// of the vector we are applying the function to:
        /// reversing it twice should return the original input.
        xs == reverse(&reverse(&xs))
    }
}
```

`quickcheck` 在一个可配置迭代次数（默认为 100）的循环中调用 prop: 每次迭代时，它会生成一个新的 `Vec<u32>` 并检查 prop 是否返回 `true`。

如果 prop 返回 `false`，它会尝试将生成的输入压缩为尽可能小的失败样本（最短的失败向量），以帮助我们调试哪里出了问题。

在我们的例子中，我们希望实现如下代码:

```rs
#[quickcheck_macros::quickcheck]
fn valid_emails_are_parsed_successfully(valid_email: String) -> bool {
    SubscriberEmail::parse(valid_email).is_ok()
}
```

不幸的是，如果我们要求输入字符串类型，我们将会得到各种各样的垃圾数据，从而导致验证失败。

我们如何定制生成程序?
