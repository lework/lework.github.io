---
layout: post
title: "Ansible 小手册系列 十八（Lookup 插件）"
date: "2016-11-19 21:07:33"
categories: Ansible
excerpt: "file：获取文件内容 password：生成密码字符串 如果文件已存在，则不会向其写入任何数据。 如果文件有内容，那些内容将作为密码读入。 空..."
auth: lework
---
* content
{:toc}
{% raw %}
## file：获取文件内容
---
```
---
- hosts: all
  vars:
     contents: "{{ lookup('file', '/etc/foo.txt') }}"
tasks:
- debug: msg="the value of foo.txt is {{ contents }}"
```

## password：生成密码字符串
---

```
---
- hosts: all
tasks:
# 使用只有ascii字母且长度为8的随机密码创建一个mysql用户:
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', '/tmp/passwordfile chars=ascii_letters length=8') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
# 使用只有数字的随机密码创建一个mysql用户:
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', '/tmp/passwordfile chars=digits') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
# 使用许多不同的字符集使用随机密码创建一个mysql用户：
    - mysql_user: name={{ client }}
                  password="{{ lookup('password', '/tmp/passwordfile chars=ascii_letters,digits,hexdigits,punctuation') }}"
                  priv={{ client }}_{{ tier }}_{{ role }}.*:ALL
```

如果文件已存在，则不会向其写入任何数据。 如果文件有内容，那些内容将作为密码读入。 空文件导致密码以空字符串返回。

## csvfile ：读取csv文件
---

```
f.csv 
Symbol,Atomic Number,Atomic Mass
H,1,1.008
He,2,4.0026
Li,3,6.94
Be,4,9.012
B,5,10.81
```
```
- debug: msg="The atomic number of Lithium is {{ lookup('csvfile', 'Li file=elements.csv delimiter=,') }}"
- debug: msg="The atomic mass of Lithium is {{ lookup('csvfile', 'Li file=elements.csv delimiter=, col=2') }}"

```
|参数|	默认值	|描述|
|:----|:----|:----|
|file	|ansible.csv	|要加载的文件名称|
|col|	1	|要输出的列，索引从0开始|
|delimiter	|TAB	|文件的分隔符|
|default	|empty string	|如果key不在csv文件中，则为默认返回值|
|encoding	|utf-8	|使用的CSV文件的编码（字符集）(added in version 2.1)|

## ini ：读取ini文件
---

在section下查找以key1 = value1的格式来读取文件的内容。

```
users.ini
[production]
# My production information
user=robert
pass=somerandompassword

[integration]
# My integration information
user=gertrude
pass=anotherpassword
```
```
  tasks:
  - debug: msg="User in integration is {{ lookup('ini', 'user section=integration file=users.ini') }}"
  - debug: msg="User in production  is {{ lookup('ini', 'pass section=production  file=users.ini') }}"
```
ini 参数格式
```
lookup('ini', 'key [type=<properties|ini>] [section=section] [file=file.ini] [re=true] [default=<defaultvalue>]')
```
第一个值必须是ini文件里的key


|字段	|默认值|	描述|
|:----|:----|:----|
|type|	ini	|文件类型。 可以是ini或properties （对于javaproperties ）。|
|file	|ansible.ini|	要加载的文件名称|
|section|	global	|在哪里查找key|
|re	|False|	开启正则匹配|
|default	|empty string|	如果key不在文件中，则为默认返回值|


## Credstash ：用于使用AWS的KMS和DynamoDB管理secrets 
---

此模块依赖credstash库

## dig：dns查询
---

此模块依赖dnspython 库
```
- debug: msg="The IPv4 address for example.com. is {{ lookup('dig', 'example.com.')}}"
- debug: msg="The TXT record for example.org. is {{ lookup('dig', 'example.org.', 'qtype=TXT') }}"
- debug: msg="The TXT record for example.org. is {{ lookup('dig', 'example.org./TXT') }}"
```

## 其他
---

```
tasks:
- debug: msg="{{ lookup('env','HOME') }} is an environment variable"
- debug: msg="{{ lookup('pipe','date') }} is the raw result of running this command"
# redis_kv lookup requires the Python redis package
     - debug: msg="{{ lookup('redis_kv', 'redis://localhost:6379,somekey') }} is value in Redis for somekey"
# dnstxt lookup requires the Python dnspython package
     - debug: msg="{{ lookup('dnstxt', 'example.com') }} is a DNS TXT record for example.com"
- debug: msg="{{ lookup('template', './some_template.j2') }} is a value from evaluation of this template"
# loading a json file from a template as a string
     - debug: msg="{{ lookup('template', './some_json.json.j2', convert_data=False) }} is a value from evaluation of this template"
- debug: msg="{{ lookup('etcd', 'foo') }} is a value from a locally running etcd"
# shelvefile lookup retrieves a string value corresponding to a key inside a Python shelve file
     - debug: msg="{{ lookup('shelvefile', 'file=path_to_some_shelve_file.db key=key_to_retrieve') }}
```

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
{% endraw %}