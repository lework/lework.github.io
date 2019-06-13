---
layout: post
title:  "安装Git服务器"
categories: git
tags: git daemon
excerpt: 安装Git服务器
auth: lework
---
* content
{:toc}

## Git 守护进程

接下来我们将通过 “Git” 协议建立一个基于守护进程的仓库。 对于快速且无需授权的 Git 数据访问，这是一个理想之选。 请注意，因为其不包含授权服务，任何通过该协议管理的内容将在其网络上公开。

如果运行在防火墙之外的服务器上，它应该只对那些公开的只读项目服务。 如果运行在防火墙之内的服务器上，它可用于支撑大量参与人员或自动系统（用于持续集成或编译的主机）只读访问的项目，这样可以省去逐一配置 SSH 公钥的麻烦。


本次安装Git服务器的系统版本是：

```bash
CentOS release 6.7 (Final)
```

### No.1 安装Git

```bash
[root@Git-Server ~]# yum -y install git git-daemon
```

查看Git 版本号

```bash
[root@Git-Server ~]# git --version
git version 1.7.1
```

### No.2 创建Git 用户

这个用户对仓库只拥有只读权限，并且用该用户来运行守护进程

```bash
[root@Git-Server ~]# useradd git-ro -s /usr/bin/git-shell
```

### No.3 初始化Git 仓库

```bash
[root@Git-Server ~]# mkdir /opt/git/
[root@Git-Server ~]# git init --bare /opt/git/test.git
Initialized empty Git repository in /opt/git/test.git/
[root@Git-Server ~]# chown git-ro.git-ro -R /opt/git/
```


### No.4 运行守护进程

```bash
[root@Git-Server ~]# git daemon --user=git-ro --group=git-ro --reuseaddr --base-path=/opt/git/ /opt/git/
```

命令解释
- --reuseaddr 允许服务器在无需等待旧连接超时的情况下重启
- --base-path 选项允许用户在未完全指定路径的条件下克隆项目
- –-enable=receive-pack  允许push
- 结尾的路径将告诉 Git 守护进程从何处寻找仓库来导出。

如果有防火墙正在运行，你需要开放端口 9418 的通信权限。

```bash
[root@Git-Server test]# iptables -A INPUT -p tcp --dport 9418 -j ACCEPT
[root@Git-Server test]# /etc/init.d/iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]
[root@Git-Server test]# /etc/init.d/iptables restart
iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
iptables: Flushing firewall rules:                         [  OK  ]
iptables: Unloading modules:                               [  OK  ]
iptables: Applying firewall rules:                         [  OK  ]
```

### No.5 设置仓库的无授权访问

在每个仓库下创建一个名为 git-daemon-export-ok 的文件来实现，该文件将允许 Git 提供无需授权的项目访问服务。

```bash
[root@Git-Server ~]# touch /opt/git/test.git/git-daemon-export-ok
```

### No.6 客户端克隆

windows 平台下使用

![anon](/assets/images/git/tgit_anon.png)

Linux 平台下使用

```bash
[root@Git-Server test]# git clone git://192.168.77.133/test
```