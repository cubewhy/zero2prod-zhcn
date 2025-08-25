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

TODO: WIP
