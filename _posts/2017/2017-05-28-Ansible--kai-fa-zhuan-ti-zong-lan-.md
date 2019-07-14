---
layout: post
title: "Ansible 开发专题总揽"
date: "2017-05-28 16:54:16"
categories: Ansible
excerpt: "官方开发文档 http://docs.ansible.com/ansible/dev_guide/index.html 非常推荐大家看官方文档 ..."
auth: lework
---
* content
{:toc}

## 官方开发文档
----
http://docs.ansible.com/ansible/dev_guide/index.html

> 非常推荐大家看**官方文档**

## 环境
----
本次所用的环境
- ansible `2.3.0.0`
- os `Centos 6.7 X64`
- python `2.6.6`

## 介绍
----
Ansible 开发分为两大模块，一是`modules`，而是`plugins`。

首先，要记住这两部分内容在哪个地方执行？
- `modules` 文件被传送到**远端主机**并执行。
- `plugins` 是在**ansible服务器**上执行的。

再者是执行顺序？
`plugins` 先于 `modules` 执行。

然后大家明确这两部分内容是干啥用的？
- `modules` 是ansible的核心内容，它使playbook变得更加简单明了，一个task就是完成某一项功能。ansible模块是被传送到远程主机上运行的。所以它们可以用远程主机可以执行的任何语言编写modules。
- `plugins` 是在**ansible主机**上执行的，用来辅助modules做一些操作。比如连接远程主机，拷贝文件到远程主机之类的。


## ansible执行ping模块的过程。
----
![ansible运行过程.jpg](/assets/images/Ansible/3629406-cdde75580732a013.jpg)

图片看不清，移步http://upload-images.jianshu.io/upload_images/3629406-cdde75580732a013.jpg

如果想要源文件，请加入QQ群[425931784](http://shang.qq.com/wpa/qunwpa?idkey=47638ae0b21fc2b1e714939524706b1fc405bc04cbd9426a8bcc9ed3d0c83954)，至群文件下载。

## github
---
所有的脚本文件，插件，模块都会放在这个仓库中。

[https://github.com/lework/Ansible-dev](https://github.com/lework/Ansible-dev)

## 调试
----
- [Ansible 开发调试 之【pdb本地调试】](http://www.jianshu.com/p/37a0e64a8242)
- [Ansible 开发调试 之【pycharm远程调试】](http://www.jianshu.com/p/8f06b45d8e0c)
- [Ansible 开发调试 之【模块调试】](http://www.jianshu.com/p/5d65a8a0088a)

## modules 开发
----
- [Ansible 开发模块 之【模块说明】](http://www.jianshu.com/p/fd8ae373fc99)
- [Ansible 开发模块 之【构建一个简单的module】](http://www.jianshu.com/p/7d89b4f8c140)
- [Ansible 开发模块 之【添加module的文档说明】](http://www.jianshu.com/p/95a7c0a57dbc)
- [Ansible 开发模块 之【使用其他语言写module】](http://www.jianshu.com/p/6676377b6e25)
- [Ansible 开发模块 之【module的返回值】](http://www.jianshu.com/p/6771cd8851e6)
- [Ansible 开发模块 之【连接华为交换机】](http://www.jianshu.com/p/f72b79b0d3f9)
- [Ansible 开发模块 之【企业微信通知】](/2019/07/03/ansible-wechat-module)

## plugins 开发
----
- [Ansible 开发插件之【插件说明】](http://www.jianshu.com/p/c5abe3e575b5)
- [Ansible 开发插件之【callback】](http://www.jianshu.com/p/455c58cf3758)
- [Ansible 开发Callback插件之【mail】](http://www.jianshu.com/p/ed1d9d00b007)
- [Ansible 开发Callback插件之【BlackHole】](https://www.jianshu.com/p/6db8d132c15d)
- [Ansible 开发插件之【动态主机清单】](http://www.jianshu.com/p/706c98215c02)
- [Ansible 开发Filters插件之【split】](http://www.jianshu.com/p/76ecddd89aa9)
- [Ansible 开发Action插件之【le_copy】](http://www.jianshu.com/p/bd0d212d58e9)

## API使用
---
- [Ansible 开发API 之【使用api运行task任务】](http://www.jianshu.com/p/62b6d2325648)
- [Ansible 开发API 之【使用api运行playbook任务】](http://www.jianshu.com/p/c1af78d75f45)
- [Ansible 开发API 之【使用api运行task,自定义主机信息】](http://www.jianshu.com/p/2ebaedfc743e)
