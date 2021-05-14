---
title: 2020年函数计算的冷启动怎么样了
date: 2020.6.13
tags: [Serverless, cold start]
categories: [Serverless, 闲谈]
---

## 前言

自从Serverless架构被提出，函数计算这个名词变得越发的火热，甚至在很多时候有人会认为Serverless就是函数计算。

作为Serverless架构中的一个重要组成部分，云函数确实值得，也应该备受关注，无论是吐槽他的调试能力，还是抱怨他的冷启动，亦或者对他的弹性伸缩表示怀疑，但是我们不得不承认，更多人正在越来越关注Serverless，也越来越关注Serverless中的FaaS部分。

经常看到有人在吐槽函数计算的冷启动问题，时至今日，不知道各平台的冷启动是什么样子的。本文将会通过相对客观的数据来进行基本的验证。

## 冷启动验证

首先说到冷启动，就要先说明什么是冷启动，开发者提交代码之后你不知道他调不调用，函数第一次调用会有一个函数冷启动，把网络的环境全部打通，这个函数才能提供服务。如果没有优化好冷启动优化这部分，可能对于一些比较关键的产品首次启动会产生超时，体验非常不好，以前开发者本地运行函数的时候，并不会关注本地函数执行多少毫秒和微妙，但是在云函数场景下就不一样了，云函数有一个部署的过程；无论是公有云的平台上还是开源方案上，冷启动都是值得不断探讨话题和优化的方向。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-6.png)

在《Serverless: Cold Start War》这篇文章中，作者对AWS Lambda，Azure Function以及Google Cloud Function等三个工业级的Serverless架构产品的冷启动测试。作者将函数启动划分成四个部分：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-8.png)

然后作者通过对多种语言的“Hello World”与是否有依赖等进行搭配，进行测试，测试结果：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-9.png)

通过《Understanding AWS Lambda Performance—How Much Do Cold Starts Really Matter?》与《Serverless: Cold Start War》这两个文章的分析和结果，我们可以看到冷启动问题确实存在，而且不同厂商，不同语言，不同测试方法得到的冷启动数据都是有所差异的。这也充分说明，各个厂商也在通过一些规则和策略努力降低冷启动率。除此之外文章《Understanding Serverless Cold Start》、《Everything you need to know about cold starts in AWS Lambda》、《Keeping Functions Warm》、《I'm afraid you're thinking about AWS Lambda cold starts all wrong》等也均对冷启动现象等进行描述和深入的探讨，并且提出了一些业务侧应对函数冷启动的解决方案和策略。

再说回来，时至今日，主流云厂商都已经开始开发探索Serverless架构，那么各个云厂商的冷启动已经"热"到了什么程度？

由于我是国内的开发者，所以我将测试分为两部分，一部分是国内云厂商（腾讯云、阿里云、华为云），另一部分是国外云厂商（AWS、谷歌）。

由于在实际生产过程中，冷启动的诞生往往是和API网关共同体现，也就是说，实际上让用户感知相对明显，或者比较常见感知到冷启动出现的情况，通常是函数与网关结合，做了一个接口/服务，访问该服务的时候，放大/体现了冷启动。

所以，我的做法很简单和暴力，通过函数与API网关触发器的结合，针对不同厂商来创建一个API服务，通过本地机器来对该服务进行访问，在客户端判断其耗时。当然，这种做法诚然不能精确的表现出冷启动的具体数据，因为网络因素也将会是影响其准确性的一个重要因素，但是至少可以大概的对比出，不同云厂商的冷启动优化情况，以及服务稳定情况。

国内的云厂商，在测试的时候更多使用的是国内区，国外云厂商则是国外区，这样就会出现一个额外的问题，国内外云厂商的数据不具有对比性，毕竟网络因素占了很大的一部分，所以本次对比将会是国内和国内对比，国外和国外对比。

客户端程序：

```python
import time, json
import urllib.request
import matplotlib.pyplot as plt
import numpy
from multiprocessing import Process, Manager


# 测试地址
# qcloud
url = "https://service-px5f98f4-1256773370.gz.apigw.tencentcs.com/release/scf_demo"
# # huaweicloud
# url = "https://2937587fe6ce4e6eb22d521d1d9b811c.apig.cn-east-2.huaweicloudapis.com/demo"
# # aliyun
# url = "https://50155512.cn-shanghai.fc.aliyuncs.com/2016-08-15/proxy/guide-hello_world/mydemo/"
# # aws
# url = "https://di6vbxf2lk.execute-api.us-east-1.amazonaws.com/default/mydemo"
# # GoogleCloud
# url = "https://us-central1-meta-imagery-277209.cloudfunctions.net/mydemo"

# 此时
times = 200

# 串行处理
serialColdStart = []
serialHotStart = []

for i in range(0,times):
    timeStart = time.time()
    responseAttr = urllib.request.urlopen(url)
    endTime = time.time()

    response = json.loads(responseAttr.read().decode("utf-8"))

    if response['isNew']:
        serialColdStart.append(endTime-timeStart)
    else:
        serialHotStart.append(endTime-timeStart)


# 并行处理
def worker(url, return_list):
    timeStart = time.time()
    responseAttr = urllib.request.urlopen(url)
    endTime = time.time()
    return_list.append({
        "duration": endTime-timeStart,
        "response": json.loads(responseAttr.read().decode("utf-8"))
    })

manager = Manager()
return_list = manager.list()
jobs = []
for i in range(times):
    p = Process(target=worker, args=(url ,return_list))
    jobs.append(p)
    p.start()

for proc in jobs:
    proc.join()

parallelColdStart = []
parallelHotStart = []
for eveData in return_list:
    if eveData['response']['isNew']:
        parallelColdStart.append(eveData['duration'])
    else:
        parallelHotStart.append(eveData['duration'])


# 数据汇总
print("-"*10, "串行测试", "-"*10)
print("总触发次数：", len(serialColdStart) + len(serialHotStart))
print("冷启动次数：", len(serialColdStart))
print("热启动次数：", len(serialHotStart))
print("最大耗时量：", max(serialColdStart + serialHotStart))
print("最小耗时量：", min(serialColdStart + serialHotStart))
print("平均耗时量：", numpy.mean(serialColdStart + serialHotStart))

print("-"*10, "并行测试", "-"*10)
print("总触发次数：", len(parallelColdStart) + len(parallelHotStart))
print("冷启动次数：", len(parallelColdStart))
print("热启动次数：", len(parallelHotStart))
print("最大耗时量：", max(parallelColdStart + parallelHotStart))
print("最小耗时量：", min(parallelColdStart + parallelHotStart))
print("平均耗时量：", numpy.mean(parallelColdStart + parallelHotStart))

plt.figure(figsize=(15,10))
plt.subplot(4, 2, 1)
plt.title('(Serial) Cold Start Time')
plt.plot(range(0, len(serialColdStart)), serialColdStart)
plt.subplot(4, 2, 3)
plt.title('(Serial) Cold Start Time')
plt.hist(serialColdStart, bins=20)
plt.subplot(4, 2, 5)
plt.title('(Serial) Hot Start Time')
plt.plot(range(0, len(serialHotStart)), serialHotStart)
plt.subplot(4, 2, 7)
plt.title('(Serial) Hot Start Time')
plt.hist(serialHotStart, bins=20)
plt.subplot(4, 2, 2)
plt.title('(Parallel) Hot Start Time')
plt.plot(range(0, len(parallelColdStart)), parallelColdStart)
plt.subplot(4, 2, 4)
plt.title('(Parallel) Cold Start Time')
plt.hist(parallelColdStart, bins=20)
plt.subplot(4, 2, 6)
plt.title('(Parallel) Hot Start Time')
plt.plot(range(0, len(parallelHotStart)), parallelHotStart)
plt.subplot(4, 2, 8)
plt.title('(Parallel) Hot Start Time')
plt.hist(parallelHotStart, bins=20)
plt.show()import time, json
import urllib.request
import matplotlib.pyplot as plt
import numpy
from multiprocessing import Process, Manager


# 测试地址
# qcloud
url = "https://service-px5f98f4-1256773370.gz.apigw.tencentcs.com/release/scf_demo"
# # huaweicloud
# url = "https://2937587fe6ce4e6eb22d521d1d9b811c.apig.cn-east-2.huaweicloudapis.com/demo"
# # aliyun
# url = "https://50155512.cn-shanghai.fc.aliyuncs.com/2016-08-15/proxy/guide-hello_world/mydemo/"
# # aws
# url = "https://di6vbxf2lk.execute-api.us-east-1.amazonaws.com/default/mydemo"
# # GoogleCloud
# url = "https://us-central1-meta-imagery-277209.cloudfunctions.net/mydemo"

# 此时
times = 200

# 串行处理
serialColdStart = []
serialHotStart = []

for i in range(0,times):
    timeStart = time.time()
    responseAttr = urllib.request.urlopen(url)
    endTime = time.time()

    response = json.loads(responseAttr.read().decode("utf-8"))

    if response['isNew']:
        serialColdStart.append(endTime-timeStart)
    else:
        serialHotStart.append(endTime-timeStart)


# 并行处理
def worker(url, return_list):
    timeStart = time.time()
    responseAttr = urllib.request.urlopen(url)
    endTime = time.time()
    return_list.append({
        "duration": endTime-timeStart,
        "response": json.loads(responseAttr.read().decode("utf-8"))
    })

manager = Manager()
return_list = manager.list()
jobs = []
for i in range(times):
    p = Process(target=worker, args=(url ,return_list))
    jobs.append(p)
    p.start()

for proc in jobs:
    proc.join()

parallelColdStart = []
parallelHotStart = []
for eveData in return_list:
    if eveData['response']['isNew']:
        parallelColdStart.append(eveData['duration'])
    else:
        parallelHotStart.append(eveData['duration'])


# 数据汇总
print("-"*10, "串行测试", "-"*10)
print("总触发次数：", len(serialColdStart) + len(serialHotStart))
print("冷启动次数：", len(serialColdStart))
print("热启动次数：", len(serialHotStart))
print("最大耗时量：", max(serialColdStart + serialHotStart))
print("最小耗时量：", min(serialColdStart + serialHotStart))
print("平均耗时量：", numpy.mean(serialColdStart + serialHotStart))

print("-"*10, "并行测试", "-"*10)
print("总触发次数：", len(parallelColdStart) + len(parallelHotStart))
print("冷启动次数：", len(parallelColdStart))
print("热启动次数：", len(parallelHotStart))
print("最大耗时量：", max(parallelColdStart + parallelHotStart))
print("最小耗时量：", min(parallelColdStart + parallelHotStart))
print("平均耗时量：", numpy.mean(parallelColdStart + parallelHotStart))

plt.figure(figsize=(15,10))
plt.subplot(4, 2, 1)
plt.title('(Serial) Cold Start Time')
plt.plot(range(0, len(serialColdStart)), serialColdStart)
plt.subplot(4, 2, 3)
plt.title('(Serial) Cold Start Time')
plt.hist(serialColdStart, bins=20)
plt.subplot(4, 2, 5)
plt.title('(Serial) Hot Start Time')
plt.plot(range(0, len(serialHotStart)), serialHotStart)
plt.subplot(4, 2, 7)
plt.title('(Serial) Hot Start Time')
plt.hist(serialHotStart, bins=20)
plt.subplot(4, 2, 2)
plt.title('(Parallel) Hot Start Time')
plt.plot(range(0, len(parallelColdStart)), parallelColdStart)
plt.subplot(4, 2, 4)
plt.title('(Parallel) Cold Start Time')
plt.hist(parallelColdStart, bins=20)
plt.subplot(4, 2, 6)
plt.title('(Parallel) Hot Start Time')
plt.plot(range(0, len(parallelHotStart)), parallelHotStart)
plt.subplot(4, 2, 8)
plt.title('(Parallel) Hot Start Time')
plt.hist(parallelHotStart, bins=20)
plt.show()
```


### 国内云厂商

#### 腾讯云

测试代码：

```python
# -*- coding: utf8 -*-
import json
import time
import uuid

requestId = None
containeId = str(uuid.uuid1())
createTime = time.time()

def main_handler(event, context):
    time.sleep(1)
    tempId = str(uuid.uuid1())
    timeStart = time.time()
    global requestId
    if not requestId:
        requestId = tempId

    response = {
            "isNew": True,
            "oldRequestId": requestId,
            "newRequestId": tempId,
            "duration": time.time() - timeStart,
            "containeId": containeId,
            "createTime": createTime
        }

    response["isNew"] =  True if requestId == tempId else False
    return response

```

输出数据：

```text
---------- 串行测试 ----------
总触发次数： 200
冷启动次数： 8
热启动次数： 192
最大耗时量： 2.3507158756256104
最小耗时量： 1.0568928718566895
平均耗时量： 1.134293702840805
---------- 并行测试 ----------
总触发次数： 186
冷启动次数： 170
热启动次数： 16
最大耗时量： 14.849930047988892
最小耗时量： 1.092796802520752
平均耗时量： 7.125524929774705

```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-1.png)


#### 阿里云

测试代码：

```python
# -*- coding: utf8 -*-
import json
import time
import uuid

requestId = None
containeId = str(uuid.uuid1())
createTime = time.time()

class AppClass:
    """Produce the same output, but using a class
    """
    def __init__(self, environ, start_response, response):
        self.environ = environ
        self.start = start_response
        self.response = response
    def __iter__(self):
        status = '200'
        response_headers = [('Content-type', 'text/html;charset=utf-8')]
        self.start(status, response_headers)
        yield  self.response.encode("utf-8")

def handler(environ, start_response):
    time.sleep(1)
    tempId = str(uuid.uuid1())
    timeStart = time.time()
    global requestId
    if not requestId:
        requestId = tempId

    response = {
            "isNew": True,
            "oldRequestId": requestId,
            "newRequestId": tempId,
            "duration": time.time() - timeStart,
            "containeId": containeId,
            "createTime": createTime
        }

    response["isNew"] =  True if requestId == tempId else False
    return AppClass(environ, start_response, json.dumps(response))
    
```

输出数据：

```text
---------- 串行测试 ----------
总触发次数： 200
冷启动次数： 1
热启动次数： 199
最大耗时量： 1.8273499011993408
最小耗时量： 1.1592700481414795
平均耗时量： 1.221251163482666
---------- 并行测试 ----------
总触发次数： 200
冷启动次数： 163
热启动次数： 37
最大耗时量： 3.184391975402832
最小耗时量： 1.1983528137207031
平均耗时量： 2.3849029302597047
```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-2.png)

#### 华为云

测试代码：

```python
# -*- coding: utf8 -*-
import json
import time
import uuid
requestId = None
containeId = str(uuid.uuid1())
createTime = time.time()

def handler(event, context):
    time.sleep(1)
    tempId = str(uuid.uuid1())
    timeStart = time.time()
    global requestId
    if not requestId:
        requestId = tempId

    response = {
            "isNew": True,
            "oldRequestId": requestId,
            "newRequestId": tempId,
            "duration": time.time() - timeStart,
            "containeId": containeId,
            "createTime": createTime
        }

    response["isNew"] =  True if requestId == tempId else False
    return json.dumps({
            'statusCode': 200,
            'isBase64Encoded': False,
            'headers': {
                "Content-type": "text/html; charset=utf-8"
            },
            'body': json.dumps(response),
        })

```

输出数据：

```text
---------- 串行测试 ----------
总触发次数： 200
冷启动次数： 1
热启动次数： 199
最大耗时量： 2.4535348415374756
最小耗时量： 1.202908992767334
平均耗时量： 1.4574852859973908
---------- 并行测试 ----------
总触发次数： 200
冷启动次数： 72
热启动次数： 128
最大耗时量： 3.8169281482696533
最小耗时量： 1.232532024383545
平均耗时量： 2.3244904506206514
```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-3.png)

### 国外云厂商


#### AWS

测试代码：

```python
# -*- coding: utf8 -*-
import json
import time
import uuid

requestId = None
containeId = str(uuid.uuid1())
createTime = time.time()

def lambda_handler(event, context):
    time.sleep(1)
    tempId = str(uuid.uuid1())
    timeStart = time.time()
    global requestId
    if not requestId:
        requestId = tempId

    response = {
            "isNew": True,
            "oldRequestId": requestId,
            "newRequestId": tempId,
            "duration": time.time() - timeStart,
            "containeId": containeId,
            "createTime": createTime
        }

    response["isNew"] =  True if requestId == tempId else False
    return {
        'statusCode': 200,
        'body': json.dumps(response)
    }
```

输出数据：

```text
---------- 串行测试 ----------
总触发次数： 200
冷启动次数： 1
热启动次数： 199
最大耗时量： 6.628237009048462
最小耗时量： 1.917238712310791
平均耗时量： 2.1634005284309388
---------- 并行测试 ----------
总触发次数： 200
冷启动次数： 176
热启动次数： 24
最大耗时量： 6.071150779724121
最小耗时量： 1.9705779552459717
平均耗时量： 2.370948977470398
```
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-5.png)

#### Google Cloud

测试代码：

```python
# -*- coding: utf8 -*-
import json
import time
import uuid

requestId = None
containeId = str(uuid.uuid1())
createTime = time.time()

def main_handler(event):
    time.sleep(1)
    tempId = str(uuid.uuid1())
    timeStart = time.time()
    global requestId
    if not requestId:
        requestId = tempId

    response = {
            "isNew": True,
            "oldRequestId": requestId,
            "newRequestId": tempId,
            "duration": time.time() - timeStart,
            "containeId": containeId,
            "createTime": createTime
        }

    response["isNew"] =  True if requestId == tempId else False
    return json.dumps(response)
```

输出数据：

```text
---------- 串行测试 ----------
总触发次数： 200
冷启动次数： 1
热启动次数： 199
最大耗时量： 4.707853078842163
最小耗时量： 1.226269006729126
平均耗时量： 1.3416448163986205
---------- 并行测试 ----------
总触发次数： 200
冷启动次数： 198
热启动次数： 2
最大耗时量： 7.694962024688721
最小耗时量： 1.296091079711914
平均耗时量： 5.523866602182388
```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-7.png)


### 测试结果

通过对上面的数据进行基本分析，可以作图：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-5-10.png)

通过这个表格可以看到，由于我是在国内测试的aws和google cloud，而且还是国外区域，所以网络因素对其影响蛮大的。

## 总结

函数冷启动问题确实是在项目中常见的一个现象，我做了一个微信公众号，后台绑定了两个函数，冷启动可怕到每次遇到冷启动，公众号额后台服务都会被微信判定为"故障，无法提供服务"，在实际项目中，冷启动不能说是地球毁灭，但是也应该是一场大灾难。

我始终认为，一个项目对开发者再友好，提供再多的功能/能力都是建立起来的高楼，如果这个高楼的地基不稳定，那么这高楼注定坍塌。