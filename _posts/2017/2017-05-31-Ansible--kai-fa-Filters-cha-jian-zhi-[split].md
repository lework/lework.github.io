---
layout: post
title: "Ansible 开发Filters插件之【split】"
date: "2017-05-31 22:36:14"
categories: Ansible
excerpt: "需求 实现python的字符串分割 实现re的正则表达式分割 Filter 类 所有的filter类都是上诉构造 filter_name  是 ..."
auth: lework
---
* content
{:toc}

## 需求
---

1. 实现python的字符串分割
2. 实现re的正则表达式分割
{% raw %}
## Filter 类
---
```
class FilterModule(object):
    ''' A filter to split a string into a list. '''
    def filters(self):
        return {
            'filter_name' : filter_function,
        }
```
所有的filter类都是上诉构造
- `filter_name`  是 Ansible Jinja2 filter的name。
- `filter_function(string)`  是处理字符的方法, 如{{ 'test' | filter_name }}， test字符串会传递给string。

## 实现代码
---
```
cat filter_plugins/split.py
import re

def split_string(string, seperator=None, maxsplit=-1):
    try:
        return string.split(seperator, maxsplit)
    except:
        return list(string)

def split_regex(string, seperator_pattern):
    try:
        return re.split(seperator_pattern, string)
    except:
        return list(string)

class FilterModule(object):
    ''' A filter to split a string into a list. '''
    def filters(self):
        return {
            'split' : split_string,
            'split_regex' : split_regex,
        }
```

## Playbook
---
```
- hosts: localhost
  gather_facts: no
  tasks:
    - debug: msg={{ 'a b c' | split }}
    - debug: msg={{ '1c2c3' | split('c') }}
    - debug: msg={{ 'dev.example.com' | split_regex('\.') }}
```
## 主机清单
```
localhost ansible_connection=local
```

## 运行playbook
![image.png](/assets/images/Ansible/3629406-54adbf9e627ff068.png)
{% endraw %}


