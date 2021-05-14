---
title: Serverless架构下怎么优雅的上传文件？
date: 2020.7.15
tags: [Serverless, 开发经验, 文件上传]
categories: [Serverless, 闲谈]
---


# 第二篇： 

## 前言

在传统开发过程中，我们对文件上传部分相对来说是比较自由：上传什么文件/怎么上传/存储到哪里等问题的决定权往往在我们自己这里，并没有太多的问题。但是在Serverless架构下，我们往往上传文件就没有这么自由了。无论是成本的原因还是某些服务的限制，我们都需要寻求一些比较"优"的解决方案。

## Serverless架构与文件上传

由于Serverless架构中的函数计算部分是没有办法做文件持久化的，因为函数执行的容器，用完过后会被回收，所以也就是说你想要存储文件需要借助对象存储等相关的服务。当然将文件上传到对象存储服务的方法有很多，这里只说两种：

* 函数计算->对象存储
* 对象存储

一般情况下，我们一个业务如果有文件上传功能，我们通常都会用`multipart/form-data`，或者将文件进行`base64`编码之后再上传。但是在Serverless架构下，这种思路要有一些变化。

首先说`函数计算->对象存储`这种方法，这种方法上传文件，是相对来说比较容易，也是比较常见的，我们将文件直接通过API网关，传送到云函数中，在云函数重做一些处理（例如压缩图像，视频转码，数据入库等），然后再由云函数将结果存储到对象存储中，做文件资源的持久化。这种做法是比较流畅的，也是正常思路。但是理想很丰满，现实很骨感：
* 首先说直接通过`multipart/form-data`上传，将文件直接通过网关传给函数的问题，函数计算通过API网关获得到的数据结构，一般来说都是一个JSON格式，或者某些字段干脆就是字符串，所以这样的一个设定，会让函数计算中对二进制的支持非常不友好；所以我们只能通过将文件转换为`base64`编码之后在进行传输，通过API网关之后，函数部分收到这个数据，再将`base64`编码的文件解码，然后做一些处理之后，持久化对象存储中。
* 其次无论是腾讯云的SCF，还是AWS的Lambda，在通过API网关触发函数的时候，都是有一个数据包大小限制的。以腾讯云为例，这个限制是6M。也就是说，你无论发送多大的数据，在API网关到函数计算部分，是有一个数据包的最大限制，如果你上传文件过大，这里就无法进行资源的传输。也就是说，通过上传文件到云函数这个case只能上传6M以下的文件。那么这个6M是一个什么概念呢？上文已经描述过，函数计算对接受二进制文件表现的非常不友好，所推荐将文件`base64`编码之后传输，编码之后的数据包通常会变大一些，一就是说通过这种方法，我们上传到云函数的数据包可能只有4M左右。那么4M又是一个什么概念呢：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-2-1.png)
上图所示，我拿出了我的手机，拍了一张图片，图片大小是6.21M，也就是说如果我想把这张图片上传到SCF来进行一些处理是“不可能”事件，或者说，我就做了一个相册的功能，我就直接把图片上传到函数计算，函数计算再将其存储到对象存储中，这个操作是因为数据包大小而被限制住。

当然，上面这种方法除了对文件大小限制之外，对成本也是有一定影响的，因为API网关相对来说并不是一个对文件进行传输的服务，为什么这么说的，我们就单纯的从流量费用来看对象存储和API网关的区别：

* API网关的收费：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-2-4.png)

* 对象存储的收费：
![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-2-5.png)

可以看到单纯根据流量来看API网关的费用就比COS高了很多，其实也可以理解，毕竟API网关更多定位可能是控制流，而真正的数据存储传输这一部分还是对象存储更合适一些。那么我们有没有什么方案可以直接上传文件等资源到对象存储呢？如果我们直接将资源上传到对象存储，这条资源数据又如何入库呢（例如用户上传图片到自己的相册功能，使用传统方法，系统接收到图片，将图片存储，将数据入库，但是如果图片直接上传到对象存储，我们怎么会知道这个图片是那个用户给我们的）？同时，将文件上传到对象存储需要写入权限，那么是将权限开发？还是使用密钥？如果是一个Web服务，这个密钥信息又应该存储在哪里？如何存储？

所以此时此刻，就衍生出了第二种解决方法：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-2-2.png)

在直传`对象存储`方法中，客户端发起三个请求，分别是获取临时上传地址、将文件上传到COS、获取处理结果（当然，如果不需要获取处理结果什么的，例如就是用户单纯的上传个文件到自己的账号下，那这种情况就不需要第三次请求了）。


相对于之前的方法，这种方法会稍微复杂一下，但是这种方法对二进制上传、对文件资源的大小以及成本控制都能，都有很好的支持。当然，也并不是说每种场景都只用这个方案，因为不同的方案，在不同的场景中，可能真的有一定的差异，但是不管怎么说，在Serverless架构下，这个方案都是较优的。

针对不同场景的的不同适用方案：

* 场景1: 用户上传头像功能
针对这样的场景，其实直接选用方案1，就可以了。因为一般情况下，头像都不是很大的，完全可以在客户端对图像进行一次压缩和裁剪，完成之后，直接带着用户的一些参数，例如用户的token等，上传到函数计算，在函数计算中对图片转存到对象存储以及将图像和用户信息进行关联，并将某些结果返回给客户端。整个流程只需要一个函数，方便快捷。


* 场景2: 用户上传图片到相册系统中
针对这样的场景，其实方案2是更好的，因为很多时候上传图片到相册，都是会希望保留原图，而不希望被压缩，那么原图大小很可能超过6M，方案1也并不是十分合理，而且APIGW+函数计算的组合，本身就不是非常适合进行文件的传输等，这个时候优先上传对象存储是比较合理的方案。
用户可以带着图像要上传的相册以及图片名称，用户的token发起获取临时密钥到函数1中，函数1将用户、相册、图片以及状态（例如待上传、待处理、已处理等）等信息关联，并且存储，然后将临时地址返回给客户端，客户端将图片上传到对象存储中，通过对象存储触发器触发函数2，函数2对图像进行压缩（一般情况下，相册列表都会显示压缩图片，点到相册详情才会有完整的无损图片），并且和之前信息进行关联，修改数据状态。在用户上传图片完成之后，如果有需要，客户端就可以发起第三次请求获取图像存储/处理结果，函数3会查询数据库状态，在某个时间阈值内，如果数据状态是完成，则表示数据已经上传并且完成了部分处理，否则会返回对应的异常信息。

## 代码实例

接下来分享上面两种方法的实现过程：
函数1，实现第一种方案，文件通过Base64，传递到SCF，由SCF转存到COS：

```python
def uploadToScf(event, context):
    print('event', event)
    print('context', context)
    body = json.loads(event['body'])

    # 可以通过客户端传来的token进行鉴权，只有鉴权通过才可以获得临时上传地址
    # 这一部分可以按需修改，例如用户的token可以在redis获取，可以通过某些加密方法获取等
    # 也可以是传来一个username和一个token，然后去数据库中找这个username对应的token是否
    # 与之匹配等，这样会尽可能的提升安全性
    if "key" not in body or "token" not in body or body['token'] != 'mytoken' or "key" not in body:
        return {"url": None}

    pictureBase64 = body["picture"].split("base64,")[1]
    with open('/tmp/%s' % body['key'], 'wb') as f:
        f.write(base64.b64decode(pictureBase64))
    region = os.environ.get("region")
    secret_id = os.environ.get("TENCENTCLOUD_SECRETID")
    secret_key = os.environ.get("TENCENTCLOUD_SECRETKEY")
    token = os.environ.get("TENCENTCLOUD_SESSIONTOKEN")
    config = CosConfig(Region=region, SecretId=secret_id, SecretKey=secret_key, Token=token)
    client = CosS3Client(config)
    response = client.upload_file(
        Bucket=os.environ.get("bucket_name"),
        LocalFilePath='/tmp/%s' % body['key'],
        Key=body['key'],
    )
    return {
        "uploaded": 1,
        "url": 'https://%s.cos.%s.myqcloud.com' % (
            os.environ.get("bucket_name"), os.environ.get("region")) + body['key']
    }

```

函数1，实现第二种方案，进行临时签名URL的获取：

```python
def getPresignedUrl(event, context):
    print('event', event)
    print('context', context)
    body = json.loads(event['body'])

    # 可以通过客户端传来的token进行鉴权，只有鉴权通过才可以获得临时上传地址
    # 这一部分可以按需修改，例如用户的token可以在redis获取，可以通过某些加密方法获取等
    # 也可以是传来一个username和一个token，然后去数据库中找这个username对应的token是否
    # 与之匹配等，这样会尽可能的提升安全性
    if "key" not in body or "token" not in body or body['token'] != 'mytoken' or "key" not in body:
        return {"url": None}

    # 初始化COS对象
    region = os.environ.get("region")
    secret_id = os.environ.get("TENCENTCLOUD_SECRETID")
    secret_key = os.environ.get("TENCENTCLOUD_SECRETKEY")
    token = os.environ.get("TENCENTCLOUD_SESSIONTOKEN")
    config = CosConfig(Region=region, SecretId=secret_id, SecretKey=secret_key, Token=token)
    client = CosS3Client(config)

    response = client.get_presigned_url(
        Method='PUT',
        Bucket=os.environ.get('bucket_name'),
        Key=body['key'],
        Expired=30,
    )
    return {"url": response.split("?sign=")[0],
            "sign": urllib.parse.unquote(response.split("?sign=")[1]),
            "token": os.environ.get("TENCENTCLOUD_SESSIONTOKEN")}

```

HTML页面基本实现：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/5-2-3.png)

HTML部分：

```html
<div style="width: 70%">
        <div style="text-align: center">
            <h3>Web端上传文件</h3>
        </div>
        <hr>
        <div>
            <p>
                方案1：通过上传到SCF，进行处理再转存到COS，这种方法比较直观，但是问题是SCF从APIGW处只能接收到小于6M的数据，而且对二进制文件处理并不好。
            </p>
            <input type="file" name="file" id="fileScf"/>
            <input type="button" onclick="UpladFileSCF()" value="上传"/>
        </div>
        <hr>
        <div>
            <p>
                方案2：
                直接上传到COS，流程是先从SCF获得临时地址，进行数据存储（例如将文件信息存到redis等），然后再从客户端进行上传COS，上传结束可通过COS触发器触发函数，从存储系统（例如已经存储到redis）读取到更对信息，在对图像进行处理。
            </p>
            <input type="file" name="file" id="fileCos"/>
            <input type="button" onclick="UpladFileCOS()" value="上传"/>
        </div>
    </div>
```

方案1上传部分JS：

```javascript
function UpladFileSCF() {
    var oFReader = new FileReader();
    oFReader.readAsDataURL(document.getElementById("fileScf").files[0]);
    oFReader.onload = function (oFREvent) {
        const key = Math.random().toString(36).substr(2);
        var xmlhttp = window.XMLHttpRequest ? (new XMLHttpRequest()) : (new ActiveXObject("Microsoft.XMLHTTP"))
        xmlhttp.onreadystatechange = function () {
            if (xmlhttp.readyState == 4 && xmlhttp.status == 200) {
                if (JSON.parse(xmlhttp.responseText)['uploaded'] == 1) {
                    alert("上传成功")
                }
            }
        }
        var url = " https://service-f1zk07f3-1256773370.bj.apigw.tencentcs.com/release/upload/cos"
        xmlhttp.open("POST", url, true);
        xmlhttp.setRequestHeader("Content-type", "application/json");
        var postData = {
            picture: oFREvent.target.result,
            token: 'mytoken',
            key: key,
        }
        xmlhttp.send(JSON.stringify(postData));
    }

}
```

方案2上传部分JS：

```javascript
function doUpload(key, bodyUrl, bodySign, bodyToken) {
    var fileObj = document.getElementById("fileCos").files[0];
    xmlhttp = window.XMLHttpRequest ? (new XMLHttpRequest()) : (new ActiveXObject("Microsoft.XMLHTTP"));
    xmlhttp.open("PUT", bodyUrl, true);
    xmlhttp.onload = function () {
        console.log(xmlhttp.responseText)
        if (!xmlhttp.responseText) {
            alert("上传成功")
        }
    };
    xmlhttp.setRequestHeader("Authorization", bodySign);
    xmlhttp.setRequestHeader("x-cos-security-token", bodyToken);
    xmlhttp.send(fileObj);
}

function UpladFileCOS() {
    const key = Math.random().toString(36).substr(2);

    var xmlhttp = window.XMLHttpRequest ? (new XMLHttpRequest()) : (new ActiveXObject("Microsoft.XMLHTTP"))
    xmlhttp.onreadystatechange = function () {
        if (xmlhttp.readyState == 4 && xmlhttp.status == 200) {
            var body = JSON.parse(xmlhttp.responseText)
            if (body['url']) {
                doUpload(key, body['url'], body['sign'], body['token'])
            }
        }
    }
    var getUploadUrl = 'https://service-f1zk07f3-1256773370.bj.apigw.tencentcs.com/release/upload/presigned'
    xmlhttp.open("POST", getUploadUrl, true);
    xmlhttp.setRequestHeader("Content-type", "application/json");
    xmlhttp.send(JSON.stringify({
        token: 'mytoken',
        key: key,
    }));
}
```

这里面可以看到获取用户密钥信息的方法是os.environ.get("TENCENTCLOUD_SECRETID")，想要通过这种方法获取密钥信息，需要给予函数相关的角色和对角色进行相关的权限，以Serverless Framework为例，可以使用tencent-cam-role，例如创建一个全局组件：

```yaml
Conf:
  component: "serverless-global"
  inputs:
    region: ap-beijing
    runtime: Python3.6
    role: SCF_UploadToCOSRole
    bucket_name: scf-upload-1256773370
```

然后创建一个增加Role的组件：

```yaml
UploadToCOSRole:
  component: "@gosls/tencent-cam-role"
  inputs:
    roleName: ${Conf.role}
    service:
      - scf.qcloud.com
    policy:
      policyName:
        - QcloudCOSFullAccess
```

接下来就是函数的创建，函数创建时需要绑定刚才的这个role:

```yaml
getUploadPresignedUrl:
  component: "@gosls/tencent-scf"
  inputs:
    name: Upload_getUploadPresignedUrl
    role: ${Conf.role}
    codeUri: ./fileUploadToCos
    handler: index.getPresignedUrl
    runtime: ${Conf.runtime}
    region: ${Conf.region}
    description: 获取cos临时上传地址
    memorySize: 64
    timeout: 3
    environment:
      variables:
        region: ${Conf.region}
        bucket_name: ${Conf.bucket_name}
```

同时将这个函数绑定APIGW：

```yaml
UploadService:
  component: "@gosls/tencent-apigateway"
  inputs:
    region: ${Conf.region}
    protocols:
      - http
      - https
    serviceName: UploadAPI
    environment: release
    endpoints:
      - path: /upload/cos
        description: 通过SCF上传cos
        method: POST
        enableCORS: TRUE
        function:
          functionName: Upload_uploadToSCFToCOS
      - path: /upload/presigned
        description: 获取临时地址
        method: POST
        enableCORS: TRUE
        function:
          functionName: Upload_getUploadPresignedUrl
```

另外，本例子还需要一个COS存储桶来作为测试使用，由于Web服务可能存在跨域问题，所以需要对COS进行跨域设置：

```yaml
SCFUploadBucket:
  component: '@gosls/tencent-cos'
  inputs:
    bucket: ${Conf.bucket_name}
    region: ${Conf.region}
    cors:
      - id: abc
        maxAgeSeconds: '10'
        allowedMethods:
          - POST
          - PUT
        allowedOrigins:
          - '*'
        allowedHeaders:
          - '*'
```

完成之后，可以快速部署：

```text
(venv) DFOUNDERLIU-MB0:test dfounderliu$ sls --debug

  DEBUG ─ Resolving the template's static variables.
  DEBUG ─ Collecting components from the template.
  DEBUG ─ Downloading any NPM components found in the template.
  ... ...
    apis: 
      - 
        path:   /upload/cos
        method: POST
        apiId:  api-0lkhke0c
      - 
        path:   /upload/presigned
        method: POST
        apiId:  api-b7j5ikoc

  15s › uploadToSCFToCOS › done
```

至此，我们完成了项目部署，可以进行测试与适用。

## 总结

Serverless可以看作是一个新的技术，一个新的架构。我们在接触新鲜事物的时候，或多或少都要有一个适应期，我认为如何在Serverless架构下上传文件，就是一个需要适应的部分。习惯了直接将文件上传到服务器的行为，在接触Serverless架构之后，由于网关->函数对二进制支持和数据包大小问题，由于前端为了安全不方便直接放放密钥信息等问题我们就要讲某些事情稍微复杂化。当然，Serverless架构下上传文件的方法很多，不同厂商解决方案可能也不一样，我这里更多也是抛砖引玉，希望更多大佬分享经验，让我们一起Serverless。