---
layout: post
title: "Ansible 小手册系列 六（Patterns 匹配模式）"
date: "2016-11-19 16:19:08"
categories: Ansible
excerpt: "Patterns 是定义Ansible要管理的主机。但是在playbook中它指的是对应主机应用特定的配置或IT流程。 命令格式 命令行 ans..."
auth: lework
---
* content
{:toc}

Patterns 是定义Ansible要管理的主机。但是在playbook中它指的是对应主机应用特定的配置或IT流程。

## 命令格式
---

命令行

**`ansible <host-pattern> [options]`**

playbook 中

**`- hosts: <host-pattern>`**

## 使用示例
---

```bash
ansible * -m service -a "name=httpd state=restarted"
```

## Patterns 使用
---

** 匹配所有的主机 **

```
all
*
```
> 以上两个Patterns 均表示匹配所有的主机

**精确匹配**

```
192.168.77.121
```

> 以上Patterns 表示只匹配192.168.77.121这一个主机

**或匹配**

```
web:db
```

> 以上Patterns 表示匹配的主机在web组或db组中

**非模式匹配**

```
"web:\!db"
```

> 命令下需转义特殊符号，以上Patterns 表示匹配的主机在web组，不在db组中，包含在web组，又在db中的用户


**交集匹配**

```
"web:&db"
```

> 以上Patterns 表示匹配的主机同时在db组和dbservers组中

** 通配符匹配**

```*.com
web-*.com:dbserver
webserver[0]
webserver[0:25]
```

> *表示所有字符，[0]表示组第一个成员，[0:25] 表示组第1个到第24个成员，类似python中得切片

**正则表达式匹配**

```
~(web|db).*\.example\.com
```
> 在开头的地方使用“~”，表示这是一个正则表达式

**组合匹配**

```
"webservers:dbservers:&staging:!phoenix"
```

> 在webservers 或者dbservers 组中，必须还存在于staging 组中，但是不在phoenix 组中

在ansible-palybook 命令中，你也可以使用变量来组成这样的表达式，但是你必须使用“-e”的选项来指定这个表达式

```
webservers:!{{excluded}}:&{{required}}
```

**排除条件**


> 只执行-l后的主机

```bash
ansible-playbook site.yml -l 192.168.77.129
ansible-playbook site.yml --l  @retry_hosts.txt
```

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
