---
title: 云函数中使用Python-ORM-Peewee
date: 2020.6.10
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework, ORM, Peewee]
categories: [Serverless, 腾讯云, 领域实战]
---

## 前言

ORM（Object Ralational Mapping，对象关系映射）用来把对象模型表示的对象映射到基于SQL的关系模型数据库结构中去。这样，我们在具体的操作实体对象的时候，就不需要再去和复杂的SQL语句打交道，只需简单的操作实体对象的属性和方法。ORM技术是在对象和关系之间提供了一条桥梁，前台的对象型数据和数据库中的关系型的数据通过这个桥梁来相互转化。

在传统的开发过程中，无论是Java还是Python抑或Go，Nodejs等，都会有自己对应的一些工具，让我们可以快速、简单的使用数据库。以Python为例，Django是常见的Python Web框架，其提供的ORM组件就是非常方便的。除此之外，Python语言还有很多常见的ORM工具，例如SQLObject、Storm、SQLAlchemy以及peewee。 

本文将会通过在云函数中使用peewee，来进行简单的操作，将POST请求中的某些参数存储到数据库中，再读取出来。

## 入门peewee

Peewee是一个简单小巧的Python ORM，它非常容易学习，并且使用起来很直观。在其官方文档中，我们可以看到这样一个Demo：

```python
from peewee import *

db = SqliteDatabase('people.db')

class Person(Model):
    name = CharField()
    birthday = DateField()

    class Meta:
        database = db # This model uses the "people.db" database.
```

通过这样一段代码，我们可以基本明确几件事：

使用的时候可以直接导入依赖：`from peewee import *`，需要先初始化数据库链接：`SqliteDatabase('people.db')`，需要建立一个`class`用来描述结构，并且要继承`Model`。

通过对文档阅读，我们可以看到peewee的增删改查和Django所提供的ORM增删改查很是相似。此处不进行更多说明。

## 本地调试

按照腾讯云的Serverless分类下的云函数格式要求，我们先完成一个基础框架:

```python
import json
from peewee import *

# 连接数据库

# 定义ServerlessFramework

def main_handler(event, context):
    # 相关操作
    return None
```

这里需要注意，由于函数是无状态的，且存在容器的复用，所以这里链接数据库的操作，要在入口方法外进行，这样在一定程度下，可以提升性能和维持稳定。

* 方法内连接数据库：函数每次被触发（方法每次被调用）都会进行数据库连接操作
* 方法外连接数据库：只会在容器启动时进行初始化/数据库连接，之后函数的每次触发，都会复用当前的数据库连接

为了在本地调试的更加愉快，此处增加一个新的`test()`方法: 

```python
def test():
    event = {'body': json.dumps({"name": "hello world"}), 'headerParameters': {}, 'headers': {
        'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
        'accept-encoding': 'gzip, deflate', 'accept-language': 'zh-CN,zh;q=0.9', 'cache-control': 'no-cache',
        'connection': 'keep-alive', 'content-length': '27', 'content-type': 'application/x-www-form-urlencoded',
        'cookie': 'Hm_lvt_a0c900918361b31d762d9cf4dc81ee5b=1574491278,1575257377', 'endpoint-timeout': '15',
        'host': 'blog.0duzhan.com', 'origin': 'http://blog.0duzhan.com', 'pragma': 'no-cache',
        'proxy-connection': 'keep-alive', 'referer': 'http://blog.0duzhan.com/admin/tag/new/?url=%2Fadmin%2Ftag%2F',
        'upgrade-insecure-requests': '1',
        'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36',
        'x-anonymous-consumer': 'true', 'x-api-requestid': '656622f3b008a0d406a376809b03b52c',
        'x-b3-traceid': '656622f3b008a0d406a376809b03b52c', 'x-qualifier': '$LATEST'}, 'httpMethod': 'POST',
             'path': '/admin/tag/new/', 'pathParameters': {}, 'queryString': {'url': '/admin/tag/'},
             'queryStringParameters': {},
             'requestContext': {'httpMethod': 'ANY', 'identity': {}, 'path': '/admin', 'serviceId': 'service-23ybmuq7',
                                'sourceIp': '119.123.224.87', 'stage': 'release'}}
    print(main_handler(event, None))


if __name__ == "__main__":
    test()
```

这样做的目的是，可以直接通过执行该文件，模拟线上的API网关触发器，例如当我的`main_handler`方法为以下代码时：

```python
def main_handler(event, context):
    print("Event: ", event)
```

执行当前文件会输出：

```text
Event:  {'body': '{"name": "hello world"}', 'headerParameters': {}, 'headers': {'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9', 'accept-encoding': 'gzip, deflate', 'accept-language': 'zh-CN,zh;q=0.9', 'cache-control': 'no-cache', 'connection': 'keep-alive', 'content-length': '27', 'content-type': 'application/x-www-form-urlencoded', 'cookie': 'Hm_lvt_a0c900918361b31d762d9cf4dc81ee5b=1574491278,1575257377', 'endpoint-timeout': '15', 'host': 'blog.0duzhan.com', 'origin': 'http://blog.0duzhan.com', 'pragma': 'no-cache', 'proxy-connection': 'keep-alive', 'referer': 'http://blog.0duzhan.com/admin/tag/new/?url=%2Fadmin%2Ftag%2F', 'upgrade-insecure-requests': '1', 'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36', 'x-anonymous-consumer': 'true', 'x-api-requestid': '656622f3b008a0d406a376809b03b52c', 'x-b3-traceid': '656622f3b008a0d406a376809b03b52c', 'x-qualifier': '$LATEST'}, 'httpMethod': 'POST', 'path': '/admin/tag/new/', 'pathParameters': {}, 'queryString': {'url': '/admin/tag/'}, 'queryStringParameters': {}, 'requestContext': {'httpMethod': 'ANY', 'identity': {}, 'path': '/admin', 'serviceId': 'service-23ybmuq7', 'sourceIp': '119.123.224.87', 'stage': 'release'}}
None
```

此时我们建立一个数据库：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-1-1.png)

并且完成代码：

```python
import json
from peewee import *

# 连接数据库
database = MySQLDatabase('数据库名', user='数据库用户', host='数据库Host', port=数据库端口,
                         password="数据库密码")


# 定义ServerlessFramework
class ServerlessFramework(Model):
    componentName = CharField()

    class Meta:
        database = database


# 建立表（此处推荐不在函数中建立，最好初始化之后，屏蔽掉该语句）
# ServerlessFramework.create_table()

def main_handler(event, context):
    print("Event: ", event)
    
    body = json.loads(event["body"])
    print("Body: ", body)

    try:
        ServerlessFramework(componentName=body["name"]).save()
        return [eve.componentName for eve in ServerlessFramework.select()]
    except Exception as e:
        return str(e)


def test():
    event = {'body': json.dumps({"name": "hello world"}), 'headerParameters': {}, 'headers': {
        'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
        'accept-encoding': 'gzip, deflate', 'accept-language': 'zh-CN,zh;q=0.9', 'cache-control': 'no-cache',
        'connection': 'keep-alive', 'content-length': '27', 'content-type': 'application/x-www-form-urlencoded',
        'cookie': 'Hm_lvt_a0c900918361b31d762d9cf4dc81ee5b=1574491278,1575257377', 'endpoint-timeout': '15',
        'host': 'blog.0duzhan.com', 'origin': 'http://blog.0duzhan.com', 'pragma': 'no-cache',
        'proxy-connection': 'keep-alive', 'referer': 'http://blog.0duzhan.com/admin/tag/new/?url=%2Fadmin%2Ftag%2F',
        'upgrade-insecure-requests': '1',
        'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36',
        'x-anonymous-consumer': 'true', 'x-api-requestid': '656622f3b008a0d406a376809b03b52c',
        'x-b3-traceid': '656622f3b008a0d406a376809b03b52c', 'x-qualifier': '$LATEST'}, 'httpMethod': 'POST',
             'path': '/admin/tag/new/', 'pathParameters': {}, 'queryString': {'url': '/admin/tag/'},
             'queryStringParameters': {},
             'requestContext': {'httpMethod': 'ANY', 'identity': {}, 'path': '/admin', 'serviceId': 'service-23ybmuq7',
                                'sourceIp': '119.123.224.87', 'stage': 'release'}}
    print(main_handler(event, None))


if __name__ == "__main__":
    test()

```

此时我们本地执行可以得到结果：

```text
Event:  {'body': '{"name": "hello world"}', 'headerParameters': {}, 'headers': {'accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9', 'accept-encoding': 'gzip, deflate', 'accept-language': 'zh-CN,zh;q=0.9', 'cache-control': 'no-cache', 'connection': 'keep-alive', 'content-length': '27', 'content-type': 'application/x-www-form-urlencoded', 'cookie': 'Hm_lvt_a0c900918361b31d762d9cf4dc81ee5b=1574491278,1575257377', 'endpoint-timeout': '15', 'host': 'blog.0duzhan.com', 'origin': 'http://blog.0duzhan.com', 'pragma': 'no-cache', 'proxy-connection': 'keep-alive', 'referer': 'http://blog.0duzhan.com/admin/tag/new/?url=%2Fadmin%2Ftag%2F', 'upgrade-insecure-requests': '1', 'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36', 'x-anonymous-consumer': 'true', 'x-api-requestid': '656622f3b008a0d406a376809b03b52c', 'x-b3-traceid': '656622f3b008a0d406a376809b03b52c', 'x-qualifier': '$LATEST'}, 'httpMethod': 'POST', 'path': '/admin/tag/new/', 'pathParameters': {}, 'queryString': {'url': '/admin/tag/'}, 'queryStringParameters': {}, 'requestContext': {'httpMethod': 'ANY', 'identity': {}, 'path': '/admin', 'serviceId': 'service-23ybmuq7', 'sourceIp': '119.123.224.87', 'stage': 'release'}}
Body:  {'name': 'hello world'}
['test', 'test2', 'test2', 'hello world']
```

可以看到，我们的程序是没问题，数据也可以成功被写入到数据库：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-1-2.png)

## 部署到线上

部署到线上，我们可以将数据库所在子网和云函数所在子网设置成一致，通过内网直接使用：

* 数据库子网：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-1-4.png)

* 云函数VPC：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-1-3.png)

完成之后，我们需要在项目当前目录下安装相关依赖：`pip3 install peewee -t ./`

同时我们需要将数据的链接地址，修改成内网地址，最后上传代码包函数：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-1-5.png)

通过PostMan进行测试：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-1-6.png)

当然，如果想通过Serverless命令行工具部署也是比较容易的：

```yaml
MyPeeweeTest:
  component: "@serverless/tencent-scf"
  inputs:
    name: mypeewee_test
    codeUri: ./src
    handler: index.main_handler
    runtime: Python3.6
    region: ap-guangzhou
    vpcConfig:
      subnetId: 'subnet-j2ga1kew'
      vpcId: 'vpc-h3933w5f'
    events:
      - apigw:
          name: serverless_test
          parameters:
            serviceId: service-cyjmc4eg
            protocols:
              - http
            description: the serverless service
            environment: release
            endpoints:
              - path: /users
                method: POST
```

## 总结

至此，我们完成了在云函数中使用peewee的Demo，这个Demo还有很多可以优化的地方，例如将数据库等信息写入到环境变量中，这样便于我们更新维护等。

云函数其实是可以使用绝大部分框架和依赖的，就看我们如何去使用，当然这个组件是比较简单的，不需要进行改造，有一些组件就算不能用，也是可以根据一些改造，将其跑在云函数中。