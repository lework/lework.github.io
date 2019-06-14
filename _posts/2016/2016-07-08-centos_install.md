---
layout: post
title:  "安装git"
categories: git
tags: git install centos
excerpt: 安装git
auth: lework
---
* content
{:toc}

本次安装Git服务器的系统版本是：

```bash
CentOS release 6.7 (Final)
```

其他系统安装方法，请移步到[其他系统安装方法](other_install.html)


### No.1 安装Git

```bash
[root@Git-Server ~]# yum -y install git
```

查看Git 版本号

```bash
[root@Git-Server ~]# git --version
git version 1.7.1
```

### No.2 创建Git 用户

```bash
[root@Git-Server ~]# useradd git -s /usr/bin/git-shell
```

### No.3 创建证书登录

收集所有需要登录的用户的公钥，就是他们自己的id_rsa.pub文件，把所有公钥导入到/home/git/.ssh/authorized_keys文件里，一行一个。

关于密钥生成，windows使用Puttygen生成；linux使用ssh-keygen生成。

这里以TortoiseGit 用户为例。

在安装TortoiseGit的时候，默认还会安装有Puttygen.exe这个程序，这个程序是可以生成putty密钥的。
Puttygen默认路径在C:\Program Files\TortoiseGit\bin\puttygen.exe，启动后按照下图所示进行操作。

![puttykey](/assets/images/git/puttykey.png)

#### 然后在服务器上添加客户端的公钥信息

```bash
[root@Git-Server ~]# mkdir /home/git/.ssh
[root@Git-Server ~]# echo ssh-rsa AAAAB3NzaC1yc2EAAAABJQAAAQEAvoZ5kTxuDb1JePox5GPTONRGeU5BcASw7D2BYczcHcnMnWjPWfAnzwy9r/9xE656dRnb+/qOacfO6oe9VyzixkKM+iCfdzIOe7BpRt9049jPmmMiHL+mhHOqbwU4i7jSodh86xEWSDCXEgWqjUQzWs6g9gWOLgc+oUVPnWqjiPKncFsuGoPDrSFTjfCB0uDiWXWjB7Y/NEv/d/Wr60fw2WBrwlpGDBFCKKI3ja+7re1uQqFWLzYdhwiOLLM8Tib4OJsx/P/4K/shXvo8hqGz8+mwrplXBud0z96KUoCXlIhXLt81v4mnJdZi1ny+kZypwWRCJu8tzB9g9aDyblShBw== rsa-key-20160708 >> /home/git/.ssh/authorized_keys
[root@Git-Server ~]# chown git.git -R /home/git/.ssh
```

### No.4 初始化Git仓库

Git就会创建一个裸仓库，裸仓库没有工作区，因为服务器上的Git仓库纯粹是为了共享，所以不让用户直接登录到服务器上去改工作区，并且服务器上的Git仓库通常都以.git结尾。

```bash
[root@Git-Server ~]# mkdir /svn_data
[root@Git-Server ~]# git init --bare /svn_data/sample.git
Initialized empty Git repository in /svn_data/sample.git/
[root@Git-Server ~]# chown git.git -R /svn_data/sample.git/
```

### No.5 客户端克隆仓库

使用TortoiseGit克隆远程仓库

![clone](/assets/images/git/tclone.png)


克隆成功的提示信息

![sucess](/assets/images/git/tclone_success.png)


### 扩展

- 要方便管理公钥，用[Gitosis](https://github.com/sitaramc/gitolite)
- 要像SVN那样变态地控制权限，用[Gitolite](https://github.com/sitaramc/gitolite)

## 其他方式安装

### Debian/Ubuntu
$ apt-get install git

### Fedora
$ yum install git (up to Fedora 21)
$ dnf install git (Fedora 22 and later)

### Gentoo
$ emerge --ask --verbose dev-vcs/git

###　Arch Linux
$ pacman -S git

### openSUSE
$ zypper install git

### FreeBSD
$ cd /usr/ports/devel/git
$ make install

### Solaris 9/10/11 (OpenCSW)
$ pkgutil -i git

### Solaris 11 Express
$ pkg install developer/versioning/git

### OpenBSD
$ pkg_add git