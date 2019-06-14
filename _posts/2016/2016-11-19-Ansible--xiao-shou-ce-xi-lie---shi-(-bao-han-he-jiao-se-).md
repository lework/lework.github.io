---
layout: post
title: "Ansible 小手册系列 十（包含和角色）"
date: "2016-11-19 17:29:16"
categories: Ansible
excerpt: "包含 使用include模块来包含foo文件 include 还允许传递变量 动态包含 循环引用3次 还可以使用动态变量引入task文件 动态包..."
auth: lework
---
* content
{:toc}

## 包含
---

使用`include`模块来包含foo文件

```
tasks:
  - include: foo.yml

--- foo.yml
- name: test foo
  command: echo foo
```

`include` 还允许传递变量
```
- include: wordpress.yml wp_user=timmy
- include: wordpress.yml
  vars:
      wp_user: timmy
      ssh_keys:
        - keys/one.txt
        - keys/two.txt
```

## 动态包含
---

循环引用3次
```
- include: foo.yml param={{item}}
  with_items:
  - 1
  - 2
  - 3
```

还可以使用动态变量引入task文件

```
- include: "{{inventory_hostname}}.yml"
```

**动态包含的一些限制**

• 您不能使用notify来触发来自动态包含的处理程序名称。
• 您不能使用--start-at-task在动态包含内的任务开始执行。
• 仅存在于动态包含内的标记不会显示在-list-tags输出中。
• 只存在于动态包含内的任务将不会显示在-list-tasks输出中。

为了解决上面限制，2.1版本后引入了`static`

```
- include: foo.yml
  static: <yes|no|true|false>
```
默认情况下，在Ansible 2.1及更高版本中，include包含符合以下条件时会自动被视为静态而不是动态：

* include不使用任何循环
* 包含的文件名不使用任何变量
* 静态选项没有显式禁用（即static：no）
* 强制静态包含（见下文）的ansible.cfg选项被禁用

ansible.cfg配置中有两个选项可用于静态包括：

* `task_includes_static` - 将所有在tasks部分中包含的内容都设置为静态。
* `handler_includes_static` - 强制所有包括在处理程序部分是静态的。

这些选项允许用户强制playbook的行为与他们在1.9.x和之前一样。

## 变量包含
---

`include_vars` 在task中动态加载yaml或json文件类型中的变量

```
- include_vars: myvars.yml
```

根据操作系统类型加载变量文件，如果找不到，则为默认值。

```
- include_vars: "{{ item }}"
  with_first_found:
   - "{{ ansible_distribution }}.yml"
   - "{{ ansible_os_family }}.yml"
   - "default.yml"
```

## 角色 ROLE
---

角色是基于已知文件结构自动加载某些vars_files，任务和处理程序的方法。 按角色分组内容还允许轻松与其他用户共享角色。

**文件结构如下**

![Paste_Image.png](/assets/images/Ansible/3629406-2f960202a9b3b8d1.png)



**结构说明**

- site.yml   主要的playbook
- webservers.yml   webservers 得playbook
- hosts.ini     主机清单
- ibrary          如果有任何自定义模块，将其放在这里（可选）
- filter_plugins  如果有任何自定义过滤器插件，将其放在这里（可选）
- 如果group_var/all存在，其中列出的变量将被添加到所有的主机组中
- 如果group_var/groupname1存在，其中列出的变量将被添加到groupname1主机组中- 
- 如果host_vars/hostname1存在，其中列出的变量将被添加到hostname1主机组中

这个 playbook 为一个角色 ‘x’ 指定了如下的行为

- 如果 roles/x/tasks/main.yml 存在, 其中列出的 tasks 将被添加到 play 中
- 如果 roles/x/handlers/main.yml 存在, 其中列出的 handlers 将被添加到 play 中
- 如果 roles/x/vars/main.yml 存在, 其中列出的 variables 将被添加到 play 中
- 如果  roles/ x/defaults /main.yml存在，其中列出的变量将被添加到play 中
- 如果 roles/x/meta/main.yml 存在, 其中列出的 “角色依赖” 将被添加到 roles 列表中
- 所有 copy tasks 可以引用 roles/x/files/ 中的文件，不需要指明文件的路径。
- 所有 script tasks 可以引用 roles/x/files/ 中的脚本，不需要指明文件的路径。
- 所有 template tasks 可以引用 roles/x/templates/ 中的文件，不需要指明文件的路径。
- 所有 include tasks 可以引用 roles/x/tasks/ 中的文件，不需要指明文件的路径。


> 如果 roles 目录下有文件不存在，这些文件将被忽略。

这些目录的加载顺序
  1.  meta/main.yml
  1. tasks/main.yml
  1. handlers/main.yml
  1. vars/main.yml
  1. defaults/main.yml

**如果区分环境使用角色，可以使用下列文档结构**

![Paste_Image.png](/assets/images/Ansible/3629406-56284ed25795ae17.png)


- production/inventory 主机清单

** 运行playbook**

ansible-playbook -i production site.yml

## 角色定义
---

在playbook中定义角色

```
---
- hosts: webservers
  roles:
     - x
```

定义角色参数

```
---
- hosts: webservers
  roles:
    - { role: x, dir: '/opt/a',  app_port: 5000 }
```

使用条件判断，当主机是Redhat时，才执行角色任务
```
---
- hosts: webservers
  roles:
    - { role: some_role, when: "ansible_os_family == 'RedHat'" }
```

定义角色标签

```
---
- hosts: webservers
  roles:
    - { role: x, tags: ["bar", "baz"] }
```
定义执行角色任务前后执行的动作
```
---
- hosts: webservers

  pre_tasks:
    - shell: echo 'hello'

  roles:
    - { role: some_role }

  tasks:
    - shell: echo 'still busy'

  post_tasks:
    - shell: echo 'goodbye'
```

>pre_tasks里的task在roles执行前执行的任务
post_tasks里的task在roles执行完成后执行的任务

**角色默认变量**

定义角色默认变量，只需在角色目录中添加一个defaults/main.yml文件。角色默认变量优先级最低。

## 角色依赖
---

角色依赖性允许您在使用角色时自动提取其他角色。 角色依赖关系存储在角色目录中包含的meta/main.yml文件中。 此文件应包含要在指定角色之前插入的角色和参数的列表，例如角色/myapp/meta/main.yml中的以下内容：

```
---
dependencies:
  - { role: common, some_parameter: 3 }
  - { role: apache, apache_port: 80 }
  - { role: postgres, dbname: blarg, other_parameter: 12 }
  - { role: '/path/to/common/roles/foo', x: 1 }
```

** 角色依赖执行顺序**

角色依赖性始终在包含角色的角色之前执行，并且是递归的。
如上面内容，按照common，apache，postgres，'/path/to/common/roles/foo顺序依次执行，再执行此角色


** 角色依赖嵌套**

默认情况下，在添加依赖其他角色的时候，如果其他角色内也有依赖关系，是不执行其他角色内的依赖关系的。

可以通过`allow_duplicates: yes`设置来实现执行其他角色内的依赖关系。

实际测试，`2.1`，`2.2`版本的ansible，无论加不加这个设置，就只会执行第一次依赖角色，后续的则不执行。

![Paste_Image.png](/assets/images/Ansible/3629406-bd5b4079152d1077.png)

## 例子
---

ansible examples ：[https://github.com/ansible/ansible-examples](https://github.com/ansible/ansible-examples)


---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
