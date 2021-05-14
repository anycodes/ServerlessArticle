---
title: 入门Serverless Framework开发者工具
date: 2019.12.15
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework]
categories: [Serverless, 腾讯云, 基本介绍]
---

## 前言

Serverless架构是云发展的产物，是一种去服务器化更加明显的架构或者说是一种新技术，有人说Serverless才是真正的云计算，但是无论如何，Serverless确实在不断的发展，而且越来越火热。

细心的人都会发现，在Serverless发展的越发火热的过程中，有一个开发者工具也叫做Serverless，那么Serverless到底是一个架构，还是一个开发者工具呢？这个开发者工具和Serverless架构又有什么关系呢？

## 初探Serverless开发者工具

Serverless架构开始发展没多久，有这样一群人，注册了一个叫Serverless的公司，并且买了一个域名：serverless.com，同时他们又做了一个工具或者说是软件，也起名叫做Serverless！

就相当于我们说：近年，中国电信行业发展迅速。这里面的中国电信行业实际上说的是中国的电信行业，而不是说中国电信这个运营商。同样的道理，Serverless这个开发者工具和Serverless架构其实也是两个东西。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-1.png)

通过Serverless这个公司的名字我们大概可以猜到：他们推出的产品，自然而然就是和热点技术Serverless架构紧密相关！

在各个云厂商都有自己函数计算业务的时候，Serverless团队做了一个类似多云管理平台的工具，可以认为是多Serverless管理的工具。通过这个工具，你可以快速直接使用AWS的Lambda，Azure的Funtions以及腾讯云SCF等众多云厂商的函数计算相关服务，大体支持的功能如下（部分工业化的云厂商）：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-2.png)


通过这个表，大家可以看到，其实这个公司开发的这个工具，或者说开发的这个软件实际上就是一个开发者工具， 帮助我们可以快速使用多个云厂商的函数服务：帮大家打包、部署、回滚......当然，各个厂商也都有自己的类似的工具，例如AWS的SAM，腾讯云的SCFCLI等。

除了一个以函数计算为核心的多云开发者工具之外，这个公司还推出了组件化工具：Components。就是说，Serverless这个开发者工具不仅仅关注Serverless中的FaaS，也要关注Serverless中的BaaS，即将API网关，对象存储，CDN，数据库......众多的后端服务和函数计算有机集合，让用户可以一站式开发，一站式部署，一站式更新，一站式维护！

所以，这个叫做Serverless Framework的开发者工具被一分为二：Plugin和Components。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-3.png)

如果说最初的Serverless Cli更多是一种以插件（Plugin）形式提供各个云厂商的函数计算功能，那么这个叫Components的功能更多就是以各个云厂商整体服务为基础，来帮助用户快速将项目部署到Serverless架构上。

所谓的Components可以认为是很多的Component的组合，就是很多组件的组合，例如我们要部署一个网站，可能会有几个部分：静态资源部分、函数计算部分、API网关部分、CDN部分、域名解析部分等，那么这个Components就可以同时帮我们一站式部署这些资源，例如将静态资源部署到对象存储中，将函数计算部分部署到函数中，将API网关、CDN等业务部署到对应的产品或者服务中，如果有域名解析需求，会自动帮助我们解析域名，同时将整个项目的所有资源进行关联。

除了帮我们一键部署，自动关联之外，Components还提供了若干的传统Web框架部署到Serverless架构的解决方案，例如腾讯云的Components中就有很多传统框架与Serverless架构的结合：tencent-koa、tencent-express、tencent-flask......，用户可将自己已有的，或者使用这些框架新开发的项目，直接一键部署到云端，这对于开发者来说，这将会是一个巨大的方便。

那么用户如何使用Plugin和Components呢？其实这两个功能都是Serverless Cli作为承载，也就是说，只要我们安装了Serverless Framework这个开发者工具，你就可以同时使用这两个功能。

安装Serverless Framework开发者工具的过程也是很简单的：

* 安装Nodejs，官方说的nodejs只需要6以上就好，但是在实际使用过程中，发现6不行，貌似至少8以上才可以。所以第一步就是安装Nodejs和NPM。

* 安装Serverless开发者工具：`npm install -g serverless`，安装完成之后可以通过`serverless -v`查看版本号，来确定是否成功的安装该工具。

至于如何使用Serverless Framework开发者工具，可以参考加下来的Plugin和Components部分。

## 什么是Serverless Plugin

首先，什么是Plugin，Serverless Framework Plugin实际上是一个函数的管理工具，无论针对AWS还是腾讯云，Plugin的目标都是对函数进行管理。使用这个工具，你可以很轻松的部署函数/删除函数/触发函数/查看函数信息/查看函数日志/回滚函数/查看函数数据等。

Plugin的使用时比较简单的，你可以直接使用Serverlss Framework进行创建，例如：

    serverless create -t tencent-python -p mytest

就可以看到这样的图案生成：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-4.png)

这其中，-t指的是模板，-p指的是路径，在Serverless Plugin操作下，你可以在任何指令中使用-h查看帮助信息，例如查看Serverless Plugin的全部指令，可以直接：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-5.png)

如果想查看Create的帮助：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-6.png)

其实这样看还是比较方便的。接下来继续说我们刚才的部分，创建完Serverless Plugin的项目之后，我们可以看一下他的Yaml长什么样子：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-7.png)

通过这个Yaml，我们可以看到，从上到下实际上它包括了几个主要的Key：Service、Provider、Plugins以及Functions。

Service可以认为是一个服务，也可以认为是一个分组，就是说在一个Service下面的函数，是可以被统一管理的，例如部署/删除/查看统计信息等，都是可以按照这个Service层面来统一进行的。

Provider可以认为是供应商以及全局变量的定义场景，这里使用的是腾讯云的云函数，供应商是腾讯云，所以就要写tencent，同时在这里还可以定义全局变量，这样在部署的时候，会将这些全局变量分别配置到不同的函数中。

Plugin就是插件的意思，就是说你在这个项目下，你会用到那些Plugin，因为Serverless团队提供了超级多的Plugin，当然这个例子里面是需要使用serverless-tencent-scf这个Plugin，因为我们要部署腾讯云的云函数，你要使用其他厂商的可能就要替换上面Provider中的name和这里Plugin了。

最后就是Functions，这就是我们定义函数的地方，这里可以定义很多函数。

接下来继续体验，我们创建项目之后，完成代码编写和Yaml的配置，我们可以继续操作，接下来就是要安装Plugin（是的，就是我们刚才在Yaml中写的Plugin）：

	npm install

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-8.png)

然后就可以进行功能的使用，例如部署服务：

	serverless deploy

在我们使用这个工具部署的时候，我们并没事先指定我们的账号信息，所以它会自动唤起扫码登录：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-9.png)

我们只需要扫一扫就可以完成登陆，登陆之后会继续进行操作：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-10.png)

操作完成会看到我们的Service信息，这里要注意，如果我们走CICD的时候，是没办法扫码的，那么这个时候我们就可以手动配置这个账户信息，格式是：

    [default]
    tencent_appid = appid
    tencent_secret_id = secretid
    tencent_secret_key = secretkey

配置完成之后，在Yaml中指定这个文件路径就好：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-11.png)

完成部署之后，我们可以触发函数：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-12.png)

还可以服务信息：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-13.png)

很多操作，大家有兴趣的都可以试一下：

    创建服务
    打包服务
    部署服务
    部署函数
    云端调用
    查看日志
    回滚服务
    删除服务
    获取部署列表
    获取服务详情
    获取统计数据
    ......

可以说是把函数的管理和操作是做的淋漓尽致，非常不错。但是大家也注意下，这里只是对函数资源的管理（触发器除外），不包括API网关，不包括COS，不包括数据库，不包括CDN.....只是函数。是的，Plugin就是函数开发者工具。

另外，这里要注意，虽然在腾讯云函数中只有命名空间和函数的概念，但是在Serverless Framework Plugin中却有Service、Stage以及函数的三层概念，同时云函数在Plugin这里不支持命名空间，所以可以认为是云函数只有函数的概念，而工具却有服务，阶段和函数的三层概念，这就会产生问题：

Service和Stage是什么？在函数中怎么体现？

以我们刚才部署的hello_world而言：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-14.png)

可以看到，在两个地方体现，一个函数名：服务名-阶段-函数名，另一个是标签，这里我对标签的体现没有太多的想法，但是对这个函数名的变化就要额外提醒：可能我们在简单使用的时候没有影响，但是如果涉及到函数间调用，或者通过云API使用函数的时候，那么这里面就要注意，我们的函数名并不是我们在Yaml中的函数名！这也导致另外一个问题，如果你已经有一个函数，且不是按照这种三段式命名的函数，那么你可能没办法很好的使用Plugin进行部署，除非说把函数进行迁移，将原函数删掉，使用Serverless重新进行部署。

## 什么是Serverless Component

刚才说了Plugin主要是对函数的管理，那么Component呢？可以认为Component是云产品的工具，因为通过Componnt你可以对你所有的组件进行组合使用，甚至你还可以很简单的开发出自己的Component来满足自己的需求。

Component的Yaml是一段一段的，刚才可以看到Plugin的Yaml是一个整体，从上到下实际上是一个Service里面的内容。但是Component完全可能是前后的两个组件没有任何关系，例如：

```yaml
test1:
  component: "@gosls/tencent-website"
  inputs:
    code:
      src: ./public
      index: index.html
      error: index.html
    region: ap-shanghai
    bucketName: test1


test2:
  component: "@gosls/tencent-website"
  inputs:
    code:
      src: ./public
      index: index.html
      error: index.html
    region: ap-shanghai
    bucketName: test2
```

通过这样的一个Yaml我们可以看到，实际上这里是两部分，是把一个网站的代码放到了不同的Bucket。目前腾讯云的Component的基础组件包括：

```yaml
@serverless/tencnet-scf
@serverless/tencnet-cos
@serverless/tencnet-cdn
@serverless/tencnet-apigateway
@serverless/tencnet-cam-role
@serverless/tencnet-cam-policy
```

封装的上层Component包括：

```yaml
@serverless/tencnet-express
@serverless/tencnet-bottle
@serverless/tencnet-django
@serverless/tencnet-egg
@serverless/tencnet-fastify
@serverless/tencnet-flask
@serverless/tencnet-koa
@serverless/tencnet-laravel
@serverless/tencnet-php-slim
@serverless/tencnet-pyramid
@serverless/tencnet-tornado
@serverless/tencnet-website
```

基础Component就是说，你可通过相关的Component部署相关的资源，例如tencent-scf就可以部署云函数，tencent-cos就可以部署一个存储桶；上层的Component实际上就是对底层Component的组合，并且增加一些额外的逻辑，实现一些高阶功能，例如tencent-django就通过对请求的WSGI转换，将Django框架部署到云函数上，其底层依赖了tencent-scf/tencent-apigateway等组件。

相对于Plugin而言，Component并没有那么多的操作，他只有两个：部署和移除。例如部署操作：

	serverless --debug

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-15.png)

移除操作：

	serverless remove --debug

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-2-16.png)

所以相对于Plugin而言，Component的产品纬度是增加了，但是实际功能数量是缩减了。

当然，我觉得这些都不是什么问题，毕竟我们可以Plugin和Component混用，但是问题来了：他们的Yaml长的不一样，我怎么混用？确实很尴尬的问题，我相信Serverless团队在不久的将来应该会解决这个问题吧。

## 总结

接下来说一下我的个人想法：

* Plugin部署到线上的函数，会自动变更名字，例如我的函数是myFunction，我的服务和阶段是myService-Dev，那么函数部署到线上就是myService-Dev-myFunction，这样的函数名，很可能会让我的函数间调用等部分产生很多不可控因素。例如我现在的环境是Dev，我函数间调用就要写函数名是myService-Dev-myFunction，如果是我的环境是Test，此时就要写myService-Test-myFunction，我始终觉得，我更改环境应该只需要更改配置，而不是更深入的代码逻辑。所以我对Plugin的这个换名字问题很烦躁；

* Plugin也是有优势的，例如他有Invoke、Remove以及部署单个函数的功能，同时Plugin也有全局变量，我觉得这个更像一个开发者工具，我可以开发、部署、调用、查看一些信息、指标以及删除回滚等操作，都可以通过Plugin完成，这点很给力，我喜欢；

* Components可以看作是一个组件集，这里面包括了很多的Components，可以有基础的Components，例如cos、scf、apigateway等，也有一些拓展的Components，例如在cos上拓展出来的website，可以直接部署静态网站等，还有一些框架级的，例如Koa，Express，这些Components说实话，真的蛮方便的，腾讯官方也是有他们的最佳实践；

* Components除了刚才所说的支持的产品多，可以部署框架之外，对我来说，最大吸引力在于这个东西，部署到线上的函数名字就是我指定的名字，不会出现额外的东西，这个我非常看重；

* Components相对Plugin在功能上略显单薄，除了部署和删除，再没有其他，例如Plugin的Invoke，Rollback等等一切都没有，同时，我们如果有多个东西要部署，写到了一个Components的yaml上，那么我们每次部署都要部署所有的，如果我们认为，我们只修改了一个函数，并且不想重新部署其他函数从而注释掉其他函数，那么很抱歉告诉你，不行！他会看到你只有一个函数，并且帮你把你注释掉的函数在线上删除；

* Components更多的定义是组件，所以每个组件就是一个东西，所以在Components上面，是没有全局变量这一说法，这点我觉得很坑。

我接触Serverless有一段时间了，无论是Serverless架构还是Serverless的这个开发者工具，我也不断的在社区中贡献一些Serverless相关的经验、分享，以及Serverless这个工具的相关组件和功能。经历长久的和用户的沟通交流，我发现很多用户对Serverless架构和Serverless的这个开发者工具很模糊很迷茫，好不容易弄懂了，又对这个工具提供的Plugin和Components感到模糊。所以一直都想写一篇文章，来描述一下什么是Serverless架构，什么是Serverless开发者工具，以及Serverless开发者工具中的Plugin和Components有什么区别。并不想说明什么事情，只是希望，我们在使用这些东西的时候，头脑是清楚了，知道我在做什么，我为什么这么做。


