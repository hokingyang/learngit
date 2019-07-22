Delivery Hero HQ 的k8s经历:从YAML的地狱到Helm的天堂

![标题图](http://dockone.io/uploads/article/20190722/d60c2e5129032dc6901c013d0d8b76b7.jpg)

如果这几年没听说过k8s的人一定是太宅了，k8s最近几年成为高科技公司平台首选，Delivery Hero就是其中之一。DH技术团队最大特点就是人员完全分散，说不同的语言，住在不同的大洲，解决不同的问题。

分散带来很多挑战，同时也让我们聚合各种方法解决普遍性问题。本文中您将看到如何通过在k8s弹性平台上用某些工具解决普遍性问题的技术细节：
 - 轻松管理很多k8s资源
 - 以有状态方式管理k8s资源
 - 进入生产之前对k8s资源进行检视
 - k8s中处理敏感数据
刚开始使用k8s，跟大多数人一样，是从YAML文件开始的。对初级使用者足够了，但是当系统扩展到多集群，很多应用运行在复杂环境中时，随着文件增加，YAML管理就变得越来越困难了。对一个普通应用运行需要的资源包括：Deployment,Service,Ingree,ConfigMap,Secret,HorizontalPodAutoscaler和一到几个CronJob。同时，还有大量代码并且要保持诸多文件中同步。

一开始，团队成员通过简单的shell和自动化代码（例如sed,envsubst,Sprig,Jinja）解决了这些问题，但是单一工具渊源无法适应k8s的发展速度....

## Helm ##
[Helm](https://helm.sh/) 图表允许用文本模板和形如逻辑值和循环等基本编程功能将多个应用打包在一起。几乎DH每个团队都将其应用打包成Helm图表模式。有了公开图表库，Helm还可以直接安装集群工具方案，例如cluster-autoscaler, nginx-ingress, metrics-server 等。DH工程师也参与创建了这些公开图表，例如cluster-overprovisioner。

一旦选择了Helm，很多问题就变得很明显。集群安装了什么图表？集群上运行了什么版本的Prometheus？如何才能在集群上同时升级多个图表？由此引出了第二个工具....

## Helmfile ##
[Helmfile](https://github.com/roboll/helmfile)可以用来定义与具体用kubectl控制的集群上下文相关的Helm赋值文件中使用特定图表，并且可以用一条命令同步到整个集群上。参见如下示例：
```
context: eks_staging

releases:
- chart: charts/apps/event-processor
  name: staging-event-processor
  values:
  - chart/sapps/event-processor/values/staging/values.yaml

- chart: charts/apps/delivery-manager
  name: staging-delivery-manager
  values:
  - chart/sapps/delivery-manager/values/staging/values.yaml
```
同步图表到eks_staging集群：
```
helmfile --file helmfiles/staging.yaml sync
```

Helmfile也是一个很好的集群刷新工具。例如，有一套安装在所有集群上的标准图表：cluster-autoscaler, fluentd, nginx-ingress, metrics-server, external-dns, oauth2-proxy, prometheus, cluster-overprovisioner 和node-problem-detector。在某个单独Helmfile中指定这些图表以及版本，并将这些文件放到git中，使得我们可以很安全地在多个集群中跟踪和升级图表，例如在非生产集群中首先测试新图标版本。

看来用Helmfile可以将应用打包成图表，并且用有状态方式管理这些图表，但是是否能保证在集群中可以完全同步？Helm只是将数据发送到k8s API，并不知道下一步会发生什么。因此需要引入Helm 插件....

## helm-diff ##
[helmdiff](https://github.com/databus23/helm-diff)会将Helm将使用的diff变化用不同颜色标注出来，非常好用！其易用性有点像另外一个DH每天都在用的工具：Terraform，参见如下示例（nginx-ingress的configmap中有些改变）:
```
$ helmfile --selector name=ingress01 --file helmfiles/staging/infra.yaml diff
exec: helm diff upgrade --allow-unreleased ingress01 stable/nginx-ingress --version 1.0.1 --values nginx-ingress/staging/values.yaml --kube-context eks_staging
default, ingress01-nginx-ingress-controller, ConfigMap (v1) has changed:
  # Source: nginx-ingress/templates/controller-configmap.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    labels:
      app: nginx-ingress
      chart: nginx-ingress-1.0.1
      component: "controller"
      heritage: Tiller
      release: ingress01
    name: ingress01-nginx-ingress-controller
  data:
    enable-vts-status: "true"
    log-format-escape-json: "true"
    proxy-next-upstream: error timeout http_502
-   proxy-next-upstream-tries: "3"
+   proxy-next-upstream-tries: "2"
    use-geoip: "true
```
这个插件可以用于处理k8s资源的工作流，那么在Helm图表中如何处理敏感数据？例如API键值和认证信息？

## helm-secrets ##
本文第四个工具也是Helm插件:helm-secrets。

很多时候，git中的API键值或者密码都是用明文保存，或者不得不使用PGP工具加密，或者不得不将敏感数据从图表中分拆，尽管有很多方法解决这个问题，但是都无法与本文中描述的工作流整合。helm-secrets插件与Helmfile整合，加密图表文件在本地通过有AWS/GCP KMS提供的服务无缝地解密，这样做有如下好处：
 - 加密信息文件可以安全存放在git中
 - 对云提供商授权需要解密
 - YAML键值仍然未加密，因此pull请求仍然会滥用敏感字段
 - Helm图表中加密字段可以如其他键值对一样被使用
如下示例是一个采用加密值的文件：
```
context: eks_staging

releases:
- chart: charts/apps/event-processor
  name: staging-event-processor
  values:
  - charts/apps/event-processor/values/staging/values.yaml
  secrets:
  - charts/apps/event-processor/values/staging/secrets.yaml
```

secrets.yaml文件看起来如下：
```
secrets:
    API_KEY: ENC[AES256_GCM,data:xxxxxxxxx=,tag:xxxxxx==,type:str]
```

另外还需要添加一个.sops.yaml文件，这样helm-secrets会知道使用哪个KMS键值。一旦以上配置完成而且被授权访问在云提供商处的KMS键值，就可以使用最简单方式同步图表，并且secrets.yaml中的键值会被无缝加密。

## 结论 ##
如果不仔细设计，k8s带来的管理问题会很快超过其优势。但是使用本文中这些工具可以简化我们的管理，使之变得更加安全。

如果喜欢本文内容，而且喜欢k8s架构，也许你会对在DH公司的日常发现和推荐系统工作感兴趣。

我们还有其他超棒的工作职位，可以从[这里获得](https://www.deliveryhero.com/careers/jobs/#/)。






