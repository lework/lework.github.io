---
layout: post
title: "Ansible 开发API 之【使用api运行task任务】"
date: "2017-06-24 15:09:53"
categories: Ansible
excerpt: "ansible 依赖于子进程，因此API不是线程安全的。 环境 本文使用的是ansible api 2.0ansible 2.3.0.0os C..."
auth: lework
---
* content
{:toc}

> ansible 依赖于子进程，因此API不是线程安全的。

## 环境
---

本文使用的是`ansible api 2.0`
ansible `2.3.0.0`
os `Centos 6.7 X64`
python `2.6.6`

下面是一个执行task的例子

## 主机清单
---
task任务执行的主机
```
~]# cat /etc/ansible/hosts
[node1]
192.168.77.129 ansible_ssh_pass=123456
```

## api
---
```
~]# cat test.py 
#!/bin/env python
#! coding: utf-8

import json
from collections import namedtuple
from ansible.parsing.dataloader import DataLoader
from ansible.vars import VariableManager
from ansible.inventory import Inventory
from ansible.playbook.play import Play
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.task_result import TaskResult
from ansible.plugins.callback import CallbackBase

# 自定义 callback，即在运行 api 后调用本类中的 v2_runner_on_ok()，在这里会输出 host 和 result 格式
class ResultCallback(CallbackBase):
    def v2_runner_on_ok(self, result, **kwargs):
        # result 包含 '_check_key', '_host', '_result', '_task', '_task_fields', 'is_changed', 'is_failed', 'is_skipped', 'is_unreachable', 'task_name'
        host = result._host
        print u"%s 执行结果" % result._task
        print json.dumps({host.name: result._result}, indent=4)
        print "-----------------------------------------------"

# 初始话需要的类
variable_manager = VariableManager()
# 用来管理变量，包括主机、组、扩展等变量，该类在之前的Inventory内置

loader = DataLoader()
# 用来加载解析yaml文件或JSON内容,并且支持vault的解密

# 定义选项 
Options = namedtuple('Options', ['connection', 'module_path', 'forks', 'become', 'become_method', 'become_user', 'check'])

# 定义连接远端的额方式为smart
options = Options(connection='smart', module_path=None, forks=100, become=None, become_method=None, become_user=None, check=False)

# 定义默认的密码连接，主机未定义密码的时候才生效，conn_pass指连接远端的密码，become_pass指提升权限的密码
passwords = dict(conn_pass='123456', become_pass='123456')

# Instantiate our ResultCallback for handling results as they come in
# 结果回调类实例化
results_callback = ResultCallback()

# create inventory and pass to var manager
# 创建inventory、并带进去参数
inventory = Inventory(loader=loader, variable_manager=variable_manager, host_list='/etc/ansible/hosts')  
# hosts文件，也可以是 ip列表 '10.1.162.18:322, 10.1.167.36' 或者 ['10.1.162.18:322', '10.1.167.36']，如果不设置，默认取ansible.cfg配置文件中的inventory值，默认为/etc/ansible/hosts.
variable_manager.set_inventory(inventory)
# 把inventory传递给variable_manager管理

# create play with tasks
# 创建要执行play的内容并引入上面的变量
play_source =  dict(
        name = "Ansible Play", 
        hosts = 'node1',   # 匹配host_list中的主机的正则表达式
        gather_facts = 'no',
	# 定义任务列表
        tasks = [
            dict(action=dict(module='shell', args='hostname'), register='shell_out',async=0,poll=15),
            dict(action=dict(module='debug', args=dict(msg='{{shell_out.stdout}}')),async=0,poll=15)
         ]
    )

play = Play().load(play_source, variable_manager=variable_manager, loader=loader)

# actually run it
# TaskQueueManager 是创建进程池，负责输出结果和多进程间数据结构或者队列的共享协作
tqm = None
try:
    tqm = TaskQueueManager(
              inventory=inventory,
              variable_manager=variable_manager,
              loader=loader,
              options=options,
              passwords=passwords,
              stdout_callback=results_callback,  # Use our custom callback instead of the ``default`` callback plugin
              # 如果注释掉 callback 则会调用原生的 DEFAULT_STDOUT_CALLBACK，输出 task result的output，同 ansible-playbook debug
          )
    result = tqm.run(play)
    print u'任务执行返回码: %s' % result  # 返回码，只要有一个 host 出错就会返回 非0 数字
finally:
    if tqm is not None:
        tqm.cleanup()
    if loader:
        print 'end'    
        loader.cleanup_all_tmp_files()
    shutil.rmtree(C.DEFAULT_LOCAL_TMP, True)
```

##执行结果
---
```
~]# python test.py 
TASK: command 执行结果
{
    "192.168.77.129": {
        "_ansible_parsed": true, 
        "stderr_lines": [], 
        "cmd": "hostname", 
        "end": "2017-06-22 17:52:18.111433", 
        "_ansible_no_log": false, 
        "stdout": "node1", 
        "changed": true, 
        "rc": 0, 
        "start": "2017-06-22 17:52:18.109322", 
        "stderr": "", 
        "delta": "0:00:00.002111", 
        "invocation": {
            "module_args": {
                "warn": true, 
                "executable": null, 
                "_uses_shell": true, 
                "_raw_params": "hostname", 
                "removes": null, 
                "creates": null, 
                "chdir": null
            }
        }, 
        "stdout_lines": [
            "node1"
        ]
    }
}
-----------------------------------------------
TASK: debug 执行结果
{
    "192.168.77.129": {
        "msg": "node1", 
        "changed": false, 
        "_ansible_verbose_always": true, 
        "_ansible_no_log": false
    }
}
-----------------------------------------------
任务执行返回码: 0
```

## TaskQueueManager 字段说明
---

inventory --> 由ansible.inventory模块创建，用于导入inventory文件
variable_manager --> 由ansible.vars模块创建，用于存储各类变量信息
loader --> 由ansible.parsing.dataloader模块创建，用于数据解析
options --> 存放各类配置信息的数据字典
passwords --> 登录密码，可设置加密信息
stdout_callback --> 回调函数

## TaskQueueManager.run方法的返回状态码
 ---
    RUN_OK                = 0    
    RUN_ERROR             = 1  
    RUN_FAILED_HOSTS      = 2
    RUN_UNREACHABLE_HOSTS = 4
    RUN_FAILED_BREAK_PLAY = 8
    RUN_UNKNOWN_ERROR     = 255
