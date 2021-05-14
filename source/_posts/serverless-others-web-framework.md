---
title: 通过Component实现高可用的Web服务（多地域部署容灾）
date: 2020.7.21
tags: [Serverless, Serverless Framework, 高可用]
categories: [Serverless, 闲谈]
---

## 前言

在实际生产中，单点故障的情况不可避免，而且单副本的存储方案早已无法满足业务的可靠性要求,因此一般情况下我们至少也会做双机存储架构。说到双机存储结构，就可能会涉及到主备、主从以及主主模式。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-1.png)

但是，在Serverless架构下，我们的高可用方案或者说容灾方案，是否还需要主备，主从以及主主等模式呢？如果还需要，那么他又是什么样子的呢？

## Serverless与多地域部署

每个云厂商都会给我们一个服务可用性的承诺，但即使给我们了一个非常高的值作为承诺，我们已让不能认为业务不会在运营场级别出现故障导致不可用。所以在很多时候容灾方案还是必须要有的。

在传统主机时代，可能要有主备，主从以及主主模式，这个模式更多针对的是单台机器或者某个集群，然而在Serverless架构下，只相对应的有函数概念，并没有所谓的机器和集群概念，至少在用户层面是没有这个概念的，因为这两个部分，都是交给云厂商来维护的，那么我们在Serverless架构下，是否就没办法做容灾了？

按照道理来说，一般情况下云厂商会保证我们整个一个服务的可用性，如果云厂商管理的某个机器出现故障，机器也会被及时剔除，确保新的函数在安全，稳定，健康的环境下启动，正常提供服务，那么如果云厂商的整个地域的服务出现故障怎么？

一般情况下，可能由于某些原因，云厂商确实会在某个地域出现大规模故障，无论是AWS还是Google，还是国内的一些云厂商，都出现过类似的事情，那么如果出现类似的事情，我们怎么确保我们的服务依旧可用？而不是要苦苦等待云厂商的恢复？

所以对于但地域解析的网站，我们可以实现多地域的主备方案，在云函数中，多地域的主备方案显得更加经济实惠，因为函数是按量付费的，我完全可以将函数复制到其他的地域，只要不进行触发，就不会产生额外的费用。所以相对传统主机时代的主备模式，这种主备显得更加人性化。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-2.png)

 一般情况下，我们的单地域部署的服务，是这样一个逻辑（以部署在上海区为例），这里面比较可能出现问题的有API网关服务以及云函数服务，数据库服务。
 
 其中数据库服务我们可以考虑快地域主从，然后跨区域同步，一旦出现问题，我们可以在函数中做一个负载，确保整体数据不会出现问题，在腾讯云的云数据库描述下可以看到：
 
 > 本地 IDC 机房 MySQL 数据库与云数据库 MySQL 之间可以通过数据迁移服务实时同步数据，本地IDC机房如遇到断电、网络故障等引起数据库服务中断，可迅速切换数据库服务至作为灾难备份的云数据库 MySQL 实例，实现数据库容灾；同时，云数据库 MySQL 可支持同城多可用区灾备/跨城灾备，保障高可用。
 
 所以说，数据库相关服务的容灾可以通过备用数据库来提高可用性。
 
 那么云函数和API网关的呢？
 
 此时，我们以多地域部署容灾为方案，可以考虑这样的架构：
 
 ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-3.png)

同样作为单地域解析服务，但是这个多地域部署相对来说就安全稳定的多，一旦某个地域的服务出现问题（例如API网关，云函数），那么都可以通过监控程序及时发现，并且迅速切换解析到其他地域，所以我们完全可以通过多地域部署函数监控脚本以及多地域部署业务来实现这样完整的一套功能。

我们多地域部署的监控函数与时间触发器进行结合，定期进行网站可用性的排查，一旦出现问题，我们就在云解析层面，进行解析的切换，实现单地域服务的多地域部署容灾方案：

 ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-4.png)
 
 
 在这个逻辑中，可以看到，我们需要先请求服务是否可以用，如果不可用，我们则会获取容灾列表，剔除不可用的服务，并通过云解析进行可用区的解析。当然，在生产中，我们确定了某个服务不可用了，还要进行精确的告警，所以可以在获得到不可用的解析记录对应的服务之后，通过邮件或者企业微信，短信等方法进行告警。
 
至此为止，我们的一个简单版的“高可用”服务就算做成了，有的读者可能有所疑问：

1.	切换解析不需要时间吗？怎么确保TTL按时生效？
2.	对服务进行修复，是否比切换解析更加靠谱呢？

针对问题1，其实切换解析是需要时间，虽然说有一小部分的机房可能不会按照我们设定的TTL进行生效，但是实际上大部分的机房都是可以按照我们的设定及时生效，所以这里我们要注意，在添加解析的时候要尽可能地确保TTL时间短一些，目前腾讯云的云解析付费版最低TTL可以设置为1s，免费版是600s。针对问题2，我们在云函数上运行我们服务，很少会说因为流量太高导致我们的服务不可用，或者我们服务中存在bug导致整个项目不可用，因为Serverless架构下，云厂商会帮我们解决很大一部分的可用性，例如流量并发问题等，同时就算是我们程序中存在bug，但是由于函数的前一次后一次运行实际上关联性不大，确切来说无状态性让函数的前后执行变得解耦，所以针对程序中有bug这种情况，我们完全可以通过告警来确定，然后再人为排查，消灭隐患。所以，在Serverless架构下常出现大规模服务性灾难应该很大一部分原因就是云厂商的问题（此处已经排除掉用户代码层面的bug），那么这种问题一旦出现，就不是我们能够决定是否可以修复以及什么时候修复，所以这个时候：


 ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-5.png)

如果是API网关层面出现问题，我们就可以通过上一层来进行解决，例如云解析的切换，如果是函数层面出现问题，我们可以考虑切换到API网关到同区域的备用函数，如果是函数服务的整个区域性故障（概率非常低），那么我们依旧可以考虑切换解析到备用区。


当然，如果我们的服务并非是上述的单地域提供服务，我们可以考虑多地域部署多地域就近接入以及多地域容灾方案。

 ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-6.png)
 
 通过图上图中，我们可以看到，我们可以将一个域名，划分地域进行解析，例如华北、华东两个地区，就可分别解析到北京1区，北京2区的两个不同的机房中两台不同的服务器上。当我们其中一个服务无法提供服务时，其他所有区域不受影响，此时，我们可以使用云函数对解析进行修改，将其解析到每个地区的备用服务上，这时整个流程是：
 
  ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-7.png)
  
  整体结构图：
  
   ![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-4-8.png)
   
对于传统的服务而言，即每个地区需要部署多套服务，这无疑是大大增加了成本支持。但是在Serverless架构上使用了，即使我们部署多套云函数仍然是按量付费，成本可以大幅度减少。

当我们采用Serverless架构作为后端服务时，以华北地区为例，华北地区用户在访问后端服务的时候，通过DNS重定向到北京区的API网关，然后再由API网关触发北京区云函数，此时我们需要两个云函数对服务进行监控，一个函数对云函数进行监控，当云函数服务失效之后，可以将API网关绑定到备用的函数上，另一个监控是对API网关进行监控，当某个地域的API网关失效之后，可以对解析进行修改，重定向到备用地域的API网关上，尽可能快速的保证服务的可用性。

当然，如果想要让服务更稳定，也可以在增加数据库/对象存储的主备与监控，以及数据库/对象存储封装函数的主备与监控，这样就可以在四个层次做监控告警以及服务保障：在我们的服务中，通常数据库或者存储模块不会同时在多个区域部署，也可能只有一套主备服务，此时，可以通过对数据库和对象存储做一层封装，即在外层增加一个对应的函数，通过内网对数据库以及对象存储进行数据转发等，通过外部云函数调用invoke（专线调用）大幅度降低由于网络原因造成的延时，当云数据库/对象存储出现问题，在接入层（数据库/对象存储封装函数）一侧，进行切换，将云数据库/对象存储切换到备用服务上，并进行告警；当接入层发生故障，无法继续服务时，在逻辑函数初（北京区/上海区/广州区），切换封装函数，即通过内部专线（函数间调用）调用备用接入层函数；同理，当逻辑函数/业务函数出现故障，监控脚本对API网关侧的后端服务进行切换，切换到备用逻辑函数/业务函数上；如果当API网关出现故障，无法继续提供服务，则只能在解析层面做切换。在整个这样的一个流程中，每一阶段或者说每一层面都有自身的负载机制，主备策略，可以根据不同组件出现故障的实际情况，进行多层级的自动切换，进而保证业务可用时长的一个最大化。

此处，可能会有读者有些疑问：为什么某个函数会无法提供服务？底层服务的容灾机制，不是云厂商要提供的么？其实按照道理来说，这个容灾机制是云厂商提供的，按照道理来说，函数是无状态的，只要确保自己的业务逻辑无问题，理论是不需要进行某些层级的主备容灾等，但是实际上，各个云厂商均没有办法保证我们他的某个服务可以100%可用。例如2019年6月2日凌晨2点，亚马逊云AWS突发受大规模故障，直到当天下午1点48分故障解除，故障时间将近12小时。所以，虽然说云厂商会为我们提供相对安全与可靠的容灾机制和服务，但是我们还是很有必要在自身层面额外处理一下，尽可能保证服务安全与稳定。



## 通过Serverless Framework进行多地域部署与解析

### 函数的多地域部署

以腾讯云为例，基础的Component如果跨地域部署可能不是很容易实现，因为我们是可以实现修改区域将函数部署到多个地域，但是实际中，每个区域的函数可能还会有一些额外的配置，所以这个时候可以借助多地域部署的组件来实现：`tencent-scf-multi-region`

相对于传统的`tencent-scf`组件而言，这个组件的yaml将`region`字段变成了兼容`list`形式，同时增加了子地域的额外配置，例如yaml：

```yaml
hello_world:
  component: '@serverless/tencent-scf-multi-region'
  inputs:
    codeUri: ./
    description: This is a template function
    region: 
      -ap-guangzhou
      - ap-shanghai
    environment:
      variables:
        ENV_FIRST: env1
        ENV_SECOND: env2
    handler: index.main_handler
    memorySize: 128
    name: hello_world
    runtime: Python3.6
    timeout: 3
    events:
      - apigw:
          name: serverless_test
          parameters:
            protocols:
              - http
            description: the serverless service
            environment: release
            endpoints:
              - path: /users
                method: POST
              - path: /usersss
                method: POST
    ap-guangzhou:
      environment:
        variables:
          ENV_FIRST: env2
    ap-shanghai:
      events:
        - apigw:
            name: serverless_test
            parameters:
              protocols:
                - http
              description: the serverless service
              environment: release
              endpoints:
                - path: /usersd
                  method: POST
```

测试部署效果：

```
$ sls --debug
  
    DEBUG ─ Resolving the template's static variables.
    DEBUG ─ Collecting components from the template.
    DEBUG ─ Downloading any NPM components found in the template.
    DEBUG ─ Analyzing the template's components dependencies.
    DEBUG ─ Creating the template's components graph.
    DEBUG ─ Syncing template state.
    DEBUG ─ Executing the template's components graph.
    DEBUG ─ Compressing function hello_world file to /Users/dfounderliu/Desktop/ServerlessComponents/test/scf_test/.serverless/hello_world.zip.
    DEBUG ─ Compressed function hello_world file successful
    DEBUG ─ Uploading service package to cos[sls-cloudfunction-ap-guangzhou-code]. sls-cloudfunction-default-hello_world-1583816589.zip
    DEBUG ─ Uploaded package successful /Users/dfounderliu/Desktop/ServerlessComponents/test/scf_test/.serverless/hello_world.zip
    DEBUG ─ Creating function hello_world
    DEBUG ─ Updating code... 
    DEBUG ─ Updating configure... 
    DEBUG ─ Created function hello_world successful
    DEBUG ─ Setting tags for function hello_world
    DEBUG ─ Creating trigger for function hello_world
    DEBUG ─ Starting API-Gateway deployment with name hello_world.ap-guangzhou-hello_world.serverless_test in the ap-guangzhou region
    DEBUG ─ Service with ID service-p14470dc created.
    DEBUG ─ API with id api-pg3ihnoi created.
    DEBUG ─ Deploying service with id service-p14470dc.
    DEBUG ─ Deployment successful for the api named hello_world.ap-guangzhou-hello_world.serverless_test in the ap-guangzhou region.
    DEBUG ─ API with id api-op4jqvba created.
    DEBUG ─ Deploying service with id service-p14470dc.
    DEBUG ─ Deployment successful for the api named hello_world.ap-guangzhou-hello_world.serverless_test in the ap-guangzhou region.
    DEBUG ─ Deployed function hello_world successful
    DEBUG ─ Compressing function hello_world file to /Users/dfounderliu/Desktop/ServerlessComponents/test/scf_test/.serverless/hello_world.zip.
    DEBUG ─ Compressed function hello_world file successful
    DEBUG ─ Uploaded package successful /Users/dfounderliu/Desktop/ServerlessComponents/test/scf_test/.serverless/hello_world.zip
    DEBUG ─ Creating function hello_world
    DEBUG ─ Updating code... 
    DEBUG ─ Updating configure... 
    DEBUG ─ Created function hello_world successful
    DEBUG ─ Setting tags for function hello_world
    DEBUG ─ Creating trigger for function hello_world
    DEBUG ─ Starting API-Gateway deployment with name hello_world.ap-shanghai-hello_world.serverless_test in the ap-shanghai region
    DEBUG ─ Service with ID service-7daktopz created.
    DEBUG ─ API with id api-4v40ce4u created.
    DEBUG ─ Deploying service with id service-7daktopz.
    DEBUG ─ Deployment successful for the api named hello_world.ap-shanghai-hello_world.serverless_test in the ap-shanghai region.
    DEBUG ─ API with id api-emkl7ov4 created.
    DEBUG ─ Deploying service with id service-7daktopz.
    DEBUG ─ Deployment successful for the api named hello_world.ap-shanghai-hello_world.serverless_test in the ap-shanghai region.
    DEBUG ─ Starting API-Gateway deployment with name hello_world.ap-shanghai-hello_world.serverless_test in the ap-shanghai region
    DEBUG ─ Using last time deploy service id service-7daktopz
    DEBUG ─ Updating service with serviceId service-7daktopz.
    DEBUG ─ API with id api-2zag45hq created.
    DEBUG ─ Deploying service with id service-7daktopz.
    DEBUG ─ Deployment successful for the api named hello_world.ap-shanghai-hello_world.serverless_test in the ap-shanghai region.
    DEBUG ─ Deployed function hello_world successful
  
    hello_world: 
      ap-guangzhou: 
        Name:        hello_world
        Runtime:     Python3.6
        Handler:     index.main_handler
        MemorySize:  128
        Timeout:     3
        Region:      ap-guangzhou
        Namespace:   default
        Description: This is a template function
        APIGateway: 
          - serverless_test - POST - http://service-p14470dc-1256773370.gz.apigw.tencentcs.com/release/users
          - serverless_test - POST - http://service-p14470dc-1256773370.gz.apigw.tencentcs.com/release/usersss
      ap-shanghai: 
        Name:        hello_world
        Runtime:     Python3.6
        Handler:     index.main_handler
        MemorySize:  128
        Timeout:     3
        Region:      ap-shanghai
        Namespace:   default
        Description: This is a template function
        APIGateway: 
          - serverless_test - POST - http://service-7daktopz-1256773370.sh.apigw.tencentcs.com/release/users
          - serverless_test - POST - http://service-7daktopz-1256773370.sh.apigw.tencentcs.com/release/usersss
          - serverless_test - POST - http://service-7daktopz-1256773370.sh.apigw.tencentcs.com/release/usersd
  
    35s › hello_world › done

```

通过部署，可以成功将函数部署到不同的地域，并且针对不同地域进行额外的配置。

### 上层服务的多地域部署与解析

但是就目前来看，`tencent-scf`更多是一个基础组件，更多人则是在使用Koa、Express等组件，那么对于这种相对高阶的组件，是否可以多地域部署呢？就目前文档的描述来看，是可以进行多地域部署，且可以进行自动解析，以`tencent-tornado`为例：

```yaml
TornadoTest:
  component: '@serverless/tencent-tornado'
  inputs:
    region:
      - ap-guangzhou
      - ap-shanghai
    functionName: tornadoFunctionTest
    tornadoProjectName: app
    code: ./
    functionConf:
      timeout: 10
      memorySize: 256
      environment:
        variables:
          TEST: vale
      vpcConfig:
        subnetId: ''
        vpcId: ''
    apigatewayConf:
      protocols:
        - http
      environment: release
      customDomain:
        - domain: anycodestest1.com
          isDefaultMapping: 'FALSE'
          pathMappingSet:
            - path: /
              environment: release
          protocols:
            - http
        - domain: anycodestest2.com
          isDefaultMapping: 'FALSE'
          pathMappingSet:
            - path: /
              environment: release
          protocols:
            - http
    cloudDNSConf:
      ttl: 603
      status: enable
    ap-guangzhou:
      functionConf:
        timeout: 20
      apigatewayConf:
        protocols:
          - https
      cloudDNSConf:
        recordLine:
          - 电信
          - 联通

```

部署结果：

```text
  TornadoTest: 
    functionName: tornadoFunctionTest
    ap-shanghai: 
      apiGatewayServiceId: service-mdnjhsp3
      url:                 http://service-mdnjhsp3-1256773370.sh.apigw.tencentcs.com/release/
    ap-guangzhou: 
      apiGatewayServiceId: service-nh6xgutk
      url:                 https://service-nh6xgutk-1256773370.gz.apigw.tencentcs.com/release/
    DNS:          Please set your domain DNS: f1g1ns1.dnspod.net | f1g1ns1.dnspod.net

```

通过这样一个组件，就可以完成框架的跨地域部署与解析。

## 总结


至此为止，我们基本完成了一个基于Serverless的高可用框架的案例探讨，通过对单解析的分析，到对多解析的策略制定，再到最后的基于Serverless架构的高可用模型，虽然仅仅可以作为学习参考使用，在实际生产中还会面临很多难题，有些地方还需要针对实际项目做一些特殊的修改和修正，但是这样的一个简单的基于Serverless架构的高可用模型，基本上可以发挥出Serverless架构的很多优秀的特点，其中包括Serverless架构的按量付费功能，在不使用的时候则不需要收费，通过这样一个特性可以保证我们在部署多套云函数的时候不会因为部署量增加而产生额外的费用；其次在这个项目中，包括API网关等触发器对函数进行触发，也会包括函数间的编排和调用，更有FaaS与BaaS紧密结合，通过专线跨地域Invoke，在外层还会增加多套监控告警函数以及DNS切换函数、云函数切换函数等尽可能保证服务的稳定与可用性。

同时为了更加简单的进行多地域部署，还通过了Serverless Framework实现了多地域部署方案。

在实际生产生活中，多地域部署容灾无论是对单地域服务还是对多地域就近接入服务来说，都是蛮重要的，尤其在Serverless架构下，按量付费让主备模式成本骤降，更是可以尝试这种策略与方案，当然服务的100%可用性很难保证或者是是不可能事件，但是我们可以共同探讨相对优秀的高可用方案。诚然，我上面的方案在很多地方可能也有不足之处，但是我相信随着云计算的不断发展，Serverless架构的不断"进化"上面的方案也会被更多人完善补充，修改斧正。