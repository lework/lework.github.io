---
layout: post
title: "Ansible 开发模块 之【使用其他语言写module】"
date: "2017-05-28 16:49:59"
categories: Ansible
excerpt: "语言 使用bash语言开发modules 功能实现 创建一个文件。 module 返回值一定是json dumps的字符串。ansible的参数..."
auth: lework
---
* content
{:toc}

## 语言
---
使用bash语言开发modules

## 功能实现
---

创建一个文件。

## module
---
```
cd  /etc/ansible
cat library/touch.sh

#!/bin/sh

args_file=$1

[ ! -f "$args_file" ] && echo -n '{"failed": true, "msg": "missing required arguments: file"}' && exit 1
args_result=$(cat $args_file | gawk -F'file=' '{print $2}' | gawk -F' ' '{print $1}')

[ ! -n "$args_result" ] && echo -n "{\"failed\": true, \"msg\": \"file () is absent, cannot continue\", \"file\": \"$args_result\"}" && exit 1

touch $args_result && echo -n "{\"changed\": true, \"rc\": $?,\"file\": \"$args_result\"}" || echo -n "{\"failed\": true, \"rc\": $?, \"file\": \"$args_result\"}"
exit $?
```
返回值一定是json dumps的字符串。
ansible的参数都会被写入一个名为args的文件，上图的$1 就是这个文件的路径，读取这个文件的内容，就能获取file参数的值。


## playbook
---
```
cat touch.yml 
---

- hosts: node1
  tasks:
  - touch: file=/tmp/123
```

## 主机清单
---
```
cat hosts
[node1]
192.168.77.129 ansible_ssh_pass=123456 ansible_sh_interpreter=/bin/sh
```
指出执行模块的可执行文件

## 执行结果
---

![image.png](/assets/images/Ansible/3629406-d77a4e11b6dfe42d.png)

文件已被创建
