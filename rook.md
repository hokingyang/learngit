#Rook：k8s中Ceph自动化工具

Rook是运行在k8s集群中自动化编排工具，在0.8版本中，Rook已经变成Beta发行版，如果还没有尝试过Rook，可以现在尝鲜。

Rook是什么，为什么很重要？Ceph运行在k8s集群中很久了，为什么要有这么大的变动？如果以前玩过ceph集群，肯定深知维护ceph集群的复杂性，Rook就是为此而生，使用k8s分布式平台简化大量针对Ceph存储的操作和维护工作。
Rook通过一个操作器(operator)完成后续操作，只需要定义需要的状态就可以了。rook通过操作器监控状态需求变化，并将配置文件分配到集群上生效。操作器关注包括各种集群运行和启停所需的状态信息。本文将详细讨论这些细节。

## Mons ##
Ceph集群中最重要的信息是Mons的quorum，一般集群中有有三个Mons，保持高可用以及quorum可用性以防数据不可用，当创建集群是，Rook将会：
 - 启动特定节点上的Mons,确保他们在quorum中
 - 定期确定Mons状态,确保他们在quorum中
 - 如果某个Mon出现故障，并且没有重新启动，操作器会往quorum中添加一个新的mon，并将失效的移除quorum
 - 更新Ceph客户端和Daemons的IP地址。


## mgr ##
mgr是一个无状态服务，提供集群信息。除了启动核心功能外，Rook还参与配置其它两个mgr插件：
 - 搜集prometheus状态
 - 启动Ceph面板，并启动服务点

## OSDs ##
集群中最具挑战的是OSD部分，存储部分的核心部件。大规模集群会在上线前大量使用OSD。Rook会根据如何使用，以两种模式配置他们，可以参考更多OSD[专题](https://rook.io/docs/rook/v0.8/ceph-cluster-crd.html#storage-selection-settings)。

### 完全自动化 ###
最简单使用方式就是“使用所有资源”模式。意味着操作者自动在所有节点上启动OSD设备，Rook会用如下标准监控并发现可用设备：
 - 设备没有分区
 - 设备没有格式化的文件系统
Rook不会使用不满足以上标准的设备。操作完毕后（一般要几分钟），就拥有一套OSD配置完毕的存储集群。

### 自声明模式 ###
第二种模式给用户更大的选择控制权限，用户可以指定哪些节点或者设备会被使用。有几个层级方式进行配置：

 - 节点
    - 声明哪些节点上会启动OSD
    - 采用“标签”方式用k8s来声明节点
 - 设备
    - 声明启动OSD的设备名
    - 声明设备过滤规则，并在其上启动OSD
    - 采用SSD或者NVME设备创建bluestore的metadata分区，bluestore数据分区分布在各种设备上
这种模式很灵活，可以选择需要启动OSD的设备。

## 客户访问 ##
k8s中，需要存储的客户端都会使用PV并挂载到pod上，Rook提供FlexVolume插件，此插件可以使访问Ceph集群更加简便，只需要声明存储类相关的池，然后在pod上声明指向存储类的PVC，[本例](https://rook.io/docs/rook/v0.8/block.html)将解释在pod中挂载RBD映像的具体步骤。

## RGW ##
除了基础的RADOS集群，Rook还会帮助管理对象空间。如果声明需要一个对象存储空间，Rook将：
 - 为对象空间创建metadata和数据池
 - 启动RGW Daemon，如果需要还可以运行高可用实例
 - 通过RGW Daemon创建k8s设备提供负载均衡。对象存储则在存储内部提供

## MDS ##
最后，但不仅限如此，rook还可以配置CephFS提供共享存储空间服务。当生命在集群中需要文件系统时，Rook会：
 - 为CephFS创建metadata和数据池
 - 创建文件系统
 - 使用MDS启动期望数量的实例
 - 文件系统可以被集群中pod使用，通过相关或者独立路径为每个pod提供访问方式。参看[如下示例](https://rook.io/docs/rook/v0.8/filesystem.html#consume-the-shared-file-system-k8s-registry-sample)

## Ceph工具 ##
即使实现了自动化，仍然需要使用Ceph工具维护正常运转。但是随着越来越自动化，工具的依赖度会慢慢降低。同时，如果需要运行Ceph工具，要么启动一个工具箱，或者通过连接到Mon Daemon容器执行这些工具。

## 下一步工作 ##
Rook运行前提是有一套存储需要配置的k8s集群。Rook的目标就是让配置存储的工作越来越简单，当然目前我们还只是处在这一工作的开始。

我们正在积极开发[Rook项目](https://github.com/rook/rook)，期待有更多功能出现。我们也希望更多专家出现在社区，如果有问题，可以通过Rook [Slack](https://rook-slackin.herokuapp.com/)和我们沟通。

