# 适配kong的k8s ingress控制器发布
今天我们很高兴发布[k8s Ingress控制器](https://github.com/Kong/kubernetes-ingress-controller)。

容器编排发展很快，以满足基础架构更加灵活可靠，更加有效的要求。这些工具前端都是依托于k8s，k8s编排器不仅使得操作和应用团队部署扩展灵活，同时也让给开发者更好的开发体验并使用自服务模式。

这种模式关键是一套网络栈，能够提供跨集群容器编排的高度动态部署和动态扩展。

[Kong](https://konghq.com/)就是满足这样要求，提供了高性能，开源API网关、流量控制和微服务管理层的功能，支撑工作负载对网络的要求。然而，Kong并非一个通用型解决方案。对于不同软件和企业级生态系统需求，Kong支持丰富插件提供认证，流量控制等功能。

k8s上部署Kong相对比较简单，但是将Kong服务整合到k8s是手动过程。这也就是为什么发布Kong Ingress Controller for Kubernetes。实现此功能后，Kong被嵌合进k8s生命周期。一旦应用和新服务创建，Kong自动生成配置来适配这些服务。
【图一】

秉承k8s哲学，即使用资源自己实现期望功能，而不是告诉服务实现功能的具体步骤。简单说，只定义最终状态而不是在集群上定义具体步骤。

当使用负载均衡器重启，重载时，自动配置会消耗大量资源更新路由。开源nginx ingress controller就是这样一个场景（每次配置文件更新时需要重新载入）。在一个高可用动态环境，配置重载，nginx重新配置后重新载入配置文件有可能造成宕机或者无效路由。开源版本Kong和Kong Ingress Controller内置管理层和API，在线配置目标，通过一个高可用横向扩展状态存储（可能是Postgres或者Cassandra）确保Kong实例无延迟地同步。

## 配置Kong Ingress Controller

下一步，将展示Kong Ingress Controller易用性。有一个[Github示例](https://github.com/Kong/kubernetes-ingress-controller/blob/master/deploy/README.md)，一步步演示如何去做。也可以跟随Marco Palladino，Kong CTO和Co-Founder在视频中演示的具体步骤。
【视频】

<p><a href="https://konghq.com/blog/kubernetes-ingress-controller-for-kong/?wvideo=031s5hisu2"><img src="https://embedwistia-a.akamaihd.net/deliveries/612848cb8c1cff658300a48712c188a5a91bc3f2.jpg?image_play_button_size=2x&amp;image_crop_resized=960x540&amp;image_play_button=1&amp;image_play_button_color=54bbffe0" width="400" height="225" style="width: 400px; height: 225px;"></a></p><p><a href="https://konghq.com/blog/kubernetes-ingress-controller-for-kong/?wvideo=031s5hisu2">Announcing the Kubernetes Ingress Controller for Kong</a></p>

安装步骤基本就是安装所需的k8s manifests，例如前端服务：ingress controller部署，Kong需要跟所有RBAC模块打交道以便正常访问k8s API路径。

这些menifests可以在任意k8s集群上工作。一旦开始，建议使用[minikube](https://github.com/kubernetes/minikube)开发。minikube是可以运行在个人电脑虚机啥个，官方提供的单节点k8s集群，作为应用开发者来说最容易的切入点。

安装步骤很容易：
{{{curl https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml \
| kubectl create -f -}}}

安装Kong Ingress Controller，并部署服务和ingress资源后，就可以利用Kong将服务请求转发到集群资源上。我们可以部署一个测试服务，其功能就是将pod头和基础信息发回。

{{{curl https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/manifests/dummy-application.yaml \
| kubectl create -f -}}}

部署应用后，需要Ingress资源提供访问压力，可以用如下manifest向测试服务模拟生成负载

{{{$ echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
name: foo-bar
spec:
rules:
- host: foo.bar
http:
paths:
- path: /
backend:
serviceName: http-svc
servicePort: 80
" | kubectl create -f -}}}

到这一步，可以开始向负载输出测试压力：

{{{$ export PROXY_IP=$(minikube   service -n kong kong-proxy --url --format "{{ .IP }}" | head -1)
$ export HTTP_PORT=$(minikube  service -n kong kong-proxy --url --format "{{ .Port }}" | head -1)

$ curl -vvvv $PROXY_IP:$HTTP_PORT -H "Host: foo.bar"}}}

## 添加插件
Kong Ingress Controller中的[插件](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/custom-types.md)暴露为Custom Resource Definitions(CRDs)，[CRDs](https://kubernetes.io/docs/concepts/api-extension/custom-resources/)在k8s API服务中属于第三方API对象，操作员可以定义，允许随机数据在可以被用在客制化控制回路中，例如the Kong Ingress Controllers.''

在我们的示例中加入码率控制插件，并将其捆绑为Ingress的工具。插件跟ingress资源是一一对应的，使得我们通过使用插件对应用具有细粒度控制。
{{{
$ echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
name: add-ratelimiting-to-route
config:
hour: 100
limit_by: ip
second: 10
" | kubectl create -f -

$ kubectl patch ingress foo-bar \
-p '{"metadata":{"annotations":{"rate-limiting.plugin.konghq.com":"add-ratelimiting-to-route\n"}}}'

}}}
使用cURL访问服务端点，得到如下响应：

{{{
    $ curl -vvvv $PROXY_IP:$HTTP_PORT -H "Host: foo.bar"

> GET / HTTP/1.1

> Host: foo.bar

> User-Agent: curl/7.54.0

> Accept: */*

>

< HTTP/1.1 200 OK

< Content-Type: text/html; charset=ISO-8859-1

< Transfer-Encoding: chunked

< Connection: keep-alive

< X-RateLimit-Limit-hour: 100

< X-RateLimit-Remaining-hour: 99

< X-RateLimit-Limit-second: 10

< X-RateLimit-Remaining-second: 9
}}}

立刻看出码率限制的头通过服务端点生效了。如果部署另外一个服务或者ingress，码率插件不会被使用，除非创建一个新的自带配置资源的KongPlugin，将注释加入新ingress中。

到这里，我们有个全功能API网关ingress栈，并实现了一个实例。可以在自己的微服务中尝试使用Kong。

关注Kong ingress controller，可以在[Github库](https://github.com/Kong/kubernetes-ingress-controller)中报告问题或者请求功能。期望在这里与你沟通。
