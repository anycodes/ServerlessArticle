---
title: Serverless与传统Web框架的迁移
date: 2020.12.13
tags: [Serverless, 传统框架, 迁移]
categories: [Serverless, 闲谈]
---

## 前言

Serverless的确是在非常快速的发展，快速发展到什么程度呢？就通过我们国内排行前列的两个云厂商来看，腾讯云在过去的几个月内，平均每个月都会推出2-3个能力或者特性，尽管在部分能力的输出上是在追平AWS和阿里（例如Custom Runtime的发布、硬盘挂载、增加Nodejs12运行时、预置并发功能等都是在追平友商），但是其也在积极打磨自己的一些特色能力，例如发布了基于云函数 SCF 的 Ckafka to Ckafka 转储功能等。而阿里云则是比较激进，继之前全球首推了硬盘挂载能力之后，今年也算是引领行业先后发布/支持了事件总线（Event Bridge）、性能实例（更大的规格，更长的执行时间）和容器镜像等若干能力。在两个云厂商能力对比上来看，二者可谓是各有千秋。就我个人而言，我觉得腾讯在体验层做的更好，例如有Serverless Framework，例如支持依赖在线安装，拥有层的概念等；阿里云则是在底层能力上更强一些，包括之前做的性能测试对比，同时阿里云在产品能力的纬度上，也是具有前瞻性的，例如目前容器镜像的支持，性能实例的支持，以及云栖云栖大会持续发布的事件总线（Event Bridge）、函数工作流（Workflow）等。


> 有人可能会觉得腾讯云云开发支持了应用托管等，这个应该也是容器镜像，但是实际上托管和阿里云推出的Runtime是不同的，这个托管更类似于阿里云的SAE产品，而且通过他对自动扩缩容能力的描述，基本可以确定他的弹性伸缩本身就是基于K8S HPA搞的，对外表现实际上是利用了采集上来的CPU等指标进行伸缩，而腾讯云SCF对外表现则是根据请求粒度来做弹性的，在这一点和SCF本身貌似就是两个体系了，和阿里云的容器镜像支持也本身就是两个体系，当然这也是猜测，猜测错了也别喷我。

当然，通过产品能力上的描述来说明Serverless发展速度快，各个厂商都重视可能不是很直观，我们不妨通过招聘信息来看一下：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/othermaterial/1-1.png)

通过腾讯老巢和阿里老巢两个地区的招聘信息对比，可以看到这两个城市都有（相对于去年来说）大量Serverless岗位，并且岗位的薪资水平也是蛮不错的，这其中阿里下的"血本"貌似还更大一些。两个厂商，都是下了血本在抓Serverless，Serverless也如愿以偿的有了高速的发展，那么用户使用情况是怎么样子的呢？

一方面是云厂商们的争先恐后提升能力，生怕一不留神错失先机，另一方面则是广大用户/开发者们努力上云的艰辛过程。

包括我自己在内，也是一个传统Web应用的开发者，我自己做过一个应用，目前已经有过百万的下载量了：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/othermaterial/1-2.png)

我近期也是在非常迅速的想把想把项目完完全全迁移到Serverless架构上，之前也是做过一些工作的，如今再做这个工作，突然有了不一样的体会，当然这期间我也和很多开发者聊过类似的事情：迁移传统的Web应用上Serverless架构，有担心，也有犹豫，有诱惑力，又让人极其无语。



## 请求/响应 集成

首先说一下我心中阿里云和腾讯云两个厂商的函数计算在Web侧的最直接区别是什么：那就是集成（请求集成、响应集成）。

什么是请求集成呢，就是说我们有一个请求过来，是否可以被直接的发送给我们的应用，而不是再做一层奇奇怪怪的转换。

阿里云： 阿里云拥有一个Http函数，这个应该是Web迁移的一大杀器，他是直接具备将请求发给应用的能力，应用本身无需再做各种转化。同时在阿里云官网也是有着比较友好的提示和操作：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/othermaterial/1-3.png)

当然，在其开发者工具Funcruft中，也是支持将框架直接部署到函数计算上的。

之所以说他是一大杀器，是因为，我们的很多Web应用，在请求层面，是可以无需修改，即可部署到阿里云函数计算上的，无论你是上传了二进制的文件，还是做了什么奇奇怪怪的请求，这些Request对象都会直接发到你的应用，这种感觉有点类似于，他给我们提供了一个Nginx的能力。

腾讯云：相对阿里云而言，腾讯云在Web框架支持上，天生比较弱，因为他是依靠API网关的，并不是说腾讯云的API网管比较），而API网关至今还没有推出请求集成能力，也就是说，用户发起的所有请求，到函数计算侧都是字符串，所以这点就有一点难受了，尤其是迁移过程，在请求层面就会显得很"无助"，很"孤独"，为什么这么说呢？因为我们的很多上传能力，以及很多奇奇怪怪的Request都会被打成字符串的形式，那么原有项目本身用了二进制上传的小伙伴，不得不将其改成Base64或者干脆修改结构直接上传到对象存储中。

并且，在腾讯云上如果不使用Serverless Framework，一个传统的Web项目想要迁移到函数计算上，实际上，我们是要做一个非常庞大的转换工作的，以一个Flask项目的转换为例：

```python
def make_environ(event):
    environ = {}

    for hdr_name, hdr_value in event['headers'].items():
        hdr_name = hdr_name.replace('-', '_').upper()
        if hdr_name in ['CONTENT_TYPE', 'CONTENT_LENGTH']:
            environ[hdr_name] = hdr_value
            continue

        http_hdr_name = 'HTTP_%s' % hdr_name
        environ[http_hdr_name] = hdr_value

    apigateway_qs = event['queryStringParameters']
    request_qs = event['queryString']
    qs = apigateway_qs.copy()
    qs.update(request_qs)

    body = ''
    if 'body' in event:
        body = event['body']

    environ['REQUEST_METHOD'] = event['httpMethod']
    environ['PATH_INFO'] = event['path']
    environ['QUERY_STRING'] = urlencode(qs) if qs else ''
    environ['REMOTE_ADDR'] = 80
    environ['HOST'] = event['headers']['host']
    environ['SCRIPT_NAME'] = ''

    environ['SERVER_PORT'] = 80
    environ['SERVER_PROTOCOL'] = 'HTTP/1.1'

    environ['CONTENT_LENGTH'] = str(len(body))

    environ['wsgi.url_scheme'] = ''
    environ['wsgi.input'] = StringIO(body)
    environ['wsgi.version'] = (1, 0)
    environ['wsgi.errors'] = sys.stderr
    environ['wsgi.multithread'] = False
    environ['wsgi.run_once'] = True
    environ['wsgi.multiprocess'] = False

    BaseRequest(environ)

    return environ


class LambdaResponse(object):
    def __init__(self):
        self.status = None
        self.response_headers = None

    def start_response(self, status, response_headers, exc_info=None):
        self.status = int(status[:3])
        self.response_headers = dict(response_headers)


class FlaskLambda(Flask):
    def __call__(self, event, context):
        if 'httpMethod' not in event:
            print('httpMethod not in event')
            # In this "context" `event` is `environ` and
            # `context` is `start_response`, meaning the request didn't
            # occur via API Gateway and Lambda
            return super(FlaskLambda, self).__call__(event, context)

        response = LambdaResponse()
        # print response.start_response

        body = next(self.wsgi_app(
            make_environ(event),
            response.start_response
        ))

        return {
            'statusCode': response.status,
            'headers': response.response_headers,
            'body': body
        }
```

同样的Flask项目，在阿里云，我们只需要做的是：

1: 上传代码
2: 在设置一个handler即可

所以，由于腾讯云没有请求集成，实际上在整个传统Web项目迁移的过程中，是逊色阿里云的，但是有一件事是非常值得开心的，那就是腾讯在工具链层面建设的相对完善，目前可以通过其开发的一些Serverless Framework组件来直接迁移一些框架：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/othermaterial/1-4.png)

但是由于请求集成和响应集成的问题，这种转换层的能力也是非常难以满足更多框架的，例如Python著名的Web框架Tornado在腾讯云中，就没办法得以更好的支持，而在阿里云，只需要通过自定义runtime，可直接搞定：

1: 增加bootstrap文件，例如：
```
#!/usr/bin/env bash
export PORT=9000
export DEFAULTAPP=index.py
python $DEFAULTAPP
```
2：创建函数，上传代码即可。

当然，在阿里云也是有Funcruft这样的开发者工具，可以直接帮助用户一键部署/迁移Web项目的。

就目前为止，我更关心的其实已经不是这些普通的Runtime如何更好的兼容我的传统Web项目了，而是更加关注有了容器镜像的支持，阿里云在传统Web迁移过程中，会不会有更好，更便捷的表现？我相信，会的。



## 总结描述

通过上面的描述，我们可以看到这样一件事情：虽然腾讯云和阿里云都支持Web项目的迁移，但是实际上两者还是有一定差距的：

- 阿里云，更加注重底层的能力，更想从根本解决一些问题，包括响应集成、请求集成这些天然能力的支持，以及目前发布的容器镜像的支持。这两部分在实际使用过程中，无疑是非常棒的，无论是开发者的学习成本、迁移成本、使用成本都会变得很低。但是，阿里云为此引入了Http的概念，也会对体验造成一定的负面印象，例如在阿里云就有一些奇怪的设计
  - 有http函数，不能创建其他触发器（当然，我一般http函数貌似也不会创建其他触发器，我只是好奇这不都是触发器么？为啥要在这里区分的这么详细？技术的问题要靠用户额外去理解去平衡么）
  - 引入了http函数，那么他和apigateway的区别是什么？包括定位上的区别和能力上的区别，这都会是未来让用户可能比较困惑的一点

- 腾讯云，更加注重上层的用户体验，所谓的就是底层可能稍微逊色，但是我要在用户体验层面尽可能的追平，虽然表面上看是治标不治本的方法，但是不可否定的是也是非常有效果的，至少现在用户迁移Express等框架的时候，可以不用关心非常糟糕的转换层了，而是可以通过工具一键部署和迁移。只不过在某些比较细节的操作上，或者某些比较特殊的框架上还是很难更好的支持，例如：
  - 二进制上传以及例如Tornado这样的框架支持
  - 如果你用的某个框架他没有提供迁移能力，那么要不等待官方提供，要不自己动手丰衣足食，但是我相信90%的用户，如果看到没有对应组件支持，会直接放弃
  

当然，我在这里并不会说腾讯云和阿里云谁的做法会更好一些，因为在很多时候真的算是，各有千秋，各有优劣：

如果你更想通过根本，简单快速的迁移一些Web项目，那么我觉得阿里云可能会更合适，毕竟有一个http触发器，可以天然和传统的框架结合，无需更多转换层，同时还支持容器镜像，可以更加以"开发者"的姿态，将项目部署/迁移到云端；

如果你不在乎底层的支持和转换层的复杂又很反感阿里云引入的更多概念，而又恰好不用一些"奇奇怪怪"的能力（例如二进制上传等），所需要的框架也恰好是Serverless Framework提供的，那么我觉得，你也可以使用腾讯云，因为Serverless Framework在体验层做的真的比阿里云Funcruft做得好一些，至少站在我的角度，是这样的。



## 踩坑分享

最后，我要分享一下，我心中，迁移传统Web项目可能需要的一些注意事项：

1：整个项目迁移，是否会真的带来极致弹性和费用节约是未知的，因为有一些web项目中，拥有大量静态资源，如果将这些静态资源也放在了函数计算，我觉得这并不是一个最佳实践，因为这样的做法，可能会带来更大的并发量，更多的请求次数，更多的计费次数；
2：项目是否真的可以一键迁移，这个所谓的真的一键迁移并不是说厂商是否提供这样的能力，而是我们要自己问自己的，因为我觉得如果一个项目中，有一些接口或者功能的资源消耗，明显大于其他更多的接口或者功能，那么我觉得，要把这个部分单独提取出来，毕竟我们部署到函数计算上是要评估内存，超时时间这些配置信息的，而内存就是和计费挂钩的，如果盲目一键迁移，我相信，成本未必得到更好的控制；
3：项目中有上传等操作的部分，这里已经要注意，因为函数计算是实例级别的，一旦实例销毁，你上传的东西可能就丢了，所以这里推荐的是使用对象存储或者硬盘挂载会好一些；
4：状态存储，很多项目，尤其是中小型项目，都是直接缓存一些信息在本地的，但是Serverless架构下，更推荐这些缓存信息存储到Redis等地方（让然你要用对象存储等也都可以，就看自己的需求了），因为你存储函数计算中可能面临着下次请求是否会恰好到这个实例中，下次请求这个实例是否还会存在以及如果你有你缓存的目录，这个目录是否可读写；

Serverless架构确实在飞速发展，但是也确实还有很长的路要走。无论是阿里云还是腾讯云，我希望的是都可以在用户层面多思考，通过用户层面做事情，因为Serverless架构，不是厂商的玩具，厂商之间的彼此竞争，不如全都站在用户角度，为用户考虑，为用户办事，一切为了用户，用户的目光是雪亮的。

道阻且长，终有荣光，Serverless，未来可期！不，是未来已来，Serverless正当时。

-------

## 结束语

本文写于桂林，十一期间和朋友一起来桂林走一走，早晨起床，一直就在想这段时间迁移Web的过程，充满心酸和挑战，所以就写了这篇文章。另外实名推荐桂林的阳朔，这是一个非常棒的地方，大家以后有机会可以过来走一走，也祝大家十一快乐，好好休息。

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/othermaterial/1-5.jpeg)