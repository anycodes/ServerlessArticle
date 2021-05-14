---
title: 基于Serverless架构的Git代码统计工具
date: 2020.5.20
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework, Git, 统计]
categories: [Serverless, 腾讯云, 领域实战]
---

# 

## 前言

自己毕业也有一年多了，很想统计一下过去一年自己贡献了多少的代码。想了一下，可能要用`git log`，简单的做了一下，感觉不是很爽，正直自己想通过Serverless做一个工具合集，就想能不能通过Serverless做一个Git的代码统计功能？

## 代码实现

其实整个逻辑是比较简单的，只需要将`git`整合到云函数中，再加一个API网关作为触发器，就可以轻松实现。整体代码为：

```python
import random
import subprocess
import os
import json
import uuid
import tarfile

randomStr = lambda num: "".join(random.sample('zyxwvutsrqponmlkjihgfedcba',num))

def return_msg(error, msg):
    return_data = {
        "uuid": str(uuid.uuid1()),
        "error": error,
        "message": msg
    }
    print(return_data)
    return return_data


def installGit():
    targetDir = "/tmp/git"
    dir_path = os.path.dirname(os.path.realpath(__file__))
    tar = tarfile.open(os.path.join(dir_path, 'git-2.14.0.tar'))
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

def doPopen(gitStr, path):
    child = subprocess.Popen("cd %s && %s" % (path, gitStr), shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    return child.stdout.read().decode("utf-8")


def main_handler(event, context):
    try:
        path = "/tmp/%s" % (randomStr(5))
        print("git clone %s %s" % (json.loads(event["body"])["url"], path))
        child = subprocess.Popen("git clone %s %s" % (json.loads(event["body"])["url"], path), shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        print(child.stdout.read().decode("utf-8"))
        users = {}
        for eveCommit in doPopen("git log --format='%aN'", path).split("\n"):
            if eveCommit:
                if eveCommit not in users:
                    users[eveCommit] = {"commits": 1,
                                        "added_lines": 0,
                                        "removed_lines": 0,
                                        "total_lines": 0}
                    for eveItem in doPopen('git log --author="%s" --pretty=tformat: --numstat'%eveCommit, path).split("\n"):
                        if eveItem:
                            eveItemList = eveItem.split("\t")
                            users[eveCommit]["added_lines"] = users[eveCommit]["added_lines"] + int(eveItemList[0])
                            users[eveCommit]["removed_lines"] = users[eveCommit]["removed_lines"] + int(eveItemList[1])
                            users[eveCommit]["total_lines"] = users[eveCommit]["added_lines"] - users[eveCommit]["removed_lines"]

                else:
                    users[eveCommit]['commits'] = users[eveCommit]['commits'] + 1
        return return_msg(False, users)
    except Exception as e:
        return return_msg(True, e)
```

在这段代码中，主要就是将Git的压缩包，解压到`/tmp/`目录，然后将其装载到环境变量中，确保git指令可用。

接下来通过`git clone`指令来进行用户提供的仓库的clone，将其同样存放在`/tmp/`目录下。

然后通过`git log --format='%aN'`获得该仓库所有提交过过的人的列表，并且可以统计出每个人提交的次数。

最后通过`git log --author="user" --pretty=tformat: --numstat`来统计user的代码行数，最后进行组装并且返回。

Serverless的Yaml文件为：

```yaml
GitStatistics:
  component: "@serverless/tencent-scf"
  inputs:
    name: GitStatistics
    codeUri: ./
    exclude:
      - .gitignore
      - .git/**
      - .serverless
      - .env
    handler: index.main_handler
    runtime: Python3.6
    region: ap-beijing
    description: git提交次数统计
    namespace: serverless_tools
    memorySize: 64
    timeout: 1800
    events:
      - apigw:
          name: serverless
          parameters:
            serviceId: service-8d3fi753
            environment: release
            endpoints:
              - path: /git/statistics
                serviceTimeout: 1800
                description: git提交次数统计
                method: POST
                enableCORS: true
```

代码的接口信息：

请求地址： http://service-8d3fi753-1256773370.bj.apigw.tencentcs.com/release/git/statistics   
请求类型：POST   
入参类型：JSON   
入参参数：url: git地址，例如：{"url": "https://github.com/serverless-components/tencent-scf.git"}   
出参列表：
```json
{
    "uuid": "6ff62096-819d-11ea-a6a2-0242cb007105",
    "error": false,
    "message": {
        "yugasun": {
            "commits": 23,
            "added_lines": 435,
            "removed_lines": 137,
            "total_lines": 298
        }
    }
}
```
       

完整之后，我们可以通过POSTMAN测试一下：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-3-1.png)

## 总结

Serverless架构有很多有趣的应用，通过将Git集成到函数计算中，不仅仅可以更加简单快速，方便的集成CICD流程，也可以额实现代码行数统计，还可以做很多有趣的事情，希望通过我的抛砖引玉，可以让大家有更多的想法，创新性的提出更多的应用。

## 体验

* [体验地址](https://serverlesschina.com/32.html)

## 附件

* [Git完整功能的Tar](https://others-1256773370.cos.ap-chengdu.myqcloud.com/resource/git-2.14.0.tar)