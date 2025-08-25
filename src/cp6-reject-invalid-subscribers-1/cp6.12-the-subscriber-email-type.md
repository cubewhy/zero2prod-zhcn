# SubscriberEmail 类型

我们将遵循与名称验证相同的策略——将不变量（“此字符串代表有效的电子邮件”）编码到新的 `SubscriberEmail` 类型中。

## 破坏 domain 子模块

不过，在开始之前，我们先腾出一些空间——将域子模块 (domain.rs) 拆分成多个较小的文件，每个类型一个，类似于我们在第三章中对路由所做的操作。

我们当前的文件夹结构（在 src 下）如下:

```plaintext
src/
  routes/
    [...]
  domain.rs
    [...]
```

我们想要

```plaintext
src/
  routes/
    [...]
  domain/
    subscriber_name.rs
    subscriber_email.rs
    new_subscriber.rs
  domain.rs
  [...]
```

单元测试应该位于与其引用的类型相同的文件中。最终结果如下:

```rs
//! src/domain.rs
cmod subscriber_name;
mod subscriber_email;
mod new_subscriber;

pub use subscriber_name::SubscriberName;
pub use new_subscriber::NewSubscriber;
```

```rs
//! src/domain/subscriber_name.rs
use unicode_segmentation::UnicodeSegmentation;

#[derive(Debug)]
pub struct SubscriberName(String);

impl SubscriberName {
    // [...]
}

impl AsRef<str> for SubscriberName {
    // [...]
}

#[cfg(test)]
mod tests {
    // [...]
}
```

```rs
//! src/domain/subscriber_email.rs

// Still empty, ready for us to get started!
```

```rs
//! src/domain/new_subscriber.rs
use crate::domain::SubscriberName;

pub struct NewSubscriber {
    pub email: String,
    pub name: SubscriberName,
}
```

我们项目中的其他文件无需进行任何更改——由于 domain.rs 中的 `pub use` 语句，我们模块的 API 并未改变。

## 新类型骨架

让我们添加一个简单的 `SubscriberEmail` 类型: 没有验证，只是一个 String 的包装器和一个方便的 AsRef 实现:

```rs
//! src/domain/subscriber_email.rs
#[derive(Debug)]
pub struct SubscriberEmail(String);

impl SubscriberEmail {
    pub fn parse(s: String) -> Result<Self, String> {
        // TODO: add validation!
        Ok(Self(s))
    }
}

impl AsRef<str> for SubscriberEmail {
    fn as_ref(&self) -> &str {
        &self.0
    }
}
```

```rs
//! src/domain.rs
// [...]
pub use subscriber_email::SubscriberEmail;
```

这次我们从测试开始: 我们先举几个应该被拒绝的无效邮件的例子。

```rs
//! src/domain/subscriber_email.rs
#[derive(Debug)]
pub struct SubscriberEmail(String);

#[cfg(test)]
mod tests {
    use claim::assert_err;

    use crate::domain::SubscriberEmail;

    #[test]
    fn empty_string_is_rejected() {
        let email = "".to_string();
        assert_err!(SubscriberEmail::parse(email));
    }

    #[test]
    fn email_missing_at_symbol_is_rejected() {
        let email = "example.com".to_string();
        assert_err!(SubscriberEmail::parse(email));
    }

    #[test]
    fn email_missing_subject_is_rejected() {
        let email = "@example.com".to_string();
        assert_err!(SubscriberEmail::parse(email));
    }
}
```

不出意料的, `cargo test` 全部失败

是时候加入 `validator` 了

```shell
cargo add validator
```

我们的解析方法将把所有繁重的工作委托给[`validator::validate_email`](https://docs.rs/validator/latest/validator/?search=validate_email):

```rs
//! src/domain/subscriber_emial.rs
impl SubscriberEmail {
    pub fn parse(s: String) -> Result<Self, String> {
        if validator::ValidateEmail::validate_email(&s) {
            Ok(Self(s))
        } else {
            Err(format!("{s} is not a valid subscriber email."))
        }
    }
}

// [...]
```

就这么简单——现在我们所有的测试都通过了!
需要注意的是——我们所有的测试用例都在检查无效的邮箱地址。我们还应该至少进行一次测试，检查有效的邮箱地址是否通过。

我们可以在测试中硬编码一个已知的有效邮箱地址，并检查它是否被成功解析——例如，`ursula@domain.com`。

但是，从这个测试用例中我们能得到什么值呢? 它只能再次确认某个特定的邮箱地址被正确解析为有效地址。
