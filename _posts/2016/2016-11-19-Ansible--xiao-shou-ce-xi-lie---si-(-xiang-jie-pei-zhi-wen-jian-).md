---
layout: post
title: "Ansible 小手册系列 四（详解配置文件）"
date: "2016-11-19 16:02:45"
categories: Ansible
excerpt: "配置文件存在不同的位置，但只有一个可用。在下列列表中，ansible从上往下依次检查，检查到哪个可用就用哪个 ANSIBLE_CFG  环境变量..."
auth: lework
---
* content
{:toc}

```bash
# ansible --version
ansible 2.1.2.0
  config file = /etc/ansible/ansible.cfg
  configured module search path = Default w/o overrides
```
配置文件存在不同的位置，但只有一个可用。在下列列表中，ansible从上往下依次检查，检查到哪个可用就用哪个
- ANSIBLE_CFG  环境变量，可以定义配置文件的位置
- ansible.cfg 存在于当前工作目录
- ansible.cfg 存在与当前用户家目录
- /etc/ansible/ansible.cfg

ansible 配置文件默认存使用   `/etc/ansible/ansible.cfg`
hosts文件默认存使用 `/etc/ansible/hosts`

ansible.cfg 配置项说明

**[defaults]**

| 配置 | 说明|
|:---|:---|
|#inventory      = /etc/ansible/hosts|	指定主机清单文件|
|#library        = /usr/share/my_modules/	|指定模块地址|
|#remote_tmp     = $HOME/.ansible/tmp	|指定远程执行的路径|
|#local_tmp      = $HOME/.ansible/tmp	|ansible 管理节点得执行路径|
|#forks          = 5	|置默认情况下Ansible最多能有多少个进程同时工作，默认设置最多5个进程并行处理|
|#poll_interval  = 15	|轮询间隔|
|#sudo_user      = root	|sudo默认用户|
|#ask_sudo_pass = True|	是否需要用户输入sudo密码|
|#ask_pass      = True	|是否需要用户输入连接密码|
|#transport      = smart||
|#remote_port    = 22	|远程链接的端口|
|#module_lang    = C	|这是默认模块和系统之间通信的计算机语言,默认为’C’语言.|
|#module_set_locale = True||
|#gathering = implicit||
|#gather_subset = all	|定义获取fact的子集，默认全部|
|#roles_path    = /etc/ansible/roles	|角色存储路径|
|#host_key_checking = False	|跳过ssh 首次连接提示验证部分，False表示跳过。|
|#stdout_callback = skippy||
|#callback_whitelist = timer, mail||
|#task_includes_static = True||
|#handler_includes_static = True||
|#sudo_exe = sudo|	sudo的执行文件名|
|#sudo_flags = -H -S -n	|sudo的参数|
|#timeout = 10|	连接超时时间|
|#remote_user = root|	指定默认的远程连接用户|
|#log_path = /var/log/ansible.log	|指定日志文件|
|#module_name = command|	指定ansible默认的执行模块|
|#executable = /bin/sh	|用于执行脚本得解释器|
|#hash_behaviour = replace	|如果变量重叠，优先级更高的一个是替换优先级低得还是合并在一起，默认为替换|
|#private_role_vars = yes	|默认情况下，角色中的变量将在全局变量范围中可见。 为了防止这种情况，可以启用以下选项，只有tasks的任务和handlers得任务可以看到角色变量。|
|#jinja2_extensions = jinja2.ext.do,jinja2.ext.i18n	|jinja2的扩展应用|
|#private_key_file = /path/to/file	|指定私钥文件路径|
|#vault_password_file = /path/to/vault_password_file	|指定vault密码文件路径|
|#ansible_managed = Ansible managed: {file} modified on %Y-%m-%d %H:%M:%S by {uid} on {host}	|定义一个Jinja2变量，可以插入到Ansible配置模版系统生成的文件中|
|#display_skipped_hosts = True	|如果设置为False,ansible 将不会显示任何跳过任务的状态.默认选项是显示跳过任务的状态|
|#display_args_to_stdout = False||
|#error_on_undefined_vars = False	|如果所引用的变量名称错误的话, 是否让ansible在执行步骤上失败|
|#system_warnings = True| |
|#deprecation_warnings = True| |
|#command_warnings = False| |
|#action_plugins     = /usr/share/ansible/plugins/action| action模块的存放路径|
|#callback_plugins   = /usr/share/ansible/plugins/callback|callback模块的存放路径|
|#connection_plugins = /usr/share/ansible/plugins/connection|connection模块的存放路径|
|#lookup_plugins     = /usr/share/ansible/plugins/lookup|lookup模块的存放路径|
|#vars_plugins       = /usr/share/ansible/plugins/vars|vars模块的存放路径|
|#test_plugins       = /usr/share/ansible/plugins/test|test模块的存放路径|
|#strategy_plugins   = /usr/share/ansible/plugins/strategy|strategy模块的存放路径|
|#bin_ansible_callbacks = False| |
|#nocows = 1| |
|#cow_selection = default||
|#cow_selection = random||
|#cow_whitelist=bud-frogs,bunny,cheese,daemon,default,dragon,elephant-in-snake,elephant,eyes,\||
|#nocolor = 1	|默认ansible会为输出结果加上颜色,用来更好的区分状态信息和失败信息.如果你想关闭这一功能,可以把’nocolor’设置为‘1’:|
|#fact_caching = memory	|fact值默认存储在内存中，可以设置存储在redis中，用于持久化存储|
|#retry_files_enabled = False	|当playbook失败得情况下，一个重试文件将会创建，默认为开启此功能|
|#retry_files_save_path = ~/.ansible-retry	|重试文件的路径，默认为当前目录下.ansible-retry|
|#squash_actions = apk,apt,dnf,package,pacman,pkgng,yum,zypper	|Ansible可以优化在循环时使用列表参数调用模块的操作。 而不是每个with_项调用模块一次，该模块会一次调用所有项目一次。该参数记录哪些action是这样操作得。|
|#no_log = False	|任务数据的日志记录，默认情况下关闭|
|#no_target_syslog = False	|防止任务的日志记录，但只在目标上，数据仍然记录在主/控制器上|
|#allow_world_readable_tmpfiles = False||
|#var_compression_level = 9	|控制发送到工作进程的变量的压缩级别。 默认值为0，不使用压缩。 此值必须是从0到9的整数。|
|#module_compression = 'ZIP_DEFLATED'	|指定压缩方法，默认使用zlib压缩，可以通过ansible_module_compression来为每个主机设置|
|#max_diff_size = 1048576	|控制--diff文件上的截止点（以字节为单位），设置0则为无限制（可能对内存有影响）|


**[privilege_escalation]**
\#become=True	
\#become_method=sudo	
\#become_user=root	
\#become_ask_pass=False	

**[paramiko_connection]**
\#record_host_keys=False	
\#pty=False	


**[ssh_connection]**

| 配置 | 说明|
|:---|:---|
|#ssh_args = -o ControlMaster=auto -o ControlPersist=60s|	ssh连接时得参数|
|#control_path = %(directory)s/ansible-ssh-%%h-%%p-%%r	|保存ControlPath套接字的位置|
|#pipelining = False	|SSH pipelining 是一个加速 Ansible 执行速度的简单方法。ssh pipelining 默认是关闭，之所以默认关闭是为了兼容不同的 sudo 配置，主要是 requiretty 选项。如果不使用 sudo，建议开启。打开此选项可以减少 ansible 执行没有传输时 ssh 在被控机器上执行任务的连接数。不过，如果使用 sudo，必须关闭 requiretty 选项。|
|#scp_if_ssh = True	|该项为True时，如果连接类型是ssh，使ansible使用scp，为False是，ansible使用sftp。默认为sftp|
|#sftp_batch_mode = False	|该项为False时，sftp不会使用批处理模式传输文件。 这可能导致一些类型的文件传输失败而不可捕获，但应该只有在您的sftp版本在批处理模式上有问题时才应禁用|

**[accelerate]**
加速配置
\#accelerate_port = 5099
\#accelerate_timeout = 30
\#accelerate_connect_timeout = 5.0
\#accelerate_daemon_timeout = 30
\#accelerate_multi_key = yes

**[selinux]**

| 配置 | 说明|
|:---|:---|
|#special_context_filesystems=nfs,vboxsf,fuse,ramfs	|文件系统在处理安全上下文时需要特殊处理，定义复制现有上下文的文件系统|
|#libvirt_lxc_noseclabel = yes	|将此设置为yes，以允许libvirt_lxc连接在没有SELinux的情况下工作。|

**[colors]**
定义输出颜色
\#highlight = white
\#verbose = blue
\#warn = bright purple
\#error = red
\#debug = dark gray
\#deprecate = purple
\#skip = cyan
\#unreachable = red
\#ok = green
\#changed = yellow
\#diff_add = green
\#diff_remove = red
\#diff_lines = cyan

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
