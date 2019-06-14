---
layout: post
title: "Ansible 小手册系列 二十二（异步操作和轮询）"
date: "2017-06-22 22:05:31"
categories: Ansible
excerpt: "默认情况下playbook中的任务执行时会一直保持连接,直到该任务在每个节点都执行完毕.有时这是不必要的,比如有些操作运行时间比SSH超时时间还..."
auth: lework
---
* content
{:toc}

默认情况下playbook中的任务执行时会一直保持连接,直到该任务在每个节点都执行完毕.有时这是不必要的,比如有些操作运行时间比SSH超时时间还要长.

解决该问题最简单的方式是异步执行它们,然后轮询直到任务执行完毕.

你也可以对执行时间非常长（有可能遭遇超时）的操作使用异步模式.

## 指定任务为异步执行
---

为了异步启动一个任务,指定其async最大超时时间以及轮询其状态的频率.如果你没有为 poll 指定值,那么默认的轮询频率是`15`秒钟。pool设置为0时，任务会立即返回，而不等待命令执行的结果，继续执行下面的任务。

例如下面的palybook
```
# asynctest.yml
---

- hosts: node1
  tasks:
   - shell: sleep 100 && hostname
     async: 100
     poll: 0
     register: result

   - debug: var=result

   - async_status: jid={{ result.ansible_job_id }}
     register: job_result
     until: job_result.finished
     retries: 30
```

第一个任务，指定shell任务为异步执行，100秒后任务超时失败。
第二个任务，获取异步任务的返回值，其目的是获取jid。
第三个任务，检查jid异步任务的状态，当异步任务的finished不为0时，异步任务执行成功。检查次数为30次，间隔5秒。


使用ansible-playbook执行任务
```
# ansible-playbook asynctest.yml -vv
Using /etc/ansible/ansible.cfg as config file

PLAYBOOK: asynctest.yml ********************************************************************************************************
1 plays in asynctest.yml

PLAY [node1] *******************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [192.168.77.129]
META: ran handlers

TASK [command] *****************************************************************************************************************
task path: /etc/ansible/asynctest.yml:5
changed: [192.168.77.129] => {"ansible_job_id": "402058509351.34914", "changed": true, "finished": 0, "results_file": "/root/.ansible_async/402058509351.34914", "started": 1}

TASK [debug] *******************************************************************************************************************
task path: /etc/ansible/asynctest.yml:10
ok: [192.168.77.129] => {
    "changed": false, 
    "result": {
        "ansible_job_id": "402058509351.34914", 
        "changed": true, 
        "finished": 0, 
        "results_file": "/root/.ansible_async/402058509351.34914", 
        "started": 1
    }
}

TASK [async_status] ************************************************************************************************************
task path: /etc/ansible/asynctest.yml:12
FAILED - RETRYING: async_status (30 retries left).
FAILED - RETRYING: async_status (29 retries left).
FAILED - RETRYING: async_status (28 retries left).
FAILED - RETRYING: async_status (27 retries left).
FAILED - RETRYING: async_status (26 retries left).
FAILED - RETRYING: async_status (25 retries left).
FAILED - RETRYING: async_status (24 retries left).
FAILED - RETRYING: async_status (23 retries left).
FAILED - RETRYING: async_status (22 retries left).
FAILED - RETRYING: async_status (21 retries left).
FAILED - RETRYING: async_status (20 retries left).
FAILED - RETRYING: async_status (19 retries left).
FAILED - RETRYING: async_status (18 retries left).
FAILED - RETRYING: async_status (17 retries left).
FAILED - RETRYING: async_status (16 retries left).
FAILED - RETRYING: async_status (15 retries left).
FAILED - RETRYING: async_status (14 retries left).
FAILED - RETRYING: async_status (13 retries left).
changed: [192.168.77.129] => {"ansible_job_id": "402058509351.34914", "attempts": 19, "changed": true, "cmd": "sleep 100 && hostname", "delta": "0:01:40.005387", "end": "2017-06-21 17:07:31.015694", "finished": 1, "rc": 0, "start": "2017-06-21 17:05:51.010307", "stderr": "", "stderr_lines": [], "stdout": "node1", "stdout_lines": ["node1"]}
META: ran handlers
META: ran handlers

PLAY RECAP *********************************************************************************************************************
192.168.77.129             : ok=4    changed=2    unreachable=0    failed=0 
```

使用ansible来执行异步
```
[root@master ansible]# ansible node1 -B 3600 -P 0  -m yum -a "name=ansible" -vv
Using /etc/ansible/ansible.cfg as config file
META: ran handlers
192.168.77.129 | SUCCESS => {
    "ansible_job_id": "23974611070.37468", 
    "changed": true, 
    "finished": 0, 
    "results_file": "/root/.ansible_async/23974611070.37468", 
    "started": 1
}
META: ran handlers
META: ran handlers
```
参数说明
 `-B 3600` 启用异步，超时时间3600,
` -P 0 ` 轮询时间为0


使用async_status来获取异步状态信息

```
[root@master ansible]# ansible node1 -m async_status -a "jid=23974611070.37468"
192.168.77.129 | SUCCESS => {
    "ansible_job_id": "23974611070.37468", 
    "changed": false, 
    "finished": 1, 
    "msg": "", 
    "rc": 0, 
    "results": [
        "ansible-2.3.1.0-1.el6.noarch providing ansible is already installed"
    ]
}
```

> 注意：在使用`command`, `win_command`, `shell`, `win_shell`, `raw`模块时，默认是不会返回信息的，可以使用`-o`显示。

## 异步任务的状态文件
---

异步任务的状态文件以jid命名的方式存放在远端主机的用户目录下的.ansible_async目录，本次使用的是root连接远端，所以目录是/root/.ansible_async。

查看状态文件
```
[root@node1 ~]#  tail  /root/.ansible_async/402058509351.34914 
{"changed": true, "end": "2017-06-21 17:07:31.015694", "stdout": "node1", "cmd": "sleep 100 && hostname", "start": "2017-06-21 17:05:51.010307", "delta": "0:01:40.005387", "stderr": "", "rc": 0, "invocation": {"module_args": {"creates": null, "executable": null, "_uses_shell": true, "_raw_params": "sleep 100 && hostname", "removes": null, "warn": true, "chdir": null}}, "warnings": []}
```

## 异步任务的返回值
---

```
{
    "changed": false, 
    "result": {
        "ansible_job_id": "402058509351.34914", 
        "changed": true, 
        "finished": 0, 
        "results_file": "/root/.ansible_async/402058509351.34914", 
        "started": 1
    }
}
```

## 异步任务状态的返回值
---

可通过 async_status 获取，async_status 是读取远端的异步任务状态文件来获得任务状态，如果async设置的时间太短，可能会导致获取不到状态文件，因为还没生成这个文件。
例： ansible node1 -m async_status -a "jid=347674660626.31272"

执行中的返回值
```
{
    "ansible_job_id": "801846426045.31757",
    "changed": false,
    "finished": 0,
    "started": 1
}
```
执行后的返回值
```
{
    "ansible_job_id": "402058509351.34914",
    "attempts": 19,
    "changed": true,
    "cmd": "sleep 100 && hostname",
    "delta": "0:01:40.005387",
    "end": "2017-06-21 17:07:31.015694",
    "finished": 1,
    "rc": 0,
    "start": "2017-06-21 17:05:51.010307",
    "stderr": "",
    "stderr_lines": [],
    "stdout": "node1",
    "stdout_lines": [
        "node1"
    ]
}
```
