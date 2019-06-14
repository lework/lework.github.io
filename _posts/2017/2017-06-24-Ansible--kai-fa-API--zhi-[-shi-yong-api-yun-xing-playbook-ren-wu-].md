---
layout: post
title: "Ansible 开发API 之【使用api运行playbook任务】"
date: "2017-06-24 15:14:25"
categories: Ansible
excerpt: "环境 本文使用的是ansible api 2.0ansible 2.3.0.0os Centos 6.7 X64python 2.6.6 本文介..."
auth: lework
---
* content
{:toc}

## 环境
---

本文使用的是`ansible api 2.0`
ansible `2.3.0.0`
os `Centos 6.7 X64`
python `2.6.6`

本文介绍 api 运行一个playbook项目

## playbook文件
---

```
~]# cat /etc/ansible/test.yml
---

- hosts: node1
  gather_facts: no

  tasks:
    - command: echo 123
      notify: test1
  handlers:
    - name: test1
      debug: msg="456"
```

## 主机清单
---

```
~]# cat /etc/ansible/hosts
[node1]
192.168.77.129 ansible_ssh_pass=123456
```
## API
---
```
~]# cat test_playbook.py 
#!/bin/env python
#! coding: utf-8

import os
import sys
from collections import namedtuple
from ansible.parsing.dataloader import DataLoader
from ansible.vars import VariableManager
from ansible.inventory import Inventory
from ansible.executor.playbook_executor import PlaybookExecutor

# 在指定文件时，不能使用列表指定多个。
host_path = '/etc/ansible/hosts'
if not os.path.exists(host_path):
  print '[INFO] The [%s] inventory does not exist' % host_path
  sys.exit()

loader = DataLoader()
variable_manager = VariableManager()
inventory = Inventory(loader=loader, variable_manager=variable_manager,host_list=host_path)
variable_manager.set_inventory(inventory)
passwords=None

# 初始化需要的对象
Options = namedtuple('Options',
                     ['connection',
                      'remote_user',
                      'ask_sudo_pass',
                      'verbosity',
                      'ack_pass', 
                      'module_path', 
                      'forks', 
                      'become', 
                      'become_method', 
                      'become_user', 
                      'check',
                      'listhosts', 
                      'listtasks', 
                      'listtags',
                      'syntax',
                      'sudo_user',
                      'sudo'])
# 初始化需要的对象
options = Options(connection='smart', 
                       remote_user='root',
                       ack_pass=None,
                       sudo_user='root',
                       forks=5,
                       sudo='yes',
                       ask_sudo_pass=False,
                       verbosity=5,
                       module_path=None,  
                       become=True, 
                       become_method='sudo', 
                       become_user='root', 
                       check=None,
                       listhosts=None,
                       listtasks=None, 
                       listtags=None, 
                       syntax=None)

# 多个yaml文件则以列表形式
playbook_path=['/etc/ansible/test.yml']
for i in playbook_path:
  if not os.path.exists(i):
    print '[INFO] The [%s] playbook does not exist' % i
    sys.exit()


playbook = PlaybookExecutor(playbooks=playbook_path,inventory=inventory,
              variable_manager=variable_manager,
              loader=loader,options=options,passwords=passwords)

# 执行playbook
result = playbook.run()

# 返回执行结果
print '执行结果: %s' % result
```

playbook_executor 是不能自定义 callback的，可通过环境变量或者ansible.cfg中配置callback。所以输出是标准输出，result 是返回码，0表示全部运行成功

## 输出
---
```
~]# python test_playbook.py

PLAY [node1] ***********************************************************************************************************

TASK [command] *********************************************************************************************************
changed: [192.168.77.129]

RUNNING HANDLER [test1] ************************************************************************************************
ok: [192.168.77.129] => {
    "changed": false, 
    "msg": "456"
}

PLAY RECAP *************************************************************************************************************
192.168.77.129                  : ok=2    changed=1    unreachable=0    failed=0   

执行结果: 0
```
