---
layout: post
title: "Ansible 小手册系列 二（安装）"
date: "2016-11-19 15:22:56"
categories: Ansible
excerpt: "通过yum(CentOS, RHEL)安装 注：上列是centos 6.7的安装步骤，目前yum的版本是2.1的。 通过apt(Ubuntu )..."
auth: lework
---
* content
{:toc}

## 通过yum(CentOS, RHEL)安装
---

```bash
rpm -ivh  http://mirrors.ustc.edu.cn/epel/epel-release-latest-6.noarch.rpm
wget --no-check-certificate -O  /etc/yum.repos.d/epel.repo  https://lug.ustc.edu.cn/wiki/_export/code/mirrors/help/epel?codeblock=0
yum install ansible
```
> 注：上列是centos 6.7的安装步骤，目前yum的版本是2.1的。
	
## 通过apt(Ubuntu )安装
---

```bash
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get install ansible
```

> 在早期Ubuntu发行版中, “software-properties-common” 名为 “python-software-properties”.
	
## 通过 pkg (FreeBSD)安装
---

```bash
$ sudo pkg install ansible
```

## 通过pip安装
---

安装easy_install

```bash
yum –y install easy_install
```
修改easy_install源

```bash
cat >> ~/.pydistutils.cfg  <<EOF
[easy_install]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
EOF
````

修改pip源

```bash
mkdir ~/.pip
cat >>  ~/.pip/pip.conf  <<EOF
[global]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
format = columns
EOF
```
安装

```bash
easy_install pip
pip install ansible
```
## 通过源码安装
---

> cetnos 6.7安装方式

修改epel源

```bash
wget -O /etc/yum.repos.d/epel.repo   https://lug.ustc.edu.cn/wiki/_export/code/mirrors/help/epel?codeblock=0
```

修改easy_install源

```bash
cat >> ~/.pydistutils.cfg  <<EOF
[easy_install]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
EOF
```

修改pip源

```bash
mkdir ~/.pip
cat >>  ~/.pip/pip.conf  <<EOF
[global]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
format = columns
EOF
```
	
安装依赖

```bash
yum -y install gcc gcc-c++ make python-devel  python-setuptools  sshpass
easy_install pip
git clone git://github.com/ansible/ansible.git --recursive
cd ./ansible
python setup.py install
mkdir /etc/ansible/
cp examples/{ansible.cfg,hosts} /etc/ansible/
```

友情提醒：

如果下载得是release版本的zip/tar.gz文件。执行命令的时候如果出现：

```
localhost | FAILED! => {
    "failed": true, 
    "msg": "The module ping was not found in configured module paths. Additionally, core modules are missing. If this is a checkout, run 'git submodule update --init --recursive' to correct this problem."
}
```
那就 需要再次下载下面这两个仓库，放在/lib/ansible/modules/目录下，再进行安装

https://github.com/ansible/ansible-modules-core
https://github.com/ansible/ansible-modules-extras

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
