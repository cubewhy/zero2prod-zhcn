# 深刻的断言错误: claim

我们的大多数断言都类似于 `assert!(result.is_ok())` 或 `assert!(result.is_err())`。

使用这些断言时，Cargo Test 失败时返回的错误消息非常糟糕。

到底有多糟糕？让我们来做个快速实验!

如果你在这个虚拟测试上运行 `cargo test`:

```rs
#[test]
fn dummy_fail() {
    let result: Result<&str, &str> = Err("The app crashed due to an IO error");
    assert!(result.is_ok());
}
```

将会输出

```plaintext
---- dummy_fail stdout ----
thread 'dummy_fail' panicked at 'assertion failed: result.is_ok()'
```

我们无法获得任何有关错误本身的详细信息——这会让调试过程变得相当痛苦。

我们将使用 claim crate 来获取更多信息丰富的错误消息:

```shell
cargo add claim
```

claim 提供了相当全面的断言，可以处理常见的 Rust 类型——特别是 Option 和 Result。

如果我们重写 `dummy_fail` 测试并使用 `claim` crate

```rs
#[test]
fn dummy_fail() {
    let result: Result<&str, &str> = Err("The app crashed due to an IO error");
    claim::assert_ok!(result);
}
```

将会输出

```plaintext
---- dummy_fail stdout ----
thread 'dummy_fail' panicked at 'assertion failed, expected Ok(..),
  got Err("The app crashed due to an IO error")'
```

好多了。
