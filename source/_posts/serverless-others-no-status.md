---
title: Serverless架构中的无状态性指的是什么?
date: 2020.7.1
tags: [Serverless, 无状态性]
categories: [Serverless, 闲谈]
---


## 前言

接触Serverless架构的人，或者说接触函数计算的人，很多都会听过这样一句话：Serverless是无状态。

众所周知，无状态就是没有状态的意思，也就是说我们没办法用它保存状态，因为用完即销毁。那么这句话是不是说，在Serverless架构下（此处特指FaaS平台），函数的前一次运行和这一次运行，不会有联系呢？或者前一次运行的结果不会影响这一次呢？这里的无状态指的是什么？

## 函数的无状态探索

首先要明白，Serverless的几个关键特性：运行成本更低、自动扩缩容、事件驱动、无状态性。这里面的无状态性是说开发者可以直接将服务业务逻辑代码部署，运行在第三方提供的无状态计算容器中。这里的无状态如果强行说前一次不影响后一次，没有状态的话，也只能是说在容器没有被复用的情况下，是这样的。但是在实际的项目中，为了降低冷启动率，提高瞬时产生的高并发应对能力，容器的复用可能会让这个“无状态性“变得比较扑朔迷离，此处以腾讯云的SCF为例，我们在控制台创建一个函数，然后我们使用以下的代码进行测试：

```python
# -*- coding: utf8 -*-
import json
def main_handler(event, context):
    print("Test")
    return("Hello World")
```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-1.png)

我们可以看到，通过点击测试按钮，输出了日志：`Test`，接下来，我们多次点击：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-2.png)

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-3.png)

可以看到，随着我们点击测试按钮，每次都在日志准确输出了`Test`。接下来，我们变换一下代码：

```python
# -*- coding: utf8 -*-
import json
print("Not in main_handler")
def main_handler(event, context):
    print("Test")
    return("Hello World")
```

接下来同样的方法,连续点击三次测试，并且记录结果：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-4.png)

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-5.png)

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-6.png)

通过这一组测试，我们发现，这三个结果有点不太一样：只有第一次请求的时候，执行了这条语句：

```python
print("Not in main_handler")
```

那么为什么后几次都没有执行这条语句呢？是没执行到这里？还是因为容器复用的原因，在接下来的几次跳过了这个步骤？为什么会跳过这个步骤？为了让程序更加有趣，我们来做这样一个测试：

```python
# -*- coding: utf8 -*-
import json
print("此处给tempNumber赋值")
tempNumber = 100
def main_handler(event, context):
    print("temp number: ", tempNumber)
    return("Hello World")
```

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-7.png)

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-8.png)

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-9.png)

可以看到，在第一次测试的时候，我们这个程序执行的时候，先执行了：

```python
print("此处给tempNumber赋值")
tempNumber = 100
```

执行完成之后，`tempNumber`这个变量就会存在，在接下来的几次调用中，都直接取了这个值。所以可以这样认为：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-10.png)

也就是说，实际上函数在复用容器的情况下被执行（或者说是被触发），实际上可以认为是已经有一个进程被启动，每次触发，是通过这个进程来调用我们的入口方法，所以在方法之外写的各种操作，实际上是冷启动的时候，在启动进程的时候，会被执行。

所以说，实际上函数的无状态性，并不是说函数的前一次操作对后一次被触发没有影响。那么所谓的无状态是什么？

在CNCF发布的serverlss白皮书中，这样描述过Serverless架构的优点：Serverless架构通常是无状态、不可变和短暂的。每个函数都以指定的角色和明确定义有限的资源访问权限运行。同时在白皮书中，也说了什么样的程序或者服务适合Serverless架构，其中有这样一个描述：无状态，短暂的，对瞬间冷启动时间没有过多需求的程序适合使用Serverless架构。

所以说，这里的函数是无状态实际上可以认为是：函数是运行在第三方提供的无状态计算容器中的，并且在无复用的情况下，函数会存在冷启动，这个时候函数可以认为是无状态；因为各个厂商的不同容器降低冷启动方案，以及容器复用方案也都是未公开的，所以什么时候可能会复用这个容器，怎么复用也是未知的，这就要求我们函数的功能本身要保证是无状态的。例如说，在函数中，保存某些数据到缓存中，下次触发的时候从缓存中获得对应内容就是容易产生异常的操作，因为云厂商无法保证这次请求，是否复用了已有容器，以及复用的已有容器是否就是上次进行缓存的容器。

## 拓展

那么根据我们上面讨论的内容，在进行实践化的应用：

1. 通过容器复用，做一些初始化操作
刚刚说过了，如果在容器复用的前提下，那么在函数外面执行的内容是可以直接使用的，所以这里我们实际上是可以在外层进行一些初始化的，例如：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-1-11.png)
以上图的代码为例，通过这样的初始化，就不用每次调用函数的时候，都进行一次数据库的初始化/链接等而是可以复用已有的链接，如果在main_handler中进行数据库的初始化/链接，会影响函数性能，在高并发的情况下更容易把数据库的链接打满，造成极其恶劣的影响。

2. 小心容器复用不要掉进坑里
之前写了一个SCF打包Python依赖的小工具，运行在SCF中，我在测试之处是好好的，但是项目上线之后，我发现这样一个问题：只有冷启动的情况下，依赖是可以被打包的，如果出现容器复用的情况，就会出现依赖打包失败的问题。
经过仔细排查才发现，实际上是一个对象在使用完成之后未被清理，由于容器是被复用，或者说是“这个对象也被复用了”，在执行指定方法的时候，看到对象已存在，就会直接用这个对象，导致本次函数的触发使用了上次残留的对象，导致异常的发生。
所以说，当我们的程序在云函数中，连续执行多次的时候，开始成功后来失败，很可能就是由于某些资源复用，导致我们程序出错。

3. 我就想要一种状态
有的人在使用云函数的时候，可能真的就需要有一种状态来记录某些事情，例如我的博客系统判断管理员用户是否登录。本来可以直接放到缓存中的操作，此时不能放进去，那应该怎么处理，我怎么记录管理员是否已经登陆了后台，或者说我怎么确定这个用户是否是管理员？
这种情况其实就比较常见了，我们完全可以融合两套方案：
	* 方案1: 采用Token机制
	* 方案2: 采用缓存机制
所谓的采用Token机制和缓存机制融合方案，就是说管理员用户登陆之后，会生成一个Token，这个Token就记录到数据库中，同时这个Token也会被写到缓存中。当用户请求发起后，函数先尝试在缓存中获取结果，如果没获得到，就连接数据库进行获取。



## 总结

Serverless架构可以被看成是一个新的技术，一种新的框架，很多时候，我们真的不能用已有的态度去衡量这样的新鲜事物。同样，一个特性也很难直接用好坏去形容。就这个无状态性来说真的是有几种钟爱，有几种迷茫。