# Kaniko介绍：在k8s内部生成容器映像以及不需特权的Google容器生成器

通常标准Dockerfile的生成需要与Docker后台进程交互访问，因此需要本机root权限。在Docker后台进程无法暴露的场景下（例如k8s集群，详细内容可以参见[如下内容](https://github.com/kubernetes/kubernetes/issues/1806)）生成容器映像就很困难。

kaniko就是为解决这类问题而生的，它是一个不许root特权就可以从Dockerfile中生成映像，并将映像推送到注册库的开源工具。因为kaniko不需要特权，因此用户可以在标准k8s集群、google k8s引擎、以及其它无法访问Docker后台进程环境中运行。

## kaniko工作原理

kaniko运行时被当做自带三个参数的容器，它们是：Dockerfile，创建上下文和最终映像需要上传入的注册库。映像是新创建的，只包含静态Go库和需要下拉上载映像的配置文件。
！[kinako原理图](https://github.com/hokingyang/learngit/blob/master/kinako%E5%8E%9F%E7%90%86.png)

kaniko执行器获取并展开基础映像（在Dockerfile中FROM一行定义），按顺序执行每条命令，每条命令执行完毕后为文件系统做快照。快照是在用户空间创建，并与内存中存在的上一个状态进行对比，任何改变都会作为对基础镜像的修改，并以新层级对文件系统进行增加扩充，并将任何修改都写入映像的元数据中。当Dockerfile中每条命令都执行完毕后，执行器将新生成的映像上载入注册库中。
Kaniko解压文件系统，执行命令，在执行器映像的用户空间中对文件系统做快照，都是为什么kaniko不需要特权访问的原因，以上操作中没有引入任何Docker后台进程或者CLI操作。

## 在k8s集群中运行kaniko

在标准k8s集群中运行kaniko，pod配置文件需要做如下修改。本例中，[google cloud storage](https://cloud.google.com/storage/) bucket提供创建上下文
{{{
    apiVersion: v1
kind: Pod
metadata:
 name: kaniko
spec:
 containers:
 - name: kaniko
   image: gcr.io/kaniko-project/executor:latest
   args: ["--dockerfile=<path to Dockerfile>",
           "--bucket=<GCS bucket>",
           "--destination=<gcr.io/$PROJECT/$REPO:$TAG"]
   volumeMounts:
     - name: kaniko-secret
       mountPath: /secret
   env:
     - name: GOOGLE_APPLICATION_CREDENTIALS
       value: /secret/kaniko-secret.json
 restartPolicy: Never
 volumes:
   - name: kaniko-secret
     secret:
       secretName: kaniko-secret
}}}

本例中，需要挂载k8s [secret](https://kubernetes.io/docs/concepts/configuration/secret/)（其中包含了将映像上载入注册库中所需的授权），可以参见[下载secret的方法](https://github.com/GoogleCloudPlatform/kaniko)

## 在Google Cloud Container Builder中运行kaniko

运行[Google Cloud Container Builder](https://cloud.google.com/container-builder/docs/)，可以在配置文件中定义如下步骤：
{{{
    steps:
 - name: gcr.io/kaniko-project/executor:latest
   args: ["--dockerfile=<path to Dockerfile>",
          "--context=<path to build context>",
          "--destination=<gcr.io/[PROJECT]/[IMAGE]:[TAG]>"]
}}}
kaniko执行器会根据步骤定义自动创建和上载映像。

## 与其它工具比较

与kaniko类似的工具包括[img](https://github.com/genuinetools/img)和[orca-build](https://github.com/cyphar/orca-build)。这些工具都是从Dockerfile开始生成映像，但是采用不同方法和安全策略。在非特权环境下，img以非特权用户身份在容器中生成映像，而kaniko则是以root用户身份在容器内生成映像。orca-build工具则是通过包装[runc](https://github.com/opencontainers/runc)，用内核空间技术执行RUN命令生成映像，kinako可以在容器内以root身份执行命令来实现同样的功能。

## 结论

在我们的[github库](https://github.com/GoogleCloudPlatform/kaniko)中可以找到很多相关文档，如果发现发现bug可以开问题记录。并可以在[google group](https://groups.google.com/forum/#!forum/kaniko-users)中找到我们。

