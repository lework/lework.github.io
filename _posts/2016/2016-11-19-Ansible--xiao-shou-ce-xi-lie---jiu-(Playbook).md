---
layout: post
title: "Ansible 小手册系列 九（Playbook）"
date: "2016-11-19 17:06:00"
categories: Ansible
excerpt: "playbook是由一个或多个play组成的列表。play的主要功能在于将事先归并为一组的主机装扮成事先通过ansible中的task定义好..."
auth: lework
---
* content
{:toc}

playbook是由一个或多个"play"组成的列表。play的主要功能在于将事先归并为一组的主机装扮成事先通过ansible中的task定义好的角色。从根本上来讲所谓task无非是调用ansible的一个module。将多个play组织在一个playbook中即可以让它们联同起来按事先编排的机制同唱一台大戏。

其主要有以下四部分构成:
* Target section：   定义将要执行 playbook 的远程主机组
* Variable section： 定义 playbook 运行时需要使用的变量
* Task section：     定义将要在远程主机上执行的任务列表
* Handler section：  定义 task 执行完成以后需要调用的任务

** 开始书写我们第一个playbook**

## 第一步：定义我们的主机清单
---

```
[web]
192.168.77.129 ansible_ssh_pass=123456
192.168.77.130 ansible_ssh_pass=1234567
```

> 注：这里如果做了ssh免密码登陆，可以去掉


## 第二步：明确playbook做哪些任务
---
web组的主机完成下列任务
1. 远程执行用户为root
2. 安装httpd
3. apache配置文件实现自定义http端口和客户端连接数
4. 启动httpd，并设置其开机自启动

## 第三步：书写playbook
---
```
---
#httpd.yml
- hosts: web
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: name=httpd state=latest
  - name: write the apache config file
    template: src=/etc/ansible/httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf backup=yes
    notify:
    - restart apache
  - name: ensure apache is running (and enable it at boot)
    service: name=httpd state=started enabled=yes
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

## 第四步：运行playbok
---

```bash
ansible-playbook -i hosts test.yml
```

输出结果

![Paste_Image.png](/assets/images/Ansible/3629406-1895c3c1c3d062e2.png)


** 在执行playbook前，可以做些检查**

1. 检查palybook语法
```bash
ansible-playbook -i hosts httpd.yml --syntax-check
```
2. 列出要执行的主机
```bash
ansible-playbook -i hosts httpd.yml --list-hosts
```
3. 列出要执行的任务
```bash
ansible-playbook -i hosts httpd.yml --list-tasks
```

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
