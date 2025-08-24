# 类型驱动开发

让我们在项目中添加一个新模块，域，并在其中定义一个新的结构, `SubscriberName`:

```rs
//! src/lib.rs
// [...]
pub mod domain;
```

```rs
//! src/domain.rs

pub struct SubscriberName(String);
```

`SubscriberName` 是一个元组结构体，它是一种新类型，包含一个 String 类型的（未命名）字段。

`SubscriberName` 是一个真正的新类型，而不仅仅是一个别名——它不继承 String 上的任何方法，尝试将 String 赋值给 `SubscriberName` 类型的变量会触发编译器错误，例如:

```rs
let name: SubscriberName = "A string".to_string();
```

```plaintext
error[E0308]: mismatched types
  --> src/main.rs:10:32
   |
10 |     let name: SubscriberName = "A string".to_string();
   |               --------------   ^^^^^^^^^^^^^^^^^^^^^^ expected `SubscriberName`,
 found `String`
   |               |
   |               expected due to this

For more information about this error, try `rustc --explain E0308`.
 ```

根据我们当前的定义, `SubscriberName` 的内部字段是私有的：它只能根据 Rust 的可见性规则从域模块内的代码访问。

一如既往，信任但要验证：如果我们尝试在订阅请求处理程序中构建一个 `SubscriberName` 会发生什么?

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let subscriber_name = crate::domain::SubscriberName(form.name.clone());
    // [...]
}
```

编译器会报错

```plaintext
error[E0603]: tuple struct constructor `SubscriberName` is private
  --> src/routes/subscriptions.rs:22:42
   |
22 |     let subscriber_name = crate::domain::SubscriberName(form.name.clone());
   |                                          ^^^^^^^^^^^^^^ private tuple struct con
structor
   |
  ::: src/domain.rs:1:27
   |
1  | pub struct SubscriberName(String);
   |                           ------ a constructor is private if any of the fields i
s private
   |
note: the tuple struct constructor `SubscriberName` is defined here
  --> src/domain.rs:1:1
   |
1  | pub struct SubscriberName(String);
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

因此，就目前情况而言，在域模块之外构建 `SubscriberName` 实例是不可能的。

让我们为 `SubscriberName` 添加一个新方法:

```rs
//! src/domain.rs
use unicode_segmentation::UnicodeSegmentation;

pub struct SubscriberName(String);

impl SubscriberName {
    /// Returns an instance of `SubscriberName` if the input satisfies all
    /// our validation constraints on subscriber names.
    /// It panics otherwise.
    pub fn parse(s: String) -> SubscriberName {
        // `.trim()` returns a view over the input `s` without trailing
        // whitespace-like characters.
        // `.is_empty` checks if the view contains any character.
        let is_empty_or_whitespace = s.trim().is_empty();
        // A grapheme is defined by the Unicode standard as a "user-perceived"
        // character: `å` is a single grapheme, but it is composed of two characters
        // (`a` and `̊`).
        //
        // `graphemes` returns an iterator over the graphemes in the input `s`.
        // `true` specifies that we want to use the extended grapheme definition set,
        // the recommended one.
        let is_too_long = s.graphemes(true).count() > 256;
        // Iterate over all characters in the input `s` to check if any of them matches
        // one of the characters in the forbidden array.
        let forbidden_characters = ['/', '(', ')', '"', '<', '>', '\\', '{', '}'];
        let contains_forbidden_characters = s.chars().any(|g| forbidden_characters.contains(&g));

        if is_empty_or_whitespace || is_too_long || contains_forbidden_characters {
            panic!("{s} is not a valid subscriber name");
        } else {
            Self(s)
        }
    }
}
```

是的，你说得对——这简直就是对 `is_valid_name` 的厚颜无耻的复制粘贴。

不过，有一个关键的区别：返回类型。

`is_valid_name` 返回一个布尔值，而 `parse` 方法如果所有检查都成功，则会返回一个 `SubscriberName`。

还有更多!

`parse` 是在域模块之外构建 `SubscriberName` 实例的唯一方法——我们
在前面几段中已经验证过这一点。

因此，我们可以断言，任何 `SubscriberName` 实例都将满足我们所有的验证约束。

我们已经确保 `SubscriberName` 实例不可能违反这些约束。

让我们定义一个新的结构体，NewSubscriber:

```rs
//! src/domain.rs
// [...]
pub struct NewSubscriber {
    pub email: String,
    pub name: SubscriberName,
}

pub struct SubscriberName(String);

// [...]
```

如果我们将 `insert_subscriber` 改为接受 `NewSubscriber` 类型的参数而不是 `FormData` 类型，会发生什么情况?

```rs
pub async fn insert_subscriber(
    pool: &PgPool,
    new_subscriber: &NewSubscriber,
) -> Result<(), sqlx::Error> {
    // [...]
}
```

有了新的签名，我们可以确保 `new_subscriber.name` 非空——**不可能**通过传递空的订阅者名称来调用 `insert_subscriber`。

我们只需查找函数参数的类型定义即可得出这个结论——我们可以再次进行本地判断，而无需去检查函数的所有调用点。

花点时间回顾一下刚刚发生的事情：我们从一组需求开始（所有订阅者名称都必须验证一些约束），我们发现了一个潜在的陷阱（我们可能会在调用 insert_subscriber 之前忘记验证输入），然后**我们利用 Rust 的类型系统彻底消除了这个陷阱**。

我们通过构造使一个错误的使用模式变得不可表示——它将无法编译。

这种技术被称为类型驱动开发。

类型驱动开发是一种强大的方法，它可以将我们试图在类型系统内部建模的领域的约束进行编码，并依靠编译器来确保它们得到强制执行。

我们的编程语言的类型系统越具有表达力，我们就能越严格地限制我们的代码，使其只能表示在我们所工作的领域中有效的状态。

Rust 并没有发明类型驱动开发——它已经存在了一段时间，尤其是在函数式编程社区（Haskell、F#、OCaml 等）。Rust“只是”为您提供了一个具有足够表达力的类型系统，可以充分利用过去几十年来在这些语言中开创的许多设计模式。我们刚刚展示的特定模式在 Rust 社区中通常被称为“新型模式”。

在实现过程中，我们将逐步涉及类型驱动开发，但我强烈建议您查看本章脚注中提到的一些资源：
它们是任何开发人员的宝库。
