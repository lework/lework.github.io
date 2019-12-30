---
layout: post
title: 'rabbitmq自动化安装'
date: '2019-12-27 20:00:00'
category: rabbitmq
tags: rabbitmq
author: lework
---
* content
{:toc}

使用ansible的role方式进行自动化安装rabbitmq单实例和集群。



## 系统环境

os: `Centos 7.4 X64`


## 下载rabbitmq role

```bash
yum install -y ansible git
git clone https://github.com/lework/Ansible-roles.git /etc/ansible/roles/
```

## 添加主机清单

```bash
#/etc/ansible/hosts
[rabbitmq-single]
192.168.77.130 ansible_ssh_pass=123456

[rabbitmq-cluster]
192.168.77.131 ansible_ssh_pass=123456
192.168.77.132 ansible_ssh_pass=123456
192.168.77.133 ansible_ssh_pass=123456
```

## 安装Playbook示例

### 使用到的role

- `hostnames` 设置主机名称
- `erlang`   安装erlang环境
- `rabbitmq`  安装rabbitmq

### 单实例安装rabbitmq

**单实例安装rabbitmq**

```yaml
---
- hosts: rabbitmq-single
  roles:
   - erlang
   - rabbitmq
```

**单实例安装rabbitmq, 并指定版本和启用插件和添加用户**

```yaml
- hosts: rabbitmq-single
  vars:
   - rabbitmq_version: "3.7.23"
   - rabbitmq_plugins: ['rabbitmq_top', 'rabbitmq_mqtt']
   - rabbitmq_server_users: [{user: 'test', pass: '123456', role: 'administrator'}]
  roles:
   - erlang
   - rabbitmq
```

### 集群安装

**集群安装rabbitmq, 采用镜像策略**
```yaml
---
- hosts: rabbitmq-cluster
  vars:
   - ipnames:
       '192.168.77.131': 'node1'
       '192.168.77.132': 'node2'
       '192.168.77.133': 'node3'
   - rabbitmq_plugins: ['rabbitmq_top', 'rabbitmq_mqtt']
   - rabbitmq_server_users: [{user: 'test', pass: '123456', role: 'administrator'}]
   - rabbitmq_cluster: true
  roles:
   - hostnames
   - erlang
   - rabbitmq
```

**集群安装rabbitmq, 采用镜像策略, 使用静态集群配**
```yaml
---
- hosts: rabbitmq-cluster
  vars:
   - ipnames:
       '192.168.77.131': 'node1'
       '192.168.77.132': 'node2'
       '192.168.77.133': 'node3'
   - rabbitmq_plugins: ['rabbitmq_top', 'rabbitmq_mqtt']
   - rabbitmq_server_users: [{user: 'test', pass: '123456', role: 'administrator'}]
   - rabbitmq_cluster: true
   - rabbitmq_cluster_discovery_classic: true
  roles:
   - hostnames
   - erlang
   - rabbitmq
```

根据以上情况，配置playbook，运行即可安装。

ansible role 详细介绍请看 [rabbitmq](https://github.com/lework/Ansible-roles/tree/master/rabbitmq)