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

## 实现 Arbitrary Trait

让我们回到前面的例子——quickcheck 是如何知道生成 `Vec<u32>` 的？一切都建立在 `quickcheck` 的 [`Arbitrary`](https://docs.rs/quickcheck/latest/quickcheck/trait.Arbitrary.html) trait之上:

```rs
pub trait Arbitrary: Clone + 'static {
    // Required method
    fn arbitrary(g: &mut Gen) -> Self;

    // Provided method
    fn shrink(&self) -> Box<dyn Iterator<Item = Self>> { ... }
}
```

我们有两个方法：

- arbitrary：给定一个随机源 (g)，返回该类型的一个实例；
- shrink：返回一个逐渐“缩小”的该类型实例序列，以帮助 quickcheck

找到最小的可能失败情况。

`Vec<u32>` 实现了 Arbitrary 接口，因此 quickcheck 知道如何生成随机的 u32 向量。

我们需要创建自己的类型，我们称之为 ValidEmailFixture，并为其实现 Arbitrary 接口。

如果你查看 `Arbitrary` 的特征定义，你会注意到 shrinking 是可选的：有一个默认的实现（使用 `empty_shrinker`），它会导致 `quickcheck` 输出遇到的第一个失败，而不会尝试使其变得更小或更优。因此，我们只需要为我们的 `ValidEmailFixture` 提供一个 `Arbitrary::arbitrary` 的实现。

让我们将 `quickcheck` 和 `quickcheck-macros` 都添加为开发依赖项:

```shell
cargo add quickcheck quickcheck-macros --dev
```

然后

```rs
//! src/domain/subscriber_email.rs
// [...]
#[cfg(test)]
mod tests {
    use claim::assert_err;
    use fake::locales::{self, Data};

    use crate::domain::SubscriberEmail;

    // Both `Clone` and `Debug are required by `quickcheck`
    #[derive(Debug, Clone)]
    struct ValidEmailFixture(pub String);

    impl quickcheck::Arbitrary for ValidEmailFixture {
        fn arbitrary(g: &mut quickcheck::Gen) -> Self {
            let username = g
                .choose(locales::EN::NAME_FIRST_NAME)
                .unwrap()
                .to_lowercase();
            let domain = g.choose(&["com", "net", "org"]).unwrap();
            let email = format!("{username}@example.{domain}");
            Self(email)
        }
    }

    // ...
}
```

这是一个令人惊叹的例子，它展现了通过在 Rust 生态系统中共享关键特性而获得的互操作性。

我们如何让 fake 和 quickcheck 完美地协同工作?

在 Arbitrary::arbitrary 中，我们以 g 作为输入，它是一个 G 类型的参数。

G 受特性边界 G: quickcheck::Gen 的约束，因此它必须实现 quickcheck 中的 Gen 特性，

其中 Gen 代表“生成器”。

注: 等等...你注意到了吗? 在新的 quickcheck crate 中 Gen不再是trait, 而是一个 struct, 前面这一段代码是我自己(译者)的解决方案, 并非原作, 解决方案来自 [这个 GitHub issue](https://github.com/BurntSushi/quickcheck/issues/320)

两个 crate 的维护者可能彼此了解，也可能不了解，但 rand-core 中一套社区认可的 trait 为我们提供了轻松的互操作性。太棒了!

现在您可以运行 cargo test domain 了——结果应该会是绿色，这再次证明了我们的邮箱验证机制
并没有过于死板。

如果您想查看生成的随机输入，请在测试中添加 `dbg!(&valid_email.0);`

语句，然后运行 `​​cargo test valid_emails -- --nocapture` ——数十个有效的邮箱地址
应该会弹出到您的终端中!
