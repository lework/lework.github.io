---
layout: post
title: "利用ansible来做tomcat应用的持续交付"
date: "2017-08-19 17:28:38"
categories: Ansible
excerpt: "在做持续交付这件事，想必大家都是用jenkins这款程序来做基石。当然，我们这次也是用jenkins作为承载工具，jenkins强大的插件是有目..."
auth: lework
---
* content
{:toc}

在做持续交付这件事，想必大家都是用jenkins这款程序来做基石。当然，我们这次也是用jenkins作为承载工具，jenkins强大的插件是有目共睹的，有些ansible做起来不容易的事情交给jenkins反而简单有效。下面我会详细说明怎么持续交付tomcat应用。

> 希望本实验可以引导大家在持续交付的过程中使用ansible工具，也希望本实验能帮助到有需要的人，更希望给到大家一个简单的持续交付思想和启发。如想继续交流的，还请加入QQ群：425931784。

## 应用架构
---

本次使用的应用架构是常见的负载均衡实例。

![image.png](/assets/images/Ansible/3629406-83bed54dc6debcb4.png)

## 软件版本
---
os： `centos 6.7 X64`
ansible: `2.3.1.0`
python: `2.6.6`
ant: `10.1`
java:   `1.8.0_13`
tomcat: `8.5.14`
jenkins: `2.73`

## Ansible roles
---
- [Ansible Role 系统环境 之【ant】](http://www.jianshu.com/p/75a129e71685)
- [Ansible Role 系统环境 之【java】](http://www.jianshu.com/p/1be92c3f65ec)
- [Ansible Role 系统环境 之【iptables】](http://www.jianshu.com/p/1ce357af03bf)
- [Ansible Role WEB 之【tomcat】](http://www.jianshu.com/p/fd7cca20c227)
- [Ansible Role WEB 之【nginx】](http://www.jianshu.com/p/447df6f2335a)
- [Ansible Role 持续集成 之【jenkins】](http://www.jianshu.com/p/f4bac35454b4)
- [Ansible Role 持续交付 之【deploy-tomcat】](http://www.jianshu.com/p/eadbdb861fa2)

## 服务器角色
---
| 主机 | 角色 |
| --- |---|
| node1 | nginx,jenkins |
| node130 | tomcat |
| node131 | tomcat |

## 集群搭建
---

本次使用anisble playbook 
```yml
---

- hosts: node130 node131
  vars:
   - java_version: "1.8"
   - tomcat_version: "8.5.14"
   - iptables_allowed_tcp_ports: ["8080"]
  roles:
  - java
  - { role: tomcat, java_home: "/usr/java/jdk1.8.0_131" }
  - iptables

- hosts: node1
  vars:
   - java_version: "1.8"
   - nginx_version: "1.12.1"
   - nginx_upstreams:
     - name: upstremtest
       servers:
       - 192.168.77.130:8080 max_fails=2 fail_timeout=2
       - 192.168.77.131:8080 max_fails=2 fail_timeout=2
   - nginx_vhosts:
     - listen: 80
       locations:
       - name: /
         proxy_pass: http://upstremtest
   - jenkins_version: "2.73"
   - jenkins_plugins_extra:
     - ansible
     - ansicolor
   - iptables_allowed_tcp_ports: ["80","8080"]
  roles:
  - ant
  - java
  - nginx
  - jenkins
  - iptables
  tasks:
  - name: install ansible
    package: name=ansible
```
怎么使用ansible roles，请移步到 [Ansible Role【怎么用？】](http://www.jianshu.com/p/585303ab4b02)

确保正常访问以下服务：
- nginx http://192.168.77.129/lework
- jenkins http://192.168.77.129:8080 帐号密码：admin/admin
- tomcat http://192.168.77.130:8080/lework http://192.168.77.131:8080/lework

## node1服务器操作
---
在服务器上配置ansible playbook
```
# cd /etc/ansible/
# cat tomcat-deploy.yml
---

- hosts: all
  serial: 1
  roles:
   - deploy-tomcat

# cat hosts
[node130]
192.168.77.130

[node131]
192.168.77.131

[testservers:children]
node130
node131

[testservers:vars]
ansible_ssh_user=root
ansible_ssh_pass=123456

# git clone https://github.com/lework/Ansible-roles.git /etc/ansible/roles/
# chown jenkins.jenkins /etc/ansible/
```

## jenkins 操作
---

**登录jenkins之后，设置工具**
点击“系统管理”==》“Global Tool Configuration”

![image.png](/assets/images/Ansible/3629406-6cb1c49217015393.png)

![image.png](/assets/images/Ansible/3629406-6b8235de45e91003.png)

![image.png](/assets/images/Ansible/3629406-fd46196dcb799b2c.png)


**创建发布项目**
![image.png](/assets/images/Ansible/3629406-8879cadf761da721.png)

配置参数化构建
![image.png](/assets/images/Ansible/3629406-e329100b334ca355.png)

配置源码仓库地址
![image.png](/assets/images/Ansible/3629406-8121b503685875e2.png)

> repo: https://github.com/lework/AntSpringMVC.git


配置构建环境
![image.png](/assets/images/Ansible/3629406-d3c2e9ae3a3c4210.png)

配置编译

![image.png](/assets/images/Ansible/3629406-c79ab16e4c9d68a1.png)


配置`ansible`
![image.png](/assets/images/Ansible/3629406-a34c7e8b3f0d267e.png)


配置ansible变量
![image.png](/assets/images/Ansible/3629406-9dad672c8a038c05.png)


这里就不配置邮件通知了。

**创建回滚项目**

![image.png](/assets/images/Ansible/3629406-a5e4c6f0b95cb1f4.png)
配置参数化构建
![image.png](/assets/images/Ansible/3629406-fb26e628e34daee8.png)

配置构建环境
![image.png](/assets/images/Ansible/3629406-7dfd1eb2756a0422.png)

配置`ansible`
![image.png](/assets/images/Ansible/3629406-5e00c52575293913.png)
配置`anisble`变量
![image.png](/assets/images/Ansible/3629406-77e5f415cd983248.png)


## 测试
---
**执行tomcat_deploy任务**

![](/assets/images/Ansible/3629406-dec48f19a8a3d7e0.png)
>选择发布的节点，默认all

任务执行的日志
```
Started by user admin
Building in workspace /var/lib/jenkins/workspace/tomcat_deploy
Cloning the remote Git repository
Cloning repository https://github.com/lework/AntSpringMVC.git
 > git init /var/lib/jenkins/workspace/tomcat_deploy # timeout=10
Fetching upstream changes from https://github.com/lework/AntSpringMVC.git
 > git --version # timeout=10
 > git fetch --tags --progress https://github.com/lework/AntSpringMVC.git +refs/heads/*:refs/remotes/origin/*
 > git config remote.origin.url https://github.com/lework/AntSpringMVC.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git config remote.origin.url https://github.com/lework/AntSpringMVC.git # timeout=10
Fetching upstream changes from https://github.com/lework/AntSpringMVC.git
 > git fetch --tags --progress https://github.com/lework/AntSpringMVC.git +refs/heads/*:refs/remotes/origin/*
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 989ea3a6549e16e3dd4cd329ab969b47658c9d67 (refs/remotes/origin/master)
Commit message: "Create README.md"
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 989ea3a6549e16e3dd4cd329ab969b47658c9d67
First time build. Skipping changelog.
[tomcat_deploy] $ ant -file build.xml -Ddeploy_node=all
Buildfile: /var/lib/jenkins/workspace/tomcat_deploy/build.xml

clean:
   [delete] Deleting directory /var/lib/jenkins/workspace/tomcat_deploy/war/WEB-INF/classes

init:
    [mkdir] Created dir: /var/lib/jenkins/workspace/tomcat_deploy/target
    [mkdir] Created dir: /var/lib/jenkins/workspace/tomcat_deploy/war/WEB-INF/classes

resolve:
     [echo] Getting dependencies...
[ivy:retrieve] :: Apache Ivy 2.4.0 - 20141213170938 :: http://ant.apache.org/ivy/ ::
[ivy:retrieve] :: loading settings :: url = jar:file:/usr/local/ant/lib/ivy-2.4.0.jar!/org/apache/ivy/core/settings/ivysettings.xml
[ivy:retrieve] :: resolving dependencies :: org.apache#WebProject;working@node1
[ivy:retrieve] 	confs: [compile, runtime, test]
[ivy:retrieve] 	found org.slf4j#slf4j-api;1.7.6 in public
[ivy:retrieve] 	found jstl#jstl;1.2 in public
[ivy:retrieve] 	found ch.qos.logback#logback-classic;1.1.2 in public
[ivy:retrieve] 	found ch.qos.logback#logback-core;1.1.2 in public
[ivy:retrieve] 	found org.springframework#spring-core;4.1.3.RELEASE in public
[ivy:retrieve] 	found commons-logging#commons-logging;1.2 in public
[ivy:retrieve] 	found org.springframework#spring-beans;4.1.3.RELEASE in public
[ivy:retrieve] 	found org.springframework#spring-context;4.1.3.RELEASE in public
[ivy:retrieve] 	found org.springframework#spring-aop;4.1.3.RELEASE in public
[ivy:retrieve] 	found aopalliance#aopalliance;1.0 in public
[ivy:retrieve] 	found org.springframework#spring-expression;4.1.3.RELEASE in public
[ivy:retrieve] 	found org.springframework#spring-web;4.1.3.RELEASE in public
[ivy:retrieve] 	found org.springframework#spring-webmvc;4.1.3.RELEASE in public
[ivy:retrieve] downloading https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.6/slf4j-api-1.7.6.jar ...
[ivy:retrieve] ............ (28kB)
[ivy:retrieve] .. (0kB)
..... 省略下载的信息
[ivy:retrieve] :: resolution report :: resolve 74135ms :: artifacts dl 120701ms
	---------------------------------------------------------------------
	|                  |            modules            ||   artifacts   |
	|       conf       | number| search|dwnlded|evicted|| number|dwnlded|
	---------------------------------------------------------------------
	|      compile     |   13  |   13  |   13  |   0   ||   13  |   13  |
	|      runtime     |   13  |   13  |   13  |   0   ||   13  |   13  |
	|       test       |   13  |   13  |   13  |   0   ||   13  |   13  |
	---------------------------------------------------------------------
[ivy:retrieve] :: retrieving :: org.apache#WebProject
[ivy:retrieve] 	confs: [compile, runtime, test]
[ivy:retrieve] 	13 artifacts copied, 0 already retrieved (5920kB/79ms)

compile:
    [javac] Compiling 1 source file to /var/lib/jenkins/workspace/tomcat_deploy/war/WEB-INF/classes
copy-resources:
     [copy] Copying 1 file to /var/lib/jenkins/workspace/tomcat_deploy/war/WEB-INF/classes
package:
[ivy:retrieve] :: retrieving :: org.apache#WebProject
[ivy:retrieve] 	confs: [runtime]
[ivy:retrieve] 	0 artifacts copied, 13 already retrieved (0kB/5ms)
      [war] Building war: /var/lib/jenkins/workspace/tomcat_deploy/target/helloproject-20170819172002.war

main:

BUILD SUCCESSFUL
Total time: 3 minutes 19 seconds
[tomcat_deploy] $ /usr/bin/ansible-playbook /etc/ansible/tomcat-deploy.yml -i /etc/ansible/hosts -l all -f 5 -e deploy_port=8080 -e deploy_file=/var/lib/jenkins/workspace/tomcat_deploy/target/helloproject-*.war

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : check | 发布文件是否存在] ****************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : check | 目标应用服务的家目录是否存在] **********************************
ok: [192.168.77.130]

TASK [deploy-tomcat : check | 工作目录如果不存在则创建] ************************************
changed: [192.168.77.130] => (item=/tmp/tomcat-ansible-snap/new)
changed: [192.168.77.130] => (item=/tmp/tomcat-ansible-snap/pre)
changed: [192.168.77.130] => (item=/tmp/tomcat-ansible-snap/old)

TASK [deploy-tomcat : deloy | 解压代码至目标服务器] **************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : deloy | 关闭服务] ********************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : deloy | 等待端口关闭] ******************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : deloy | 移动线上代码] ******************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : deloy | 部署最新代码] ******************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : deloy | 启动服务] ********************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : deloy | 等待端口开启] ******************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : backup | 创建存储备份的文件夹] *************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : backup | 备份上线的代码] ****************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : rollback | 检查/tmp/tomcat-ansible-snap/old是否存在代码] *********
skipping: [192.168.77.130]

TASK [deploy-tomcat : rollback | 关闭服务] *****************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : rollback | 等待端口关闭] ***************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : rollback | 部署上一版代码] **************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : rollback | 启动服务] *****************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : rollback | 等待端口开启] ***************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
skipping: [192.168.77.130]

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : check | 发布文件是否存在] ****************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : check | 目标应用服务的家目录是否存在] **********************************
ok: [192.168.77.131]

TASK [deploy-tomcat : check | 工作目录如果不存在则创建] ************************************
ok: [192.168.77.131] => (item=/tmp/tomcat-ansible-snap/new)
ok: [192.168.77.131] => (item=/tmp/tomcat-ansible-snap/pre)
ok: [192.168.77.131] => (item=/tmp/tomcat-ansible-snap/old)

TASK [deploy-tomcat : deloy | 解压代码至目标服务器] **************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : deloy | 关闭服务] ********************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : deloy | 等待端口关闭] ******************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : deloy | 移动线上代码] ******************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : deloy | 部署最新代码] ******************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : deloy | 启动服务] ********************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : deloy | 等待端口开启] ******************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : backup | 创建存储备份的文件夹] *************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : backup | 备份上线的代码] ****************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : rollback | 检查/tmp/tomcat-ansible-snap/old是否存在代码] *********
skipping: [192.168.77.131]

TASK [deploy-tomcat : rollback | 关闭服务] *****************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : rollback | 等待端口关闭] ***************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : rollback | 部署上一版代码] **************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : rollback | 启动服务] *****************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : rollback | 等待端口开启] ***************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
skipping: [192.168.77.131]

PLAY RECAP *********************************************************************
192.168.77.130             : ok=14   changed=8    unreachable=0    failed=0   
192.168.77.131             : ok=14   changed=7    unreachable=0    failed=0   

Finished: SUCCESS
```
**执行tomcat_rollback任务**

![image.png](/assets/images/Ansible/3629406-598eee8dc9bfa337.png)

> 选择回滚的节点，默认all

执行的日志
```
Started by user admin
Building in workspace /var/lib/jenkins/workspace/tomcat_rollback
[tomcat_rollback] $ /usr/bin/ansible-playbook /etc/ansible/tomcat-deploy.yml -i /etc/ansible/hosts -l all -f 5 -e deploy_rollback=true

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : check | 发布文件是否存在] ****************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : check | 目标应用服务的家目录是否存在] **********************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : check | 工作目录如果不存在则创建] ************************************
skipping: [192.168.77.130] => (item=/tmp/tomcat-ansible-snap/new) 
skipping: [192.168.77.130] => (item=/tmp/tomcat-ansible-snap/pre) 
skipping: [192.168.77.130] => (item=/tmp/tomcat-ansible-snap/old) 

TASK [deploy-tomcat : deloy | 解压代码至目标服务器] **************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : deloy | 关闭服务] ********************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : deloy | 等待端口关闭] ******************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : deloy | 移动线上代码] ******************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : deloy | 部署最新代码] ******************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : deloy | 启动服务] ********************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : deloy | 等待端口开启] ******************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : backup | 创建存储备份的文件夹] *************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : backup | 备份上线的代码] ****************************************
skipping: [192.168.77.130]

TASK [deploy-tomcat : rollback | 检查/tmp/tomcat-ansible-snap/old是否存在代码] *********
changed: [192.168.77.130]

TASK [deploy-tomcat : rollback | 关闭服务] *****************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : rollback | 等待端口关闭] ***************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : rollback | 部署上一版代码] **************************************
changed: [192.168.77.130]

TASK [deploy-tomcat : rollback | 启动服务] *****************************************
fatal: [192.168.77.130]: FAILED! => {"changed": true, "cmd": "/etc/init.d/tomcat start", "delta": "0:00:20.035003", "end": "2017-08-19 17:24:47.586469", "failed": true, "rc": 1, "start": "2017-08-19 17:24:27.551466", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}
...ignoring

TASK [deploy-tomcat : rollback | 等待端口开启] ***************************************
ok: [192.168.77.130]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
ok: [192.168.77.130]

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : check | 发布文件是否存在] ****************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : check | 目标应用服务的家目录是否存在] **********************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : check | 工作目录如果不存在则创建] ************************************
skipping: [192.168.77.131] => (item=/tmp/tomcat-ansible-snap/new) 
skipping: [192.168.77.131] => (item=/tmp/tomcat-ansible-snap/pre) 
skipping: [192.168.77.131] => (item=/tmp/tomcat-ansible-snap/old) 

TASK [deploy-tomcat : deloy | 解压代码至目标服务器] **************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : deloy | 关闭服务] ********************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : deloy | 等待端口关闭] ******************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : deloy | 移动线上代码] ******************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : deloy | 部署最新代码] ******************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : deloy | 启动服务] ********************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : deloy | 等待端口开启] ******************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : backup | 创建存储备份的文件夹] *************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : backup | 备份上线的代码] ****************************************
skipping: [192.168.77.131]

TASK [deploy-tomcat : rollback | 检查/tmp/tomcat-ansible-snap/old是否存在代码] *********
changed: [192.168.77.131]

TASK [deploy-tomcat : rollback | 关闭服务] *****************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : rollback | 等待端口关闭] ***************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : rollback | 部署上一版代码] **************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : rollback | 启动服务] *****************************************
changed: [192.168.77.131]

TASK [deploy-tomcat : rollback | 等待端口开启] ***************************************
ok: [192.168.77.131]

TASK [deploy-tomcat : verify | 查看http状态.] **************************************
ok: [192.168.77.131]

PLAY RECAP *********************************************************************
192.168.77.130             : ok=8    changed=4    unreachable=0    failed=0   
192.168.77.131             : ok=8    changed=4    unreachable=0    failed=0   

Finished: SUCCESS
```

至此，持续交付实验就完成了，但是持续之路还是很漫长了。望大家永远前进。 大家也可在发的过程中，测试发布是否是灰度发布。
```
for i in `seq 10000`;do curl -s -I http://192.168.77.129 | head -1;sleep 1;done;
```
