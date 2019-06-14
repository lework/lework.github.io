---
layout: post
title: "Ansible 插件 之 【统计任务处理时间】"
date: "2016-11-19 21:40:38"
categories: Ansible
excerpt: "在做性能优化之前首先需要做的是收集一些统计数据，这样才能为后面做的性能优化提供数据支持，对比优化前后的结果。非常不错的是，在 github 发现..."
auth: lework
---
* content
{:toc}

> 在做性能优化之前首先需要做的是收集一些统计数据，这样才能为后面做的性能优化提供数据支持，对比优化前后的结果。非常不错的是，在 github 发现一个 Ansible 任务计时插件“ansible-profile”，安装这个插件后会显示 ansible-playbook 执行每一个任务所花费的时间。

> ** 在`ansible2.2`版本以上，ansible自带了`/usr/lib/python2.6/site-packages/ansible/plugins/callback/profile_tasks.py`文件，所以，只需在callback_whitelist开启这个插件，从而不需要下载这个文件，就可以实现统计任务处理时间的功能。**

Github 地址：https://github.com/jlafon/ansible-profile

这个插件安装很简单，只需要简单的三个命令即可完成安装。在你的 playbook 文件的目录下创建一个目录，目录名 callback_plugins 然后将下载的 profile_tasks.py 文件放到该目录下。

```
cd /etc/ansible 
mkdir callback_plugins 
cd callback_plugins 
wget https://raw.githubusercontent.com/jlafon/ansible-profile/master/callback_plugins/profile_tasks.py
```


ansible 2.0版本需要在ansible.cfg 中加入
```
callback_whitelist = profile_tasks
```


现在，执行 ansible-playbook 命令就会看到 playbook 中每个 tasks 的用时情况。

![Paste_Image.png](/assets/images/Ansible/3629406-598bb35e440a1a7c.png)

在这里，我设置了 2 个 task，1 个 task sleep2 秒，另 1 个 task sleep4秒，在 PLAY RECAP 处会汇总所有 task 执行消耗的时间，并按照耗费时间排序。

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
