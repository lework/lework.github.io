---
layout: post
title: "Ansible 小手册系列 三（命令介绍）"
date: "2016-11-19 15:48:03"
categories: Ansible
excerpt: "Ansible 版本：2.1.2.0 ansible ansible是指令核心部分，其主要用于执行ad-hoc命令，即单条命令。默认后面需要跟主..."
auth: lework
---
* content
{:toc}

> Ansible 版本：`2.1.2.0`

## ansible
---

ansible是指令核心部分，其主要用于执行ad-hoc命令，即单条命令。默认后面需要跟主机和选项部分，默认不指定模块时，使用的是command模块。

**`Usage: ansible <host-pattern> [options]`**

选项：

|参数  |说明  |
|:------------- |:-------------|
|-a MODULE_ARGS, --args=MODULE_ARGS|	模块的参数。|
|--ask-vault-pass|	vault 密码。|
|-B SECONDS, --background=SECONDS|	异步运行时，多长时间超时。|
|-C, --check|           	只是测试一下会改变什么内容，不会真正去执行;相反,试图预测一些可能发生的变化。|
|-D, --diff   |         	当更改文件和模板时，显示这些文件得差异，比--check效果好。|
|-e EXTRA_VARS, --extra-vars=EXTRA_VARS	|添加附加变量，比如key=value，yaml，json格式。|
|-f FORKS, --forks=FORKS	|指定定要使用的并行进程数，默认为5个。|
|-h, --help            	|显示此帮助信息。|
|-i INVENTORY, --inventory-file=INVENTORY	|指定主机清单文件或逗号分隔的主机，默认为/etc/ansible/hosts。|
|-l SUBSET, --limit=SUBSET	|进一步限制所选主机/组模式，只执行-l 后的主机和组。|
|--list-hosts|	输出匹配主机的列表。|
|-m MODULE_NAME, --module-name=MODULE_NAME	|要执行的模块，默认为command。|
|-M MODULE_PATH, --module-path=MODULE_PATH	|要执行的模块的路径。|
|--new-vault-password-file=NEW_VAULT_PASSWORD_FILE|	新vault密钥文件。|
|-o, --one-line	|压缩输出，摘要输出.尝试一切都在一行上输出。|
|--output=OUTPUT_FILE	|加密或解密输出文件名 用于标准输出。|
|-P POLL_INTERVAL, --poll=POLL_INTERVAL|	如果使用-B，则设置轮询间隔。|
|--syntax-check	|对playbook进行语法检查，且不执行playbook。|
|-t TREE, --tree=TREE	|将日志内容保存在该目录中,文件名以执行主机名命名。|
|--vault-password-file=VAULT_PASSWORD_FILE|	vault密码文件|
|-v, --verbose    |    	输出执行的详细信息，使用-vvv获得更多，-vvvv 启用连接调试 |
|--version |           	显示程序版本号|

连接选项:

|参数  |说明  |
|:------------- |:-------------|
|-k, --ask-pass|	要求用户输入请求连接密码|
|--private-key=PRIVATE_KEY_FILE, --key-file=PRIVATE_KEY_FILE	|私钥路径，使用这个文件来验证连接|
|-u REMOTE_USER, --user=REMOTE_USER|	连接用户|
|-c CONNECTION, --connection=CONNECTION	|连接类型，默认smart|
|-T TIMEOUT, --timeout=TIMEOUT	|指定默认超时时间，默认是10S|
|--ssh-common-args=SSH_COMMON_ARGS	|指定要传递给sftp / scp / ssh的常见参数  （例如 ProxyCommand）|
|--sftp-extra-args=SFTP_EXTRA_ARGS	|指定要传递给sftp，例如-f -l|
|--scp-extra-args=SCP_EXTRA_ARGS	|指定要传递给scp，例如 -l|
|--ssh-extra-args=SSH_EXTRA_ARGS	|指定要传递给ssh，例如 -R|

特权升级选项：

|参数  |说明  |
|:------------- |:-------------|
|-s, --sudo     |     	使用sudo (nopasswd)运行操作 ， 不推荐使用|
|-U SUDO_USER, --sudo-user=SUDO_USER	|sudo 用户，默认为root， 不推荐使用|
|-S, --su       |    	使用su运行操作， 不推荐使用|
|-R SU_USER, --su-user=SU_USER|	su 用户，默认为root，不推荐使用|
|-b, --become |    运行操作|
|--become-method=BECOME_METHOD|权限升级方法使用 ，默认为sudo，有效选择：sudo,su,pbrun,pfexec,runas,doas,dzdo
|--become-user=BECOME_USER 	|使用哪个用户运行，默认为root|
|--ask-sudo-pass    | 	sudo密码，不推荐使用|
|--ask-su-pass       |	su密码，不推荐使用|
|-K, --ask-become-pass|	权限提升密码|

示例：

```bash
ansible all -m ping
ansible 192.168.77.* -m ping
ansible all -m command -a ifconfig
ansible all -m shell -a "ifconfig eth0 |grep 'inet addr' "
ansible -i "192.168.77.129,192.168.77.130" 192.168.77.129  -m ping
ansible -i hosts  all --list-host
ansible -i hosts -l 192.168.77.130 all -m ping -t /tmp -vvvv
ansible web -l @retry_hosts.txt --list-hosts
```

## ansible-console
---

交互式命令执行界面

**`Usage: ansible-console <host-pattern> [options]`**

> 选项与ansible一致



## ansible-doc 
---

该指令用于查看模块信息，常用参数有两个`-l`和 `-s` 

**`Usage: ansible <host-pattern> [options]`**

选项

|参数  |说明  |
|:------------- |:-------------|
|  -h, --help     |       	显示此帮助信息|
|  -l, --list         |   	列出可用的模块|
 | -M MODULE_PATH, --module-path=MODULE_PATH	|指定到模块库的路径|
| -s, --snippet    |    	显示playbook制定模块的用法|
|  -v, --verbose   |     	详细模式（-vvv表示更多，-vvvv表示启用连接调试）|
 | --version         |    	显示程序版本号|

示例：

```
ansible-doc -l
ansible-doc yum
ansible-doc yum -s
```


## ansible-galaxy 
---

`ansible-galaxy` 指令用于方便的从 https://galaxy.ansible.com/  站点下载第三方扩展模块，我们可以形象的理解其类似于centos下的yum、python下的pip或easy_install

**`Usage: ansible-galaxy [delete|import|info|init|install|list|login|remove|search|setup] [--help] [options] …`**


示例：
```
ansible-galaxy install aeriscloud.docker  # 下载
ansible-galaxy  init abc   # 创建角色模板
```

## ansible-playbook
---

对于需反复执行的、较为复杂的任务，我们可以通过定义 Playbook 来搞定。Playbook 是 Ansible 真正强大的地方，它允许使用变量、条件、循环、以及模板，也能通过角色 及包含指令来重用既有内容。

**`Usage: ansible-playbook playbook.yml `**

相对于ansible，增加了下列选项：

|参数  |说明  |
|:------------- |:-------------|
| --flush-cache        	|清除fact缓存|
|--force-handlers	|如果任务失败，也要运行handlers|
|--list-tags|	列出所有可用的标签|
|--list-tasks|	列出将要执行的所有任务|
|--skip-tags=SKIP_TAGS|	跳过运行标记此标签的任务|
|--start-at-task=START_AT_TASK|	在此任务处开始运行   |     
|--step         |      	一步一步：在运行之前确认每个任务|
|-t TAGS, --tags=TAGS  |	只运行标记此标签的任务|
  
示例：
```
ansible-playbook -i hosts ssh-addkey.yml    # 指定主机清单文件
ansible-playbook -i hosts ssh-addkey.yml  --list-tags   # 列出tags
ansible-playbook -i hosts ssh-addkey.yml  -T install  # 执行install标签的任务
```

## ansible-pull 
---

pull模式在被配置的机器上运行，速度很快。在这种模式下，你需要提供一个git仓库来供Ansible下载来配置你的机器。

**`Usage: ansible-pull -U <repository> [options]`**

## ansible-vault 
---

ansible-vault主要应用于配置文件中含有敏感信息，又不希望他能被人看到，vault可以帮你加密/解密这个配置文件。这种playbook文件在执行时，需要加上 –ask-vault-pass参数，同样需要输入密码后才能正常执行。

**`Usage: ansible-vault [create|decrypt|edit|encrypt|rekey|view] [--help] [options] vaultfile.yml`**

示例：
```
ansible-vault create /tmp/123 # 创建加密文件
ansible-vault view  /tmp/123 # 查看加密文件
ansible-vault encrypt  /tmp/abc # 加密文件
ansible-vault decrypt  /tmp/abc # 解密文件
```
---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
