---
layout: post
title: "Ansible 开发调试 之【pycharm远程调试】"
date: "2017-05-28 16:52:24"
categories: Ansible
excerpt: "介绍 PyCharm是一种Python IDE，带有一整套可以帮助用户在使用Python语言开发时提高其效率的工具，比如调试、语法高亮、Proj..."
auth: lework
---
* content
{:toc}

## 介绍
---
PyCharm是一种Python IDE，带有一整套可以帮助用户在使用Python语言开发时提高其效率的工具，比如调试、语法高亮、Project管理、代码跳转、智能提示、自动完成、单元测试、版本控制。此外，该IDE提供了一些高级功能，以用于支持Django框架下的专业Web开发。
本地调试有许多不方便的地方。pycharm提供了所见及所得的调试界面。调试更加轻松方便。

## 配置pycharm远程调试
---

1. 打开pycharm--》RUN==》Edit Configuration
![image.png](/assets/images/Ansible/3629406-01b174e36a1d8328.png)
1. 点击+号按钮，选择Python Remote Debug
![image.png](/assets/images/Ansible/3629406-3b8da6ddea172f05.png)
1. 设置远程debug的监听地址。
 ![image.png](/assets/images/Ansible/3629406-ef85449d5438103b.png)
   - **Local host name** 是本机的IP。
   - **Port**在保证不冲突的情况下可以任意指定。
1. 启动pycharm调试
![image.png](/assets/images/Ansible/3629406-d4fece9f68ba6ff5.png)
可以看到console里的监听信息，正在等待远程主机连接。
![image.png](/assets/images/Ansible/3629406-4a85f69a8a7ad0bb.png)


## 在远程服务器上安装远程调试插件
---

1. 将pycharm-debug.egg文件拷贝到远程主机的python的site-packages目录下，并安装。
![image.png](/assets/images/Ansible/3629406-78ad90e8d9f0bca1.png)
安装pycharm-debug.egg
![image.png](/assets/images/Ansible/3629406-b4bcfd92a32955da.png)

2. 在需要调试的代码中加入远程调试所需的代码
查找到ansible执行文件
![](/assets/images/Ansible/3629406-1d6c326286f984e0.png)

3. 在程序入口添加下面两行代码
import pydevd
pydevd.settrace('192.168.77.1', port=9999, stdoutToServer=True, stderrToServer=True)
![image.png](/assets/images/Ansible/3629406-bc980fcc580f8224.png)
4. 启动ansible命令
![image.png](/assets/images/Ansible/3629406-cae6e5aa34974bff.png)


## 使用pycharm调试远程代码
---

1. 查看pycharm窗口，可以看到有链接进来。
![image.png](/assets/images/Ansible/3629406-a1a806b20856147a.png)
2. 此时可点击”Download”下载源码
![image.png](/assets/images/Ansible/3629406-9f881a88d2a0a40d.png)
3. 点击完成后，就可以看到远程的ansible代码。
![image.png](/assets/images/Ansible/3629406-b210d89660eb4739.png)
4. 调试的一些常用按钮
![image.png](/assets/images/Ansible/3629406-cd427cecdc81da3b.png)

