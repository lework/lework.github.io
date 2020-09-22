---
layout: post
title: "Ansible 小手册系列 一（介绍）"
date: "2016-11-19 15:15:30"
categories: Ansible
excerpt: "介绍 Ansible 是一个配置管理和应用部署工具，功能类似于目前业界的配置管理工具 Chef,Puppet,Saltstack。Ansible..."
auth: lework
---
* content
{:toc}

## 介绍
----

Ansible 是一个配置管理和应用部署工具，功能类似于目前业界的配置管理工具 Chef,Puppet,Saltstack。Ansible 是通过 Python 语言开发。Ansible 平台由 Michael DeHaan 创建，他同时也是知名软件 Cobbler 与 Func 的作者。Ansible 的第一个版本发布于 2012 年 2 月，相比较其它同类产品来说，Ansible 还是非常年轻的，但这并不影响他的蓬勃发展与大家对他的热爱。

Ansible 默认通过 SSH 协议管理机器，所以 Ansible 不需要安装客户端程序在服务器上。您只需要将 Ansible 安装在一台服务器，在 Ansible 安装完后，您就可以去管理控制其它服务器。不需要为它配置数据库，Ansible 不会以 daemons 方式来启动或保持运行状态。

Ansible 的目标有如下：
• 自动化部署应用
• 自动化管理配置
• 自动化的持续交付
• 自动化的（AWS）云服务管理。

根据 Ansible 官方提供的信息，当前使用 Ansible 的用户有：evernote、rackspace、NASA、Atlassian、twitter 等。

##  Ansible是怎么工作的
---

![Paste_Image.png](/assets/images/Ansible/3629406-84b4a2e7285d74bf.png)

从上图可以看出，运行ansible的先决条件是，安装ansible到管理节点，定义主机清单，并有一些playbooks定义。

让我们来看看我们如何使用Ansible将我们的Ubuntu虚拟机转换为Web服务器。

您在管理节点上运行Ansible Playbook，它查看您在playbook中定义的命令参数，并通知我们定位到网络组中的节点。 Ansible然后读取主机清单以查找分配给Web组的节点。在这一点上，Ansible已经准备好开始工作，所以它将通过ssh远程连接到定义的机器，通常你会想要通过预共享密钥建立一些类型的ssh信任，这样你就不必在进行ssh登陆的时候输入密码。然后Ansible将开始逐步执行playbook中的任务，一次一个任务，从顶部到底部的顺序遍历它们，就像你手动登录执行任务一样。所以，它安装软件包，更新配置文件，使用git部署我们的网站代码，最后启动我们的Web服务。当Ansible很愉快的把一切都按预期的完成，你会得到一个执行成功的状态报告。

可以用动图说明下此次过程。


![46-ansible-playbook-haproxy-nginx1.gif](/assets/images/Ansible/3629406-cd0eaeac69698452.gif)

## 对管理主机的要求
---

目前,只要机器上安装了 Python 2.6 或 Python 2.7 (windows系统不可以做控制主机),都可以运行Ansible.
主机的系统可以是 Red Hat, Debian, CentOS, OS X, BSD的各种版本,等等.

## 对节点主机的要求
---

通常我们使用 ssh 与托管节点通信，默认使用 sftp.如果 sftp 不可用，可在 ansible.cfg 配置文件中配置成 scp 的方式. 在托管节点上也需要安装 Python 2.4 或以上的版本.如果版本低于 Python 2.5 ,还需要额外安装一个模块:

`python-simplejson`

## Ansible 与其它配置管理的对比
---

选择了目前几款主流的与 Ansible 功能类似的配置管理软件 Puppet、Saltstack，这里所做的对比不针对各个软件的性能作比较，只是对各个软件的特性做个对比。


|     	|Puppet|	Saltstack |	Ansible |
| ---------------- |:-------------:|:-------------:|:-------------:|
|开发语言	|Ruby|	Python|	Python|
|是否有客户端|	有|	有	|无|
|是否支持二次开发|	不支持|	支持	|支持|
|服务器与远程机器是否相互验证|	是|	是|	是|
|服务器与远程机器通信是否加密|	是，标准 SSL 协议	|是，使用 AES 加密|是，使用 OpenSSH|
|平台支持|	支持 AIX、BSD、HP-UX、Linux、 MacOSX、Solaris、 Windows	|支持 BSD、Linux、Mac OS X、Solaris、 Windows	|支持 AIX、BSD、 HP-UX、 Linux、Mac OSX、Solaris|
|是否提供 web| ui	提供|	提供|	提供，不过是商业版本|
|配置文件格式|	Ruby 语法格式	|YAML|	YAML|
|命令行执行|	不支持，但可通过配置模块实现	|支持|	支持|

## 资源
---

官方文档： http://docs.ansible.com/
中文文档： http://www.ansible.com.cn/    http://ansible-tran.readthedocs.io/
Jinja2 中文文档： http://docs.jinkan.org/docs/jinja2/
yaml语法： http://www.yaml.org/
书籍： https://www.ansible.com/ebooks   链接：http://pan.baidu.com/s/1qYazeos 密码：28p2
ansible  examples ：https://github.com/ansible/ansible-examples
ansible-vim： https://github.com/pearofducks/ansible-vim （可以高亮显示，语法检查）

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
