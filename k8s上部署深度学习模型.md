# 用Python/Keras/Flask/Docker在k8s上部署深度学习模型#

## 简单到老板也可以亲自部署 ##
这篇博文演示了如何通过Docker和k8s，用Keras部署深度学习模型，并且通过Flask提供REST API服务。

这个模型并不是强壮可供生产的模型，而是给k8s新手一个尝试的机会。我在Google Cloud上部署了这个模型，而且工作的很好。另外用户可以用同样的步骤重现以上功能。如果用户担心成本，Google提供了大量免费机会，这个演示基本没有花钱。

## 为什么用k8s来做机器学习和数据科学 ##
k8s,也叫cloud-native，正在席卷整个世界，而我们正在感受它，我们正处在一个由AI/Big Data/Cloud驱动的技术风暴中心，k8s也正在加入这个中心。

但是如果从数据科学角度看并没有使用k8s的特殊原因。但是从部署，扩展和管理REST API方面来看，k8s正在实现简易化的特性。

## 步骤预览 ##
1. 在 google cloud上创建用户
2. 使用keras/Flask/Docker搭建一个REST API的机器学习模型服务
3. 用k8s部署上述模型
4. enjoy it

### 步骤一 在google Cloud上创建用户 ###
我再Google Compute Engine上创建了一个对外提供服务的容器化深度学习模型，当然Google平台并不是必须的，只要能够安装Docker，随便选择平台模式。

【图片一】

进入google云平台，点击左侧屏幕选择Compute Engine，启动Google Cloud VM。然后选择“Create Instance”，可以看到已经运行的实例。
【图片二】

下一步选择计算资源。默认设置就足够，因为只是演示，我选择了4vCPUs和15G内存。
【图片三】

选择操作系统和磁盘大小。我选择了Centos7，100G硬盘。建议磁盘大于10G，因为每个Docker容器有1G大小。
【图片四】

最后一步是配置允许HTTP/S工作的防火墙策略。建议选择全部透明，以便减少麻烦。
【图片五】

选择“Create”，一切进展顺利。
【图片六】

### 步骤二 用keras创建深度学习模型 ###
SSH登录到虚机开始建立模型。最简单方式就是点击虚机下方的SSH图标，会在浏览器中打开一个终端。
【图片七】

1. 删除预装Docker
`sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine`

2. 安装最新Docker版本
```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager — add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce 
```

3. 启动容器运行测试脚本
```
sudo systemctl start docker
sudo docker run hello-world
```
以下是正确输出
>Hello from Docker!
This message shows that your installation appears to be working correctly.To generate this message, Docker took the following steps: 1. The Docker client contacted the Docker daemon. 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.    (amd64) 3. The Docker daemon created a new container from that image which runs the    executable that produces the output you are currently reading. 4. The Docker daemon streamed that output to the Docker client, which sent it    to your terminal.

4. 创建深度学习模型
这里会借用Adrian Rosebrock的一个脚本，他提供了使用keras的深度学习模型并通过Flask提供服务的教程，可以从[这里](https://blog.keras.io/building-a-simple-keras-deep-learning-rest-api.html)访问。

这个模型可以直接执行。但是我修改了两个配置信息：
首先，改变了容器配置，默认flask使用127.0.0....作为默认服务地址，这会在容器内部运行时出现问题。我将它修改成0.0.0.0，这样就可以实现对外和对内都可以工作的IP地址。

第二是关于Tensorflow的配置，可以从Github中找到这个[问题描述](https://github.com/tensorflow/tensorflow/issues/14356)。
```
global graph
graph = tf.get_default_graph()
...
with graph.as_default():
    preds = model.predict(image)
```

运行脚本，首先创建专用目录：
```
mkdir keras-app
cd keras-app
```

创建app.py文件：
`vim app.py`

```
# USAGE
# Start the server:
#   python app.py
# Submit a request via cURL:
#   curl -X POST -F image=@dog.jpg 'http://localhost:5000/predict'

# import the necessary packages
from keras.applications import ResNet50
from keras.preprocessing.image import img_to_array
from keras.applications import imagenet_utils
from PIL import Image
import numpy as np
import flask
import io
import tensorflow as tf

# initialize our Flask application and the Keras model
app = flask.Flask(__name__)
model = None

def load_model():
    # load the pre-trained Keras model (here we are using a model
    # pre-trained on ImageNet and provided by Keras, but you can
    # substitute in your own networks just as easily)
    global model
    model = ResNet50(weights="imagenet")
    global graph
    graph = tf.get_default_graph()

def prepare_image(image, target):
    # if the image mode is not RGB, convert it
    if image.mode != "RGB":
        image = image.convert("RGB")

    # resize the input image and preprocess it
    image = image.resize(target)
    image = img_to_array(image)
    image = np.expand_dims(image, axis=0)
    image = imagenet_utils.preprocess_input(image)

    # return the processed image
    return image

@app.route("/predict", methods=["POST"])
def predict():
    # initialize the data dictionary that will be returned from the
    # view
    data = {"success": False}

    # ensure an image was properly uploaded to our endpoint
    if flask.request.method == "POST":
        if flask.request.files.get("image"):
            # read the image in PIL format
            image = flask.request.files["image"].read()
            image = Image.open(io.BytesIO(image))

            # preprocess the image and prepare it for classification
            image = prepare_image(image, target=(224, 224))

            # classify the input image and then initialize the list
            # of predictions to return to the client
            with graph.as_default():
                preds = model.predict(image)
                results = imagenet_utils.decode_predictions(preds)
                data["predictions"] = []

                # loop over the results and add them to the list of
                # returned predictions
                for (imagenetID, label, prob) in results[0]:
                    r = {"label": label, "probability": float(prob)}
                    data["predictions"].append(r)

                # indicate that the request was a success
                data["success"] = True

    # return the data dictionary as a JSON response
    return flask.jsonify(data)

# if this is the main thread of execution first load the model and
# then start the server
if __name__ == "__main__":
    print(("* Loading Keras model and Flask starting server..."
        "please wait until server has fully started"))
    load_model()
    app.run(host='0.0.0.0')
```    
```


5. 创建requirements.txt文件
为了在容器内运行代码，需要创建requirements.txt文件，其中包括需要运行的包，例如keras、flask、一起其它相关包。这样无论在哪里运行代码，依赖包都保持一致。

```
keras
tensorflow
flask
gevent
pillow
requests
```

6. 创建Dockerfile

```
FROM python:3.6
WORKDIR /app
COPY requirements.txt /app
RUN pip install -r ./requirements.txt
COPY app.py /app
CMD ["python", "app.py"]~
```

首先让容器自行下载Python3安装image，然后让python调用pip安装requirements.txt中的依赖包，最后运行python app.py。

7. 创建容器
`sudo docker build -t keras-app:latest .`

在keras-app目录下创建容器，后台开始安装python3 image等在步骤6中定义的操作。

8. 运行容器
`sudo docker run -d -p 5000:5000 keras-app`

用`sudo docker ps -a`检查容器状态，应该看到如下输出：
>[gustafcavanaugh@instance-3 ~]$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                    PORTS                    NAMES
d82f65802166        keras-app           "python app.py"     About an hour ago   Up About an hour          0.0.0.0:5000->5000/tcp   nervous_northcutt

9. 测试模型
现在可以测试此模型。用狗图像作为输入，可以返回狗类型。在Adrian的示例中都有[详细代码和图片](https://github.com/jrosebr1/simple-keras-rest-api/blob/master/dog.jpg)，我们也使用它们，并保存自工作目录下，命名为dog.jpg。

执行命令`curl -X POST -F image=@dog.jpg 'http://localhost:5000/predict'`
应该得到如下输出：
>{"predictions":[{"label":"beagle","probability":0.987775444984436},{"label":"pot","probability":0.0020967808086425066},{"label":"Cardigan","probability":0.001351703773252666},{"label":"Walker_hound","probability":0.0012711131712421775},{"label":"Brittany_spaniel","probability":0.0010085132671520114}],"success":true}

可以看到此模型成功将够归类为比格犬。下一步，我们用k8s部署容器模型。

### 第三步 用k8s部署模型 ###
1. 创建Docker Hub账号
第一步需要在[Docker hub](https://hub.docker.com/)上传模型，以便使用k8s集中管理。

2. 登录到Docker Hub
`sudo docker login`， 登录到Docker Hub，应该看到如下输出：
>Login Succeeded

3. 给容器打标签
给模型容器命名，上传前先给它打标签。

`sudo docker images`，应该得到容器的id，输出如下：
>REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE keras-app           latest              ddb507b8a017        About an hour ago   1.61GB

打标签命令如下：
```
#Format
sudo docker tag <your image id> <your docker hub id>/<app name>
#My Exact Command - Make Sure To Use Your Inputs
sudo docker tag ddb507b8a017 gcav66/keras-app
```

4. 将模型容器上传到Docker Hub
运行命令如下：
```
#Format
sudo docker push <your docker hub name>/<app-name>
#My exact command
sudo docker push gcav66/keras-app
```

5. 创建k8s集群
在Google Cloud Home界面，选择k8s Engine
【图片八】
创建新集群
【图片九】
选择集群内节点资源，因为要启动三个节点（每个节点4vCPU和15G内存），至少需要12vCPU和45G内存。
【图片十】
连接集群，google k8s自动会在VM上安装k8s。
【图片十一】
在k8s中运行容器， `kubectl run keras-app --image=gcav66/keras-app --port 5000`，确认是否pod正确运行`kubectl get pods`，输出如下：
>gustafcavanaugh@cloudshell:~ (basic-web-app-test)$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
keras-app-79568b5f57-5qxqk   1/1       Running   0          1m

为了安全起见，将服务端口暴露与80端口：
`kubectl expose deployment keras-app --type=LoadBalancer --port 80 --target-port 5000`
确认服务正常启动：`kubectl get service`，正常输出如下：
>gustafcavanaugh@cloudshell:~ (basic-web-app-test)$ kubectl get service
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
keras-app    LoadBalancer   10.11.250.71   35.225.226.94   80:30271/TCP   4m
kubernetes   ClusterIP      10.11.240.1    <none>          443/TCP        18m

提取cluster-IP，并将其合并于服务提交命令：`curl -X POST -F image=@dog.jpg 'http://<your service IP>/predict'`，得到正常输入如下：

>$ curl -X POST -F image=@dog.jpg 'http://35.225.226.94/predict'
{"predictions":[{"label":"beagle","probability":0.987775444984436},{"label":"pot","probability":0.0020967808086425066},{"label":"Cardigan","probability":0.001351703773252666},{"label":"Walker_hound","probability":0.0012711131712421775},{"label":"Brittany_spaniel","probability":0.0010085132671520114}],"success":true}


### 第四步 总结 ###
本文提供了一个使用Keras和Flask提供REST API服务的深度学习模型，并把它集成到容器内部，上传到Docker Hub，并用k8s部署，非常容易地实现了对外提供服务和访问。

当然会有大量的提高可能，对于初学者，可以改变本地python服务到更加强壮大gunicorn；可以横向扩展k8s，实现服务扩容；也可以从头搭建一套k8s环境。