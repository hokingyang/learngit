# 容器化JDE #
你可能刚看完Josh Long的演讲，想访问start.spring.io创建第一个应用；或者你喜欢Eclipse MicroProfile，想通过start.microprofile.io创建第一个应用；更奢侈一点儿，你想通过红帽的Quarkus项目使用Supersonic Subatomic Java。

但是你忘了不同电脑的开发环境可能不太一样；或许你想尝试各种不同架构，不想被不同版本工具、CLI、JDK等淹没。

实际情况是，开发环境并不是很容配置，有时候很杂乱。对Java开发者，至少需要安装JDK，一个编辑器，以及类似于Maven或Gradle的开发工具。

幸运的是，现在只需要安装Visual Studio Code，远程开发扩展包，以及Docker桌面就够了。有了这个组合，你就可以在电脑上运行以上这些框架。
我们来看看具体情况。

一旦安装了远程开发扩展包和VS Code，访问start.spring.io，下载一个网络依赖包，解压此文件并记下目录。

图一、图二

到了这一步，打开command palette，选择远程容器：打开容器中的目录....，选择刚才记下的目录。系统会问选择哪个容器镜像作为开发环境。如果不更改Java版本，那么需要Java8。选择了Java8镜像后，VSC将重新加载窗口，并打开一个拥有基本功能的Java开发环境（JDK 8, Maven, and Gradle），并挂载解压目录。

图三、图四

刚才选择的Docker容器内会安装Java项目所需的任何扩展，这些扩展都会在dev容器中定义。现在你可以安全地修改Java类，添加Rest控制器，然后点击在main上方的RUN。

图五、图六

现在如果访问http://localhost:8080，会报错。访问之前需要将端口从dev容器中指向宿主机。

从command palette选择远程容器：转发端口，选择8080端口。

图七、图八

VS Code会弹出访问URL，这样访问Spring Boot应用是不是很简单？现在，访问基于Maven的vanilla Eclipse MicroProfile也同样简单。如果你倾向于用Gradle，dev容器中都预装了他们。如果想用Quarkus的话，由于创建Quarkus应用需要使用Maven从Quarkus原型中搭建，因此有些不同；这也适用于Micronaut，它使用特殊的CLI，下面我们会提供一个快速指南。

启动一个全新VS Code窗口，不需要打开目录。从command palette选择远程容器：在容器中打开目录..，这次，创建一个全新目录。

图九

系统会问使用哪个dev容器镜像；再次选择Java8。 目录只包含一个叫 .devcontainer的子目录，用来保存开发环境的配置和Dockerfile。后续可以修改它们，甚至提交到版本控制系统。在Mac中打开Terminal，创建一个新的带有如下maven命令参数的Quarkus项目:

```
mvn io.quarkus:quarkus-maven-plugin:0.14.0:create \
 -DprojectGroupId=org.acme \
 -DprojectArtifactId=getting-started \
 -DclassName="org.acme.quickstart.GreetingResource" \
 -Dpath="/hello"
```

你现在可以进入启动目录，带着*mvn package quarkus:dev* 启动Quarkus。服务启动运行后，可以通过http://localhost:8080/hello 访问。

如果使用Micronaut，唯一需要做的是安装SDKMAN，然后在Terminal中使用Micronaut命令行。做完这一步，运行*apt-get update && apt-get install zip* 安装SDKMAN，之后运行Micronaut:
```
curl -s “https://get.sdkman.io" | bash \
&& source “/root/.sdkman/bin/sdkman-init.sh” \
&& sdk install micronaut
```

随后，创建Micronaut应用，并运行*mn create-app example*，后续可以参照Micronaut的安装指南。

如果想了解更多，可以通过SSH登入远程开发环境。更多信息可以参考：

 - [Remote Development with Visual Studio Code](https://code.visualstudio.com/docs/remote/remote-overview?wt.mc_id=vscode-medium-brborges)
 - [Definitions of devcontainers on GitHub](https://github.com/Microsoft/vscode-dev-containers?wt.mc_id=vscode-medium-brborges)
 - [Remote Development Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack&wt.mc_id=vscode-medium-brborges)

如果有其它问题或者建议，可以访问我的[Twitter](https://twitter.com/brunoborges)。如果喜欢这期内容，欢迎留下鼓励。

