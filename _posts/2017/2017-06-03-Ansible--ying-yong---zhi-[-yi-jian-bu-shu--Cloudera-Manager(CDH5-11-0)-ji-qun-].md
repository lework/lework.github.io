---
layout: post
title: "Ansible 应用 之【一键部署 Cloudera Manager（CDH5.11.0）集群】"
date: "2017-06-03 21:54:54"
categories: Ansible
excerpt: "CDH 介绍 CDH是Apache Hadoop及相关项目中最完整，且经过测试和受欢迎的发行版。 CDH提供Hadoop的核心元素 - 可扩展存..."
auth: lework
---
* content
{:toc}

## CDH 介绍
---

CDH是Apache Hadoop及相关项目中最完整，且经过测试和受欢迎的发行版。 CDH提供Hadoop的核心元素 - 可扩展存储和分布式计算 - 以及基于Web的用户界面和重要的企业功能。 CDH是Apache授权的开源，是唯一提供统一处理，交互式SQL和交互式搜索以及基于角色的访问控制的Hadoop解决方案。

Cloudera Manager是用于管理CDH群集的端到端应用程序。 Cloudera Manager通过为CDH集群的每个部分提供细致的可见性和控制权，为企业部署设置了标准，从而提高运营商的性能，提高服务质量，提高合规性并降低管理成本。 通过Cloudera Manager，您可以轻松部署和集中操作完整的CDH堆栈和其他托管服务。 应用程序自动执行安装过程，将部署时间从几周缩短到几分钟; 为您提供运行的主机和服务的集群范围的实时视图; 提供了一个单一的中央控制台来在集群中实现配置更改; 并集成了全面的报告和诊断工具，以帮助您优化性能和利用率。 

Cloudera Manager架构
cloudera manager的核心是管理服务器，该服务器承载管理控制台的Web服务器和应用程序逻辑，并负责安装软件，配置，启动和停止服务，以及管理上的服务运行群集。

![image.png](/assets/images/Ansible/3629406-693027522626d110.png)

## ansible介绍
---
Ansible 是一个配置管理和应用部署工具，功能类似于目前业界的配置管理工具 Chef,Puppet,Saltstack。Ansible 是通过 Python 语言开发。Ansible 平台由 Michael DeHaan 创建，他同时也是知名软件 Cobbler 与 Func 的作者。

利用ansible的playbook安装集群，可实现一次编写，多次安装部署。所以使用ansible安装cdh5不仅仅是省一次的时间，而是省下了每个人安装cdh5的时间。

更多内容： http://www.jianshu.com/p/c56a88b103f8

## 本次使用的cdh集群架构

![image.png](/assets/images/Ansible/3629406-c0fa9423936645a7.png)

系统版本： `Centos 6.7 X64`

Server内存配置：8G，磁盘100G
Agent内存配置： 4G， 磁盘100G

>  本次安装的仅仅是cdh集群，而不是hadoop集群，hadoop集群在cm web页面可配置。

## 本次用到的ansible rules
---
- [Ansible Role 数据库 之【mysql】](http://www.jianshu.com/p/9ef27a8c1989)
    mysql 角色用于安装mysql数据库，用于cdh5的数据存储
- [Ansible Role 大数据 之【cdh5-pre】](http://www.jianshu.com/p/e84b78247969)
   cdh5-pre 角色用于在安装前做些环境设置操作，如ntp，java配置等等
- [Ansible Role 大数据 之【cdh5-agent】](http://www.jianshu.com/p/ef6f99afe08a)
  cdh5-agent 角色用于安装cm angent服务，连接server
- [Ansible Role 大数据 之【cdh5-server】](http://www.jianshu.com/p/e789d1d9295e)
  cdh5-server 角色用于安装cm server服务，管理agent

## Ansible系统环境
---

os:  `Centos 6.7 X64`
python: `2.6.6`

# 安装ansible和git
---

```bash
[root@node ~]# yum install -y ansible git
[root@node ~]# ansible --version
ansible 2.3.0.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = Default w/o overrides
  python version = 2.6.6 (r266:84292, Jul 23 2015, 15:22:56) [GCC 4.4.7 20120313 (Red Hat 4.4.7-11)]
```

## 设置ansible
---

关闭主机ssh known_hosts检查
配置文件：/etc/ansible/ansible.cfg
```vim
 59 host_key_checking = False
```

## 使用git 克隆ansible role
---

```
git clone https://github.com/lework/Ansible-roles.git /etc/ansible/roles/
```

## 添加主机清单
---
根据上面的集群架构图，定义cdh集群架构中的主机角色
```
# cat /etc/ansible/hosts
[cdh5-all]
192.168.77.129
192.168.77.130
192.168.77.131

[cdh5-server]
192.168.77.129

[cdh5-agent]
192.168.77.129
192.168.77.130
192.168.77.131

[all:vars]
ansible_ssh_pass=123456
```

## 添加playbook
---
利用ansible安装cdh集群，就是要写playbook。下面的playbook分为3个步骤
1. 对集群所有主机进行安装服务器预处理。利用role：cdh-pre
2. 安装cm server，注意这里先安装的是mysql，后续才是server，mysql的binlog format 必须是MiXED
3. 最后在安装cm agent，agent启动后会自动连接server。

```
# cat /etc/ansible/cdh5.yml 
---

- hosts: cdh5-all
  vars:
   - ipnames:
      '192.168.77.129': 'master'
      '192.168.77.130': 'node1'
      '192.168.77.131': 'node2'
  roles:
   - { role: cdh5-pre }

- hosts: cdh5-server
  vars:
   - mysql_host: 192.168.77.129
   - mysql_user: root
   - mysql_password: 123456
   - mysql_binlog_format: 'MiXED'
  roles: 
   - { role: mysql }
   - { role: cdh5-server }

- hosts: cdh5-agent
  vars:
   - cdh5_server_host: '192.168.77.129'
  roles:
   - { role: cdh5-agent }
```

## 执行playbook
---

```
# cd /etc/ansible/
# ansible-playbook cdh5.yml
```

## web配置
---

http://192.168.77.129:7180/cmf/login

默认用户名/密码: admin/admin

然后跟着web页面一步一步的部署hadoop集群

## 建议
---
>  在安装集群服务后，务必重启下机器。
