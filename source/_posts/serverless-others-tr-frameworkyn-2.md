---
title: 传统框架部署到Serverless架构的利与弊
date: 2020.7.4
tags: [Serverless, 传统框架部署]
categories: [Serverless, 闲谈]
---

## 前言

Serverless架构是一个新的概念，也可以说是一个新的架构或者技术，但是无论他有多新，都不能一下子完成现有都开发习惯到Serverless架构的过渡，让现有的工程师放弃现有的Express、Koa、Flask、Django等框架直接在Serverless架构上开发项目，显然是不可能，就算可能，这也需要时间进行适应和过渡。

那么在这个过渡的期间我们是否可以考虑将现有的框架部署到Serverless架构上？如果要上，我们应该怎么顺利将我们的函数上云呢？

## Web框架在Serverless的表现

接下来，我们以Flask框架进行一个简单的测试：

* 测试四种接口：
	* Get请求（可能涉及到通过路径传递参数）
	* Post请求（通过Formdata传递参数）
	* Get请求（通过url参数进行参数传递）
	* Get请求（带有jieba等计算功能）

* 测试两种情况：
	* 本地表现
	* 通过Flask-Component部署表现

* 测试两种性能：
	* 传统云服务器上的性能表现
	* 云函数性能表现

首先是测试代码：

```python
from flask import Flask, redirect, url_for, request
import jieba
import jieba.analyse

app = Flask(__name__)


@app.route('/hello/<name>')
def success(name):
    return 'hello %s' % name


@app.route('/welcome/post', methods=['POST'])
def welcome_post():
    user = request.form['name']
    return 'POST %s' % user


@app.route('/welcome/get', methods=['GET'])
def welcome_get():
    user = request.args.get('name')
    return 'GET %s' % user


@app.route('/jieba/', methods=['GET'])
def jieba_test():
    str = "Serverless Framework 是业界非常受欢迎的无服务器应用框架，开发者无需关心底层资源即可部署完整可用的 Serverless 应用架构。Serverless Framework 具有资源编排、自动伸缩、事件驱动等能力，覆盖编码、调试、测试、部署等全生命周期，帮助开发者通过联动云资源，迅速构建 Serverless 应用。"
    print(", ".join(jieba.cut(str)))
    print(jieba.analyse.extract_tags(str, topK=20, withWeight=False, allowPOS=()))
    print(jieba.analyse.textrank(str, topK=20, withWeight=False, allowPOS=('ns', 'n', 'vn', 'v')))
    return 'success'


if __name__ == '__main__':
    app.run(debug=True)
```

这段测试代码是比较有趣的，它包括了最常用的请求方法、传参方法，也包括简单的接口和稍微复杂的接口。


## 本地表现

本地运行之后，通过Postman进行三个接口简单测试：

* Get请求：   
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-1.jpeg)

* Post参数传递：   
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-2.jpeg)

* Get参数传递：   
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-3.jpeg)

### 通过Flask-Component部署表现

接下来，我们将这个代码部署到云函数中：

通过Flask-Component部署，可以参考Tencent给予的文档，Github地址https://github.com/serverless-components/tencent-flask

Yaml文档内容：

```yaml
FlaskComponent:
  component: '@gosls/tencent-flask'
  inputs:
    region: ap-beijing
    functionName: Flask_Component
    code: ./flask_component
    functionConf:
      timeout: 10
      memorySize: 128
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
```

部署完成

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-4.jpeg)

接下来测试我们的目标三个接口

* Get通过路径传参：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-5.jpeg)

* Post参数传递：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-6.jpeg)

* Get参数传递：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-7.jpeg)

通过上面的测试，我们可以看出，通过Flask-Component部署的云函数，也是可以具备常用的几种请求形式和传参形式。

可以这样说，一般情况下，用户的Flask项目可以直接通过腾讯云提供的Flask-component快速部署到Serverless架构上，可以得到比较良好的运行。

### 简单的性能测试

接下来对性能进行一波简单的测试，首先购买一个云服务器，将这个部分代码部署到云服务器上。
在云上购买服务器，保守一点买了1核2G

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-8.jpeg)
然后配置环境，到服务可以跑起来：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-9.jpeg)
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-10.jpeg)

通过Post设置一下简单的Tests：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-11.jpeg)

然后对接口进行测试：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-12.jpeg)

非常顺利完成了接口测试：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-13.png)

可以通过接口测试结果进行部分可视化：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-14.jpeg)

同时对数据进行统计：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-15.jpeg)

可以看到，通过上图和上表，服务器测的整体响应时间都快于云函数的响应时间。而且可以看到函数存在冷启动，一按出现冷启动，其响应时间会增长20余倍。在由于上述测试，仅仅是非常简单的接口，接下来我们来测试一下稍微复杂的接口，使用了jieba分词的接口，因为jieba分词接口存在：

测试结果：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-16.png)

可视化结果：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-17.jpeg)
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-18.jpeg)

通过对Jieba接口的测试，可以看到虽然服务器也会有因分词组件进行初始化而产生比较慢的响应时间，但是整体而言，速度依旧是远远低于云函数。

那么问题来了，是函数本身的性能有问题，还是增加了Flask框架+APIGW响应集成之后才有问题？

接下来，做一组新的接口测试，在函数中，直接返回内容，而不进行额外处理，看看函数+API网关性能和正常情况下的服务器性能对比

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-19.jpeg)
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-20.jpeg)

可以看出虽然最小和平均耗时的区别不是很大，但是最大耗时基本上是持平。可以看出来，框架的加载会导致函数冷启动时间长度变得异常可怕。
接下来通过Python代码，对Flask框架进行并发测试：
对函数进行3次压测，每次并发301:

```
===========task end===========
total:301,succ:301,fail:0,except:0
response maxtime: 1.2727971077
response mintime 0.573610067368

===========task end===========
total:301,succ:301,fail:0,except:0
response maxtime: 1.1745698452
response mintime 0.172255039215

===========task end===========
total:301,succ:301,fail:0,except:0
response maxtime: 1.2857568264
response mintime 0.157210826874
```

对服务器进行3次压测，同样是每次并发301:

```
===========task end===========
total:301,succ:301,fail:0,except:0
response maxtime: 3.41151213646
response mintime 0.255661010742

===========task end===========
total:301,succ:301,fail:0,except:0
response maxtime: 3.37784004211
response mintime 0.212490081787

===========task end===========
total:301,succ:301,fail:0,except:0
response maxtime: 3.39548277855
response mintime 0.439364910126
```

通过这一波压测，我们可以看到这样一个奇怪现象，那就是在函数和服务器预热完成之后，连续三次并发301个请求。函数的整体表现，反而比服务器的要好。这也说明了在Serverless架构下，弹性伸缩的一个非常重要的表现。传统服务器，我们如果出现了高并发现象，很容易会导致整体服务受到严重影响，例如响应时间变长，无响应，甚至是服务器直接挂掉，但是在Serverless架构下，这个弹性伸缩的能力是云厂商帮助我们做的，所以在并发量达到一定的时候，其实Serverless架构的优势变得会更加明显。


## 传统Web框架上云方法（以Python Web框架为例）

### 分析已有Component（Flask为例）

首先第一步，我们要知道其他的框架是怎么运行的，例如Flask等，我们先通过腾讯云的Flask-Component，按照他的说明部署一下：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-21.jpeg)

非常简单轻松愉快的部署上线，然后在函数的控制台，我们把部署好的下载下来，研究一下：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-22.jpeg)

下载解压之后，我们可以看这样一个目录结构：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-23.jpeg)

蓝色框起来的，是依赖包，黄色的app.py是我们的自己写的代码，那么红色圈起来的是什么？这两个文件从哪里出来的？
api_server.py文件内容：

```python
import app  # Replace with your actual application
import severless_wsgi

# If you need to send additional content types as text, add then directly
# to the whitelist:
#
# serverless_wsgi.TEXT_MIME_TYPES.append("application/custom+json")

def handler(event, context):
    return severless_wsgi.handle_request(app.app, event, context)
```

可以看到，这里面是将我们创建的app.py文件引入，并且拿到了app这个对象，并且将event和context同时传递给severless_wsgi.py中的handle_reques方法中，那么问题来了，这个方法是什么？

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-24.png)

这个方法内容好多......看着有点眼晕，但是，我们可以直接发现这一段代码：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-25.jpeg)

这一段是什么呢？这一段实际上就是将我们拿到的参数（event和context）进行转换，转换之后统一environ中，然后接下来通过werkzeug这个依赖，将这个内容变成request对象，并且与我们刚才说的app对象一起调用from_app方法。获得到反馈：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-26.jpeg)

并且按照API网关的响应集成的格式，将结果返回。
此时此刻，各位看官可能有点想法了，貌似有一丢丢灵感出现了，那么我们不妨看一下Flask/Django这些框架的实现原理：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-27.jpeg)

通过这个简版的原理图，和我刚才说的内容，我们可以想到，实际上正常用的时候要通过web_server，进入到下一个环节，而我们云函数更多是一个函数，本不需要启动web server，所以我们就可以直接调用wsgi_app这个方法，其中这里的environ就是我们刚才的通过对event/context等进行处理后的对象，start_response可以认为是我们的一种特殊的数据结构，例如我们的response结构形态等。所以，如果我们自己想要实现这个过程，不使用腾讯云flask-component，可以这样做：

```python
import sys

try:
    from urllib import urlencode
except ImportError:
    from urllib.parse import urlencode

from flask import Flask

try:
    from cStringIO import StringIO
except ImportError:
    try:
        from StringIO import StringIO
    except ImportError:
        from io import StringIO

from werkzeug.wrappers import BaseRequest

__version__ = '0.0.4'


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
            return super(FlaskLambda, self).__call__(event, context)

        response = LambdaResponse()

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

这样一个流程，就会变得更加简单，清楚。整个实现过程，可以认为是对web server部分进行了一种“截断”或者是“替换”：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-28.png)

这就是对Flask-Component的基本分析思路，那么按照这个思路，我们是否可以将Django框架部署上Serverless架构呢？那么Flask和Django有什么区别呢？我这里的区别特指的是在运行启动过程中。

## 拓展思路：实现Django-component

仔细想一下，貌似并没有区别，那么我们是不是可以直接用Flask这个转换逻辑，将flask的app替换成django的app呢？
把：

```python
from flask import Flask
app = Flask(__name__)
```

替换成：

```python
import os
from django.core.wsgi import get_wsgi_application
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mydjango.settings')
application = get_wsgi_application()
```

是否就能解决问题呢？
我们不妨试一下：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-29.jpeg)

建立好Django项目，直接增加index.py：

```python
# -*- coding: utf-8 -*-

import os
import sys
import base64
from werkzeug.datastructures import Headers, MultiDict
from werkzeug.wrappers import Response
from werkzeug.urls import url_encode, url_unquote
from werkzeug.http import HTTP_STATUS_CODES
from werkzeug._compat import BytesIO, string_types, to_bytes, wsgi_encoding_dance
import mydjango.wsgi

TEXT_MIME_TYPES = [
    "application/json",
    "application/javascript",
    "application/xml",
    "application/vnd.api+json",
    "image/svg+xml",
]


def all_casings(input_string):
    if not input_string:
        yield ""
    else:
        first = input_string[:1]
        if first.lower() == first.upper():
            for sub_casing in all_casings(input_string[1:]):
                yield first + sub_casing
        else:
            for sub_casing in all_casings(input_string[1:]):
                yield first.lower() + sub_casing
                yield first.upper() + sub_casing


def split_headers(headers):
    """
    If there are multiple occurrences of headers, create case-mutated variations
    in order to pass them through APIGW. This is a hack that's currently
    needed. See: https://github.com/logandk/serverless-wsgi/issues/11
    Source: https://github.com/Miserlou/Zappa/blob/master/zappa/middleware.py
    """
    new_headers = {}

    for key in headers.keys():
        values = headers.get_all(key)
        if len(values) > 1:
            for value, casing in zip(values, all_casings(key)):
                new_headers[casing] = value
        elif len(values) == 1:
            new_headers[key] = values[0]

    return new_headers


def group_headers(headers):
    new_headers = {}

    for key in headers.keys():
        new_headers[key] = headers.get_all(key)

    return new_headers


def encode_query_string(event):
    multi = event.get(u"multiValueQueryStringParameters")
    if multi:
        return url_encode(MultiDict((i, j) for i in multi for j in multi[i]))
    else:
        return url_encode(event.get(u"queryString") or {})


def handle_request(application, event, context):

    if u"multiValueHeaders" in event:
        headers = Headers(event["multiValueHeaders"])
    else:
        headers = Headers(event["headers"])

    strip_stage_path = os.environ.get("STRIP_STAGE_PATH", "").lower().strip() in [
        "yes",
        "y",
        "true",
        "t",
        "1",
    ]
    if u"apigw.tencentcs.com" in headers.get(u"Host", u"") and not strip_stage_path:
        script_name = "/{}".format(event["requestContext"].get(u"stage", ""))
    else:
        script_name = ""

    path_info = event["path"]
    base_path = os.environ.get("API_GATEWAY_BASE_PATH")
    if base_path:
        script_name = "/" + base_path

        if path_info.startswith(script_name):
            path_info = path_info[len(script_name) :] or "/"

    if u"body" in event:
        body = event[u"body"] or ""
    else:
        body = ""

    if event.get("isBase64Encoded", False):
        body = base64.b64decode(body)
    if isinstance(body, string_types):
        body = to_bytes(body, charset="utf-8")

    environ = {
        "CONTENT_LENGTH": str(len(body)),
        "CONTENT_TYPE": headers.get(u"Content-Type", ""),
        "PATH_INFO": url_unquote(path_info),
        "QUERY_STRING": encode_query_string(event),
        "REMOTE_ADDR": event["requestContext"]
        .get(u"identity", {})
        .get(u"sourceIp", ""),
        "REMOTE_USER": event["requestContext"]
        .get(u"authorizer", {})
        .get(u"principalId", ""),
        "REQUEST_METHOD": event["httpMethod"],
        "SCRIPT_NAME": script_name,
        "SERVER_NAME": headers.get(u"Host", "lambda"),
        "SERVER_PORT": headers.get(u"X-Forwarded-Port", "80"),
        "SERVER_PROTOCOL": "HTTP/1.1",
        "wsgi.errors": sys.stderr,
        "wsgi.input": BytesIO(body),
        "wsgi.multiprocess": False,
        "wsgi.multithread": False,
        "wsgi.run_once": False,
        "wsgi.url_scheme": headers.get(u"X-Forwarded-Proto", "http"),
        "wsgi.version": (1, 0),
        "serverless.authorizer": event["requestContext"].get(u"authorizer"),
        "serverless.event": event,
        "serverless.context": context,
        # TODO: Deprecate the following entries, as they do not comply with the WSGI
        # spec. For custom variables, the spec says:
        #
        #   Finally, the environ dictionary may also contain server-defined variables.
        #   These variables should be named using only lower-case letters, numbers, dots,
        #   and underscores, and should be prefixed with a name that is unique to the
        #   defining server or gateway.
        "API_GATEWAY_AUTHORIZER": event["requestContext"].get(u"authorizer"),
        "event": event,
        "context": context,
    }

    for key, value in environ.items():
        if isinstance(value, string_types):
            environ[key] = wsgi_encoding_dance(value)

    for key, value in headers.items():
        key = "HTTP_" + key.upper().replace("-", "_")
        if key not in ("HTTP_CONTENT_TYPE", "HTTP_CONTENT_LENGTH"):
            environ[key] = value

    response = Response.from_app(application, environ)

    returndict = {u"statusCode": response.status_code}

    if u"multiValueHeaders" in event:
        returndict["multiValueHeaders"] = group_headers(response.headers)
    else:
        returndict["headers"] = split_headers(response.headers)

    if event.get("requestContext").get("elb"):
        # If the request comes from ALB we need to add a status description
        returndict["statusDescription"] = u"%d %s" % (
            response.status_code,
            HTTP_STATUS_CODES[response.status_code],
        )

    if response.data:
        mimetype = response.mimetype or "text/plain"
        if (
            mimetype.startswith("text/") or mimetype in TEXT_MIME_TYPES
        ) and not response.headers.get("Content-Encoding", ""):
            returndict["body"] = response.get_data(as_text=True)
            returndict["isBase64Encoded"] = False
        else:
            returndict["body"] = base64.b64encode(response.data).decode("utf-8")
            returndict["isBase64Encoded"] = True

    return returndict



def main_handler(event, context):
    return handle_request(mydjango.wsgi.application, event, context)
```

然后我们部署到函数上，看一下效果：
函数信息：

```python
from django.shortcuts import render
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt

# Create your views here.
@csrf_exempt
def hello(request):
    if request.method == "POST":
        return HttpResponse("Hello world ! " + request.POST.get("name"))
    if request.method == "GET":
        return HttpResponse("Hello world ! " + request.GET.get("name"))

```

通过部署完成，并绑定apigw触发器，然后在postman中进行测试：
get：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-30.png)

post：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-31.png)

可以看到，通过我们对运行原理的基本剖析和对django的改造，我们已经通过增加一个文件和相关依赖的方法，实现了Django上Serverless的过程。

接下来，我们看一下，如何将这个代码写成一个Component：
首先Clone下来Flask-Component的代码：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-32.jpeg)

然后，我们按照Django的部分模式进行修改：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-33.jpeg)

第一部分，是我们可能会依赖的一个依赖包，以及我们刚才放入的index.py文件。在用户调用这个Component的时候，我们会把这两个文件，放入用户的代码中，一并上传。
第二部分是Serverless.js部分，这里的一个基本格式：

```javascript
const { Component } = require('@serverless/core')
class TencentDjango extends Component {
  async default(inputs = {}) {
  }
  async remove(inputs = {}) {
  }
}
module.exports = TencentDjango
```

用户在执行sls的时候，会默认调用default的方法，在执行sls remove的时候会调用remove的方法，所以可以认default的内容是部署，而remove的内容是移除。

部署这里主要流程也蛮简单的，首先将文件进行复制和处理，然后直接调用云函数的组件，通过函数中的include参数将这些文件额外加入，再通过调用apigw的组件来进网关的管理，而用户写的yaml中inpust的内容，会在inputs中获取，我们要做的就是对应的传给不同的组件：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-3-34.jpeg)

当然除了这两部分对应放过去，上面的region等一些信息也要对应的进行处理。而调用底层组件方法也很简单：

```javascript
const tencentCloudFunction = await this.load('@serverless/tencent-scf'
const tencentCloudFunctionOutputs = await tencentCloudFunction(inputs)
```

处理好这里之后，只需要修改一下package.json和readme就可以了。至此，我们完成了一个Django Component的开发，当我们发布到NPM之后，在使用的时候，只需要引入这个Component就好：

```yaml
DjangoTest:
  component: '@serverless/tencent-django'
  inputs:
    region: ap-guangzhou
    functionName: DjangoFunctionTest
    djangoProjectName: mydjango
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

```

## 总结：

* Flask是可以通过很简单的方法上Serverless架构，用户基本上可以按照原生Flask开发习惯来开发Flask项目，尤其是使用Flask开发接口服务的项目，更是可以比较容易的迁移到Serverless架构。
* 整体框架迁移上Serverless架构可能要要注意几个额外的点：
1.  如果接口比较多，可能要按照资源消耗比较大的那个接口来设置内存大小，以我例子中的情况，非jieba接口使用的时候，可以使用最小内存（64M），jieba接口使用的时候，需要256M的内存，而整个项目是一体的，只能设置一个内存，所以为了保证项目可用性，就会整体设置为256M的内存，这样一来如果另外三个接口访问比较多的前提下，可能资源消耗会相对增加比较大，所以，如果有条件的话，可考虑将资源消耗比较大的接口额外提取出来；
2.  云函数+API网关的组合对静态资源以及文件上传等的支持可能并不是十分友好，尤其是云函数+API网关的双重收费，所以这里建议将Flask中的一些静态资源统一放在对象存储中，同时将文件上传逻辑修改成优先上传到对象存储中，可以参考之前的文章：【实践与踩坑】用Serverless怎么上传文件？
* 框架越大，或者框架内的资源越多函数冷启动的时间可能会越大。这一点是非常值得重视的。在刚才测试过程中，非框架下，最高耗时是平均耗时的3倍，而在加载Flask框架和Jieba的前提下，最高耗时是平均的10+倍！如果可以保证函数都是热启动还好，一旦出现冷启动，可能会有一定的影响。
* 由于用户发起请求是客户端到API网关再到函数，然后从函数回到API网关，再回到客户端，这个过程相对直接访问服务器获得结果的链路明显长了一些，所以在实际测试过程中小用户量对的表现发而不是很好，几次测试，基本上1核2G的服务器都是优于函数表现。但是当并发量上来的之后可以看到函数的表现实现了大超车，一度超越这台1核2G的服务器。那么这里有一个有趣的结论：对于极小规模请求，函数是按量付费，虽然性能上有一定的劣势，但是按量付费在价格上有一定的优势；当流量逐渐变大之后，函数在性能上的优势也逐渐凸显。

除了上面对传统Web框架部署到Serverless架构的一个利弊分析之外，通过对Flask框架进行分析，我们可以总结出Web框架上Serverless架构的原理思路，虽然说Serverless团队很活跃，各个云厂商也很活跃，但是我们在实际生产中，必然会用到很多定制化组件，Serverless Component本身就是组件化的产品，如果深度使用，我们就可以定制化开发Component满足自身需求，这时候就要知道一些原理和技巧。

我相信，Serverless架构会随着时间的发展，越发的成熟，目前可能还有或多或少的问题，但是不久的将来，一定不负众望。

