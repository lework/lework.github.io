---
layout: post
title: "Ansible Tower系列 五（安装 tower 3.1）"
date: "2017-05-31 21:58:27"
categories: Ansible
excerpt: "系统要求 RHEL 7，CentOS 7或Ubuntu 14.04 LTS或16.04 LTS上，并且是64位操作系统 内存最小 2 GB RA..."
auth: lework
---
* content
{:toc}

## 系统要求
---

- RHEL 7，CentOS 7或Ubuntu 14.04 LTS或16.04 LTS上，并且是64位操作系统
- 内存最小 2 GB RAM
- /var 分区最小 20GB
- Ansible Core 2.1.X或更高版本

## tower 用到的组件
---

- postgres
- memcached
- rabbitmq
- nginx
- supervisord
- uwsgi
- django
- celeryd

## 本次的环境
---
```
[root@localhost ~]# cat /etc/centos-release
CentOS Linux release 7.2.1511 (Core) 
[root@localhost ~]# python --version
Python 2.7.5
```

## 安装
---
下载安装包
```
wget http://releases.ansible.com/ansible-tower/setup-bundle/ansible-tower-setup-bundle-3.1.3-1.el7.tar.gz
tar zxf ansible-tower-setup-bundle-3.1.3-1.el7.tar.gz 
cd  ansible-tower-setup-bundle-3.1.3-1.el7
```
单实例配置tower

```
# cat inventory 
[tower]
localhost ansible_connection=local

[database]

[all:vars]
admin_password='admin'

pg_host=''
pg_port=''

pg_database='awx'
pg_username='awx'
pg_password='awx'

rabbitmq_port=5672
rabbitmq_vhost=tower
rabbitmq_username=tower
rabbitmq_password='tower'
rabbitmq_cookie=cookiemonster

# Needs to be true for fqdns and ip addresses
rabbitmq_use_long_name=false
```
配置admin的密码，pg的密码，rabbitmq的密码。
pg和rabbitmq 如果本机没有安装的话，默认会进行安装。


执行安装

```
./setup.sh
```

## 获取license
访问web页面，默认80端口

![image.png](/assets/images/Ansible/3629406-946c0cc66dbcb0de.png)


选择第二项，填写信息

![image.png](/assets/images/Ansible/3629406-422bf1d50bfc36e9.png)

填写完成后，ansible官方会发一份邮件到你的邮箱

![image.png](/assets/images/Ansible/3629406-48fcf904d5529566.png)

下载邮箱中的license，提交到页面。

> 这里提供一份enterprise的key，**谨记：此key只能用于测试和学习使用，切勿在生产环境使用，如有使用，后果自负。**
```
{
    "company_name": "Test ansible. ",
    "contact_email": "test@test.com",
    "contact_name": "test",
    "hostname": "f90080e78b3f48b7a29ea1877d212503",
    "instance_count": 1000000,
    "license_date": 2127363580,
    "license_key": "235c3abdf402716bc5fceae1a6832f7a2f9a1ed54efe6be1e8b188d246bfbde2",
    "license_type": "enterprise",
    "subscription_name": ""
}
```

点击提交后，就进入了后台页面

![image.png](/assets/images/Ansible/3629406-575c4ddfadacd173.png)





