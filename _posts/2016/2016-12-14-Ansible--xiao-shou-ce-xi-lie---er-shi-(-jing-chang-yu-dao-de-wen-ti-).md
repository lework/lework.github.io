---
layout: post
title: "Ansible 小手册系列 二十（经常遇到的问题）"
date: "2016-12-14 10:21:59"
categories: Ansible
excerpt: "(1). 怎么为任务设置环境变量？ (2). 不同的用户登录不同的主机？ 在主机清单里设置 也可以指定连接类型 (3). 通过跳转主机访问无法访..."
auth: lework
---
* content
{:toc}
{% raw %}
(1). 怎么为任务设置环境变量？

```yml
- name: set environment
  shell: echo $PATH $SOME >> /tmp/a.txt
  environment:
    PATH: "{{ ansible_env.PATH }}:/thingy/bin"
    SOME: value
```

(2). 不同的用户登录不同的主机？

在主机清单里设置
```yml
[webservers`]
asdf.example.com  ansible_port=5000   ansible_user=alice  ansible_pass=123456
jkl.example.com   ansible_port=5001   ansible_user=bob   ansible_pass=654321
```
也可以指定连接类型
```yml
[testcluster]
localhost           ansible_connection=local
/path/to/chroot1    ansible_connection=chroot
foo.example.com
bar.example.com
```	
(3). 通过跳转主机访问无法访问的主机
```yml
	ansible_ssh_common_args: '-o ProxyCommand="ssh -W %h:%p -q user@gateway.example.com"'
	ansible_ssh_common_args: '-o ProxyCommand="sshpass -f /etc/tpasswd ssh xx@10.10.10.1 -p 66677 nc %h %p"' 
```

(4). 关闭cowsay功能

```bash
export ANSIBLE_NOCOWS=1
```	

(5). 关闭ssh在首次连接时出现检查keys 的提示
```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```	
(6). 查看主机名的所有清单变量？
```bash
ansible -m debug -a "var=hostvars['hostname']" localhost
```	
(7). 通过拼接字符串，来获取接口ip地址
```yml
{{ hostvars[inventory_hostname]['ansible_' + which_interface]['ipv4']['address'] }}
```
(8). 获取组中第一个主机的ip地址
```yml
{{ hostvars[groups['webservers'][0]]['ansible_eth0']['ipv4']['address'] }}
```	
(9). 在任务中设置变量
```yml
- set_fact: headnode={{ groups[['webservers'][0]] }}
- debug: msg={{ headnode}}
```	
(10). 如何获取shell变量？
```yml
vars:
  local_home: "{{ lookup('env','HOME') }}"
tasks:
   - debug: var=local_home
```
在ansible1.4版本以上，可以使用以下方式获取
```
- debug: var=ansible_env.HOME
```	
(11). 在模板中如何遍历某一组内的所有主机？
```yml
{% for host in groups['db_servers'] %}
  {{ host }}
{% endfor %}
```
获取ip地址
```yml
{% for host in groups['db_servers'] %}
  {{ hostvars[host]['ansible_eth0']['ipv4']['address'] }}
{% endfor %}
```
(12). 加密hosts主机清单文件

 **有时候我们主机清单里面会有些密码信息，但是不想让别人看到。这种情况可以用ansible-vault来达到此目的。**
```bash
[root@node1 ansible]# cat db_hosts
localhost ansible_connection=local
[root@node1 ansible]# ansible-vault encrypt db_hosts 
New Vault password: 
Confirm New Vault password: 
Encryption successful
[root@node1 ansible]# ansible -i db_hosts localhost -m ping
ERROR! Decryption failed
Decryption failed
[root@node1 ansible]# ansible -i db_hosts --ask-vault-pass localhost -m ping
Vault password: 
localhost | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
[root@node1 ansible]# cat db_hosts 
$ANSIBLE_VAULT;1.1;AES256
61663966666265363465653064386666326234353433346163633838366532366236313032303636
6437313333333936396164663031633566613233343161650a333163333732616130343762636135
30303864663138643661393234336433313465623830333832663165393964353961323261373130
3135626236626435640a396338616563646532623966333337366365636665663563666432333539
61663632633130623733316232353836663366623136636432616332376266383263356264303765
6133616235363066356164653232326139643862653464623037
```
(13). service 模块启动服务没效果？
首先检查下service httpd status的信息，是不是有
```
httpd is stopped
```
这种字符，没有的话，在服务启动脚本里，在case语句里添加以下方法
```
status)
    status -p ${pidfile} $httpd
    RETVAL=$?
    ;;
# bash变量
# httpd=${HTTPD-/usr/sbin/httpd}
# pidfile=${PIDFILE-/var/run/httpd/httpd.pid}
```
从而达到service http status有stopped的字样。

(14). 递归目录中的模版文件
```
- name: Copying the templated jinja2 files
  template: src={{item}} dest={{RUN_TIME}}/{{ item | regex_replace(role_path+'/templates','') | regex_replace('\.j2', '') }}
  with_items: "{{ lookup('pipe','find {{role_path}}/templates -type f').split('\n') }}"
```

(15). 目标主机的python为2.7版本，且需要使用yum模块
需要增加下列变量，指定python版本为2.6
```
- hosts: servers
  vars:
    - ansible_python_interpreter: /usr/bin/python2.6.6
```
(16). 远程遍历拷贝文件
```yml
- name    : get files in /path/
  shell   : ls /path/*
  register: path_files
	
- name: fetch these back to the local Ansible host for backup purposes
  fetch:
  src : /path/"{{item}}"
  dest: /path/to/backups/
  with_items: "{{ path_files.stdout_lines }}"
```
(17). 获取主机清单中组的ip地址
```
- shell: "ping -c 1 {{item}} | grep icmp_seq | gawk -F'[()]'  '{print $2}'"
  with_inventory_hostnames: test2
  register: testip

 - debug: "msg={{ item.stdout }}"
  with_items: "{{ testip.results }}"
  ```

(18). 保留ansbile远程执行的模块文件，并调试模块

 添加`ANSIBLE_KEEP_REMOTE_FILES=1` 环境变量

	`$ ANSIBLE_KEEP_REMOTE_FILES=1 ansible localhost -m ping -a 'data=debugging_session' -vvv`
  ```
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
模块文件是由base64编码的字符串文件，可使用explode将字符串转换成可执行的python文件
 ```
 $ python /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py explode
Module expanded into:
/root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/debug_dir
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
`ansible_module_ping.py`  是模块本身的代码。
`args`文件包含一个JSON字符串。 该字符串是一个包含模块参数和其他变量的字典。
`ansible`目录包含由ansible_module_ping模块使用的ansible.module_utils的代码文件。
如果修改了`debug_dir`文件中的代码之后，需要使用execute执行代码
```
$ python /root/.ansible/tmp/ansible-tmp-1489477306.61-275734926719932/ping.py execute
{"invocation": {"module_args": {"data": "debugging_session"}}, "changed": false, "ping": "debugging_session"}
```
(19). 提升权限

Ansible ad-hoc命令
```
ansible -i hosts node1 -m shell -a "whoami" --become  --become-method=su --become-user=root --ask-su-pass
SU password: 
192.168.77.130 | SUCCESS | rc=0 >>
root
```
Ansible-playbook命令
```
cat test.yml
---
- hosts: node1
  gather_facts: no
  tasks:
  - name: I'm become to root.
    shell: whoami
    register: w
    become: true
    become_user: "root"
    become_method: "su"
  - debug: var=w.stdout

[root@base ~]# ansible-playbook -i hosts test.yml --ask-su-pass
SUDO password: 

PLAY [node1] ****************************************************************************************************************************************

TASK [I'm become to root.] **************************************************************************************************************************
changed: [192.168.77.130]

TASK [debug] ****************************************************************************************************************************************
ok: [192.168.77.130] => {
    "w.stdout": "root"
}

PLAY RECAP ******************************************************************************************************************************************
192.168.77.130             : ok=2    changed=1    unreachable=0    failed=0  
```
如果不想在执行过程中输入提升用户的密码，可以在hosts文件中配置ansible_become_pass变量设置密码。
```
# cat hosts 
[node1]
192.168.77.130 ansible_ssh_user=test ansible_ssh_pass=123456 ansible_become_pass=123456
```
参数解释
- become 开启提升权限
- become-method  提升权限的方式，有sudo，su，runas等。
- become-user  提升权限的用户
- ansible_become_pass 提升权限的用户密码
- ask-su-pass  告诉程序提升权限的用户密码

(20). 变量嵌套

在动态取变量的时候，我们第一时间就会写出`"{{ t_var[{{ n }}] }}"`的引用命令，但这类引用在jinja2的语法中是错误的，可以使用下列方式解决此引用问题。

```
- hosts: localhost
  gather_facts: no
  vars:
   - t_var: ['1','2']
   - n: "1"

  tasks:
   - shell: "echo {% if n %} {% set number = n | int %} {{ t_var[number]}} {% endif %}"
```
```bash
ansible-playbook test.yml -vv
Using /etc/ansible/ansible.cfg as config file
 [WARNING]: provided hosts list is empty, only localhost is available


PLAYBOOK: test.yml ***************************************************************************************************************
1 plays in test.yml

PLAY [localhost] *****************************************************************************************************************
META: ran handlers

TASK [command] *******************************************************************************************************************
task path: /etc/ansible/test.yml:10
changed: [localhost] => {"changed": true, "cmd": "echo   2 ", "delta": "0:00:00.012834", "end": "2017-09-12 10:40:44.959595", "rc": 0, "start": "2017-09-12 10:40:44.946761", "stderr": "", "stderr_lines": [], "stdout": "2", "stdout_lines": ["2"]}
META: ran handlers
META: ran handlers

PLAY RECAP ***********************************************************************************************************************
localhost                  : ok=1    changed=1    unreachable=0    failed=0 
```

(21). 环境变量找不到的问题

我们在ansible直接执行命令，不带有绝对路径，就会报出找不到命令的提示信息：
```
ansible node2 -m shell -a "openresty -v"
192.168.77.130 | FAILED | rc=127 >>
/bin/sh: openresty: 未找到命令
```

此时我们应该使用下列命令避免。
```
ansible node2 -m shell -a "source /etc/profile; openresty -v"
192.168.77.130 | SUCCESS | rc=0 >>
nginx version: openresty/1.11.2.3
```
ansible 的ssh登陆属于交互式的非登陆shell

详细说明请移步到 [ssh连接远程主机执行脚本的环境变量问题](http://www.jianshu.com/p/a6c4dd6e75e3)

(22). 获取redis的info信息

```
- hosts: localhost
  gather_facts: false
  tasks:
  - name: "query redis info"
    expect:
      command: "telnet 127.0.0.1 6379"
      responses:
        "Escape":
           - "auth test\ninfo\nquit\n"
    ignore_errors: true
    register: result
  - name: "show the variable"
    debug: 
      var: result
```
---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
{% endraw %}