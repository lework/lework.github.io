---
layout: post
title: "Ansible Tower系列 三（使用tower执行一个任务）"
date: "2016-11-19 21:34:08"
categories: Ansible
excerpt: "创建playbook Tower playbook 项目默认存在  /var/lib/awx/projects/ 创建登陆凭据 创建项目 创建主..."
auth: lework
---
* content
{:toc}

## 创建playbook
---

Tower playbook 项目默认存在  `/var/lib/awx/projects/`
```
su - awx
cd projects/
mkdir ansible-for-devops && cd ansible-for-devops
cat main.yml << EOF
---
- hosts: all
  gather_facts: no
 
  tasks:
  - name: Check the date on the server.
	command: date
  - name: Check the eth0 ip on the server.
	command: ifconfig eth0
EOF

```
## 创建登陆凭据
---

![Paste_Image.png](/assets/images/Ansible/3629406-755329a7a7001304.png)

## 创建项目
---

![Paste_Image.png](/assets/images/Ansible/3629406-0b0a38c2210d4a39.png)

## 创建主机清单
---

![Paste_Image.png](/assets/images/Ansible/3629406-fea7a6dae4a494e5.png)

## 在主机清单里添加主机
---

点击主机清单名称，就可以进入添加主机的页面

![Paste_Image.png](/assets/images/Ansible/3629406-95a29edd6df14575.png)


点击 +ADD HOST

![Paste_Image.png](/assets/images/Ansible/3629406-cb3f86552d11073c.png)

本次添加了2个主机

![Paste_Image.png](/assets/images/Ansible/3629406-ca5b94914ee16adb.png)


## 创建任务模板
---

Inventory 选择 ops_主机清单
PROJECT 选择 Test_Project
PALYBOOK 选择 man.yml
MACHINE CREDENTIAL 选择 ssh登陆账号
其他默认

![Paste_Image.png](/assets/images/Ansible/3629406-21d7ad7922777022.png)


## 运行模板
---

点击任务右侧得火箭按钮

![Paste_Image.png](/assets/images/Ansible/3629406-b2c78ccc7ee3c801.png)


## 查看任务运行情况
---

![Paste_Image.png](/assets/images/Ansible/3629406-819ff36fa335680f.png)

DETAILS 里面可以查看任务得详细信息

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
