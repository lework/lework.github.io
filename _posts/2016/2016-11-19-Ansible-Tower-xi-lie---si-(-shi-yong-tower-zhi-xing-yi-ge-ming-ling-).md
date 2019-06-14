---
layout: post
title: "Ansible Tower系列 四（使用tower执行一个命令）"
date: "2016-11-19 21:35:58"
categories: Ansible
excerpt: "在主机清单页面中，选择一个主机清单，进入后，选择hosts里的主机 点击 RUN COMMANDS MODULE 选择 commandARGUM..."
auth: lework
---
* content
{:toc}

## 在主机清单页面中，选择一个主机清单，进入后，选择hosts里的主机
---

![Paste_Image.png](/assets/images/Ansible/3629406-8f0c95eab39a810a.png)

## 点击 RUN COMMANDS
---

MODULE 选择 command
ARGUMENTS 填写 ifconfig eth0
MACHINE CREDENTIAL 选择 ssh登陆账号
Verbosity 选择 3 (Debug)


![Paste_Image.png](/assets/images/Ansible/3629406-3fca45da07b3990f.png)

## 点击 Launch，查看输出
---

![Paste_Image.png](/assets/images/Ansible/3629406-881302e4e7461f93.png)

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
