---
layout: post
title: "Ansible 开发插件之【插件说明】"
date: "2017-05-28 16:49:40"
categories: Ansible
excerpt: "action action插件是在同名的modules之前运行的，且是在本地执行。目录提供的一些action插件在/usr/lib/python..."
auth: lework
---
* content
{:toc}
{% raw %}
## action
---

action插件是在同名的modules之前运行的，且是在本地执行。目录提供的一些action插件在/usr/lib/python2.6/site-packages/ansible/plugins/action/目录中

## cache
---
cache插件用于保留“fact”数据的操作。目前提供的方式是redis,memcached,memory,jsonfile,pickle,yml。这些插件可以在/usr/lib/python2.6/site-packages/ansible/plugins/cache找的到。

## callback
---
callback 插件可以在事件执行的时候增加新行为，目前提供的一些callback在/usr/lib/python2.6/site-packages/ansible/plugins/callback/目录中。

## connection
---
ansible通过使用connection插件来连接远程系统，可以通过配置connection来选择用哪种方式连接远程系统。ansible提供的connection插件可以在/usr/lib/python2.6/site-packages/ansible/plugins/connection找的到。

## filter
---
filter插件允许你在playbook和模版内操作数据，ansible使用filter plugin来扩展jinja2模版的功能。插件在/usr/lib/python2.6/site-packages/ansible/plugins/filter目录中

使用方式: "{{ statement | cloud_truth }}"

## lookup
---
用于从外部数据中提取数据并返回到变量或参数中。比如循环with_*的用法。插件在/usr/lib/python2.6/site-packages/ansible/plugins/lookup/
使用方式：{{ lookup('file', '/etc/foo.txt') }}

## shell
---
很像connection插件，ansible使用shell插件在shell环境中执行，目前支持的shell有csh，fish，powershell，sh。这些插件可以在/usr/lib/python2.6/site-packages/ansible/plugins/shell找的到

## strategy
---
控制任务执行流程，插件在/usr/lib/python2.6/site-packages/ansible/plugins/strategy目录

## terminal
---
用来连接cli的硬件设备，像交换机，路由器，防火墙。插件在/usr/lib/python2.6/site-packages/ansible/plugins/terminal/目录。

## test
---
用于验证数据，属于jinja2的功能

## vars
---
用来解析主机清单中的变量，像host_vars, group_vars 都是有var插件来完成的。插件在/usr/lib/python2.6/site-packages/ansible/inventory/vars_plugins目录


## 自定义的插件存放位置
---
1. 在ansible.cfg配置的插件目录。
2. 当您的Playbook同目录下或角色中有以下子文件夹之一时，插件会自动加载：
```
	'./shell_plugins'
	'./module_utils'
	'./test_plugins'
	'./callback_plugins'
	'./vars_plugins
	'./terminal_plugins'
	'./connection_plugins'
	'./lookup_plugins'
	'./strategy_plugins'
	'./filter_plugins'
	'./action_plugins'
	'./cache_plugins'
```
{% endraw %}