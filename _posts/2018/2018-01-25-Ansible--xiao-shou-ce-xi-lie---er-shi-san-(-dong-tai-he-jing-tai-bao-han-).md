---
layout: post
title: "Ansible 小手册系列 二十三（动态和静态包含）"
date: "2018-01-25 15:35:37"
categories: Ansible
excerpt: "Ansible有两种可重用内容的操作模式：动态和静态。在Ansible 2.0阶段使用static来设置操作模式，Ansible 2.4则引入了..."
auth: lework
---
* content
{:toc}

> Ansible有两种可重用内容的操作模式：动态和静态。
在Ansible 2.0阶段使用static来设置操作模式，Ansible 2.4则引入了include和import的概念。

如果您使用`import*`包含Task（`import_playbook`，`import_tasks`等），它将是静态的。 
如果您使用`include*`包含Task（`include_tasks`，`include_role`等），它将是动态的。

> 使用include包含Task（用于task文件和Playbook级包括）仍然可用，但现在被认为已被弃用。

## 静态和动态之间的差异
---

Ansible预处理Playbook解析期间的所有静态导入,而动态包含是在运行期间遇到该任务时处理的。

当涉及Ansible task选项，如tags和when：

对于静态导入，父任务选项将被复制到import中包含的所有子任务。
对于动态包含，任务选项仅在评估时应用于动态任务，不会被复制到子任务。


## 优缺点
---
使用`include*`语句的主要优点是循环。当循环与`include*`一起使用时，包含的任务或角色将为循环中的每个项目执行一次。

与`import*`语句相比，使用`include*`有一些限制：

- 仅存在于动态包含内的标签不会显示在-list-tags输出中。
- 仅存在于动态包含内的任务不会显示在-list-tasks输出中。
- 您不能使用notify来触发来自动态包含内部的处理程序名称。
- 您不能使用--start-at-task开始执行动态包含内的任务。

与动态相比，使用`import*`也可能有一些限制：

- 如上所述，循环不能用于导入。
- 当使用目标文件或角色名称的变量时，不能使用来自库存源（主机/组变量等）的变量。


总而言之，没有使用with的包含，就使用import，使用了with，那就用include。
