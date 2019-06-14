---
layout: post
title: "Ansible 开发模块 之【模块说明】"
date: "2017-05-28 16:51:39"
categories: Ansible
excerpt: "在开发模块之前，现问下自己几个问题？ 官方是否有提供的类似功能模块？可从下面两个连接确定官方提供的模块，以免重复造轮子官方已发布的模块 http..."
auth: lework
---
* content
{:toc}

### 在开发模块之前，现问下自己几个问题？
1. 官方是否有提供的类似功能模块？
        可从下面两个连接确定官方提供的模块，以免重复造轮子
	官方已发布的模块 http://docs.ansible.com/ansible/modules.html
	官方正在开发的模块 https://github.com/ansible/ansible/labels/module
2. 你需要开发一个action 插件么？
	action插件是在ansible主机上运行，而不是在目标主机上运行的。对于类似file/copy/template功能的模块，在模块执行前需要在ansible主机上做一些操作的。有关action插件的开发请移步到

### 明确几点
	- 模块是传送到目标主机上运行的。
	- 模块的返回值必须是json dumps的字符串。

## 执行模块的过程

![image.png](/assets/images/Ansible/3629406-74fb47e88468c3dd.png)

首先，将模块文件读入内存，然后添加传递给模块的参数，最后将模块中所需要的类添加到内存，由zipfile压缩后，再由base64进行编码，写入到模版文件内。

通过默认的连接方式，一般是ssh。ansible通过ssh连接到远程主机，创建临时目录，并关闭连接。然后将打开另外一个ssh连接，将模版文件以sftp方式传送到刚刚创建的临时目录中，写完后关闭连接。然后打开一个ssh连接将任务对象赋予可执行权限，执行成功后关闭连接。

最后，ansible将打开第三个连接来执行模块，并删除临时目录及其所有内容。模块的结果是从标准输出stdout中获取json格式的字符串。ansible将解析和处理此字符串。如果有任务是异步控制执行的，ansible将在模块完成之前关闭第三个连接，并且返回主机后，在规定的时间内检查任务状态，直到模块完成或规定的时间超时。

使用了**管道连接**后，与远程主机只有一个连接，命令通过数据流的方式发送执行。

配置方式
```
vim /etc/ansible/ansible.cfg
pipelining = True
```
执行过程
![image.png](/assets/images/Ansible/3629406-4e456a746bc6eb4c.png)



## 模块工具
----

Ansible提供了许多模块实用程序，它们提供了在开发自己的模块时可以使用的辅助功能。 basic.py模块为程序提供访问Ansible库的主要入口点，所有Ansible模块必须至少从basic.py导入：
`from ansible.module_utils.basic import *`

其他模块工具
a10.py - Utilities used by the a10_server module to manage A10 Networks devices.
api.py - Adds shared support for generic API modules.
aos.py - Module support utilities for managing Apstra AOS Server.
asa.py - Module support utilities for managing Cisco ASA network devices.
azure_rm_common.py - Definitions and utilities for Microsoft Azure Resource Manager template deployments.
basic.py - General definitions and helper utilities for Ansible modules.
cloudstack.py - Utilities for CloudStack modules.
database.py - Miscellaneous helper functions for PostGRES and MySQL
docker_common.py - Definitions and helper utilities for modules working with Docker.
ec2.py - Definitions and utilities for modules working with Amazon EC2
eos.py - Helper functions for modules working with EOS networking devices.
f5.py - Helper functions for modules working with F5 networking devices.
facts.py - Helper functions for modules that return facts.
gce.py - Definitions and helper functions for modules that work with Google Compute Engine resources.
ios.py - Definitions and helper functions for modules that manage Cisco IOS networking devices
iosxr.py - Definitions and helper functions for modules that manage Cisco IOS-XR networking devices
ismount.py - Contains single helper function that fixes os.path.ismount
junos.py - Definitions and helper functions for modules that manage Junos networking devices
known_hosts.py - utilities for working with known_hosts file
mysql.py - Allows modules to connect to a MySQL instance
netapp.py - Functions and utilities for modules that work with the NetApp storage platforms.
netcfg.py - Configuration utility functions for use by networking modules
netcmd.py - Defines commands and comparison operators for use in networking modules
network.py - Functions for running commands on networking devices
nxos.py - Contains definitions and helper functions specific to Cisco NXOS networking devices
openstack.py - Utilities for modules that work with Openstack instances.
openswitch.py - Definitions and helper functions for modules that manage OpenSwitch devices
powershell.ps1 - Utilities for working with Microsoft Windows clients
pycompat24.py - Exception workaround for Python 2.4.
rax.py - Definitions and helper functions for modules that work with Rackspace resources.
redhat.py - Functions for modules that manage Red Hat Network registration and subscriptions
service.py - Contains utilities to enable modules to work with Linux services (placeholder, not in use).
shell.py - Functions to allow modules to create shells and work with shell commands
six/__init__.py - Bundled copy of the Six Python library to aid in writing code compatible with both Python 2 and Python 3.
splitter.py - String splitting and manipulation utilities for working with Jinja2 templates
urls.py - Utilities for working with http and https requests
vca.py - Contains utilities for modules that work with VMware vCloud Air
vmware.py - Contains utilities for modules that work with VMware vSphere VMs
vyos.py - Definitions and functions for working with VyOS networking
