---
layout: post
title: "Ansible 小手册系列 十七（特性模块）"
date: "2016-11-19 20:53:06"
categories: Ansible
excerpt: "异步操作和轮询 async: 1000  开启异步加载模式，超时时间1000秒。没有异步时间限制的默认值。 如果省略“async”关键字，任务将..."
auth: lework
---
* content
{:toc}

## 异步操作和轮询
---

```
---
# Requires ansible 1.8+
- name: 'YUM - fire and forget task'
  yum: name=docker-io state=installed
  async: 1000
  poll: 0
  register: yum_sleeper

- name: 'YUM - check on fire and forget task'
  async_status: jid={{ yum_sleeper.ansible_job_id }}
  register: job_result
  until: job_result.finished
  retries: 30
```

- async: 1000  开启异步加载模式，超时时间1000秒。没有异步时间限制的默认值。 如果省略“async”关键字，任务将同步运行。
- poll: 0  轮询时间0秒，直接跳过，执行下面的任务。默认轮询值为10秒。


## 检查模式
---

在检查模式状态下，是不会在远程系统上做出任何改变的。
```
ansible-playbook foo.yml --check
```

## 忽略错误
---
 ignore_errors 模块可以在任务执行错误时，忽略错误并继续执行任务。
```
  tasks:
  - command: /bin/false
    ignore_errors: true
  - debug: msg="false"
```

## 滚动执行
---
```
- name: test play
  hosts: webservers
  serial: 3
```
在上面的例子中，如果我们有100个主机，组“webservers”中的3个主机将完成playbook，然后再移动到接下来的3个主机。

还可以使用百分比
```
  serial: "30%"
```

## 指定最大失败数目
---

默认情况下，只要组中有尚未失败的主机，Ansible将继续执行操作。 在一些情况下，例如利用上述滚动更新，可能希望在达到失败的特定阈值时中止任务。
```
- hosts: webservers
  max_fail_percentage: 30
  serial: 10
```

## 委托facts
---
```
- hosts: app_servers
  tasks:
    - name: gather facts from db servers
      setup:
      delegate_to: "{{item}}"
      delegate_facts: True
      with_items: "{{groups['dbservers']}}"
```
以上将为dbservers组中的机器收集facts，并将facts分配给这些机器，而不是app_servers。 这样您可以查找hostvars ['dbhost1'] ['default_ipv4_addresses'] [0]，即使dbserver不是play的一部分，或者使用-limit省略。


## 任务只运行一次
---

```
---
  tasks:
    - command: /opt/application/upgrade_db.py
      run_once: true
```

## 有错误时立即中断ansbile
---

```
---
- hosts: web
  any_errors_fatal: True
```

## 设置环境变量
---
```
  tasks:
    - apt: name=cobbler state=installed
      environment:
        http_proxy: http://proxy.example.com:8080
	PATH: /var/local/nvm/versions/node/v4.2.1/bin:{{ ansible_env.PATH }}
```
 也可以在play级别使用
```
- hosts: testhost
  roles:
     - php
     - nginx
  environment:
    http_proxy: http://proxy.example.com:8080
```

## 即使任务失败，handlers也执行
---

以下3中方法均可使用
• 命令行加上 --force-handlers 参数
• 配置文件加上 force_handlers = True
• playbook里设置  force_handlers: True

## Prompts： 运行时，提示输入内容
---

```
- hosts: test
  gather_facts: no
  vars_prompt:
    - name: "name"
      prompt: "what is your name?"
      default: "user"
      private: yes
       confirm: yes
  tasks:
  - debug: msg={{ name }}
```

- name: 定义变量名称，即输入的内容赋值给此变量
- prompt：输入得提示信息
- default： 输入的默认值，没有输入任何内容的时候，把此值赋值给变量
- private：是否隐藏输入得内容
- confirm： 是否要再次确认输入

## 加密输入的内容
----

依赖Passlib包
```
- hosts: test
  gather_facts: no
  vars_prompt:
    - name: "pass"
      prompt: "Enter password"
      private: yes
      encrypt: "sha512_crypt"
      confirm: yes
      salt_size: 7
  tasks:
  - debug: msg={{ pass }}
```

可支持以下加密类型
• des_crypt - DES Crypt
• bsdi_crypt - BSDi Crypt
• bigcrypt - BigCrypt
• crypt16 - Crypt16
• md5_crypt - MD5 Crypt
• bcrypt - BCrypt
• sha1_crypt - SHA-1 Crypt
• sun_md5_crypt - Sun MD5 Crypt
• sha256_crypt - SHA-256 Crypt
• sha512_crypt - SHA-512 Crypt
• apr_md5_crypt - Apache’s MD5-Crypt variant
• phpass - PHPass’ Portable Hash
• pbkdf2_digest - Generic PBKDF2 Hashes
• cta_pbkdf2_sha1 - Cryptacular’s PBKDF2 hash
• dlitz_pbkdf2_sha1 - Dwayne Litzenberger’s PBKDF2 hash
• scram - SCRAM Hash
• bsd_nthash - FreeBSD’s MCF-compatible nthash encoding


## 标记 Tags
---

**标记一个任务**
```
tasks:
    - yum: name={{ item }} state=installed
      with_items:
         - httpd
         - memcached
      tags:
         - packages
    - template: src=templates/src.j2 dest=/etc/foo.conf
      tags:
         - configuration
```


**标记playbook**
```
- hosts: all
  tags:
    - bar
  tasks:
    ...

- hosts: all
  tags: ['foo']
  tasks:
    ...
```
**标记roles**
```
roles:
  - { role: webserver, port: 5000, tags: [ 'web', 'foo' ] }
```
**标记包含**
```
- include: foo.yml
  tags: [web,foo]
```
**始终运行标记的任务**
```
tasks:
  - debug: msg="Always runs"
    tags:
      - always
```

还有另外3个特殊关键字用于标签，'tagged'，'untagged'和'all'，它们分别是仅运行已标记，只有未标记和所有任务。

默认情况下ansible运行就像指定了'`-tags all`'。运行playbook中的未标记任务 -tags untagged

**显示playbook中的所有标记任务**
```
ansible-playbook example.yml --list-tags
```
**执行所有标记名称为packages的任务**
```
ansible-playbook example.yml --tags packages
```
**跳过所有标记名称为notification的任务**
```
ansible-playbook example.yml --skip-tags "notification"
```

## wait_for
---

等待端口可用,才能执行任务
```
- wait_for: port=8000 delay=10
```

等待直到锁定文件被删除
```
wait_for: path=/var/lock/file.lock state=absent
```

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
