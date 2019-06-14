---
layout: post
title: "Ansible 小手册系列 十六（Playbook Debug）"
date: "2016-11-19 20:44:50"
categories: Ansible
excerpt: "debug模块在执行期间打印语句，并且可用于调试变量或表达式，而不必停止playbook。 打印自定义的信息 调试变量 playbook 开启d..."
auth: lework
---
* content
{:toc}

debug模块在执行期间打印语句，并且可用于调试变量或表达式，而不必停止playbook。

## 打印自定义的信息
---
```
- debug: msg="System {{ inventory_hostname }} has uuid {{ ansible_product_uuid }}"
```
## 调试变量
---

```
- debug: var=result verbosity=2
```


## playbook 开启debug模式
---

在2.1中，我们添加了一个调试策略。 此策略使您能够在任务失败时调用调试器。 您可以访问失败任务上下文中调试器的所有功能。 然后，您可以例如检查或设置变量的值，更新模块参数，并使用新的变量和参数重新运行失败的任务，以帮助解决失败的原因。

```
- hosts: test
  strategy: debug
  gather_facts: no
  vars:
    var1: value1
  tasks:
    - name: wrong variable
      ping: data={{ wrong_var }}
```

以上playbook，在执行到错误的任务时，会进入debug模式下

![Paste_Image.png](/assets/images/Ansible/3629406-45674d3d11cfa20d.png)


**可用命令**

|命令|说明|
|:---|:---|
|p	|显示此次失败的原因|
|p task	|显示此次任务的名称|
|p task.args	|显示模块的参数|
|p host	|显示执行此次任务的主机|
|p result	|显示此次任务的结果|
|p vars	|显示当前的变量|
|vars[key] = value	|更新vars中的值|
|task.args[key] = value	|更新模块的参数。|
|r	|再次执行此任务|
|c	|继续执行|
|q	|退出debug模式|

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
