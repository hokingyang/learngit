
伴随着Azure Devops的分布，我们给开发者也提供了一个全新的叫做Azure Pipelines的CI/CD服务，这项服务实现跨平台开发测试部署；对linux，macOS和windows提供云代理，并提供本地容器工作流支持，弹性k8s、VM和无服务器环境部署。

微软承诺会大力支持开源软件开发，我们下一步就是为开源项目提供最好的CI/CD体验。今天开始，Azure Pipelines为所有开源项目免费地提供CI/CD不限时服务和10个并行作业。所有开源项目都免费运行在同样的基础架构上，也就意味着具有同样性能和服务质量。许多排名靠前的开源项目已经开始使用Azure Pipelines，例如Atom, CPython, Pipenv, Tox, Visual Studio Code, 和TypeScript，使用列表每天都在持续更新中。

下面的内容，大家将会看到Atom如何在Linux, macOS, 和 Windows 中进行持续集成。
【1.png】

## 在Github市场中的Azure Pipelines应用
Azure Pipelines在Github市场中也有下载地址，可以很容易上手。只要在Github账号中安装了应用，就可以在自己的库中运行CI/CD。

【2.png】

## Pull请求和CI检测
当Github应用设置好后，就可以看到pull请求和默认分支commit的CI/CD检查点
【3.png】

和Github API整合使得在pull请求结果中看开发结果很容易，如果有问题，调用栈和受影响文件都可以很容易显示出来。

【4.png】

## 不仅仅是开源
Azure Pipelines对私有部署也很有帮助，对来Columbia, Shell, Accenture等来说都是很好的CI/CD解决方案，同样微软自己最大项目如Azure, Office 365, 和Bing也都在使用。我们为私有项目提供每月1800分钟CI/CD云运行时间或者在自己硬件上不限时CI/CD时间，也可以在Azure DevOps或者Github 市场为私有项目购买并行作业。

除了CI，Azure Pipelines还可以弹性部署在任意平台上，包括Azure, Amazon Web Services, 和Google Cloud Platform，以及云端运行Linux, macOS 或者Windows的服务器，也有内置适配k8s、VM和serverless的部署。另外，还有丰富的语言和工具扩展生态系统。Azure Pipelines代理和任务都是开源的，也已经在Github上发布并接受反馈。



