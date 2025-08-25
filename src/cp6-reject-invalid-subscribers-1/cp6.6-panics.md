# Panics

...但是我们的测试结果并不理想:

```plaintext
running 4 tests
test subscribe_returns_a_200_when_fields_are_present_but_empty ... FAILED
test subscribe_returns_a_200_for_valid_form_data ... ok
test subscribe_returns_a_400_when_data_is_missing ... ok
test health_check_works ... ok

failures:

---- subscribe_returns_a_200_when_fields_are_present_but_empty stdout ----

thread 'actix-server worker 0' panicked at src/domain.rs:33:13:
 is not a valid subscriber name
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

thread 'subscribe_returns_a_200_when_fields_are_present_but_empty' panicked at t
ests/health_check.rs:179:14:
Failed to execute request.: reqwest::Error { kind: Request, url: "http://127.0.0
.1:42035/subscriptions", source: hyper_util::client::legacy::Error(SendRequest, 
hyper::Error(IncompleteMessage)) }
```

好的一面是：我们不再为空名称返回 200 OK 错误。

不好的一面是: 我们的 API 会突然终止请求处理，导致客户端
观察到 `IncompleteMessage` 错误。这不太优雅。

让我们修改测试以反映我们的新期望：当有效负载包含无效数据时，我们希望看到 400 Bad Request 响应。

```rs
//! tests/health_check.rs
// [...]

async fn subscribe_returns_a_200_when_fields_are_present_but_invalid() {
    // Arrange
    let app = spawn_app().await;
    let client = reqwest::Client::new();
    let test_cases = vec![
        ("name=&email=ursula_le_guin%40gmail.com", "empty name"),
        ("name=Ursula&email=", "empty email"),
        ("name=Ursula&email=definitely-not-an-email", "invalid email"),
    ];
    for (body, description) in test_cases {
        // Act
        let response = client
            .post(&format!("{}/subscriptions", &app.address))
            .header("Content-Type", "application/x-www-form-urlencoded")
            .body(body)
            .send()
            .await
            .expect("Failed to execute request.");

        // Assert
        assert_eq!(
            // Not 200 anymore!
            400,
            response.status().as_u16(),
            "The API did not return a 400 OK when the payload was {}.",
            description
        );
    }
}
```

现在，让我们看看根本原因——当 SubscriberName::parse 中的验证检查失败时，我们选择 panic:

```rs
//! src/domain.rs
// [...]
pub fn parse(s: String) -> SubscriberName {
    // [...]

    if is_empty_or_whitespace || is_too_long || contains_forbidden_characters {
        panic!("{s} is not a valid subscriber name");
    } else {
        Self(s)
    }
}
```

Rust 中的 panic 用于处理不可恢复的错误: 意料之外的故障模式，或者我们无法有效恢复的故障模式。例如，主机内存不足或磁盘已满。Rust 的 panic 并不等同于 Python、C# 或 Java 等语言中的异常。虽然 Rust 提供了一些工具来捕获（某些） panic, 但这绝对不是推荐的方法，应该谨慎使用。[Burntsushi](https://github.com/BurntSushi) 几年前在 [Reddit 的一个帖子](https://www.reddit.com/r/rust/comments/9x17hn/when_should_a_library_panic_vs_return_result/e9p5c9t?utm_source=share&utm_medium=web2x&context=3)中就曾明确指出:

> [...] 如果您的 Rust 应用程序在响应任何用户输入时出现 panic，则以下情况应该为真：您的应用程序存在错误，无论该错误存在于库中还是主应用程序代码中。

从这个角度来看，我们可以理解正在发生的事情: 当我们的请求处理程序发生恐慌时，[actix-web 会认为发生了可怕的事情](https://github.com/actix/actix-web/issues/1501)，并立即丢弃正在处理该 panic 请求的工作进程。

如果恐慌不是解决问题的办法，我们应该用什么来处理**可恢复**的错误?
