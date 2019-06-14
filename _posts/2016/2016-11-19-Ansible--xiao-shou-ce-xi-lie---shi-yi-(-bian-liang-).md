---
layout: post
title: "Ansible 小手册系列 十一（变量）"
date: "2016-11-19 17:36:18"
categories: Ansible
excerpt: "变量名约束 变量名称应为字母，数字和下划线。 变量应始终以字母开头。 变量名不应与python属性和方法名冲突。 变量使用 通过命令行传递变量（..."
auth: lework
---
* content
{:toc}

## 变量名约束
---

- 变量名称应为字母，数字和下划线。 
- 变量应始终以字母开头。
- 变量名不应与python属性和方法名冲突。


## 变量使用
---

**通过命令行传递变量（extra vars）**

```bash
ansible-playbook release.yml -e "user=starbuck"
```

**在 `inventory` 中定义变量（inventory vars）**

```
host3 http_port=80 # 定义主机变量
[webservers:vars] # 定义组的变量
ntp_server= ntp.example.com
```

**在 `playbook` 中如何定义变量（play vars）**

```
- hosts: webservers
  vars:
    http_port: 80
```

**从角色和文件包含中定义变量**

```
- hosts: webservers
   include_vars: myvars.yml

- hosts: webservers
  vars_files:
    - /vars/external_vars.yml
```
**定义角色默认的变量（role defaults）**

在角色目录中添加一个defaults/main.yml文件。文件里存储着yaml或json格式的数据。


**以交互方式获取变量值**
```
---
- hosts: server
  vars_prompt:
    - name: web
      prompt: 'Please input the web server:'
      private: no
```

**定义角色变量（role and include vars）**

```
roles:
   - { role: app_user, name: Ian    }
```

**注册变量（registered vars）**

```
---
- hosts: all 
  tasks:
  - shell: uptime
    register: result
  - name: show uptime
    debug: var=result
```

此选项将任务的结果存储在变量中，结果参数可以用在模版中。名称为result，使用debug来输出result的信息。
 
以下是一些重要的注册变量的组件：

- changed: 显示是否已更改
- cmd:  执行的命令
- rc: 命令的返回码
- stdout:命令的输出
- stdout_lines: 逐行输出
- stderr: 如果有错误，则输出错误的信息

**内置变量**

|变量名称|说明|使用|
|:---|:-----|:---|
|hostvars|包含主机得fcats信息|`{{ hostvars['db.example.com'].ansible_eth0.ipv4.address }}`|
|inventory_hostname|	当前主机的名称|	`{{ hostvars[inventory_hostname] }}`|
|groups_name	|当前主机所在组的主机列表|	`\{\% if 'webserver' in group_names \%\}# some part of a configuration file that only applies to webservers\{\% endif \%\}`|
|groups	|包含设备清单组内的所有主机|	`{% for host in groups['db_servers'] %} {{ host }}{% endfor %}`|
|play_hosts	|在当前playbook中处于活动状态的主机名列表|	`{{play_hosts}}`|
|ansible_version	|ansible版本信息|	`{{ansible_version}}`|

## 变量优先级
---

> 最后的优先级最高

• role defaults
• inventory vars
• inventory group_vars
• inventory host_vars
• playbook group_vars
• playbook host_vars
• host facts
• play vars
• play vars_prompt
• play vars_files
• registered vars
• set_facts
• role and include vars
• block vars (only for tasks in block)
• task vars (only for the task)
• extra vars (always win precedence)

如果多个组具有相同的变量，则最后一个加载获胜。


## 变量范围
---

Ansible有3个主要范围：

- 全局：这是由config，环境变量和命令行设置的
- play：每个play和包含的结构，vars条目，include_vars，角色默认和vars。
- 主机：直接与主机相关联的变量

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
