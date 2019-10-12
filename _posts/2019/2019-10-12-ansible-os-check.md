---
layout: post
title: "使用ansible tasks生成linux巡检报告"
date: "2019-10-12 18:40:00"
category: Ansible
tags:  Ansible linux
author: lework
---
* content
{:toc}

一直想做个关于资源巡检的功能，其需求就是通过邮件的形式来查看linux资源的使用情况，超出一定的阈值时高亮显示出来。也有人说啦，这个需求通过监控`zabbix`, `prometheus`都能做呀，何必自己重复造轮子做这些啊。我就是瞎折腾呗，只能说巡检报告是一总主动探测系统资源的一种手段，一般公司监控，外部都不能直接访问的，需要拨通vpn才可以，有些情况我们是无法连接到监控平台，比如放假游玩，不想打开电脑...这些情况下通过每天的巡检报告可以随时的了解系统资源的情况。




## 使用task方式获取报告

### 统计的系统资源

- Hostname
- Main IP
- OS
- CPU Used
- CPU LoadAvg
- Mem Used
- Swap Used
- Disk Size Used
- Disk Inode Used
- Tcp Connection Used
- Timestamp

### 克隆git仓库

``` 
git clone https://github.com/lework/Ansible-roles.git /etc/ansible/roles/
mv /etc/ansible/roles/filter_plugins /etc/ansible/
```

> 这里我们只用到了`filter_plugins`, `os-check` role

> 在使用role之前，一定要查看role的`README.md`

### 定义主机

```
#/etc/ansible/hosts
[node2]
192.168.77.130 ansible_ssh_pass=123456
```

### 编写playbook

```
#/etc/ansible/os-check.yaml
---
- hosts: all
  gather_facts: false
  vars:
    check_report_path: /tmp
    check_mail_host: "smtp.lework.com"
    check_mail_port: "465"
    check_mail_username: "ops@lework.com"
    check_mail_password: "le123456"
    check_mail_to: ["ops@lework.com"] 
  roles:
    - os-check
```

### 执行playbook

```
ansible-playbook /etc/ansible/os-check.yaml
```

### 执行结果

报告文件存放在`/tmp`目录下

邮件中也能看到报告内容了

![os-check](/assets/images/Ansible/os-check.png)


### 执行流程

简要的说下执行流程

1. 使用脚本`files\check_linux.sh`在远端执行获取资源数据，并以json结构体返回。
2. 使用`jinja2`模板将获取的数据渲染到模板文件中`templates\report-cssinline.html`,生成的文件存放在指定的目录中。
	- `report-cssinline.html` 是将css设置以`inline`的方式存储的html文件,`report.html`才是源模板文件，修改完源模板文件后，使用[Responsive Email CSS Inliner](https://htmlemail.io/inline/)进行转换下，才能更好的兼容邮件显示。
	- 其中模板中使用的`get_check_data`过滤器是从`hostvars`中获取每台主机的脚本执行结果，进行分析整理传递给模板，使用传递回来的数据进行渲染。
3. 获取生成的模板文件内容，并通过smtp发送给接收人。



## 使用fact数据获取报告

上面的操作是我们通过自己写脚本获取系统数据，这种方式有执行速度快，自定义强的优点，也有兼容性差的问题，对各个系统支持的不全面。使用fact数据则是相反。

那下面我们就使用ansible的fact数据来生成巡检报告

### 统计的系统资源

- Hostname
- Main IP
- OS
- Mem Used
- Swap Used
- Disk Size Used
- Disk Inode Used
- Timestamp


### 配置脚本

将下列链接中的文件下载到`ansible`控制机上
```
https://github.com/lework/script/tree/master/python/facts_os_check
```

在`ansible.py` 中我们配置fact目录和smtp信息

```
# 设置fact目录
fact_dirs = [os.path.join(current_path, 'facts')]


# 发送邮件
subject = 'System Check Report [%s]' % now_date
to_list = ['lework@ops.com']

mail_config = {
	'mail_host': 'smtp.lework.com',
	'mail_port': '465',
	'mail_user': 'ops@lework.com',
	'mail_pass': '123123'
}
```

### 执行脚本

> 使用python3环境

```
pip3 install jinja2

[ ! -d ./report ] && mkdir ./report
[ ! -d ./facts ] && mkdir ./facts

rm -rf ./facts

ansible all -m setup --tree ./facts

python3 ansible.py
```

> 在每次生成fact文件时，需要对其目录进行清空操作，避免历史数据干扰。


巡检报告文件以日期命名的方式存放在当前的`report`目录下