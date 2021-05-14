---
title: Serverless架构下用Python轻松搞定图像分类/预测
date: 2019.12.23
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework, 图像预测, 图像分类]
categories: [Serverless, 腾讯云, 领域实战]
---

## 前言

图像分类是人工智能领域的一个热门话题。通俗解释就是，根据各自在图像信息中所反映的不同特征，把不同类别的目标区分开来的图像处理方法。它利用计算机对图像进行定量分析，把图像或图像中的每个像元或区域划归为若干个类别中的某一种，以代替人的视觉判读。图像分类在实际生产生活中也是经常遇到的，而且针对不同领域或者需求有着很强的针对性。例如通过拍照花朵识别花朵信息，通过人脸匹对人物信息等。

通常情况下，这些以来图像识别或者分类的工具，都是在客户端进行数据采集，在服务端进行运算获得结果，也就是说一般情况下都是有专门的API实现图像识别的。例如各大云厂商都会为我们有偿提供类似的能力：

* 华为云图像标签
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-10.png)

* 腾讯云图像分析
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-11.png)

本文将会通过一个有趣的Python库，快速将图像分类的功能搭建在云函数上，并且和API网关结合，对外提供API功能，实现一个Serverless架构的"图像分类API"。

## 入门ImageAI

首先和大家据介绍一下需要的依赖库：`ImageAI`。

通过该依赖的官方文档我们可以看到这样的描述：

> ImageAI是一个python库，旨在使开发人员能够使用简单的几行代码构建具有包含深度学习和计算机视觉功能的应用程序和系统。
> ImageAI本着简洁的原则，支持最先进的机器学习算法，用于图像预测，自定义图像预测，物体检测，视频检测，视频对象跟踪和图像预测训练。ImageAI目前支持使用在ImageNet-1000数据集上训练的4种不同机器学习算法进行图像预测和训练。ImageAI还支持使用在COCO数据集上训练的RetinaNet进行对象检测，视频检测和对象跟踪。 最终，ImageAI将为计算机视觉提供更广泛和更专业化的支持，包括但不限于特殊环境和特殊领域的图像识别。
    
也就是说这个依赖库，可以帮助我们完成基本的图像识别和视频的目标提取，虽然他给了一些数据集和模型，但是我们也可以根据自身需要对其进行额外的训练，进行定制化拓展。通过官方给的代码，我们可以看到一个简单的Demo：

```python
from imageai.Prediction import ImagePrediction
import os
execution_path = os.getcwd()

prediction = ImagePrediction()
prediction.setModelTypeAsResNet()
prediction.setModelPath(os.path.join(execution_path, "resnet50_weights_tf_dim_ordering_tf_kernels.h5"))
prediction.loadModel()

predictions, probabilities = prediction.predictImage(os.path.join(execution_path, "1.jpg"), result_count=5 )
for eachPrediction, eachProbability in zip(predictions, probabilities):
    print(eachPrediction + " : " + eachProbability)
```

我们可以在本地进行初步运行，当我们指定图片`1.jpg`为下图的时候：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-12.png)

可以得到结果：

    convertible  :  52.459537982940674
    sports_car  :  37.61286735534668
    pickup  :  3.175118938088417
    car_wheel  :  1.8175017088651657
    minivan  :  1.7487028613686562

## 让ImageAI上云（部署到Serverless架构上）

通过上面的Demo我们可以考虑将这个模块部署到云函数：

* 首先，我们在本地创建一个Python的项目：`mkdir imageDemo`
* 然后新建文件：`vim index.py`
* 根据云函数的一些特殊形式，我们对上面的Demo进行部分改造
    * 将初始化的代码放在外层；
    * 将预测部分当道入口方法中（此处是main_handler）;
    * 云函数与API网关结合对二进制文件支持并不是十分的友善，所以此处通过base64进行图片传输；
    * 入参定为`{"picture": 图片的base64}`，出参定为：`{"prediction": 图片分类的结果}`

实现的代码如下：

```python
from imageai.Prediction import ImagePrediction
import os, base64, random, json

execution_path = os.getcwd()

prediction = ImagePrediction()
prediction.setModelTypeAsSqueezeNet()
prediction.setModelPath(os.path.join(execution_path, "squeezenet_weights_tf_dim_ordering_tf_kernels.h5"))
prediction.loadModel()


def main_handler(event, context):
    imgData = base64.b64decode(json.loads(event["body"])["pidcture"])
    fileName = '/tmp/' + "".join(random.sample('zyxwvutsrqponmlkjihgfedcba', 5))
    with open(fileName, 'wb') as f:
        f.write(imgData)
    resultData = {}
    predictions, probabilities = prediction.predictImage(fileName, result_count=5)
    for eachPrediction, eachProbability in zip(predictions, probabilities):
        resultData[eachPrediction] = eachProbability
    return resultData


```

创建完成之后，我们需要下载一下我们所依赖的模型： 

```
- SqueezeNet（文件大小：4.82 MB，预测时间最短，精准度适中）
- ResNet50 by Microsoft Research （文件大小：98 MB，预测时间较快，精准度高）
- InceptionV3 by Google Brain team （文件大小：91.6 MB，预测时间慢，精度更高）
- DenseNet121 by Facebook AI Research （文件大小：31.6 MB，预测时间较慢，精度最高）
```

此处，我们仅用于测试，所以我们只选择一个比较小的模型作为测试使用：`SqueezeNet`：

在官方文档复制模型文件地址：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-1.png)

使用`wget`直接安装：

```
wget https://github.com/OlafenwaMoses/ImageAI/releases/download/1.0/squeezenet_weights_tf_dim_ordering_tf_kernels.h5
```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-2.png)

接下来，我们就需要进行安装依赖了，这里面貌似安装的内容蛮多的：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-3.png)

由于腾讯云Serveless的产品，在Python Runtime中还不支持在线安装依赖，所以我们需要手动打包依赖，并且上传。在Python的各种依赖库中，有很多依赖可能有编译生成二进制文件的过程，这就可能导致不同环境下打包的依赖无法通用。

所以最好的方法就是通过对应的操作系统+语言版本进行打包，例如本例子就需要在CentOS+Python3.6的环境下进行依赖打包。

对于很多MacOS用户和Windows用户来说，这确实不是一个很友好的过程，所以在之前为了方便，我在Serverless架构上做了一个在线打包依赖的工具，所以这时候，直接用该工具进行打包：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-4.png)

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-5.png)

生成压缩包之后，直接下载解压，并且放到自己的项目中即可：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-6.png)

最后，一步了，我们创建`serverless.yaml`

```
imageDemo:
  component: "@serverless/tencent-scf"
  inputs:
    name: imageDemo
    codeUri: ./
    handler: index.main_handler
    runtime: Python3.6
    region: ap-guangzhou
    description: 图像识别/分类Demo
    memorySize: 256
    timeout: 10
    events:
      - apigw:
          name: imageDemo_apigw_service
          parameters:
            protocols:
              - http
            serviceName: serverless
            description: 图像识别/分类DemoAPI
            environment: release
            endpoints:
              - path: /image
                method: ANY
```

完成之后，执行我们的`sls --debug`部署，部署过程中会有扫码的登陆，登陆之后等待即可，完成之后，即可看到我们部署的地址。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-7.png)

## 基本测试

通过Python语言进行测试，接口地址就是我们刚才复制的+`/image`，例如：

```python
import json
import urllib.request
import base64

with open("1.jpg", 'rb') as f:
    base64_data = base64.b64encode(f.read())
    s = base64_data.decode()

url = 'http://service-9p7hbgvg-1256773370.gz.apigw.tencentcs.com/release/image'

print(urllib.request.urlopen(urllib.request.Request(
    url = url,
    data= json.dumps({'picture': s}).encode("utf-8")
)).read().decode("utf-8"))
```

通过网络搜索一张图片，例如我找了这个：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/2-2-8.png)

得到运行结果：

```json
{
	"prediction": {
		"cheetah": 83.12643766403198,
		"Irish_terrier": 2.315458096563816,
		"lion": 1.8476998433470726,
		"teddy": 1.6655176877975464,
		"baboon": 1.5562783926725388
	}
}
```
通过这个结果，我们可以看到成功的进行了图片的基础分类/预测，为了证明这个接口的时延情况，可以对程序进行基本改造：

```python
import urllib.request
import base64, time

for i in range(0,10):
    start_time = time.time()
    with open("1.jpg", 'rb') as f:
        base64_data = base64.b64encode(f.read())
        s = base64_data.decode()

    url = 'http://service-9p7hbgvg-1256773370.gz.apigw.tencentcs.com/release/image'
    
    print(urllib.request.urlopen(urllib.request.Request(
        url = url,
        data= json.dumps({'picture': s}).encode("utf-8")
    )).read().decode("utf-8"))

    print("cost: ", time.time() - start_time)
```

输出结果：

```
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  2.1161561012268066
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.1259253025054932
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.3322770595550537
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.3562259674072266
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.0180821418762207
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.4290671348571777
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.5917718410491943
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.1727900505065918
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  2.962592840194702
{"prediction":{"cheetah":83.12643766403198,"Irish_terrier":2.315458096563816,"lion":1.8476998433470726,"teddy":1.6655176877975464,"baboon":1.5562783926725388}}
cost:  1.2248001098632812
```

通过上面一组数据，我们可以看到，整体的耗时基本控制在1-1.5秒之间。

当然，如果想要对接口性能进行更多的测试，还可以进行并发测试等，来看并发情况下接口性能表现等。

至此，我们通过Serveerless架构搭建的Python版本的图像识别/分类小工具做好了。

## 总结

Serverless架构下进行人工智能相关的应用可以是说是非常多的，本文是通过一个已有的依赖库，实现一个图像分类/预测的接口。`imageAI`这个依赖库相对来说自由度比较高，可以根据自身需要用来定制化自己的模型。本文算是抛砖引玉，期待更多人通过Serverless架构部署自己的"人工智能"API。




