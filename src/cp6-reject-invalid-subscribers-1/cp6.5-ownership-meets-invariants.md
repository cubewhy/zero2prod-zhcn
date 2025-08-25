# 所有权与不变量

我们修改了 `insert_subscriber` 的签名，但尚未修改主体以符合新的要求——现在就修改吧。

```rs
//! src/routes/subscriptions.rs

// [...]

#[tracing::instrument(/*[...]*/)]
pub async fn subscribe(form: web::Form<FormData>, pool: web::Data<PgPool>) -> HttpResponse {
    let new_subscriber = NewSubscriber {
        email: form.0.email,
        name: SubscriberName::parse(form.0.name),
    };
    match insert_subscriber(&pool, &new_subscriber).await {
        Ok(_) => HttpResponse::Ok().finish(),
        Err(_) => HttpResponse::InternalServerError().finish(),
    }
}


#[tracing::instrument(
    name = "Saving new subscriber details in the database",
    skip(new_subscriber, pool)
)]
pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        Uuid::new_v4(),
        new_subscriber.email,
        new_subscriber.name,
        Utc::now()
    )
    .execute(pool)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(())
}
```

我们很接近了, 但 `cargo check` 还没办法通过:

```plaintext
error[E0308]: mismatched types
  --> src/routes/subscriptions.rs:46:9
   |
46 |         new_subscriber.name,
   |         ^^^^^^^^^^^^^^
   |         |
   |         expected `&str`, found `SubscriberName`
   |         expected due to the type of this binding

error[E0277]: the trait bound `SubscriberName: sqlx::Encode<'_, Postgres>` is no
t satisfied
```

这里有一个问题: 我们没有任何方法可以真正访问封装在 SubscriberName 中的字符串值!

我们可以将 `SubscriberName` 的定义从 `SubscriberName(String)` 改为 `SubscriberName(pubString)`，但这样一来，我们就会失去前两节中提到的所有好处:

- 其他开发者可以绕过解析，使用任意字符串构建 `SubscriberName`

```rs
let liar = SubscriberName("".to_string());
```

- 其他开发者可能仍然会选择使用 parse 来构建 SubscriberName，但他们随后可以选择将内部值更改为不再满足我们关心的约束的值

```rs
let mut started_well = SubscriberName::parse("A valid name".to_string());
started_well.0 = "".to_string();
```

我们可以做得更好——这正是 Rust 所有权系统的优势所在！
给定结构体中的某个字段，我们可以选择:

- 通过值暴露它，使用消耗结构体本身:

```rs
impl SubscriberName {
    pub fn inner(self) -> String {
        // The caller gets the inner string,
        // but they do not have a SubscriberName anymore!
        // That's because `inner` takes `self` by value,
        // consuming it according to move semantics
        self.0
    }
}

- 暴露可变引用

```rs
impl SubscriberName {
    pub fn inner_mut(&mut self) -> &mut str {
        // The caller gets a mutable reference to the inner string.
        // This allows them to perform *arbitrary* changes to
        // value itself, potentially breaking our invariants!
        &mut self.0
    }
}
```

暴露引用

```rs
impl SubscriberName {
    pub fn inner_ref(&self) -> &str {
        // The caller gets a shared reference to the inner string.
        // This gives the caller **read-only** access,
        // they have no way to compromise our invariants!
        &self.0
    }
}
```

inner_mut 并非我们想要的效果——失去对不变量的控制，相当于使用 SubscriberName(pub String)。

inner 和 inner_ref 都适用，但 inner_ref 更好地传达了我们的意图:

让调用者有机会读取值，但无法对其进行修改。

让我们将 inner_ref 添加到 SubscriberName 中——然后我们可以修改 insert_subscriber 来使用它:

```rs
//! src/routes/subscriptions.rs
// [...]

#[tracing::instrument(
    name = "Saving new subscriber details in the database",
    skip(new_subscriber, pool)
)]
pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        Uuid::new_v4(),
        new_subscriber.email,
        // Using `inner_ref`!
        new_subscriber.name.inner_ref(),
        Utc::now()
    )
    .execute(pool)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(())
}
```

轰隆，编译成功了!

## AsRef

虽然我们的 inner_ref 方法完成了任务，但我必须指出，Rust 的标准库公开了一个专为此类用法而设计的特性——[AsRef](https://doc.rust-lang.org/std/convert/trait.AsRef.html)。

其定义非常简洁:

```rs
pub trait AsRef<T>
where
    T: ?Sized,
{
    // Required method
    fn as_ref(&self) -> &T;
}
```

什么时候应该为某个类型实现 `AsRef<T>`？
当该类型与 T 足够相似，以至于我们可以使用 &self 来获取 T 自身的引用时!

这听起来是不是太抽象了？再看看 inner_ref 的签名：它本质上就是为 SubscriberName 实现的 `AsRef<str>`!

AsRef 可以用来提升用户体验——让我们考虑一个具有以下签名的函数:

```rs
pub fn do_something_with_a_string_slice(s: &str) {
    // [...]
}
```

没什么太复杂的，但你可能需要花点时间弄清楚 SubscriberName 是否能提供 &str ，以及如何提供，尤其是当该类型来自第三方库时。

我们可以通过更改 do_something_with_a_string_slice 的签名来让体验更加无缝:

```rs
// We are constraining T to implement the AsRef<str> trait
// using a trait bound - `T: AsRef<str>`
pub fn do_something_with_a_string_slice<T: AsRef<str>>(s: T) {
    let s = s.as_ref();
    // [...]
}
```

我们现在可以写

```rs
let name = SubscriberName::parse("A valid name".to_string());
do_something_with_a_string_slice(name)
```

它会立即编译通过（假设 `SubscriberName` 实现了 `AsRef<str>` 接口）。

这种模式被广泛使用，例如，在 Rust 标准库 std::fs 中的文件系统模块中。像 create_dir 这样的函数接受一个 P 类型的参数，并强制要求实现 `AsRef<Path>` 接口，而不是强迫用户理解如何将 String 转换为 Path，或者如何将 PathBuf 转换为 Path，或者 OsString，等等...你懂的。

该标准库中还有其他一些像 AsRef 这样的小转换特性——它们为整个生态系统提供了一个共享的接口，以便围绕它们进行标准化。为你的类型实现这些特性，可以立即解锁大量通过现有的 crate 中的泛型类型公开的功能。

我们稍后会介绍其他一些转换特性（例如 From/Into、TryFrom/TryInto）。

让我们删除 inner_ref 并为 SubscriberName 实现 `AsRef<str>`:

```rs
//! src/domain.rs
// [...]
impl AsRef<str> for SubscriberName {
    fn as_ref(&self) -> &str {
        &self.0
    }
}
```

我们还需要修改 `insert_subscriber`:

```rs
//! src/routes/subscriptions.rs
// [...]
pub async fn insert_subscriber(pool: &PgPool, new_subscriber: &NewSubscriber) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        INSERT INTO subscriptions (id, email, name, subscribed_at)
        VALUES ($1, $2, $3, $4)
        "#,
        Uuid::new_v4(),
        new_subscriber.email,
        // Using `as_ref` now!
        new_subscriber.name.as_ref(),
        Utc::now()
    )
    .execute(pool)
    .await
    .inspect_err(|e| {
        tracing::error!("Failed to execute query: {:?}", e);
    })?;

    Ok(())
}
```

该项目可以编译通过...
