---
layout: post
title: "Ansible Role 容器 之【docker】"
date: "2018-02-08 16:56:06"
categories: Ansible
excerpt: "Ansible Role: docker 安装docker 要求 此角色仅在RHEL及其衍生产品上运行。 测试环境 ansible 2.4.2...."
auth: lework
---
* content
{:toc}
{% raw %}

# Ansible Role: docker

安装docker

## 要求

此角色仅在RHEL及其衍生产品上运行。

## 测试环境

ansible `2.4.2.0`
python `2.7.5`
os `Centos 7.4 X64`

## 角色变量
    docker_repo: "https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo"

    docker_old:
      - docker
      - docker-common
      - docker-selinux
      - docker-engine

    docker_packages:
      - lvm2
      - yum-utils
      - device-mapper-persistent-data
      
    docker_start: true

## 依赖

centos 7.3 以上版本

## github地址
https://github.com/lework/Ansible-roles/tree/master/docker

## Example Playbook

    - hosts: node1
      roles:
        - docker
        
## 使用
```
systemctl start docker
systemctl stop docker
systemctl status docker
```
{% endraw %}
