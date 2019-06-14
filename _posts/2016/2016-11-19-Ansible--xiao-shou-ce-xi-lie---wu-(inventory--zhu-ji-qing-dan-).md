---
layout: post
title: "Ansible 小手册系列 五（inventory 主机清单）"
date: "2016-11-19 16:10:03"
categories: Ansible
excerpt: "Ansible 可同时操作属于一个组的多台主机,组和主机之间的关系通过 inventory 文件配置. 默认的文件路径为 /etc/ansibl..."
auth: lework
---
* content
{:toc}

Ansible 可同时操作属于一个组的多台主机,组和主机之间的关系通过 inventory 文件配置. 默认的文件路径为 `/etc/ansible/hosts`

## 主机清单示例
---

```
mail.example.com # FQDN

[webservers] # 方括号[]中是组名
host1
host2:5522  # 指定连接主机得端口号
localhost ansible_connection=local # 定义连接类型
host3 http_port=80 maxRequestsPerChild=808 # 定义主机变量
host4 ansible_ssh_port=5555 ansible_ssh_host=192.168.1.50 # 定义主机ssh连接端口和连接地址
www[1:50].example.com # 定义 1-50范围内的主机
www-[a:f].example.com # 定义a-f范围内内的主机

[dbservers]
three.example.com     ansible_python_interpreter=/usr/local/bin/python #定义python执行文件
192.168.77.123     ruby_module_host  ansible_ruby_interpreter=/usr/bin/ruby.1.9.3 # 定义ruby执行文件
 
[webservers:vars]  # 定义webservers组的变量
ntp_server= ntp.example.com
proxy=proxy.example.com


[server:children] # 定义server组的子成员
webservers
dbservers

[server:vars] # 定义server组的变量
zabbix_server:192.168.77.121
```

## Inventory 参数的说明
---

主机连接：

|参数|说明|
|:---|:---|
|ansible_connection	|与主机的连接类型.比如:local, ssh 或者 paramiko. Ansible 1.2 以前默认使用 paramiko.1.2 以后默认使用 'smart','smart' 方式会根据是否支持 ControlPersist, 来判断'ssh' 方式是否可行.|

ssh连接参数：

|参数|说明|
|:---|:---|
|ansible_ssh_host|	将要连接的远程主机名.与你想要设定的主机的别名不同的话,可通过此变量设置.|
|ansible_ssh_port	|ssh端口号.如果不是默认的端口号,通过此变量设置.|
|ansible_ssh_user	|默认的 ssh 用户名|
|ansible_ssh_pass|	ssh 密码(这种方式并不安全,我们强烈建议使用 --ask-pass 或 SSH 密钥)|
|ansible_ssh_private_key_file	|ssh 使用的私钥文件.适用于有多个密钥,而你不想使用 SSH 代理的情况.|
|ansible_ssh_common_args	|此设置附加到sftp，scp和ssh的缺省命令行|
|ansible_sftp_extra_args	|此设置附加到默认sftp命令行。|
|ansible_scp_extra_args	|此设置附加到默认scp命令行。|
|ansible_ssh_extra_args|	此设置附加到默认ssh命令行。|
|ansible_ssh_pipelining	|确定是否使用SSH管道。 这可以覆盖ansible.cfg中得设置。|


远程主机环境参数：

|参数|说明|
|:---|:---|
|ansible_shell_type	|目标系统的shell类型.默认情况下,命令的执行使用 'sh' 语法,可设置为 'csh' 或 'fish'.|
|ansible_python_interpreter	|目标主机的 python 路径.适用于的情况: 系统中有多个 Python, 或者命令路径不是"/usr/bin/python",比如  \*BSD, 或者 /usr/bin/python|
|ansible_*_interpreter	|这里的"*"可以是ruby 或perl 或其他语言的解释器，作用和ansible_python_interpreter 类似|
|ansible_shell_executable	|这将设置ansible控制器将在目标机器上使用的shell，覆盖ansible.cfg中的配置，默认为/bin/sh。 |


## 动态主机清单示例
---

`inventory.py`
```
#!/usr/bin/env python

'''
Example custom dynamic inventory script for Ansible, in Python.
'''

import os
import sys
import argparse

try:
    import json
except ImportError:
    import simplejson as json

class ExampleInventory(object):

    def __init__(self):
        self.inventory = {}
        self.read_cli_args()

        # Called with `--list`.
        if self.args.list:
            self.inventory = self.example_inventory()
        # Called with `--host [hostname]`.
        elif self.args.host:
            # Not implemented, since we return _meta info `--list`.
            self.inventory = self.empty_inventory()
        # If no groups or vars are present, return empty inventory.
        else:
            self.inventory = self.empty_inventory()

        print json.dumps(self.inventory);

    # Example inventory for testing.
    def example_inventory(self):
        return {
            'group': {
                'hosts': ['192.168.28.71', '192.168.28.72'],
                'vars': {
                    'ansible_ssh_user': 'vagrant',
                    'ansible_ssh_private_key_file':
                        '~/.vagrant.d/insecure_private_key',
                    'example_variable': 'value'
                }
            },
            '_meta': {
                'hostvars': {
                    '192.168.28.71': {
                        'host_specific_var': 'foo'
                    },
                    '192.168.28.72': {
                        'host_specific_var': 'bar'
                    }
                }
            }
        }

    # Empty inventory for testing.
    def empty_inventory(self):
        return {'_meta': {'hostvars': {}}}

    # Read the command line args passed to the script.
    def read_cli_args(self):
        parser = argparse.ArgumentParser()
        parser.add_argument('--list', action = 'store_true')
        parser.add_argument('--host', action = 'store')
        self.args = parser.parse_args()

# Get the inventory.
ExampleInventory()
```
```
$ ./inventory.py --list
{"group": {"hosts": ["192.168.28.71", "192.168.28.72"], "vars":{"ansible_ssh_user": 
"vagrant","ansible_ssh_private_key_file":"~/.vagrant.d/insecure_private_key", "example_variable": "value"}}, 
"_meta": {"hostvars": {"192.168.28.72": {"host_specific_var": "bar"}, "192.168.28.71": {"host_specific_var": "foo"}}}}

$ ansible all -i inventory.py -m ping
$ ansible all -i inventory.py -m debug -a "var=host_specific_var"

```

动态主机清单见 [Ansible 开发插件之【动态主机清单】](http://www.jianshu.com/p/706c98215c02)。

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
