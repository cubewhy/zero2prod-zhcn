# 部署到生产环境

我们已经有了一个可工作的原型通讯API，是时候让我们把他部署到**生产环境**中了！  

我们将会学习到如何去将我们的Rust程序打包成一个Docker镜像来部署到DigitalOcean的[APP Platform](https://docs.digitalocean.com/products/app-platform/)

到最后的章节，我们会拥有一个自动部署(CD)的流程：每一个提交到main分支的代码都会自动触发最新版的生产环境应用给用户们使用。

- [我们必须谈谈部署](cp5.1-we-must-talk-about-deployments.md)
- [选择我们的工具](cp5.2-choosing-our-tools.md)
- [我们项目的Dockerfile](cp5.3-a-dockerfile-for-our-application.md)
- [部署到 DigitalOcean Apps Platform](cp5.4-deploy-to-digitalocean-apps-platform.md)
