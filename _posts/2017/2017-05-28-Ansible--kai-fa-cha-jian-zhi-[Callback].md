---
layout: post
title: "Ansible 开发插件之【Callback】"
date: "2017-05-28 16:49:17"
categories: Ansible
excerpt: "callback 是经常用到的插件，而且还是自定义很强的，在任务的每个状态下执行某些动作。 触发事件的列表 可以定义的状态如下，本次不考虑使用v..."
auth: lework
---
* content
{:toc}

callback 是经常用到的插件，而且还是自定义很强的，在任务的每个状态下执行某些动作。

## 触发事件的列表
---

可以定义的状态如下，本次不考虑使用v1的方法
def v2_on_any(self, *args, **kwargs):
def v2_runner_on_failed(self, result, ignore_errors=False):  任务失败的时候
def v2_runner_on_ok(self, result):    任务成功的时候
def v2_runner_on_skipped(self, result): 任务跳过的时候 
def v2_runner_on_unreachable(self, result):
def v2_runner_on_no_hosts(self, task):
def v2_runner_on_async_poll(self, result):
def v2_runner_on_async_ok(self, result):
def v2_runner_on_async_failed(self, result):
def v2_runner_on_file_diff(self, result, diff):
def v2_playbook_on_start(self, playbook):
def v2_playbook_on_notify(self, result, handler):
def v2_playbook_on_no_hosts_matched(self):
def v2_playbook_on_no_hosts_remaining(self):
def v2_playbook_on_task_start(self, task, is_conditional):
def v2_playbook_on_cleanup_task_start(self, task):
def v2_playbook_on_handler_task_start(self, task):
def v2_playbook_on_vars_prompt(self, varname, private=True, prompt=None, encrypt=None, confirm=False, salt_size=None, salt=None, default=None):
def v2_playbook_on_setup(self):  playbook 在执行setup操作的时候执行
def v2_playbook_on_import_for_host(self, result, imported_file):
def v2_playbook_on_not_import_for_host(self, result, missing_file):
def v2_playbook_on_play_start(self, play): 
def v2_playbook_on_stats(self, stats):
def v2_on_file_diff(self, result):
def v2_playbook_on_include(self, included_file):
def v2_runner_item_on_ok(self, result):
def v2_runner_item_on_failed(self, result):
def v2_runner_item_on_skipped(self, result):
def v2_runner_retry(self, result):


## 方法的result值
---

print(result._task) 输出任务名
print(result._check_key)
print(result._host) 输出主机名
print(result._result) 输出任务执行的结果
print(result.is_changed)
print(result.is_failed)


## callback class都是继承ansible.plugins.callback.CallbackBase类，而作为一个新类存在的。
---
```
from ansible.plugins.callback import CallbackBase

class CallbackModule(CallbackBase):
    pass
```

## 定义callback说明信息
---

CALLBACK_VERSION = 2.0   插件版本
CALLBACK_TYPE = 'aggregate'  插件类型，如果是'stdout'时，只会加载一个这样的回调插件
CALLBACK_NAME = 'timer'   插件名称，需与文件名称一致。
CALLBACK_NEEDS_WHITELIST = True 插件是否需要在配置文件配置whitelist。为true是，ansible检查ansible.cfg文件中的callback_whitelist是否有插件名称，有则执行，无则跳过。
