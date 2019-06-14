---
layout: post
title: "Ansible 开发Callback插件之【BlackHole】"
date: "2018-01-05 16:45:07"
categories: Ansible
excerpt: "需求 执行playbook时，什么都不显示。把回显内容丢到黑洞去。 注： 程序的警告信息还是会出现，针对ad-hoc命令无效。 callback..."
auth: lework
---
* content
{:toc}

## 需求
---
执行playbook时，什么都不显示。把回显内容丢到黑洞去。

> 注： 程序的警告信息还是会出现，针对ad-hoc命令无效。

## callback plugin
---
```
callback_plugins/black_hole.py

from __future__ import (absolute_import, division, print_function)
__metaclass__ = type

from ansible.plugins.callback.default import CallbackModule as CallbackModule_default

class CallbackModule(CallbackModule_default):

    '''
    Magic black hole, nothing will show up.
    '''

    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'stdout'
    CALLBACK_NAME = 'black_hole'

    def on_any(self, *args, **kwargs):
        pass
    def runner_on_failed(self, host, res, ignore_errors=False):
        pass
    def runner_on_ok(self, host, res):
        pass
    def runner_on_skipped(self, host, item=None):
        pass
    def runner_on_unreachable(self, host, res):
        pass
    def runner_on_no_hosts(self):
        pass
    def runner_on_async_poll(self, host, res, jid, clock):
        pass
    def runner_on_async_ok(self, host, res, jid):
        pass
    def runner_on_async_failed(self, host, res, jid):
        pass
    def playbook_on_start(self):
        pass
    def playbook_on_notify(self, host, handler):
        pass
    def playbook_on_no_hosts_matched(self):
        pass
    def playbook_on_no_hosts_remaining(self):
        pass
    def playbook_on_task_start(self, name, is_conditional):
        pass
    def playbook_on_vars_prompt(self, varname, private=True, prompt=None, encrypt=None, confirm=False, salt_size=None, salt=None, default=None):
        pass
    def playbook_on_setup(self):
        pass
    def playbook_on_import_for_host(self, host, imported_file):
        pass
    def playbook_on_not_import_for_host(self, host, missing_file):
        pass
    def playbook_on_play_start(self, name):
        pass
    def playbook_on_stats(self, stats):
        pass
    def on_file_diff(self, host, diff):
        pass
    def v2_on_any(self, *args, **kwargs):
        pass
    def v2_runner_on_failed(self, result, ignore_errors=False):
        pass
    def v2_runner_on_ok(self, result):
        pass
    def v2_runner_on_skipped(self, result):
        pass
    def v2_runner_on_unreachable(self, result):
        pass
    def v2_runner_on_no_hosts(self, task):
        pass
    def v2_runner_on_async_poll(self, result):
        pass
    def v2_runner_on_async_ok(self, result):
        pass
    def v2_runner_on_async_failed(self, result):
        pass
    def v2_runner_on_file_diff(self, result, diff):
        pass
    def v2_playbook_on_start(self, playbook):
        pass
    def v2_playbook_on_notify(self, result, handler):
        pass
    def v2_playbook_on_no_hosts_matched(self):
        pass
    def v2_playbook_on_no_hosts_remaining(self):
        pass
    def v2_playbook_on_task_start(self, task, is_conditional):
        pass
    def v2_playbook_on_cleanup_task_start(self, task):
        pass
    def v2_playbook_on_handler_task_start(self, task):
        pass
    def v2_playbook_on_vars_prompt(self, varname, private=True, prompt=None, encrypt=None, confirm=False, salt_size=None, salt=None, default=None):
        pass
    def v2_playbook_on_setup(self):
        pass
    def v2_playbook_on_import_for_host(self, result, imported_file):
        pass
    def v2_playbook_on_not_import_for_host(self, result, missing_file):
        pass
    def v2_playbook_on_play_start(self, play):
        pass
    def v2_playbook_on_stats(self, stats):
        pass
    def v2_on_file_diff(self, result):
        pass
    def v2_playbook_on_include(self, included_file):
        pass
    def v2_runner_item_on_ok(self, result):
        pass
    def v2_runner_item_on_failed(self, result):
        pass
    def v2_runner_item_on_skipped(self, result):
        pass
    def v2_runner_retry(self, result):
        pass
```

文件保存在playbook的当前目录callback_plugins目录下。

## 配置ansible.cfg
---
```
# change the default callback
stdout_callback = black_hole
# enable additional callbacks
callback_whitelist = black_hole
```
## 目录结构
---
```
[root@manager ansible]# pwd
/etc/ansible
[root@manager ansible]# tree
.
├── ansible.cfg
├── callback_plugins
│   ├── black_hole.py
│   └── black_hole.pyc
├── hosts
├── roles
└── test.yml
```

## 执行效果
---
```
[root@manager ansible]# cat test.yml 
---

- hosts: localhost

  tasks:
   - debug: msg="hello"

[root@manager ansible]# ansible-playbook test.yml 
 [WARNING]: provided hosts list is empty, only localhost is available

[root@manager ansible]#
```
是不是很神奇呀！
