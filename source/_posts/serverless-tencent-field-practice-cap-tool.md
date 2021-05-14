---
title: 基于Serverless的验证码识别API
date: 2020.5.1
tags: [Serverless, 腾讯云, 基本介绍, Serverless Framework, 验证码识别, 人工智能]
categories: [Serverless, 腾讯云, 领域实战]
---

## 前言

之前和大家分享了很多的CV相关的例子，被很多小伙伴吐槽说我是调包侠，还连累了Serverless被很多人误以为也仅仅能"调包玩一玩"，其实在Serverless中，开发者的自由度还是非常大的，除了调包快速实现一些东西，我们也可以通过一些代码训练一些模型，然后实现一些功能，本文将会通过简单的实验，在Serverless架构上实现一个基于卷积神经网络（CNN）算法的在线验证码识别的小工具。

## 验证码与识别

验证码（CAPTCHA）是“Completely Automated Public Turing test to tell Computers and Humans Apart”（全自动区分计算机和人类的图灵测试）的缩写，是一种区分用户是计算机还是人的公共全自动程序。可以防止：恶意破解密码、刷票、论坛灌水，有效防止某个黑客对某一个特定注册用户用特定程序暴力破解方式进行不断的登陆尝试，实际上用验证码是现在很多网站通行的方式，我们利用比较简易的方式实现了这个功能。这个问题可以由计算机生成并评判，但是必须只有人类才能解答。由于计算机无法解答CAPTCHA的问题，所以回答出问题的用户就可以被认为是人类。

说白了，验证码就是用来验证的码，验证是人访问的还是机器访问的码。

验证码的发展，可以说是非常迅速的，从开始的单纯数字验证码，到后来的数字+字母验证码，再到后来的数字+字母+中文的验证码以及图形图像验证码，可以说就单纯的验证码素材已经越来越多了，从验证码的形态来看，也是各不相同，输入、点击、拖拽以及短信验证码、语音验证码......

例如腾讯云后台登陆的验证码与Bilibili的登录验证码就是滑动登录:

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-1.png)

而百度贴吧、知乎、以及Google等相关网站的验证码又各不相同，例如选择正着写的文字，选择包括指定物体的图片以及按顺序点击图片中的字符等。

验证码的识别可能会根据验证码的类型而不太一致，当然最简单的验证码可能就是最原始的文字验证码了：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-2.png)

即便是文字验证码，也是存在很多差异的，例如简单的数字验证码，简单的数字+字母验证码，文字验证码，验证码中包括计算，简单验证码中增加一些干扰成为复杂验证码.......

就这种比较简单的验证码的识别方法也有很多，除了目前比流行的端到端识别之外，之前比较常见的识别就是通过图像的切割，对验证码每一部分裁剪，然后再对每个裁剪单元进行相似度对比，获得最可能的结果，最后进行拼接，例如将验证码：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-3.png)

进行二值化等操作：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-4.png)

完成之后再进行切割：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-5.png)

切割完成在进行识别，再进行拼接，这样的做法是，针对每个字符进行识别，相对来说是比较容易容易的。但是对于某些情况，是没办法切割的，例如图片中有很多干扰线等。这个时候就可能需要深度学习，来进行端对端的识别了。

## 代码实现

本代码很多内容来源于Github，更多是通过搜集一些资料，发挥自己的想象，将该项目部署到Serverless架构上。

### 验证码生成部分

```python
# coding:utf-8
# name:captcha_gen.py

import random
import numpy as np
from PIL import Image
from captcha.image import ImageCaptcha


NUMBER = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
LOW_CASE = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u',
            'v', 'w', 'x', 'y', 'z']
UP_CASE = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U',
           'V', 'W', 'X', 'Y', 'Z']

CAPTCHA_LIST = NUMBER
CAPTCHA_LEN = 4         # 验证码长度
CAPTCHA_HEIGHT = 60     # 验证码高度
CAPTCHA_WIDTH = 160     # 验证码宽度


def random_captcha_text(char_set=CAPTCHA_LIST, captcha_size=CAPTCHA_LEN):
    """
    随机生成定长字符串
    :param char_set: 备选字符串列表
    :param captcha_size: 字符串长度
    :return: 字符串
    """
    captcha_text = [random.choice(char_set) for _ in range(captcha_size)]
    return ''.join(captcha_text)


def gen_captcha_text_and_image(width=CAPTCHA_WIDTH, height=CAPTCHA_HEIGHT, save=None):
    """
    生成随机验证码
    :param width: 验证码图片宽度
    :param height: 验证码图片高度
    :param save: 是否保存（None）
    :return: 验证码字符串，验证码图像np数组
    """
    image = ImageCaptcha(width=width, height=height)
    # 验证码文本
    captcha_text = random_captcha_text()
    captcha = image.generate(captcha_text)
    # 保存
    if save:
        image.write(captcha_text, './img/' + captcha_text + '.jpg')
    captcha_image = Image.open(captcha)
    # 转化为np数组
    captcha_image = np.array(captcha_image)
    return captcha_text, captcha_image


if __name__ == '__main__':
    t, im = gen_captcha_text_and_image(save=True)
    print(t, im.shape)      # (60, 160, 3)

```

这一部分主要用户生成验证码，目前`CAPTCHA_LIST = NUMBER`，表示只用数字验证码，如果需要英文大小写，可将`LOW_CASE`和`UP_CASE`加到`CAPTCHA_LIST`中。

### 组件

```python
# -*- coding:utf-8 -*-
# name: util.py

import numpy as np
from captcha_gen import gen_captcha_text_and_image
from captcha_gen import CAPTCHA_LIST, CAPTCHA_LEN, CAPTCHA_HEIGHT, CAPTCHA_WIDTH


def convert2gray(img):
    """
    图片转为黑白，3维转1维
    :param img: np
    :return:  灰度图的np
    """
    if len(img.shape) > 2:
        img = np.mean(img, -1)
    return img


def text2vec(text, captcha_len=CAPTCHA_LEN, captcha_list=CAPTCHA_LIST):
    """
    验证码文本转为向量
    :param text:
    :param captcha_len:
    :param captcha_list:
    :return: vector 文本对应的向量形式
    """
    text_len = len(text)    # 欲生成验证码的字符长度
    if text_len > captcha_len:
        raise ValueError('验证码最长4个字符')
    vector = np.zeros(captcha_len * len(captcha_list))      # 生成一个一维向量 验证码长度*字符列表长度
    for i in range(text_len):
        vector[captcha_list.index(text[i])+i*len(captcha_list)] = 1     # 找到字符对应在字符列表中的下标值+字符列表长度*i 的 一维向量 赋值为 1
    return vector


def vec2text(vec, captcha_list=CAPTCHA_LIST, captcha_len=CAPTCHA_LEN):
    """
    验证码向量转为文本
    :param vec:
    :param captcha_list:
    :param captcha_len:
    :return: 向量的字符串形式
    """
    vec_idx = vec
    text_list = [captcha_list[int(v)] for v in vec_idx]
    return ''.join(text_list)


def wrap_gen_captcha_text_and_image(shape=(60, 160, 3)):
    """
    返回特定shape图片
    :param shape:
    :return:
    """
    while True:
        t, im = gen_captcha_text_and_image()
        if im.shape == shape:
            return t, im


def get_next_batch(batch_count=60, width=CAPTCHA_WIDTH, height=CAPTCHA_HEIGHT):
    """
    获取训练图片组
    :param batch_count: default 60
    :param width: 验证码宽度
    :param height: 验证码高度
    :return: batch_x, batch_yc
    """
    batch_x = np.zeros([batch_count, width * height])
    batch_y = np.zeros([batch_count, CAPTCHA_LEN * len(CAPTCHA_LIST)])
    for i in range(batch_count):    # 生成对应的训练集
        text, image = wrap_gen_captcha_text_and_image()
        image = convert2gray(image)     # 转灰度numpy
        # 将图片数组一维化 同时将文本也对应在两个二维组的同一行
        batch_x[i, :] = image.flatten() / 255
        batch_y[i, :] = text2vec(text)  # 验证码文本的向量形式
    # 返回该训练批次
    return batch_x, batch_y


if __name__ == '__main__':
    x, y = get_next_batch(batch_count=1)    # 默认为1用于测试集
    print(x, y)

```

这一部分主要是进行一些组件的编写，在未来的训练和测试过程中会有所应用。

### 训练模型

```python
# -*- coding:utf-8 -*-
# name: model_train.py

import tensorflow.compat.v1 as tf
from datetime import datetime
from util import get_next_batch
from captcha_gen import CAPTCHA_HEIGHT, CAPTCHA_WIDTH, CAPTCHA_LEN, CAPTCHA_LIST

tf.compat.v1.disable_eager_execution()

def weight_variable(shape, w_alpha=0.01):
    """
    初始化权值
    :param shape:
    :param w_alpha:
    :return:
   """
    initial = w_alpha * tf.random_normal(shape)
    return tf.Variable(initial)


def bias_variable(shape, b_alpha=0.1):
    """
    初始化偏置项
    :param shape:
    :param b_alpha:
    :return:
    """
    initial = b_alpha * tf.random_normal(shape)
    return tf.Variable(initial)


def conv2d(x, w):
    """
    卷基层 ：局部变量线性组合，步长为1，模式‘SAME’代表卷积后图片尺寸不变，即零边距
    :param x:
    :param w:
    :return:
    """
    return tf.nn.conv2d(x, w, strides=[1, 1, 1, 1], padding='SAME')


def max_pool_2x2(x):
    """
    池化层：max pooling,取出区域内最大值为代表特征， 2x2 的pool，图片尺寸变为1/2
    :param x:
    :return:
    """
    return tf.nn.max_pool(x, ksize=[1, 2, 2, 1], strides=[1, 2, 2, 1], padding='SAME')


def cnn_graph(x, keep_prob, size, captcha_list=CAPTCHA_LIST, captcha_len=CAPTCHA_LEN):
    """
    三层卷积神经网络
    :param x:   训练集 image x
    :param keep_prob:   神经元利用率
    :param size:        大小 (高,宽)
    :param captcha_list:
    :param captcha_len:
    :return: y_conv
    """
    # 需要将图片reshape为4维向量
    image_height, image_width = size
    x_image = tf.reshape(x, shape=[-1, image_height, image_width, 1])

    # 第一层
    # filter定义为3x3x1， 输出32个特征, 即32个filter
    w_conv1 = weight_variable([3, 3, 1, 32])    # 3*3的采样窗口，32个（通道）卷积核从1个平面抽取特征得到32个特征平面
    b_conv1 = bias_variable([32])
    h_conv1 = tf.nn.relu(conv2d(x_image, w_conv1) + b_conv1)    # rulu激活函数
    h_pool1 = max_pool_2x2(h_conv1)     # 池化
    h_drop1 = tf.nn.dropout(h_pool1, keep_prob)      # dropout防止过拟合

    # 第二层
    w_conv2 = weight_variable([3, 3, 32, 64])
    b_conv2 = bias_variable([64])
    h_conv2 = tf.nn.relu(conv2d(h_drop1, w_conv2) + b_conv2)
    h_pool2 = max_pool_2x2(h_conv2)
    h_drop2 = tf.nn.dropout(h_pool2, keep_prob)

    # 第三层
    w_conv3 = weight_variable([3, 3, 64, 64])
    b_conv3 = bias_variable([64])
    h_conv3 = tf.nn.relu(conv2d(h_drop2, w_conv3) + b_conv3)
    h_pool3 = max_pool_2x2(h_conv3)
    h_drop3 = tf.nn.dropout(h_pool3, keep_prob)

    """
    原始：60*160图片 第一次卷积后 60*160 第一池化后 30*80
    第二次卷积后 30*80 ，第二次池化后 15*40
    第三次卷积后 15*40 ，第三次池化后 7.5*20 = > 向下取整 7*20
    经过上面操作后得到7*20的平面
    """

    # 全连接层
    image_height = int(h_drop3.shape[1])
    image_width = int(h_drop3.shape[2])
    w_fc = weight_variable([image_height*image_width*64, 1024])     # 上一层有64个神经元 全连接层有1024个神经元
    b_fc = bias_variable([1024])
    h_drop3_re = tf.reshape(h_drop3, [-1, image_height*image_width*64])
    h_fc = tf.nn.relu(tf.matmul(h_drop3_re, w_fc) + b_fc)
    h_drop_fc = tf.nn.dropout(h_fc, keep_prob)

    # 输出层
    w_out = weight_variable([1024, len(captcha_list)*captcha_len])
    b_out = bias_variable([len(captcha_list)*captcha_len])
    y_conv = tf.matmul(h_drop_fc, w_out) + b_out
    return y_conv


def optimize_graph(y, y_conv):
    """
    优化计算图
    :param y: 正确值
    :param y_conv:  预测值
    :return: optimizer
    """
    # 交叉熵代价函数计算loss 注意logits输入是在函数内部进行sigmod操作
    # sigmod_cross适用于每个类别相互独立但不互斥，如图中可以有字母和数字
    # softmax_cross适用于每个类别独立且排斥的情况，如数字和字母不可以同时出现
    loss = tf.reduce_mean(tf.nn.sigmoid_cross_entropy_with_logits(labels=y, logits=y_conv))
    # 最小化loss优化 AdaminOptimizer优化
    optimizer = tf.train.AdamOptimizer(1e-3).minimize(loss)
    return optimizer


def accuracy_graph(y, y_conv, width=len(CAPTCHA_LIST), height=CAPTCHA_LEN):
    """
    偏差计算图，正确值和预测值，计算准确度
    :param y: 正确值 标签
    :param y_conv:  预测值
    :param width:   验证码预备字符列表长度
    :param height:  验证码的大小，默认为4
    :return:    正确率
    """
    # 这里区分了大小写 实际上验证码一般不区分大小写,有四个值，不同于手写体识别
    # 预测值
    predict = tf.reshape(y_conv, [-1, height, width])   #
    max_predict_idx = tf.argmax(predict, 2)
    # 标签
    label = tf.reshape(y, [-1, height, width])
    max_label_idx = tf.argmax(label, 2)
    correct_p = tf.equal(max_predict_idx, max_label_idx)    # 判断是否相等
    accuracy = tf.reduce_mean(tf.cast(correct_p, tf.float32))
    return accuracy


def train(height=CAPTCHA_HEIGHT, width=CAPTCHA_WIDTH, y_size=len(CAPTCHA_LIST)*CAPTCHA_LEN):
    """
    cnn训练
    :param height: 验证码高度
    :param width:   验证码宽度
    :param y_size:  验证码预备字符列表长度*验证码长度（默认为4）
    :return:
    """
    # cnn在图像大小是2的倍数时性能最高, 如果图像大小不是2的倍数，可以在图像边缘补无用像素
    # 在图像上补2行，下补3行，左补2行，右补2行
    # np.pad(image,((2,3),(2,2)), 'constant', constant_values=(255,))

    acc_rate = 0.95     # 预设模型准确率标准
    # 按照图片大小申请占位符
    x = tf.placeholder(tf.float32, [None, height * width])
    y = tf.placeholder(tf.float32, [None, y_size])
    # 防止过拟合 训练时启用 测试时不启用 神经元使用率
    keep_prob = tf.placeholder(tf.float32)
    # cnn模型
    y_conv = cnn_graph(x, keep_prob, (height, width))
    # 优化
    optimizer = optimize_graph(y, y_conv)
    # 计算准确率
    accuracy = accuracy_graph(y, y_conv)
    # 启动会话.开始训练
    saver = tf.train.Saver()
    sess = tf.Session()
    sess.run(tf.global_variables_initializer())     # 初始化
    step = 0    # 步数
    while 1:
        batch_x, batch_y = get_next_batch(64)
        sess.run(optimizer, feed_dict={x: batch_x, y: batch_y, keep_prob: 0.75})
        # 每训练一百次测试一次
        if step % 100 == 0:
            batch_x_test, batch_y_test = get_next_batch(100)
            acc = sess.run(accuracy, feed_dict={x: batch_x_test, y: batch_y_test, keep_prob: 1.0})
            print(datetime.now().strftime('%c'), ' step:', step, ' accuracy:', acc)
            # 准确率满足要求，保存模型
            if acc > acc_rate:
                model_path = "./model/captcha.model"
                saver.save(sess, model_path, global_step=step)
                acc_rate += 0.01
                if acc_rate > 0.99:     # 准确率达到99%则退出
                    break
        step += 1
    sess.close()


if __name__ == '__main__':
    train()

```

这里需要额外注意，此处有两部分代码分别为：`import tensorflow.compat.v1 as tf`和`tf.compat.v1.disable_eager_execution()`，这里要吐槽一下tensorflow，他在后期的一些升级逐渐和老版本不兼容了，所以现在安装的新版本：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-6.png)

都需要增加这部分，但是在腾讯云的云函数中，自带了1.*的tensorflow版本，所以本地测试完成，部署到线上，将`import tensorflow.compat.v1 as tf`改成`import tensorflow as tf`，并且删除`tf.compat.v1.disable_eager_execution()`。

完成之后，我们可以进行训练：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/4-6-7.png)

训练完成之后，我可以保留最后（训练效果最好的模型）进行保存，并且开始编写云函数：

```python
# -*- coding:utf-8 -*-

import base64, random, json
import tensorflow as tf
from model_train import cnn_graph
from util import vec2text, convert2gray
from util import CAPTCHA_LIST, CAPTCHA_WIDTH, CAPTCHA_HEIGHT, CAPTCHA_LEN
from PIL import Image
import numpy as np


x = tf.placeholder(tf.float32, [None, CAPTCHA_HEIGHT * CAPTCHA_WIDTH])
keep_prob = tf.placeholder(tf.float32)
y_conv = cnn_graph(x, keep_prob, (CAPTCHA_HEIGHT, CAPTCHA_WIDTH))
saver = tf.train.Saver()

def captcha2text(image_list):
    """
    验证码图片转化为文本
    :param image_list:
    :return:
    """
    with tf.Session() as sess:
        saver.restore(sess, tf.train.latest_checkpoint('model/'))
        predict = tf.argmax(tf.reshape(y_conv, [-1, CAPTCHA_LEN, len(CAPTCHA_LIST)]), 2)
        vector_list = sess.run(predict, feed_dict={x: image_list, keep_prob: 1})
        vector_list = vector_list.tolist()
        text_list = [vec2text(vector) for vector in vector_list]
        return text_list


def main_handler(event, context):

    print(event)

    try:
        # 读取picture，并且保存
        imgData = base64.b64decode(json.loads(event["body"])['picture'])
        fileName = '/tmp/' + "".join(random.sample('zyxwvutsrqponmlkjihgfedcba', 5))
        with open(fileName, 'wb') as f:
            f.write(imgData)

        # 开始预测
        img = Image.open(fileName)
        img = img.resize((160, 60), Image.ANTIALIAS)
        img = img.convert("RGB")
        img = np.asarray(img)
        image = convert2gray(img)
        image = image.flatten() / 255
        pre_text = captcha2text([image])
        return {'result': pre_text}
    except Exception as e:
        return {'error': str(e)}

```

这其中有一个内容就是：我在训练的时候都是`160*60`的的大小，所以在测试时候也都是要转换成这个大小。

测试程序：

```python
import json
import urllib.request
import base64

with open("test.png", 'rb') as f:
    base64_data = base64.b64encode(f.read())
    s = base64_data.decode()

url = 'https://service-qzelhadc-1256773370.gz.apigw.tencentcs.com/release/demo'

print(urllib.request.urlopen(urllib.request.Request(
    url = url,
    data= json.dumps({'picture': s}).encode("utf-8")
)).read().decode("utf-8"))
```

测试完成：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-8.png)

可以看到已经初步可以识别成功。接下来，我们可以进行批量测试：

```python
# -*- coding:utf-8 -*-

import base64, random
import tensorflow.compat.v1 as tf
from model_train import cnn_graph
from util import vec2text, convert2gray
from util import CAPTCHA_LIST, CAPTCHA_WIDTH, CAPTCHA_HEIGHT, CAPTCHA_LEN
from PIL import Image
import numpy as np

tf.compat.v1.disable_eager_execution()

x = tf.placeholder(tf.float32, [None, CAPTCHA_HEIGHT * CAPTCHA_WIDTH])
keep_prob = tf.placeholder(tf.float32)
y_conv = cnn_graph(x, keep_prob, (CAPTCHA_HEIGHT, CAPTCHA_WIDTH))
saver = tf.train.Saver()

def captcha2text(image_list):
    """
    验证码图片转化为文本
    :param image_list:
    :return:
    """
    with tf.Session() as sess:
        saver.restore(sess, tf.train.latest_checkpoint('model/'))
        predict = tf.argmax(tf.reshape(y_conv, [-1, CAPTCHA_LEN, len(CAPTCHA_LIST)]), 2)
        vector_list = sess.run(predict, feed_dict={x: image_list, keep_prob: 1})
        vector_list = vector_list.tolist()
        text_list = [vec2text(vector) for vector in vector_list]
        return text_list


def main_handler(event, context):
    try:
        # 读取picture，并且保存
        imgData = base64.b64decode(event["body"])
        fileName = '/tmp/' + "".join(random.sample('zyxwvutsrqponmlkjihgfedcba', 5))
        with open(fileName, 'wb') as f:
            f.write(imgData)

        # 开始预测
        img = Image.open(fileName)
        img = img.resize((160, 60), Image.ANTIALIAS)
        img = img.convert("RGB")
        img = np.asarray(img)
        image = convert2gray(img)
        image = image.flatten() / 255
        pre_text = captcha2text([image])
        return {'result': pre_text}
    except Exception as e:
        return {'error': str(e)}

```

运行结果：

![](https://others-1304229895.cos.ap-shanghai.myqcloud.com/article/material/6-4-9.png)

```text
1330 {'result': ['1330']}
5142 {'result': ['5142']}
9524 {'result': ['9524']}
6867 {'result': ['6667']}
4644 {'result': ['4644']}
7023 {'result': ['7023']}
9615 {'result': ['9616']}
1684 {'result': ['1684']}
4123 {'result': ['4123']}
0135 {'result': ['0135']}
2503 {'result': ['2503']}
1112 {'result': ['1112']}
1977 {'result': ['1977']}
3242 {'result': ['3242']}
5867 {'result': ['5867']}
7143 {'result': ['7143']}
6238 {'result': ['6288']}
7049 {'result': ['7049']}
0665 {'result': ['0665']}
8557 {'result': ['8557']}

```

可以看到，基本测试之后，效果还是蛮不错的。当然，由于在训练的时候，使用的是`CAPTCHA_LIST = NUMBER`，所以目前只能识别数字，如果有兴趣，可以尝试生成混合的验证码。

## 总结

Serverless发展迅速，通过Serverless做一个验证码识别工具，我觉得这是一个非常酷的事情，在未来的数据采集等工作中，又一个优美的验证码识别工具是非常必要的额，当然验证码种类很多，针对不同类型的验证码识别，也是一项非常有挑战性的工作。
