---
layout: post
title: "Ansible 开发调试 之【模块调试】"
date: "2017-05-28 16:52:07"
categories: Ansible
excerpt: "本地调试 需要安装jinja2 库 使用官方提供的测试脚本调试 使下列命令调试modules test-module使用参数 远程调试 在前面的..."
auth: lework
---
* content
{:toc}

## 本地调试
---

需要安装jinja2 库
```
yum -y install python-jinja2
```

使用官方提供的测试脚本调试
```
git clone git://github.com/ansible/ansible.git
source ansible/hacking/env-setup
cd ansible/hacking/
```

使下列命令调试modules
```
python test-module -m /usr/lib/python2.6/site-packages/ansible/modules/system/ping.py
```
![image.png](/assets/images/Ansible/3629406-7effbb75d94d244a.png)

test-module使用参数
![image.png](/assets/images/Ansible/3629406-da9731851cd8c13f.png)


## 远程调试
----

在前面的介绍中，我们知道**modules是在远程主机上执行的**，调试模块那就需要在远程主机上执行，ansible默认在执行完modules，会自动清理在远程主机上的临时文件。
使用`ANSIBLE_KEEP_REMOTE_FILES=1`环境变量 ，可以保留ansible在远程主机的执行文件，从而在远程主机上调试模块。
```
$ ANSIBLE_KEEP_REMOTE_FILES=1 ansible localhost -m ping -a 'data=debugging_session' -vvv
sing module file /usr/lib/python2.6/site-packages/ansible/modules/core/system/ping.py
<localhost> ESTABLISH LOCAL CONNECTION FOR USER: root
<localhost> EXEC /bin/sh -c '( umask 77 && mkdir -p "` echo ~/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932 `" && echo ansible-tmp-1489477306.61-275734926719932="` echo ~/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932 `" ) && sleep 0'
<localhost> PUT /tmp/tmpv4EenK TO /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py
<localhost> EXEC /bin/sh -c 'chmod u+x /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py && sleep 0'
<localhost> EXEC /bin/sh -c '/usr/bin/python /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py && sleep 0'
localhost | SUCCESS => {
	"changed": false, 
	"invocation": {
		"module_args": {
			"data": "debugging_session"
		}, 
		"module_name": "ping"
	}, 
	"ping": "debugging_session"
}
```
模块文件是由base64编码的字符串文件，可使用`explode`将字符串转换成可执行的python文件
```
 $ python /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py explode
Module expanded into:
/root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/debug_dir
```
查看debug_dir目录
```
 $ tree  /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/debug_dir/
/root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/debug_dir/
├── ansible
│   ├── __init__.py
│   └── module_utils
│       ├── basic.py
│       ├── __init__.py
│       ├── pycompat24.py
│       ├── six.py
│       └── _text.py
├── ansible_module_ping.py
└── args
```

- ansible_module_ping.py 是模块本身的代码。
- args 文件包含一个JSON字符串。 该字符串是一个包含模块参数和其他变量的字典。
- ansible目录包含由ansible_module_ping模块使用的ansible.module_utils的代码文件。

修改了debug_dir文件中的代码之后，需要使用`execute`执行代码
```
$ python /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py execute
{"invocation": {"module_args": {"data": "debugging_session"}}, "changed": false, "ping": "debugging_session"}
```
