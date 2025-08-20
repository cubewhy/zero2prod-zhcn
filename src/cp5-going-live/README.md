# 部署到生产环境

我们已经有了一个可工作的原型通讯API，是时候让我们把他部署到**生产环境**中了！  
我们将会学习到如何去将我们的Rust程序打包成一个Docker镜像来部署到DigitalOcean的[APP Platform](https://docs.digitalocean.com/products/app-platform/)
到最后的章节，我们会拥有一个自动部署(CD)的流程：每一个提交到main分支的代码都会自动触发最新版的生产环境应用给用户们使用。
