---
layout: post
title: "Ansible 开发插件之【动态主机清单】"
date: "2017-05-28 16:46:40"
categories: Ansible
excerpt: "ansible 可以从动态源（包括云资源）中获取主机清单。我们只需要创建一个脚本或程序，可以在正确的参数输入时已正确的格式打印json字符串，就..."
auth: lework
---
* content
{:toc}

>ansible 可以从动态源（包括云资源）中获取主机清单。
我们只需要创建一个脚本或程序，可以在正确的参数输入时已正确的格式打印json字符串，就能自定义一个插件。你可以用任何语言实现。


## 脚本约定
---

当我们向脚本输入--list参数时，脚本必须将要管理的所有组以json编码的形式输出到标准输出stdout。每个组的值应该是包含每个主机/ip的列表以及定义的变量。下面给出一个简单示例
```
{
    "databases": {
        "hosts": ["host1.example.com", "host2.example.com"],
        "vars": {
            "a": true
        }
    },
    "webservers": ["host2.example.com", "host3.example.com"],
    "atlanta": {
        "hosts": ["host1.example.com", "host4.example.com", "host5.example.com"],
        "vars": {
            "b": false
        },
        "children": ["marietta", "5points"]
    },
    "marietta": ["host6.example.com"],
    "5points": ["host7.example.com"]
}
```
当我们向脚本输入 --host <hostname>参数时，脚本必须输出一个空的json字符串或一个变量的列表/字典，以便temlates和playbook可以使用。输出变量是可选的，如果脚本不希望输出，那输出一个空的列表/字典也是可以的。下面一个简单的示例
```
{
    "favcolor": "red",
    "ntpserver": "wolf.example.com",
    "monitoring": "pack.example.com"
}
```

定义hostvars的主机变量，下面一个简单的示例
```
{

    # results of inventory script as above go here
    # ...

    "_meta": {
        "hostvars": {
            "moocow.example.com": {
                "asdf" : 1234
            },
            "llama.example.com": {
                "asdf": 5678
            }
        }
    }
}
```

## 例子
---
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
                'hosts': ['192.168.77.129', '192.168.77.130'],
                'vars': {
                    'ansible_ssh_user': 'root',
                    'ansible_ssh_pass': '123456',
                    'example_variable': 'value'
                }
            },
            '_meta': {
                'hostvars': {
                    '192.168.77.129': {
                        'host_specific_var': 'foo'
                    },
                    '192.168.77.130': {
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

## 使用
---

赋予插件可执行权限
```
chmod+x test_inventory.py
```
--list信息
![image.png](/assets/images/Ansible/3629406-ac37393f591fe3dd.png)
--host信息
![image.png](/assets/images/Ansible/3629406-aea8cef51bdd5542.png)
执行
![image.png](/assets/images/Ansible/3629406-02050b63b128bf4a.png)
查看变量
![image.png](/assets/images/Ansible/3629406-7003229cbe9eaadc.png)
