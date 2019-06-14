---
layout: post
title: "Ansible 小手册系列 十二（Facts）"
date: "2016-11-19 17:42:24"
categories: Ansible
excerpt: "Facts 是用来采集目标系统信息的，具体是用setup模块来采集得。 使用setup模块来获取目标系统信息 仅显示与ansible相关的内存信..."
auth: lework
---
* content
{:toc}

> Facts 是用来采集目标系统信息的，具体是用setup模块来采集得。

使用setup模块来获取目标系统信息
```
ansible hostname -m setup
```
仅显示与ansible相关的内存信息
```
ansible all -m setup -a 'filter=ansible_*_mb'
```

## 常用的变量
---

- ansible_distribution
- ansible_distribution_release
- ansible_distribution_version
- ansible_fqdn
- ansible_hostname
- ansible_os_family
- ansible_pkg_mgr
- ansible_default_ipv4.address
- ansible_default_ipv6.address


## 关闭自动采集
---

```
- hosts: whatever
  gather_facts: no
```
## 自定义目标系统facts
---

在远程主机/etc/ansible/facts.d/目录下创建.fact 结尾的文件，也可以是json、ini 或者返回json 格式数据的可执行文件，这些将被作为远程主机本地的facts 执行

![Paste_Image.png](/assets/images/Ansible/3629406-88eb9bcb00b965b1.png)



可以通过`{{ ansible_local.preferences.test.h }}`方式来使用该变量

![Paste_Image.png](/assets/images/Ansible/3629406-c986d4114d116fd7.png)

## Facts 使用文件作为缓存
---

修改ansible配置文件
```
# /etc/ansible/ansible.cfg
fact_caching = jsonfile
fact_caching_connection = /tmp/facts_cache

mkdir /tmp/facts_cache
chmod 777 /tmp/facts_cache/
```
运行playbook
```bash
ansible-playbook facts.yml 
```
查看缓存目录

![Paste_Image.png](/assets/images/Ansible/3629406-4eff2e24c9ec5071.png)


上述文件中存储着json序列化的facts数据

## Facts 使用redis作为缓存
---

安装redis
```
yum -y install redis-server
easy_install pip
pip install redis
```
配置redis，使用密码登陆
```
#vim /etc/redis.conf
requirepass "admin"
```
启动redis
```
service redis start
```
修改ansible配置文件
```
# /etc/ansible/ansible.cfg
gathering = smart
fact_caching = redis
fact_caching_timeout = 86400
fact_caching_connection = localhost:6379:0:admin
```
运行playbook
```
ansible-playbook facts.yml
```

查看redis内容

![Paste_Image.png](/assets/images/Ansible/3629406-ac5640d463de3982.png)



## Facts 使用memcached作为缓存
---

安装  memcached
```
yum install memcached
easy_install pip
pip install python-memcached
```
启动memcached
```
/usr/bin/memcached -d -u memcached  
```
修改ansible配置文件
```
# /etc/ansible/ansible.cfg
gathering = smart
fact_caching = memcached
fact_caching_timeout = 86400
fact_caching_connection = localhost:11211
```
查看memcached内容

![Paste_Image.png](/assets/images/Ansible/3629406-7486b3aad3a76723.png)

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
