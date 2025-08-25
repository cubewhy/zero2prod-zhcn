# 单元测试

一切准备就绪——让我们向域模块添加一些单元测试，以确保我们编写的所有代码都能按预期运行。

```rs
//! src/domain.rs
// [...]

#[cfg(test)]
mod tests {
    use claim::{assert_err, assert_ok};

    use crate::domain::SubscriberName;

    #[test]
    fn a_256_grapheme_log_name_is_valid() {
        let name = "a".repeat(256);
        assert_ok!(SubscriberName::parse(name));
    }

    #[test]
    fn a_name_longer_than_256_graphemes_is_rejected() {
        let name = "a".repeat(257);
        assert_err!(SubscriberName::parse(name));
    }

    #[test]
    fn whitespace_only_names_are_rejected() {
        let name = " ".to_string();
        assert_err!(SubscriberName::parse(name));
    }

    #[test]
    fn empty_string_is_rejected() {
        let name = "".to_string();
        assert_err!(SubscriberName::parse(name));
    }

    #[test]
    fn names_containing_an_invalid_character_are_rejected() {
        let name = "".to_string();
        assert_err!(SubscriberName::parse(name));
    }

    #[test]
    fn a_valid_name_is_parsed_successfully() {
        let name = "Zhangzhiyu".to_string();
        assert_ok!(SubscriberName::parse(name));
    }
}
```

不幸的是, 它不能编译 - `cargo` 突出显示了我们对 `assert_ok/assert_err` 的所有用法

```plaintext
error[E0277]: `SubscriberName` doesn't implement `std::fmt::Debug`
  --> src/domain.rs:64:9
   |
64 |         assert_err!(SubscriberName::parse(name));
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |         |
   |         `SubscriberName` cannot be formatted using `{:?}` because it doesn'
t implement `std::fmt::Debug`
   |         required by this formatting parameter
   |
```

声明需要我们的类型实现 `Debug` trait，以便提供这些友好的错误信息。让我们在 `SubscriberName` 上添加一个 `#[derive(Debug)]` 属性:

```rs
//! src/domain.rs
// [...]

#[derive(Debug)]
pub struct SubscriberName(String);
```

编译器现在应该没问题了。测试怎么样？

```shell
cargo test
```

```plaintext
running 6 tests
test domain::tests::empty_string_is_rejected ... FAILED
test domain::tests::a_valid_name_is_parsed_successfully ... ok
test domain::tests::names_containing_an_invalid_character_are_rejected ... FAILE
D
test domain::tests::whitespace_only_names_are_rejected ... FAILED
test domain::tests::a_256_grapheme_log_name_is_valid ... ok
test domain::tests::a_name_longer_than_256_graphemes_is_rejected ... FAILED

failures:

---- domain::tests::empty_string_is_rejected stdout ----

thread 'domain::tests::empty_string_is_rejected' panicked at src/domain.rs:19:13
:
 is not a valid subscriber name
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- domain::tests::names_containing_an_invalid_character_are_rejected stdout --
--

thread 'domain::tests::names_containing_an_invalid_character_are_rejected' panic
ked at src/domain.rs:19:13:
 is not a valid subscriber name

---- domain::tests::whitespace_only_names_are_rejected stdout ----

thread 'domain::tests::whitespace_only_names_are_rejected' panicked at src/domai
n.rs:19:13:
  is not a valid subscriber name

---- domain::tests::a_name_longer_than_256_graphemes_is_rejected stdout ----

thread 'domain::tests::a_name_longer_than_256_graphemes_is_rejected' panicked at
 src/domain.rs:19:13:
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaaaaaaaa is not a valid subscriber name


failures:
    domain::tests::a_name_longer_than_256_graphemes_is_rejected
    domain::tests::empty_string_is_rejected
    domain::tests::names_containing_an_invalid_character_are_rejected
    domain::tests::whitespace_only_names_are_rejected

test result: FAILED. 2 passed; 4 failed; 0 ignored; 0 measured; 0 filtered out; 
finished in 0.00s

error: test failed, to rerun pass `--lib`
```

所有要求失败的路径测试都失败了，因为我们仍然在验证约束不满足的时候panic——让我们改变它:

```rs
//! src/domain.rs
// [...]
impl SubscriberName {
    pub fn parse(s: String) -> Result<SubscriberName, String> {
        // [...]

        if is_empty_or_whitespace || is_too_long || contains_forbidden_characters {
            // Replacing `panic!` with `Err(...)`
            Err(format!("{s} is not a valid subscriber name"))
        } else {
            Ok(Self(s))
        }
    }
}
```

现在，我们所有的领域单元测试都通过了——让我们最终解决本章开头编写的失败的集成测试。
