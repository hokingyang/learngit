

为了保证业务平稳发展，我们引入了一种使得工程师们可以动态高效快速部署上千台服务的微服务架构，这种服务大大提高了我们平台上的乘客司机的使用体验。

尽管这种模式支撑了我们快速增长的应用环境，但是由于业务规模太大也带来了痛点。为了容易维护和升级微服务，2015年采用了Docker技术，确保资源和服务被打包成一个单元。

随着Docker的发展，核心架构团队开发了一套快速高效为Apache Mesos和基于k8s的容器生态产生 Dockerfiles配置文件并将应用代码打包在Docker images中的技术。考虑到微服务技术的快速发展，我们开源了核心模块，Makisu，以便其它用户能够收到同样的效果。


# Uber的Docker之路 #
2015年早期，我们在裸服务器上部署了大约400个服务，他们共享主机上的依赖包和配置文件，仅有很少资源限制。随着工程师规模扩大，依赖管理和资源隔离成为一个问题。为了解决这个问题，核心架构团队开始将服务移植到Docker中，随之建立了标准化和流程化的容器创建流程自动化机制。

新容器化运行环境在服务生命周期管理的运行速度和可靠性方面带来很大提高。然而，访问服务依赖的密钥被打包进images，带来了潜在的安全隐患。简单说，一旦一个文件被放入Docker image中，就会永久存在。尽管可以通过其他这种手段隐藏这些密钥，但是Docker 并不提供真正将他们从image中移除的功能。

其中一个解决方案是docker-squash，一个删除Docker image 中间步骤所需文件的开源工具，但是带来的问题就是创建 image的时间翻倍，这个问题基本抵消了采用微服务架构带来的好处。

为了解决这个问题，我们决定分叉Docker自研我们需要的功能。如果在创建image时候，把所需的密钥以volume方式挂载，这样最终的Docker image中就不留任何密钥的痕迹。因为没有额外延迟，而且代码更改也很少，因此这方法很有效。架构团队和服务团队都满意了，容器架构被更好地利用起来。

# 横向编译容器image #
到了2017年，这种架构不再能满足需求。随着Uber的增长，平台规模也同样收到很大挑战。每次创建超过3000个服务，每天有若干次，大大增加了消耗时间。有些需要2个小时，大小超过10GB，消耗大量存储空间和带宽，严重影响开发者生产率。

此时，我们认识到创建可以扩展的容器image是很重要的。为此提出了三个需求：轻量创建，支持分布式cache，image大小优化。

## 轻量创建 ##
2017年，核心架构团队开始将Uber的计算负载迁移到提供自动化、可扩展和更加弹性化的统一平台上。随之而来，Dockerfile也需要能够运行在共享集群上的容器中。

不幸的是，Docker创建逻辑依赖于一种通过对比创建时间来决定不同层次之间不同的copy-on-write文件系统。这个逻辑需要特权挂载或者卸载容器内部的目录，以便给新系统提供安全机制。也就意味着，完全轻量级创建并不是解决思路。

## 支持分布式缓存(cache) ##
通过分层缓存（cache）技术，用户可以复用之前的创建版本和层级，可以减少执行时候的冗余。Docker为每个创建提供分布式缓存，但是并不支持不同分支和服务。需要依靠Docker本地缓存层，但是因为集群内的生成新版本要求不断清空缓存，造成缓存命中率大大降低。

过去，Uber采用指定机器创建指定服务的方式提高命中率，然而，这种方法对于多种服务构成的系统来说显然是不够的。考虑到我们每天会使用上千种服务，这个过程也增加了调度复杂性。

显然一个好的缓存策略对我们的方案更加重要，最终分布式缓存方案既可以解决性能问题，也能够解决创建系统时的调度问题。

## 优化image大小 ##



Smaller images save storage space and take less time to transfer, decompress, and start.

To optimize image size, we looked into using multi-stage builds in our solution. This is a common practice among Docker image build solutions: performing build steps in an intermediate image, then copying runtime files to a slimmer final image. Although this feature does require more complicated Dockerfiles, we found that it can drastically reduce final image sizes, thereby making it a requirement for building and deploying images at scale.

Another optimization tactic we explored for decreasing image size is to reasonably reduce the number of layers in an image. Fat images are sometimes the result of files being created and deleted or updated by intermediate layers. As mentioned earlier, even when a temporary file is removed in a later step, it remains in the creation layer, taking up precious space. Having fewer layers in an image decreases the chance of deleted or updated files remaining in previous layers, hence reducing the image size.






