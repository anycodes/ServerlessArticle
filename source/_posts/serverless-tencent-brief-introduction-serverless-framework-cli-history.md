---
title: Serverless Framework Cli的版本进化
date: 2019.12.20
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework]
categories: [Serverless, 腾讯云, 基本介绍]
---

## 前言

Serverless技术不断的发展，其生态也在不断的发展与完善，其中Serverless相关的开发者工具也是因为其可以提高生产效率，而变得越加重要，也逐渐被更多人所关注和使用。

在各个厂商提供自己的工具之外，Serverless Framework CLI作为多云的Serverless管理平台，同时为腾讯云，AWS，阿里云等多个厂商/开源项目提供Serverless相关的部署，监控，管理平台。

随着时间的发展，Serverless Framework也进行了多次版本变更，本文将会就目前常见的几个版本（暂时没有分析Pro版本）进行简单分析。

## 版本描述

Serverless Framework CLI就目前而言是分为Plugin、Component V1以及Component V2和Pro四个常见版本。这四个版本可以说是功能各不相同：

* Plugin版本更多是一个函数的管理工具，可以通过各个云厂商/开源项目的Plugin，来进行函数相关的管理，包括函数的部署、移除、触发、回滚、监控、日志查看等众多功能，其Yaml格式大概如下：

    ```yaml
    service: my-service # service name
    
    provider: # provider information
      name: tencent
      runtime: Nodejs8.9 # Nodejs8.9 or Nodejs6.10
      credentials: ~/credentials
    
    plugins:
      - serverless-tencent-scf
    
    functions:
      function_one:
        handler: index.main_handler
    ```
    
    可以看到，在Plugin的Yaml中，最明显的地方就是包括Service和Provider以及Plugins三个字段，一般情况，用的不同云厂商，Plugins下面所设置的插件是不同的，例如腾讯云的是`serverless-tencent-scf`，阿里云的则是`serverless-aliyun-function-compute`，系统将会根据不同的云厂商插件，帮助用户部署项目到不同的云厂商中。

* Component V1版本是Plugin版本的并行版本，可以认为Component V1和Plugin是两个不同的东西，无论是功能还是支持的产品维度，都是不同的。如果说Plugin更多的是函数管理的工具，那么Component就是Serverless的部署工具，Component V1支持部署和移除两个功能，支持云函数的部署/移除，支持API网关、对象存储......众多BaaS能力的部署和移除，同时为了帮助用户的业务快速上Serverless架构，Component V1提供了海量传统组件部署上云的组件，例如`tencent-express`,`tencent-django`,`tencent-flask`,`tencent-koa`,`tencent-nextjs`,`tencent-werobot`等在内的十余个框架。相比之下，Component V1的Yaml会比较简洁，其基本格式是：
   
    ```yaml
    cnsResolution:
      component: '@serverless/tencent-cns'
      inputs:
        domain: anycodes.cn
    ```
   
   在这个结构中，可以看到，只有一个项目名称，组件名称以及对应组件的Inputs内容。由于项目开源的，用户可以简单快速的定制化开发自己的组件，在部署项目的时候，Serverless Framework会自动检查用户所使用的Component，并且去云端（例如NPM等）进行版本对比，如果有新版本则下载新版本，没有新版本则使用本地的版本进行项目部署，这整个过程都是自动化的，同时用户也可以在一个yaml文件中，写多个项目，使用多个组件，不同组件之间进行相关关联，例如：
   
    ```yaml
    Conf:
      component: "serverless-global"
      inputs:
        mysql_host: gz-cdb-mytest.sql.tencentcdb.com
        mysql_user: mytest
        mysql_password: mytest
        mysql_port: 62580
        mysql_db: mytest
        mini_program_app_id: mytest
        mini_program_app_secret: mytest
    
    
    Album_Login:
      component: "@serverless/tencent-scf"
      inputs:
        name: Album_Login
        codeUri: ./album/login
        handler: index.main_handler
        runtime: Python3.6
        region: ap-shanghai
        environment:
          variables:
            mysql_host: ${Conf.mysql_host}
            mysql_port: ${Conf.mysql_port}
            mysql_user: ${Conf.mysql_user}
            mysql_password: ${Conf.mysql_password}
            mysql_db: ${Conf.mysql_db}
    ```
   
   在这个yaml终究使用了`serverless-global`，`@serverless/tencent-scf`两个组件，其中`@serverless/tencent-scf`组件通过`${Conf.mysql_host}`方法，引用了项目`Conf`中的`mysql_host`的值，可以说是非常方便。当然，这种做法也有一个缺点，那就是我一个yaml中写了太多的项目，可能会造成额外问题诞生：
   * 例如写了很多函数，可能造成部署的时候出现API调用超频问题；
   * 如果有一些资源比较大，你每次部署都是全量部署，没办法做到只部署其中一个内容；
   通过Component V1用户可以快速的部署API网关，云函数以及对象存储，CDN等基础业务，也可以快速部署网站，Express，Django等高级业务。
   
* Component V2版本可以看作是V1版本的升级。首先说为什么要进行这个升级，在V1版本中有几个比较恶心的缺点:
    * 组件可能会涉及到下载，不同网络环境可能对NPM等的支持并不是很理想，所以经常出现下载组件的部分被卡住，有的用户可能用的还是tnpm或者cnpm，如果这些平台对某些组件没及时同步，其用户也很难使用相关的组件，甚至是报错；
    * 组件需要在本地进行打包，对于一些小型项目还好，对于一些express，nextjs等项目来说，动不动就几万个文件（包括node_modules），几百M的文件夹，压缩过程经常出现一些不确定的问题，尤其对于一些小内存的电脑，很可能"卡死"，这个问题在社区中很多用户都遇到了；
    * Plugin的用户可以进行函数的触发，日志的查询等功能，但是Component V1并不具备这些功能，相对而言是一种功能缺失，而在我们生产过程中，查看函数日志，函数触发等相关功能还是非常有必要的，所以进行了一个比较大的升级，在V2版本支持了这些功能；
  
  所以可以认为，V2版本是在用户使用了V1版本之后，提出各类抱怨之后，做出了一次重大改变和升级，当然这次改变和升级还是有很多可圈可点的，例如腾讯云的Serverless部分增加了在线调试功能（仅限于Nodejs10.*）等。
  
  当然，V2版本的这些改变也未必是好事，首先V1版本更多是基于密钥来进行的相关操作，也就是说通过主账号、子账号的密钥，来进行相关部署行为，但是V2版本就变成了基于角色，这种不兼容性，对大客户来说并不是非常友好；其次上面说的V1版本遇到的问题，其实也并没有完全解决，例如现在也是需要本地打包上传还是可能存在打包卡住的问题，同时V1版本的所有部署行为都是在客户端，对用户来说是透明行为，而V2版本则都是云端操作，对用户而言是黑盒操作：我的代码是否安全，我的密钥信息是否安全？这些都是让人觉得没安全感的事情。按照Serverless Framework CLI的升级套路而言，Yaml果真又是不兼容的：
  
  ```yaml
    component: scf
    name: scfdemo
    org: test
    app: scfApp
    stage: dev
    
    inputs:
      name: scfFunctionName
      src: ./src
      runtime: Nodejs10.15
      region: ap-guangzhou
      handler: index.main_handler
    ```
  
   表面上看是V1版本的inputs没变化，增加了`component`，`name`，`org`，`app`，`stage`等字段，实际上inputs的变化才是致命的，因为之前所有的上传操作都是组件完成，所以以V1版本的Website组件为例，会有上传代码部分以及上传自定义域名证书部分，但是在V2版本，他只能上传一个内容，所以就必然要把这两部分放在一个目录下，然后一起打包，在云端再拆开处理，这必然就要引起yaml和目录结构的大变更，除此之外，为了解决V1不能单独部署一个项目的问题，V2版本每个Yaml中只能写一个内容，如果我们原先一个yaml写了10个函数，一个webiste，那么此时，我们就要拆成11个yaml，放在11个文件夹下，然后在上层目录一键部署全部。这种做法虽然可以兼容单个内容的部署和整体内容的部署，但是我个人觉得这个做法还是很不友好的，总之，用户从plugin升级到V1要很大的勇气，从V1升级到V2可能要有很多的坑。
   
   当然，V2版本除了上面所说的内容，据说还会有一个控制台：
   
   ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-3-1.png)
   
   可以在控制台通过点击快速部署，部署完成之后，也可以查看到部署结果的相关的信息。当然，就目前而言，这个功能还不是很成熟。但是我相信，通过一个控制台，整合一个项目（可能包括APII网关，函数，对象存储等），还是很不错的想法，值得期待。
   
## 版本优缺点

Plugin和Component的最大区别是前者是函数管理（包括触发器），后者是资源组织管理；而Component的V1和V2的最大区别是前者是本地行为，后者是云端行为。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-3-2.png)

通过这幅图，可以看出三者的基本区别，当然，正是因为你这些区别，也给三者带来了不同的优缺点，当然这些优缺点主观性比较强，所以也仅有参看靠作用。

### Plugin

* 优点

    * 可以管理包括AWS、腾讯云、阿里云等在内的8家厂商和若干开源项目的函数；
    * 支持函数的创建、移除、查看日志、触发函数、回滚、监控统计等相关操作；
    * 可以通过Service概念进行函数管理，可以通过Stage进行函数的阶段管理；
    * 可以针对单个函数进行部署，回滚等操作，也可以针对多个函数进行批量部署，移除等操作；


* 缺点

    * 函数部署到线上之后，名字/服务/空间会发生改变，以腾讯云为例，部署的函数名字实际上是`service名字-阶段名字-函数名`，以阿里云为例，部署的函数实际上是在这样的服务下`service名字-阶段名字`。这样的部署结果会让大家对函数资源的使用产生一些额外的问题，例如函数调用的参数问题等；
    
### Component V1

* 优点

    * 可以部署函数的同时，还可以部署更多的资源，例如对象存储、CDN、API网关等；
    * 可以基于现有的基础服务，诞生额外的部署能力，实现自动化脚本或者批量生产，例如一键部署Express（基于函数计算和API网关），一键部署Website（基于对象存储和CDN）；
    * 一个Yaml中可以使用多段资源，资源间可以相互关联，资源部署时也可以增加一些额外操作；例如腾讯云/AWS提供的全栈网站的例子，里面包括Express组件和Website组件，Express组件先部署，生成一个后端的Url作为变量传递给Website组件，Website组件会在本地生成一个文件，将该地址存入其中，然后执行相关的脚本进行`build`（该website使用了Vue.JS），整个项目只需要`sls --debug`就可以搞定，非常方便；
    
* 缺点
    
    * 官方本身不支持全局变量，对全局变量的支持非常差（当然后来我也提供了`serverless-global`组件）；
    * 如果一个Yaml中写了多个项目，可能会产生超频问题，同时不能支持单个资源部署，每次部署都要争个yaml部署一次（当然后来我自己也提供了一个单独部署的方案）；
    * 权限管理混乱，由于更加注重资源的自动化，给大家更多方便，但是在资源处理这里可能就有一点模糊，例如以腾讯云为例，部署Express服务，可能要对象存储的读写权限，API网关的读写全县，SCF的读写权限等，标签服务的读写权限等，如果用户并没有API网关或者对象存储的读写权限，可能就面临部署失败的问题，对子账号影响极大；
    * 每次都需要检查Npm/Cnpm等，如果有一些特殊网络环境，就没办法检查成功，会一直卡在下载组件阶段；
    * 相对Plugin而言，函数只能部署和移除，确实太多功能：没办法触发、没办法查看日志......
    * 支持的厂商太少，目前只有AWS和腾讯云支持Component V1；
    
### Component V2

* 优点

    * 不再需要下载各种组件，直接云端部署，非常快速。实际测试一个简单的Website，只需要3S完成部署，一个简单的函数项目，只需要2S；
    * 腾讯云的相关组件，支持在线调试功能（云函数-Nodejs10.*）；
    * 腾讯云的相关组件，支持控制台查看等；
    * 函数等支持了更多功能，例如触发、查看日志等；
    
* 缺点

    * 所有资源上传云端安全性未知，给用户的安全感比较低；
    * 开发过组件的开发者都会知道，数据实际上是上传给了serverless，组件开发者需要下载，也就可以认为，如果一个Express服务的实际操作在V1版本是：`用户写完代码->本地加入转换层文件->打包->上传到用户的存储桶->部署上线`，在V2版本是`用户写完代码->打包上传到Serverless->Component（组件）在云端获取下载地址并且下载解压->加入转换层文件->打包->上传到用户的存储桶->部署上线`，整个流程实际上变长了，在刚开始说的速度非常快，实际上是快在了不需要检查组件版本、不需要下载组件，但是网络足够好的，且对于越大的项目而言V2版本的这个部分反而是一种劣势，可能会因为项目足够大，文件足够多而成倍的增加部署时间，还是以Express为例，可以看到整个过程图如下：
    ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/1-3-3.png)
  可以认为，在V2版本，节省的是检查和下载组件的时间，在云端增加的是多打包一次，多上传一次，多下载一次的时间，我认为这种转换，在一定情况下是好事，也是坏事。
    * 权限管理在V1的比较乱的前提下，变得更加难以捉摸了，就目前而言，以腾讯云为例，一个主账号下，每个子账号都将会同样的操作权限，就目前而言对于权限划分细致的企业客户来说，这并不是一个好的现象；
    
## 总结

Serverless的发展确实非常迅速，Serverless周边的产品发展也是不甘落后，Serverless Framework CLI是其中一个典型的代表。

在Serverless Framework CLI发展的过程中，必然也会诞生若干个版本，从Plugin到Component，从V1到V2，每个版本都有自己的特点，每个版本都有自己的优势，人无完人，同样，任何一款产品也都会有一些缺点，不同的版本也有着不同的优劣。但是无论如何，我们可以看到其在朝着更加方便，快速，完美的方向发展。