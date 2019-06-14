---
layout: post
title: "Ansible 开发模块 之【构建一个简单的module】"
date: "2017-05-28 16:51:19"
categories: Ansible
excerpt: "需求 该模块的目的是在远程主机上将源文件复制到目标文件 步骤 将这个模块放在library目录下，命名为remote_copy可以使用 ANSI..."
auth: lework
---
* content
{:toc}

## 需求
---

该模块的目的是在远程主机上将源文件复制到目标文件

## 步骤
---

1. 将这个模块放在library目录下，命名为remote_copy
可以使用 ANSIBLE_LIBRARY环境变量来指定模块的存放位置。也可以在playbook当前目录下创建library目录。
```
[root@master library]# pwd
/etc/ansible/library
[root@master library]# ls
remote_copy.py
```
2. 在文件头部加入下列语句，表示该模块使用python运行
```
#!/usr/bin/python
#
```
3. 导入我们所需要的库，shutil模块
```
import shutil
```
4. 创建模块的入口，并使用AnsibleModule类中的argument_spec来接受参数
```
def main():
  module = AnsibleModule(
	argument_spec = dict(
	source=dict(required=True, type='str'),
	dest=dict(required=True, type='str')
  )
)
```
在我们的模块中，我们提供了两个参数，第一个参数是`source`，用来定义源文件;第二个参数是`dest`，用来定义目的地。这两个参数被标记为必须定义的。类型是字符串。

5. 使用shutil.copy拷贝文件
```
shutil.copy(module.params['source'], module.params['dest'])
```
6. 拷贝完成后，返回json数据
```
module.exit_json(changed=True)
```
此命令将退出模块执行

7. 在文件头部导入我们所需的AnsibleModule
```
from ansible.module_utils.basic import AnsibleModule
```
8. 最后，告诉程序使用main()来执行模块
```
if __name__ == '__main__':
	main()
```
9. 现在我们的模块是可以运行的了。下面将使用一个playbook来测试我们的remote_copy
```
---

- name: test remote_copy module
  hosts: node1
  gather_facts: false
  
  tasks:
   - name: ensure foo
	 file: path=/tmp/foo state=touch

   - name: do a remote copy
	 remote_copy: source=/tmp/foo dest=/tmp/bar
```
这里我们使用了一个任务，来使源文件一直存在。

![image.png](/assets/images/Ansible/3629406-5d0b77bcaef5ee6a.png)

## module 提供 fact 数据
---

与作为模块退出的一部分返回的数据类似，模块也可以直接创建fact数据，通过名为ansible_facts键返回数据到主机。无需为任务register一个变量来获取数值。
```
remote_facts = {'rc_source': module.params['source'], 'rc_dest': module.params['dest'] }
module.exit_json(changed=True, ansible_facts=remote_facts)
```
在playbook中添加查看fact的任务
```
   - name: show a fact
	 debug: var=rc_dest
```
运行playbook查看返回的数据

![image.png](/assets/images/Ansible/3629406-fd71b55d8a6b2644.png)

## 检查模式
---

模块可以选择支持检查模式<http://docs.ansible.com/ansible/playbooks_checkmode.html>。 如果用户在检查模式下运行可执行安全性，则模块应该尝试预测和报告是否发生更改，但实际上不会进行任何更改（不支持检查模式的模块也不会采取任何动作，但只是不会报告其可能的更改）。

对于您的模块支持检查模式，您必须在实例化AnsibleModule对象时传递supports_check_mode = True。 当启用检查模式时，AnsibleModule.check_mode属性将计算为True。 例如：
```
module = AnsibleModule(
  argument_spec = dict(
  source=dict(required=True, type='path'),
  dest=dict(required=True, type='path')
),
  supports_check_mode=True
)

if not module.check_mode:
  shutil.copy(module.params['source'], module.params['dest'])
```
在进行检查模式的时候，不执行拷贝动作，看下列运行状态。
![image.png](/assets/images/Ansible/3629406-64bb87858d4c8c28.png)
