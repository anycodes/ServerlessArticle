---
title: serverless-git和serverless-cicd
date: 2020.5.14
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework, git, cicd]
categories: [Serverless, 腾讯云, 领域实战]
---

## 前言

传统情况下，我们写完代码，可能面对两个事情：发布到代码仓库以及部署到线上，传统的我们会手动实现这些操作，出现误操作的概率也是蛮高的，相对来说也是比较机械化的工作。CICD的引入，大大改善了持续继承、交付以及部署的糟糕状况。

CICD实际上是持续集成（Continuous Integration）、持续交付（Continuous Delivery） 、持续部署（Continuous Deployment），在这个流程中持续集成的重点是将各个开发人员的工作集合到一个代码仓库中。通常，每天都要进行几次，主要目的是尽早发现集成错误，使团队更加紧密结合，更好地协作。持续交付的目的是最小化部署或释放过程中固有的摩擦。它的实现通常能够将构建部署的每个步骤自动化，以便任何时刻能够安全地完成代码发布（理想情况下）。持续部署是一种更高程度的自动化，无论何时对代码进行重大更改，都会自动进行构建/部署。

当然，通过上面描述，不难看出，这一切都建立与一个关键词之上："自动化"，诚然目前已经有了很多CICD的组件/平台、或者方法技术，但是基于Serverless架构的CICD貌似还不是很多。本文将会通过将git指令移植到Serverless架构的函数计算上，通过函数、API网关实现不同唯独的简单的"CICD"流程。

## Serverless-Git

如果想赋予Serverless架构CICD的能力，那么从代码仓库下载代码这个操作就是必不可少的，那么传统的主机中，或者已有的CICD平台中，GIT指令都是很容易安装或者本身默认集成的，但是Serverless架构下，各个云厂商的函数计算服务中，GIT功能并不存在，那么就需要我们通过自己封装，将其实现。

此时，我们可以通过下载Git源代码，进行编译。整个过程还是比较简单的：

* 在`https://github.com/git/git/releases`中，找到对应的版本，并且下载；
* 准备一台CentOS的机器，因为腾讯云云函数的容器环境，是CentOS的；
* 下载源码包：`wget https://github.com/git/git/archive/v2.14.0.tar.gz`
* 解压源码包：`tar -zxvf v2.14.0.tar.gz`
* 安装其他可能需要的依赖：`yum install -y curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker`
* 进入源码包进行`make`: `make prefix=/root/testgit/git all && make prefix=/root/testgit/git install`（我这里是将结果放入到`/root/testgit/git`中，此处可以根据需求自定义）
* 完成之后，我们可以将编译好的下载下来，是这三个文件夹：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-1.png)

不要小瞧这三个文件夹，这三个文件夹还是蛮大的：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-2.png)

在我们云函数中，`/tmp/`目录大小是500M，这个东西就400M，我们还怎么愉快的玩耍？所以将我们不常用的功能精简掉是蛮有必要的。我根据自己的自身需求，我认为我目前只需要少数的GIT指令，例如`git clone`等就够了，所以在目录`libexec/git-core`中，我删除其他的暂时不用的文件，只保留自己目前需要的：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-3.png)

整理完成之后，整个包的大小是（左边是完整版，右边是精简版）：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-4.png)

接下来，我们将对Serverless-git进行测试：

```python
import subprocess
import os
import tarfile


def installGit():
    targetDir = "/tmp/git"
    dir_path = os.path.dirname(os.path.realpath(__file__))
    tar = tarfile.open(os.path.join(dir_path, 'git.tar'))
    tar.extractall(targetDir)
    git_path = os.path.join(targetDir, 'git')
    bin_path = os.path.join(git_path, 'bin')

    template_dir = os.path.join(
        git_path,
        'share',
        'git-core',
        'templates'
    )

    exec_path = os.path.join(
        git_path,
        'libexec',
        'git-core'
    )
    os.environ['PATH'] = bin_path + ':' + os.environ['PATH']
    os.environ['GIT_TEMPLATE_DIR'] = template_dir
    os.environ['GIT_EXEC_PATH'] = exec_path


installGit()


def main_handler(event, context):

    # git 拉代码
    child = subprocess.run('git clone https://github.com/anycodes/ServerlessBlog.git /tmp/data',
                   stdout=subprocess.PIPE,
                   stderr=subprocess.PIPE,
                   close_fds=True,
                   shell=True)
                   
    print(child.stderr) 
    print(child.stdout) 
    print(os.listdir('/tmp/data'))
```

同时，我们还要将刚才处理好的Git打包放过来：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-5.png)

可以通过`tar -cvf 目标tar 原有路径`进行打包，打包完成和代码放在一起，一起上传到函数计算中：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-6.png)

通过对函数触发可以看到：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-7.png)

我们可以看到，我们已经成功的实现了Serverless-Git组件。

## Serverless-CICD

完成了Serverless-Git组件，就要开始进军Serverless-CICD功能了。这里将会分成几个部分，来进行示例操作。

### tencent-website组件的融入

如果我有一个个人的静态网站，我每次都需要上传到Git，还要部署到COS中。确实有点不舒服，那么通过对一个静态网站的自动化部署，来窥探一下Serverless-CICD是否可行。

首先，将我们的Serverless-CICD组件和网站放在一起：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-8.png)

内容都是比较简单的：

index.py: 

```python
import subprocess
import os
import tarfile
import random
from qcloud_cos_v5 import CosConfig
from qcloud_cos_v5 import CosS3Client


def installGit():
    targetDir = "/tmp/git"
    dir_path = os.path.dirname(os.path.realpath(__file__))
    tar = tarfile.open(os.path.join(dir_path, 'git.tar'))
    tar.extractall(targetDir)
    git_path = os.path.join(targetDir, 'git')
    bin_path = os.path.join(git_path, 'bin')

    template_dir = os.path.join(
        git_path,
        'share',
        'git-core',
        'templates'
    )

    exec_path = os.path.join(
        git_path,
        'libexec',
        'git-core'
    )
    os.environ['PATH'] = bin_path + ':' + os.environ['PATH']
    os.environ['GIT_TEMPLATE_DIR'] = template_dir
    os.environ['GIT_EXEC_PATH'] = exec_path


installGit()


def main_handler(event, context):

    targetDir = "/tmp/" + "".join(random.sample('zyxwvutsrqponmlkjihgfedcba',5))

    # git 拉代码
    subprocess.run('git clone %s %s' % (os.environ.get("git"), targetDir),
                   stdout=subprocess.PIPE,
                   stderr=subprocess.PIPE,
                   close_fds=True,
                   shell=True)

    # 部署到cos
    client = CosS3Client(CosConfig(Region=os.environ.get("region"),
                                   SecretId=os.environ.get("secret_id"),
                                   SecretKey=os.environ.get("secret_key")))
    for eveDir in os.walk(targetDir):
        if eveDir[2]:
            for eveFile in eveDir[2]:
                pathData = os.path.join(eveDir[0], eveFile)
                client.upload_file(
                    Bucket=os.environ.get("bucket_name"),
                    LocalFilePath=pathData,
                    Key=pathData.replace(targetDir, ""))

```

整个逻辑基本是，函数被触发，从GIT拉代码，并上传到COS中。由于存在容器复用的问题，所以这里写入到`/tmp/`目录的临时克隆地址是随机的。

接下来就是我们的网页部分，非常简单：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hello Serverless</title>
</head>
<body>
Hello Serverless Framework!
<br>
Test 1
</body>
</html>
```

最后就是我们的Yaml部分：

```yaml
Conf:
  component: "serverless-global"
  inputs:
    git: 'https://github.com/anycodes/ServerlessWebsiteCDDemo.git'
    secret_id: 密钥信息
    secret_key: 密钥信息
    region: ap-guangzhou
    bucket_name: my-website-cd-demo-1256773370

ServerlessWebsiteCD:
  component: "@serverless/tencent-scf"
  inputs:
    name: ServerlessWebsiteCD
    codeUri: ./website-cd
    handler: index.main_handler
    runtime: Python3.6
    region: ap-guangzhou
    description: Serverless Website CD Demo
    memorySize: 128
    timeout: 300
    environment:
      variables:
        region: ${Conf.region}
        secret_id: ${Conf.secret_id}
        secret_key: ${Conf.secret_key}
        bucket_name: ${Conf.bucket_name}
        git: ${Conf.git}
    tags:
      Website: CD-Demo
    events:
      - apigw:
          name: Serverless_Website_Demo
          parameters:
            protocols:
              - http
            description: Serverless Website Demo
            environment: release
            endpoints:
              - path: /website
                method: POST


ServerlessWebsite:
  component: '@serverless/tencent-website'
  inputs:
    code:
      src: ./website
      index: index.html
    region: ${Conf.region}
    bucketName: ${Conf.bucket_name}
    protocol: http
```

这里面还涉及到我们即将存放代码的仓库：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-9.png)

完成之后，我们通过serverless命令行工具进行部署：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-10.png)

完成部署之后，我们将API网关触发器的地址，复制下来，配置到GitHub的WebHook中：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-11.png)

完成之后，我们打开刚才部署完成的网页：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-12.png)

此时，我们对HTML页面进行修改，并且commit以及push到代码仓库中：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-13.png)

完成之后，我们刷新页面：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-2-14.png)

可以看到，我们的网页同步更改了。当然，这是一个非常简单的过程，也仅仅是抛砖引玉。

## 总结

正如大佬们所说，不要反复造轮子，其实就目前而言，CICD的工具/平台已经很多了，GITHUB也有Action可以快速将自己的项目接入，但是我仅仅觉得Serverless架构真的可以进行很多有趣的事情，例如将Git集成到Serverless架构中，实现持续部署、发布等。

我上面的文章也仅仅是抛砖引玉，真正的项目开发其实涉及到自动化的部分肯能更加复杂，例如：

* 上面不仅仅是clone仓库到函数中，还要clone之后进行一些编译，构建操作；
* 简单的编译构建无法满足需求，可能要进行一些深度学习，大数据分析等；
* 简单的clone无法满足需求，可能还需要将部分内容回推到Git上

## 附件

* [Git只有CLone功能的Tar](https://others-1256773370.cos.ap-chengdu.myqcloud.com/resource/gitgit-2.14.0-clone.tar)
* [Git完整功能的Tar](https://others-1256773370.cos.ap-chengdu.myqcloud.com/resource/git-2.14.0.tar)