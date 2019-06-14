---
layout: post
title: "Ansible 开发API 之【使用api运行task,自定义主机信息】"
date: "2017-06-24 15:20:38"
categories: Ansible
excerpt: "在做api开发的时候，我们获取主机清单的主机信息的方式，往往是从数据库中获取到的，而不是从文件中获取，下面的列子给出怎么自定义主机信息。 环境 ..."
auth: lework
---
* content
{:toc}

在做api开发的时候，我们获取主机清单的主机信息的方式，往往是从数据库中获取到的，而不是从文件中获取，下面的列子给出怎么自定义主机信息。

## 环境
---

本文使用的是`ansible api 2.0`
ansible `2.3.0.0`
os `Centos 6.7 X64`
python `2.6.6`

## API
---
```
~]# cat  test_task_inventory.py 
#!/bin/env python
#! coding: utf-8

import json
import shutil
import ansible.constants as C
from collections import namedtuple
from ansible.parsing.dataloader import DataLoader
from ansible.vars import VariableManager
from ansible.inventory import Inventory
from ansible.playbook.play import Play
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.task_result import TaskResult
from ansible.plugins.callback import CallbackBase
from ansible.inventory.host import Host


class ResultCallback(CallbackBase):
    def v2_runner_on_ok(self, result, **kwargs):
        host = result._host
        print u"%s 执行结果" % result._task
        print json.dumps({host.name: result._result}, indent=4)
        print "-----------------------------------------------"

# 定义主机列表
host_lists=['192.168.77.129']

variable_manager = VariableManager()
loader = DataLoader()

Options = namedtuple('Options', ['ssh_extra_args','ssh_common_args', 'connection', 'module_path', 'forks', 'become', 'become_method', 'become_user', 'check'])
options = Options(ssh_extra_args='', ssh_common_args='', connection='smart', module_path=None, forks=5, become=None, become_method=None, become_user=None, check=False)
passwords = dict(conn_pass='123456', become_pass='123456')

results_callback = ResultCallback()

inventory = Inventory(loader=loader, variable_manager=variable_manager, host_list=host_lists)
variable_manager.set_inventory(inventory)

# 定义ansible主机
host_info = Host(name='192.168.77.129', port=22 )
# 设置主机的用户名和密码
variable_manager.set_host_variable(host_info, 'ansible_ssh_user', 'root')
variable_manager.set_host_variable(host_info, 'ansible_ssh_pass', '123456')

# 定义匹配host_list中的主机的正则表达式,本次采用精确匹配。
host_pattern= '192.168.77.129'

play_source =  dict(
        name = "Ansible Ad-Hoc", 
        hosts = host_pattern,
        gather_facts = 'no',
        tasks = [
            dict(action=dict(module='shell', args='whoami'), register='shell_out',async=0,poll=15),
            dict(action=dict(module='debug', args=dict(msg='{{shell_out.stdout}}')),async=0,poll=15)
         ]
    )

play = Play().load(play_source, variable_manager=variable_manager, loader=loader)

tqm = None
try:
    tqm = TaskQueueManager(
              inventory=inventory,
              variable_manager=variable_manager,
              loader=loader,
              options=options,
              passwords=passwords,
              stdout_callback=results_callback
          )
    result = tqm.run(play)
    print u'任务执行返回码: %s' % result
finally:
    if tqm is not None:
        tqm.cleanup()
    if loader:
        loader.cleanup_all_tmp_files()
    shutil.rmtree(C.DEFAULT_LOCAL_TMP, True)
```

## 执行结果
---
```
~]# python test_task_inventory.py 
TASK: command 执行结果
{
    "192.168.77.129": {
        "_ansible_parsed": true, 
        "stderr_lines": [], 
        "cmd": "whoami", 
        "end": "2017-06-23 16:06:44.909307", 
        "_ansible_no_log": false, 
        "stdout": "root", 
        "changed": true, 
        "rc": 0, 
        "start": "2017-06-23 16:06:44.906586", 
        "stderr": "", 
        "delta": "0:00:00.002721", 
        "invocation": {
            "module_args": {
                "warn": true, 
                "executable": null, 
                "_uses_shell": true, 
                "_raw_params": "whoami", 
                "removes": null, 
                "creates": null, 
                "chdir": null
            }
        }, 
        "stdout_lines": [
            "root"
        ]
    }
}
-----------------------------------------------
TASK: debug 执行结果
{
    "192.168.77.129": {
        "msg": "root", 
        "changed": false, 
        "_ansible_verbose_always": true, 
        "_ansible_no_log": false
    }
}
-----------------------------------------------
任务执行返回码: 0
```

同理，运行playbook也像上面设置的一样。
