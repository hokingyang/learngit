# Cilium 1.0:将BPF革新引入k8s网络和安全系统
过去几个月对Cilimu和BPF贡献者来说是令人振奋的一段时间，我们见证了Cilium社区和BPF用户快速增长的阶段，社区由来自google，facebook，Netflix，红帽和许多公司的开发人员构成。Linux核心社区正准备用BPF替代iptables是BPF成功的重要标志。

以上因素使得BPF快速成熟。自从2017年DockerCon上发布后，Cilium经历了不可思议的发展阶段，对参与项目的所有人员致以最诚挚的感谢。其中共享代码，提供反馈和增加影响力，各种支持对项目成熟都是非常重要的。

## Cilium 1.0：稳定的API和LTS发布
今天，Cilium 1.0稳定版本发布。从这个版本开始，将会提供产品保障以及生产环境下的最佳实践。
 - [API稳定性](http://docs.cilium.io/en/doc-1.0/api/#compatibility-guarantees)，可以确保API向上或者向下兼容。
 - [稳定版本](http://docs.cilium.io/en/doc-1.0/contributing/#release-process)，提供生产环境下长期支持
 - 安全问题处理流程
 - 基于Slack和Github的及时响应，以及基于其上的问题和功能需求处理流程

## 为什么要用Cilium
本博文专注于Cilium 1.0提供的功能。在另一篇名为[Cilium - Rethinking Linux Networking and Security for the Age of Microservices](https://cilium.io/blog/2018/04/24/cilium-security-for-age-of-microservices)的博文中讨论了Cilium的背景和使用场景，以及与Service Mesh/Sidecar融合推进工作的预览。

## 什么是Cilium
以下列出了Cilium 1.0提供的所有功能描述。可以从Cilium文档中[功能概览](http://docs.cilium.io/en/doc-1.0/intro/#functionality-overview)一节找到更详细的信息。
【Cilium架构图】
 - 高效BPF数据通道：BPF通过提供高性能内核沙箱可编程性，在数据通道层处理繁重工作，在Linux底层提供的超强动力。更多信息可以参见[这篇博文](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables#bpf)或者[BPF参考指南](http://docs.cilium.io/en/doc-1.0/bpf/)
  - 全分布：所有数据通道元素在集群内都是全分布的，在每个集群节点上都运行在操作系统最有效的层级上。
  - Service Mesh数据通道: BPF允许用户为快速增长的service mesh空间建立合理的数据平面。Cilium 1.0已经提供了例如Envoy的透明内部代理。未来Cilium版本会提供Sidecar代理加速。我们已经发布了一些早期Sidecar代理的[基准指标](https://cilium.io/blog/2018/04/24/cilium-security-for-age-of-microservices)。
 - [CNI](http://docs.cilium.io/en/doc-1.0/kubernetes/)和[CMM](http://docs.cilium.io/en/doc-1.0/docker/)插件：CNI和CMM插件可以整合k8s,Mesos和Docker，提供网络，负载均衡和容器安全功能。
 - 数据包和API层面的网络安全：Cilium整合了基于数据包的网络安全以及透明API认证，为传统部署和微服务架构的安全性。
  - [基于身份](http://docs.cilium.io/en/doc-1.0/concepts/#arch-id-security)：Cilium将负载和身份信息在每个包内都打包在一起（而不是依靠源IP地址），提供高可扩展安全性。这一设计使得身份可以被嵌入任何基于IP的协议，而且与未来的[SPIFFEE](https://github.com/spiffe/spiffe)或者k8s的[Container Identity Working Group](https://github.com/kubernetes/community/tree/master/wg-container-identity)兼容。
  - [基于IP/CIDR](http://docs.cilium.io/en/doc-1.0/policy/language/#ip-cidr-based)：如果基于身份的方式不适用，那么可以采用基于IP/CIDR安全方式控制安全访问。Cilium建议在安全策略中尽量采用抽象方式，避免写入具体IP地址。其中一个实例就是定义基于[k8s服务名](http://docs.cilium.io/en/doc-1.0/policy/language/#services-based)的策略。
  - [API自感知安全机制](http://docs.cilium.io/en/doc-1.0/policy/language/#layer-7-examples)：HTTP/REST,gRPC和Kafka广泛使用暴露出基于IP和端口的服务，其安全机制明显不足。内置自感知API和数据存储相关协议在相关粒度上允许强制使用最小特权级别安全。
 - 分布式可扩展负载均衡：高性能3-4层负载均衡器在服务连接间使用BPF，具有流哈希和加权round-robin功能。基于哈希实现的BPF提供O(1)复杂度的性能，也就是性能随着服务数量增加性能并不下降。负载均衡器可以用两种方法配置实现：
  - k8s服务实现：所有k8s集群IP服务会自动在BPF中实现，为kube-proxy在集群间提供负载均衡提供了一种高可扩展性选择。
  - [API驱动](http://docs.cilium.io/en/doc-1.0/api/)：对更超前的使用场景，扩展式API可以用来直接配置负载均衡模块。
 - [简化网络模型](http://docs.cilium.io/en/doc-1.0/intro/#simple-networking)：将安全从地址层解耦出来极大简化了网络模型：一个三层网络空间为所有服务端点提供链接，在其上分段，用策略层实现安全控制。这一简化对扩展和排错很有帮助。网络可以用两种方式配置：
  - [Overlay/VXLAN](http://docs.cilium.io/en/doc-1.0/concepts/#overlay-network-mode)：此方法是在IP协议上负载身份信息的最简单整合方法。VXLAN使用硬件帮助实现最佳性能。
  - [直接路由](http://docs.cilium.io/en/doc-1.0/concepts/#direct-native-routing-mode)：直接路由将路由功能授权给已有网络模块，例如内置Linux路由层，IPVLAN或者云路由提供者。 
 - 可视化/监测：跟策略类似，可视化也在网络包和API调用两个层面实现。所有可视化信息，不仅仅是IP地址和端口号，还包括丰富的工作流元数据，例如container/pod标签和服务名。
  - [显微镜](https://github.com/cilium/microscope)：显微镜为集群层面提供基于标签，安全身份和事件类型的过滤，提供安全和转发事件的可视化。
  - 基于BPF高性能监控：高性能BPF性能循环缓冲区(perf ring buffer)的设置，就是为了追踪每秒百万级的应用事件，提供整合BPF可编程性的高效通道，允许数据可视化同时增加最小额外负载。
  - [API驱动](http://docs.cilium.io/en/doc-1.0/api/)：所有可视化都通过API提供接口，可以嵌入现有系统中。   
 - 解决问题：
  - [集群连接状态](http://docs.cilium.io/en/doc-1.0/troubleshooting/#cluster-connectivity-check)：Cilium周期性监控集群连接状态，包括节点之间延迟，判断节点失效和底层网络问题
  - [Prometheus仪表](http://docs.cilium.io/en/doc-1.0/configuration/metrics/)：可以将Cilium整合到现有监控仪表盘中。
  - 健康与状态检查：可靠健康和状态检查可以帮助评判构成元素的健康性。
  - [排错和报表工具](http://docs.cilium.io/en/doc-1.0/troubleshooting/#cluster-diagnosis-tool)帮助自动检测一般问题并搜集bug报告。

## 开始
Cilium简单易用，尤其在k8s中：
{{{
$ curl -sLO https://releases.cilium.io/v1.0.0/examples/kubernetes/cilium.yaml
$ vim cilium.yaml [provide etcd or consul address]
$ kubectl create -f cilium.yaml
$ kubectl create -f demo_app.yaml
$ kubectl create -f http_policy.yaml
$ kubectl exec -ti xwing-68c6cb4b4b-red5 -- curl -s -XPUT deathstar/v1/exhaust-port
Access denied
}}}

上例是一个[minikube简单教程](http://docs.cilium.io/en/doc-1.0/gettingstarted/minikube/)，实现了http感知网络策略的补注。更多信息可以参考[这里](http://docs.cilium.io/en/doc-1.0/gettingstarted/).

关于如何安装Cilium，可以参考[Kubernetes Quick Installation Guide](http://docs.cilium.io/en/doc-1.0/kubernetes/quickinstall/) 或者[installation guides](http://docs.cilium.io/en/doc-1.0/install/guides/#)。

## 未来路标功能
尽管Cilium1.0对所有人来说都是一个激动人心的路碑，但是我们已经开始规划Cilium1.1中的新功能，那么1.1之后版本会有哪些路标功能呢？

 - 多集群服务路由：Cilium简化网络模型、地址解耦和策略是的未来集群之间扩展很容易。Cilium不需要复杂proxy或者Ingress方案，就可以支持跨集群k8s服务路由，并提供全部基于身份和API感知的安全功能。
 - 与OpenTracing、Jaeger和ZIPkin集成：BPF增加很少额外负载的特性使得它成为追踪和控制系统最佳新技术。
 - CRI支持：考虑到很多社区的呼吁，我们希望支持CRI功能以便更好抽象container runtime环境。
 - 非容器负载：BPF数据通道不仅限于容器抽象，它只不过是第一个大力推广的场景。未来版本将会提供APIs和如何与Linux任务、虚机的文档，并提供如何将基于身份的安全空间与现存使用IP地址无法移植环境桥接的支持。

可以在[github issue](https://github.com/cilium/cilium/issues/3585)网站找到更多1.1版本规划。欢迎访问并留下建议。





