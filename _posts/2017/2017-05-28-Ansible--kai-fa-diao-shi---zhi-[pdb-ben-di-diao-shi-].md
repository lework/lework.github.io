---
layout: post
title: "Ansible 开发调试 之【pdb本地调试】"
date: "2017-05-28 16:52:41"
categories: Ansible
excerpt: "Ansible是用Python编写的，用于调试本地代码执行的工具是Python调试器 **pdb**。这个工具允许我们在Ansys中插入断点代码..."
auth: lework
---
* content
{:toc}

Ansible是用Python编写的，用于调试本地代码执行的工具是Python调试器 `**pdb**`。这个工具允许我们在Ansys中插入断点代码和交互式地逐行执行代码。


## 加入调试代码
---
找到ansible的运行文件
[root@master ~]# whereis ansible
ansible: /usr/bin/ansible /etc/ansible /usr/share/man/man1/ansible.1.gz

在/usr/bin/ansible文件开始处加入
```
import pdb; pdb.set_trace()
```
![image.png](/assets/images/Ansible/3629406-b0b525ac4e500a1c.png)

然后运行ansible命令，就进入了pdb调试状态

![image.png](/assets/images/Ansible/3629406-64a513c332105060.png)

## pdb常用命令
---

输入help 查看帮助信息

![image.png](/assets/images/Ansible/3629406-72a2a81d02f51fe1.png)


输入`n(next)`，让程序运行下一行，如果当前语句有一个函数调用，用n是不会进入被调用的函数体中的 

![image.png](/assets/images/Ansible/3629406-bf3b07f4c7ba4b36.png)


输入`l(list)`，查看代码片段

![image.png](/assets/images/Ansible/3629406-b68ce0a0191f0133.png)

输入`p(pp)` 查看变量的值

![image.png](/assets/images/Ansible/3629406-2a959c03bd9fdd75.png)

输入`s(step into)` 跟`n`相似，但是如果当前有一个函数调用，那么`s`会进入被调用的函数体中 

![image.png](/assets/images/Ansible/3629406-53e5565ec48ac117.png)


输入`r(return)` 执行代码直到从当前函数返回


![image.png](/assets/images/Ansible/3629406-a67b32137c61260c.png)

输入`b` 设置断点，例如 “b 78”，就是在当前脚本的78行打上断点，还能输入函数名作为参数，断点就打到具体的函数入口，如果只敲b，会显示现有的全部断点 


![image.png](/assets/images/Ansible/3629406-0c630bdde0584872.png)

输入`a(rgs)`，打印当前函数的参数 


![image.png](/assets/images/Ansible/3629406-e9db5daddcb74fef.png)

输入`bt(w)` 打印堆栈跟踪，最下面的帧位于底部。箭头表示“当前帧”，它决定了大多数命令的上下文。

![image.png](/assets/images/Ansible/3629406-18ebaf5655141cd4.png)


输入`c (continue)` 停止 debug 继续执行程序,直到下一个断点。


![image.png](/assets/images/Ansible/3629406-e8d2742ad65a41fa.png)

输入`q(uit)`，退出调试 


> 有时候我们只是想调试某一部分代码，如果从命令入口开始调试则很难找到相应的代码，这样就需要在某一模块的入口上添加调试点就可以了。

**调试`inventory`代码**
`inventory`负责发现，解析，加载主机清单。
`inventory`的主要入口在inventory/__init__.py文件中的Inventory类，这个文件在linux下通常存在/usr/lib/python2.6/site-packages/ansible/inventory/__init__.py，如果不知道自己的ansible路径在哪里，可以使用python -c "import ansible; print(ansible)"命令查看。
下面是在__init__()方法中添加了断点

![image.png](/assets/images/Ansible/3629406-1cead75cc48d8744.png)



**调试`Playbook`代码**
`Playbook`负责加载，解析，执行yml文件的playbook。

palybook的主要入口在playbook/__init__.py文件的Playbook类，这个文件在linux下通常存在/usr/lib/python2.6/site-packages/ansible/playbook/__init__.py.

下面是在__init__()方法中添加了断点
![image.png](/assets/images/Ansible/3629406-ed1cf6739a408ff2.png)

其他的功能模块调试方法类似。
