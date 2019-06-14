---
layout: post
title: "Ansible 小手册系列 十四（条件判断和循环）"
date: "2016-11-19 20:33:31"
categories: Ansible
excerpt: "条件判断 When 语句 在when 后面使用Jinja2 表达式，结果为True则执行任务。 若操作系统是Debian 时就执行关机操作 可以..."
auth: lework
---
* content
{:toc}
{% raw %}
## 条件判断
---

### When 语句

在when 后面使用Jinja2 表达式，结果为True则执行任务。
```
tasks:
  - name: "shut down Debian flavored systems"
    command: /sbin/shutdown -t now
    when: ansible_os_family == "Debian"
```
若操作系统是Debian 时就执行关机操作

可以对条件进行分组在比较。
```
tasks:
  - name: "shut down CentOS 6 and Debian 7 systems"
    command: /sbin/shutdown -t now
    when: (ansible_distribution == "CentOS" and ansible_distribution_major_version == "6") or
          (ansible_distribution == "Debian" and ansible_distribution_major_version == "7")
```

可以使用列表形式来表示条件为and的关系

```
tasks:
  - name: "shut down CentOS 6 systems"
    command: /sbin/shutdown -t now
    when:
      - ansible_distribution == "CentOS"
      - ansible_distribution_major_version == "6"
```

**使用jinja2过滤器**

```
tasks:
  - command: /bin/false
    register: result
    ignore_errors: True

  - command: /bin/something
    when: result|failed

  - command: /bin/something_else
    when: result|succeeded

  - command: /bin/still/something_else
    when: result|skipped
```
忽略一个语句的错误，然后决定基于成功或失败有条件地做一些事情。


**字符串转换为数字型再去比较**
```
tasks:
  - shell: echo "only on Red Hat 6, derivatives, and later"
    when: ansible_os_family == "RedHat" and ansible_lsb.major_release|int >= 6
```
**使用变量进行判断**
```
vars:
  epic: true
tasks:
    - shell: echo "This certainly is epic!"
      when: epic
tasks:
    - shell: echo "This certainly isn't epic!"
      when: not epic
```

**判断变量是否定义**
```
tasks:
    - shell: echo "I've got '{{ foo }}' and am not afraid to use it!"
      when: foo is defined

    - fail: msg="Bailing out. this play requires 'bar'"
      when: bar is undefined
```

**与循环一起使用**
```
tasks:
    - command: echo {{ item }}
      with_items: [ 0, 2, 4, 6, 8, 10 ]
      when: item > 5
```

**依次遍历列表，当列表里得数字大于5时执行任务**
```
- command: echo {{ item }}
  with_items: "{{ mylist|default([]) }}"
  when: item > 5
- command: echo {{ item.key }}
  with_dict: "{{ mydict|default({}) }}"
  when: item.value > 5
```
当变量不存在时，直接跳过


**使用自定义的facts值做判断**

```
tasks:
    - name: gather site specific fact data
      action: site_facts
    - command: /usr/bin/thingy
      when: my_custom_fact_just_retrieved_from_the_remote_system == '1234'
```
**角色包含使用when**
```
- include: tasks/sometasks.yml
  when: "'reticulating splines' in output"
- hosts: webservers
  roles:
     - { role: debian_stock_config, when: ansible_os_family == 'Debian' }
```
**基于变量选择文件和模板**
```
- name: template a file
  template: src={{ item }} dest=/etc/myapp/foo.conf
  with_first_found:
    - files:
       - {{ ansible_distribution }}.conf
       - default.conf
      paths:
       - search_location_one/somedir/
       - /opt/other_location/somedir/
```

**使用注册变量判断**

```
- name: test play
  hosts: all

  tasks:

      - shell: cat /etc/motd
        register: motd_contents

      - shell: echo "motd contains the word hi"
        when: motd_contents.stdout.find('hi') != -1
```
### failed_when

**满足条件时，使任务失败**

```
  tasks:
    - command: echo faild.
      register: command_result
      failed_when: "'faild' in command_result.stdout"
   - debug: msg="echo test"
```
还可以写成这样
```
  tasks:
    - command: echo faild.
      register: command_result
      ignore_errors: True

    - name: fail the echo
      fail: msg="the command failed"
      when: "'faild' in command_result.stdout"
    - debug: msg="echo test"
```

### changed_when

更改任务的状态。
```
- name: Install dependencies via Composer.
  command: "/usr/local/bin/composer global require phpunit/phpunit --prefer-dist"
  register: composer
  changed_when: "'Nothing to install or update' not in composer.stdout"
```
当使用PHP Composer作为安装项目依赖项的命令时，知道什么时候是有用的Composer安装了一些东西，或什么都没有改变。


## 循环
---

**标准循环**
添加多个用户
```
- name: add several users
  user: name={{ item }} state=present groups=wheel
  with_items:
     - testuser1
     - testuser2
```
添加多个用户，并将用户加入不同的组内。
```
- name: add several users
  user: name={{ item.name }} state=present groups={{ item.groups }}
  with_items:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```
**嵌套循环**

分别给用户授予3个数据库的所有权限
```
- name: give users access to multiple databases
  mysql_user: name={{ item[0] }} priv={{ item[1] }}.*:ALL append_privs=yes password=foo
  with_nested:
    - [ 'alice', 'bob' ]
    - [ 'clientdb', 'employeedb', 'providerdb' ]
```
**遍历字典**

输出用户的姓名和电话
```
 tasks:
  - name: Print phone records
    debug: msg="User {{ item.key }} is {{ item.value.name }} ({{ item.value.telephone }})"
    with_dict: {'alice':{'name':'Alice Appleworth', 'telephone':'123-456-789'},'bob':{'name':'Bob Bananarama', 'telephone':'987-654-3210'} }
```

**并行遍历列表**
```
  tasks:
  - debug: "msg={{ item.0 }} and {{ item.1 }}"
    with_together:
    - [ 'a', 'b', 'c', 'd','e' ]
    - [ 1, 2, 3, 4 ]
```
如果列表数目不匹配，用None补全

**遍历列表和索引**
```
  - name: indexed loop demo
    debug: "msg='at array position {{ item.0 }} there is a value {{ item.1 }}'"
    with_indexed_items: [1,2,3,4]
```
 item.0 为索引，item.1为值

**遍历文件列表的内容**
```
---
- hosts: all
  tasks:
       - debug: "msg={{ item }}"
      with_file:
        - first_example_file
        - second_example_file
```
**遍历目录文件**
with_fileglob匹配单个目录中的所有文件，非递归匹配模式。
```
---
- hosts: all
  tasks:
    - file: dest=/etc/fooapp state=directory
    - copy: src={{ item }} dest=/etc/fooapp/ owner=root mode=600
      with_fileglob:
        - /playbooks/files/fooapp/*
```
当在role中使用with_fileglob的相对路径时，Ansible解析相对于roles/<rolename>/files目录的路径。

**遍历ini文件**
```
lookup.ini
[section1]
value1=section1/value1
value2=section1/value2

[section2]
value1=section2/value1
value2=section2/value2
```
```
- debug: msg="{{ item }}"
  with_ini: value[1-2] section=section1 file=lookup.ini re=true
```
获取section1 里的value1和value2的值

**重试循环 until**
```
- action: shell /usr/bin/foo
  register: result
  until: result.stdout.find("all systems go") != -1
  retries: 5
  delay: 10
```

"重试次数retries" 的默认值为3，"delay"为5。

**查找第一个匹配文件**
```
  tasks:
  - debug: "msg={{ item }}"
    with_first_found:
     - "/tmp/a"
     - "/tmp/b"
     - "/tmp/default.conf"
```
依次寻找列表中的文件，找到就返回。如果列表中的文件都找不到，任务会报错。


**随机选择with_random_choice**

随机选择列表中得一个值
```
- debug: msg={{ item }}
  with_random_choice:
     - "go through the door"
     - "drink from the goblet"
     - "press the red button"
     - "do nothing"
```


**循环程序的结果**
```
  tasks:
  - debug: "msg={{ item }}"
    with_lines: ps aux 
```


**循环子元素**

定义好变量
```
#varfile
---
users:
  - name: alice
    authorized:
      - /tmp/alice/onekey.pub
      - /tmp/alice/twokey.pub
    mysql:
        password: mysql-password
        hosts:
          - "%"
          - "127.0.0.1"
          - "::1"
          - "localhost"
        privs:
          - "*.*:SELECT"
          - "DB1.*:ALL"
  - name: bob
    authorized:
      - /tmp/bob/id_rsa.pub
    mysql:
        password: other-mysql-password
        hosts:
          - "db1"
        privs:
          - "*.*:SELECT"
          - "DB2.*:ALL"
```
```
---
- hosts: web
  vars_files: varfile
  tasks:

  - user: name={{ item.name }} state=present generate_ssh_key=yes
    with_items: "{{ users }}"

  - authorized_key: "user={{ item.0.name }} key='{{ lookup('file', item.1) }}'"
    with_subelements:
      - "{{ users }}"
      - authorized

  - name: Setup MySQL users
    mysql_user: name={{ item.0.name }} password={{ item.0.mysql.password }} host={{ item.1 }} priv={{ item.0.mysql.privs | join('/') }}
    with_subelements:
      - "{{ users }}"
      - mysql.hosts
```

{{ lookup('file', item.1) }}  是查看item.1文件的内容
  with_subelements 遍历哈希列表，然后遍历列表中的给定（嵌套）的键。

**在序列中循环with_sequence**

with_sequence以递增的数字顺序生成项序列。 您可以指定开始，结束和可选步骤值。
参数应在key = value对中指定。 'format'是一个printf风格字符串。

数字值可以以十进制，十六进制（0x3f8）或八进制（0600）指定。 不支持负数。

```
---
- hosts: all

  tasks:

    # 创建组
    - group: name=evens state=present
    - group: name=odds state=present

    # 创建格式为testuser%02x 的0-32 序列的用户
    - user: name={{ item }} state=present groups=evens
      with_sequence: start=0 end=32 format=testuser%02x

    # 创建4-16之间得偶数命名的文件
    - file: dest=/var/stuff/{{ item }} state=directory
      with_sequence: start=4 end=16 stride=2

    # 简单实用序列的方法：创建4 个用户组分表是组group1 group2 group3 group4
    - group: name=group{{ item }} state=present
      with_sequence: count=4
```

**随机选择with_random_choice**
随机选择列表中得一个值
```
- debug: msg={{ item }}
  with_random_choice:
     - "go through the door"
     - "drink from the goblet"
     - "press the red button"
     - "do nothing"
```

**合并列表**
```
# 安装所有列表中的软件
- name: flattened loop demo
  yum: name={{ item }} state=installed
  with_flattened:
     - [ 'foo-package', 'bar-package' ]
     - [ ['one-package', 'two-package' ]]
     - [ ['red-package'], ['blue-package']]
```

**注册变量使用循环**
```
- shell: echo "{{ item }}"
  with_items:
    - one
    - two
  register: echo

- name: Fail if return code is not 0
  fail:
    msg: "The command ({{ item.cmd }}) did not have a 0 return code"
  when: item.rc != 0
  with_items: "{{ echo.results }}"
```


**循环主机清单**

```
# 输出所有主机清单里的主机
- debug: msg={{ item }}
  with_items: "{{ groups['all'] }}"
# 输出所有执行的主机
- debug: msg={{ item }}
  with_items: play_hosts

#输出所有主机清单里的主机
- debug: msg={{ item }}
  with_inventory_hostnames: all

# 输出主机清单中不在www中的所有主机
- debug: msg={{ item }}
  with_inventory_hostnames: all:!www
```

**改变循环的变量项**
```
# main.yml
- include: inner.yml
  with_items:
    - 1
    - 2
    - 3
  loop_control:
    loop_var: outer_item
# inner.yml
- debug: msg="outer item={{ outer_item }} inner item={{ item }}"
  with_items:
    - a
    - b
    - c
```

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
{% endraw %}