# 将错误作为值 - Result

Rust 的主要错误处理机制建立在 `Result` 类型之上:

```rs
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

Result 用作易出错操作的返回类型: 如果操作成功，则返回 `Ok(T)`; 如果失败，则返回 `Err(E)`。

我们实际上已经使用过 `Result` 了，尽管当时我们并没有停下来讨论它的细微差别。

让我们再看一下 `insert_subscriber` 的签名:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn insert_subscriber(
    pool: &PgPool,
    new_subscriber: &NewSubscriber,
) -> Result<(), sqlx::Error> {
    // [...]
}
```

它告诉我们，在数据库中插入订阅者是一个容易出错的操作——如果一切按计划进行，我们不会收到任何返回（() - 单位类型），如果出现问题，我们将收到一个 `sqlx::Error`，其中包含出错的详细信息（例如连接问题）。

将错误作为值，与 Rust 的枚举相结合，是构建健壮错误处理机制的绝佳基石。

如果你之前使用的语言支持基于异常的错误处理，那么这可能会带来翻天覆地的变化：我们需要了解的关于函数故障模式的一切都包含在它的签名中。

你无需深入研究依赖项的文档来了解某个函数可能抛出的异常（假设它事先已记录在案！）。

你不会在运行时对另一个未记录的异常类型感到惊讶。你不必“以防万一”地插入一个 catch-all 语句。

我们将在这里介绍基础知识，并将更详细的细节（Error trait）留到下一章。

## 在 `parse` 中加入 `Result` 返回值

让我们重构一下 `SubscriberName::parse` 函数，让它返回一个 Result，而不是因为输入无效而 panic。

我们先修改一下签名，但不修改函数体:

```rs
//! src/domain.rs
// [...]
impl SubscriberName {
    pub fn parse(s: String) -> Result<SubscriberName, ???> {
        // [...]
    }
}
```

我们应该使用什么类型作为 Result 的 Err 变量?

最简单的选择是字符串 - 失败时我们只会返回一条错误消息。

```rs
impl SubscriberName {
    pub fn parse(s: String) -> Result<SubscriberName, String> {
        // [...]
    }
}
```

运行 `cargo check` 会产生两个错误:

```plaintext
error[E0308]: mismatched types
  --> src/routes/subscriptions.rs:25:15
   |
25 |         name: SubscriberName::parse(form.0.name),
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `SubscriberName`,
 found `Result<SubscriberName, String>`
   |
   = note: expected struct `SubscriberName`
                found enum `Result<SubscriberName, std::string::String>`
help: consider using `Result::expect` to unwrap the `Result<SubscriberName, std:
:string::String>` value, panicking if the value is a `Result::Err`
   |
25 |         name: SubscriberName::parse(form.0.name).expect("REASON"),
   |                                                 +++++++++++++++++

error[E0308]: mismatched types
  --> src/domain.rs:20:13
   |
11 |     pub fn parse(s: String) -> Result<SubscriberName, String> {
   |                                ------------------------------ expected `Res
ult<SubscriberName, std::string::String>` because of return type
...
20 |             Self(s)
   |             ^^^^^^^ expected `Result<SubscriberName, String>`, found `Subsc
riberName`
   |
   = note: expected enum `Result<SubscriberName, std::string::String>`
            found struct `SubscriberName`
help: try wrapping the expression in `Ok`
   |
20 |             Ok(Self(s))
   |             +++       +

For more information about this error, try `rustc --explain E0308`.
error: could not compile `zero2prod` (lib) due to 2 previous errors
```

让我们关注第二个错误：在 parse 结束时，我们无法返回一个 SubscriberName 的裸实例——我们需要在两个 Result 变体中选择一个。

编译器理解了这个问题，并建议了正确的修改：使用 Ok(Self(s)) 而不是 Self(s)。

让我们遵循它的建议:

```rs
//! src/domain.rs
// [...]
impl SubscriberName {
    pub fn parse(s: String) -> Result<SubscriberName, String> {
        // [...]

        if is_empty_or_whitespace || is_too_long || contains_forbidden_characters {
            panic!("{s} is not a valid subscriber name");
        } else {
            Ok(Self(s))
        }
    }
}
```

`cargo check` 现在应该返回一个错误:

```plaintext
error[E0308]: mismatched types
  --> src/routes/subscriptions.rs:25:15
   |
25 |         name: SubscriberName::parse(form.0.name),
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `SubscriberName`,
 found `Result<SubscriberName, String>`
   |
   = note: expected struct `SubscriberName`
                found enum `Result<SubscriberName, std::string::String>`
help: consider using `Result::expect` to unwrap the `Result<SubscriberName, std:
:string::String>` value, panicking if the value is a `Result::Err`
   |
25 |         name: SubscriberName::parse(form.0.name).expect("REASON"),
   |                                                 +++++++++++++++++

For more information about this error, try `rustc --explain E0308`.
error: could not compile `zero2prod` (lib) due to 1 previous error
```

它抱怨我们在 `subscribe` 中调用 `parse` 方法: 当 parse 返回一个 `SubscriberName` 时，将其输出直接赋值给 `Subscriber.name` 是完全没问题的。

现在我们返回一个 Result —— Rust 的类型系统迫使我们处理这条不愉快的路径。我们不能假装它不会发生。

不过，我们先避免一下子讲太多——为了让项目尽快再次编译，我们暂时只在验证失败时触发 panic:

```rs
//! src/routes/subscriptions.rs
// [...]

pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let new_subscriber = NewSubscriber {
        email: form.0.email,
        // Notice the usage of `expect` to specify a meaningful panic message
        name: SubscriberName::parcse(form.0.name).expect("Name validation failed."),
    };
    // [...]
}
```

`cargo check` 现在应该很顺利了。

该进行测试了!
