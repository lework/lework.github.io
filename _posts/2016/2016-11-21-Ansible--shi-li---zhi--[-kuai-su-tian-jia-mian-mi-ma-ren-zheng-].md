---
layout: post
title: "Ansible 实例 之 【快速添加免密码认证】"
date: "2016-11-21 20:58:27"
categories: Ansible
excerpt: "第一步：将需要登陆主机得公钥添加到known_hosts 还可以使用下列简单办法： ssh在首次连接出现检查keys 的提示，通过设置 这样，在..."
auth: lework
---
* content
{:toc}
{% raw %}
## 第一步：将需要登陆主机得公钥添加到known_hosts
---
```
ssh-keyscan 192.168.77.129 192.168.77.130 >> /root/.ssh/known_hosts
```

还可以使用下列简单办法：

ssh在首次连接出现检查keys 的提示，通过设置
```
export ANSIBLE_HOST_KEY_CHECKING=False

```
这样，在执行playbook时，就**跳过**这些提示。

## 第二步：生成管理主机得私钥和公钥
---
```
ssh-keygen -t rsa -b 2048 -P '' -f /root/.ssh/id_rsa
```

## 第三步：添加主机信息到主机清单中
---
```
# hosts
[web]
192.168.77.129 ansible_ssh_pass=1234567
192.168.77.130 ansible_ssh_pass=123456
```

`ansible_ssh_pass`密码如果一样的话，这里就不需要定义了。在运行`ansible-playbook`时 加上`-k`参数，就可以输入登陆密码

## 第4步：配置palybook
---

```
# ssh-addkey.yml 
---
- hosts: all
  gather_facts: no

  tasks:

  - name: install ssh key
    authorized_key: user=root 
                    key="{{ lookup('file', '/root/.ssh/id_rsa.pub') }}" 
                    state=present
```

## 第五步，运行playbook
---
```
ansible-playbook -i hosts ssh-addkey.yml
```

![Paste_Image.png](/assets/images/Ansible/3629406-dc2512a268b0464b.png)

这样，管理节点的公钥就会添加到节点得authorized_keys文件中，再把主机清单里的ansible_ssh_pass去掉，执行ansible all -m ping 就不需要密码了。

更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
{% endraw %}