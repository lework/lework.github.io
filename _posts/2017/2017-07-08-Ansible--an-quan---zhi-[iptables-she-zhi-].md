---
layout: post
title: "Ansible 安全 之【iptables设置】"
date: "2017-07-08 18:14:50"
categories: Ansible
excerpt: "规则 只开启ssh端口。 限制管理人员登录的ip地址。 操作步骤"
auth: lework
---
* content
{:toc}

## 规则
---

1. 只开启ssh端口。
2. 限制管理人员登录的ip地址。

## 操作步骤
```
清除所有规则
iptables -F
iptables -X
iptables -Z

添加规则，只允许192.168.77.1地址访问22端口，其他一律禁止。
iptables -I INPUT -s 192.168.77.1/32 -p tcp --dport 22 -j ACCEPT  
iptables -A INPUT -p tcp --dport 22 -j DROP
iptables -A INPUT -p icmp -j DROP
iptables -P INPUT DROP

保存配置并重启iptables服务
/etc/init.d/iptables save
/etc/init.d/iptables restart
```
