---
layout: post
title: "Ansible 安全 之【控制端安全设置】"
date: "2017-07-08 18:16:05"
categories: Ansible
excerpt: "ansible控制端作为管理其他服务器的总控制器，其所处的服务器必定存在大量的密码信息，或者有权连接远程服务器。那这种情况下，如果控制端有安全隐..."
auth: lework
---
* content
{:toc}

ansible控制端作为管理其他服务器的总控制器，其所处的服务器必定存在大量的密码信息，或者有权连接远程服务器。那这种情况下，如果控制端有安全隐患，从而落到他人之手，后果不堪设想。那我们怎么避免这种情况的发生，做好安全策略是关键。

下面我列出了一些安全策略：
1. 服务器不放置在公网环境。
2. 不安装任何服务，只开启ssh端口。
3. 限制管理人员登录的ip地址。
4. 加密主机清单。
5. 命令审计。
6. ssh登录二次验证。


下面是是上面安全策略的实施方法：
实验系统： **CentOS release 6.7 (Final)**

- [Ansible 安全 之【iptables设置】](http://www.jianshu.com/p/ebe7a85579e4)
- [Ansible 安全 之【加密主机清单】](http://www.jianshu.com/p/7f29993e13a9)
- [Ansible 安全 之【命令审计】](http://www.jianshu.com/p/793e54e7e5d5)
- [Ansible 安全 之【ssh登录二次验证】](http://www.jianshu.com/p/2c7d99ada982)
