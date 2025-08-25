# 拒绝无效订阅者 第一部分

我们的新闻通讯 API 已上线，托管在云服务提供商处。

我们拥有一套基本的工具来排查可能出现的问题。

我们有一个公开的端点 (POST /subscriptions) 来订阅我们的内容。

我们已经取得了很大进展！

但我们也走了一些弯路: `POST /subscriptions` 相当...宽松。

我们的输入验证极其有限：我们只需确保姓名和邮箱字段都已提供，其他什么都不用做。

我们可以添加一个新的集成测试，用一些“棘手”的输入来探测我们的 API:

```rs
//! tests/health_check.rs
// [...]
#[tokio::test]
async fn subscribe_returns_a_200_when_fields_are_present_but_empty() {
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
            200,
            response.status().as_u16(),
            "The API did not return a 200 OK when the payload was {}.",
            description
        );
    }
}
```

不幸的是，新的测试通过了。

尽管所有这些有效载荷显然都是无效的，但我们的 API 还是欣然接受它们，并返回 200 OK。

这些麻烦的订阅者详细信息最终会直接进入我们的数据库，并在我们发送新闻通讯时给我们带来麻烦。

订阅新闻通讯时，我们需要提供两项信息：姓名和电子邮件地址。

本章将重点介绍名称验证: 我们应该注意什么?

- [需求](cp6.1-requirements.md)
- [第一个实现](cp6.2-first-implementation.md)
- [验证就像一口漏水的大锅](cp6.3-validation-is-a-leaky-cauldron.md)
- [类型驱动开发](cp6.4-type-driven-development.md)
- [所有权与不变量](cp6.5-ownership-meets-invariants.md)
- [Panics](cp6.6-panics.md)
- [将错误作为值 - Result](cp6.7-error-as-values-result.md)
- [深刻的断言错误: claim](cp6.8-insightful-assertion-errors-claim.md)
- [单元测试](cp6.9-unit-tests.md)
- [处理 Result](cp6.10-handling-a-result.md)
- [电子邮件格式](cp6.11-the-email-format.md)
- [基于属性的测试](cp6.13-property-based-testing.md)
- [Payload 验证](cp6.14-payload-validation.md)
- [小结](cp6.15-summary.md)
