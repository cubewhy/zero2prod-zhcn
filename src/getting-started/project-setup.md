# 初始化我们的项目

通过 rustup 安装的工具链会将各种组件捆绑在一起。
其中之一就是 Rust 编译器 rustc。您可以使用以下命令进行检查：

```shell
rustc --version
```

您无需花费大量时间直接使用 Rustc——您构建和测试 Rust 应用程序的主要界面将是 Rust 的构建工具 Cargo。

您可以使用以下命令再次检查一切是否正常运行

```shell
cargo --version
```

我们接下来用 **cargo** 创建项目的骨架, 在整本书里, 我们都要在这个项目上工作

```shell
cargo new zero2prod
```

项目文件夹 `zero2prod` 的结构看起来应该是这样的

```plaintext
zero2prod/
  Cargo.toml
  .gitignore
  .git
  src/
    main.rs
```

该项目本身就是一个 Git 仓库，开箱即用。 (cargo 会自动创建git仓库)
如果您计划将项目托管在 GitHub 上，只需创建一个新的空仓库并运行

```shell
cd zero2prod
git add .
git commit -am "Project skeleton"
git remote add origin git@github.com:YourGitHubNickName/zero2prod.git
git push -u origin main
```

鉴于 GitHub 的受欢迎程度以及其最近发布的用于 CI 流水线的 GitHub Actions 功能，我们将以GitHub 为参考，但您可以自由选择任何其他 git 托管解决方案 (或根本不选择任何解决方案)
