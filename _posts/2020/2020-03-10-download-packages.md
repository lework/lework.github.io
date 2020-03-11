---
layout: post
title: "一键下载软件及依赖的离线包"
date: "2020-03-10 20:40"
category: linux
tags: linux
author: lework
---
* content
{:toc}

我们有时会在不能连接外网的机器上安装软件，这种情况我们只能提前下载好软件的所有依赖包，才能顺畅的安装好软件。

一般会有两种方式，一种是自建一个软件包仓库，一种是下载所有依赖包。

我编写了一个脚本来帮助大家 **一键下载软件及依赖的离线包**，接下来我们就来介绍下这个怎么用吧。



## 使用

### 脚本帮助

```
[root@node130 ~]# wget https://raw.githubusercontent.com/lework/script/master/shell/get_packages.sh
[root@node130 ~]# bash get_packages.sh

Download Packages With Dependencies Locally.
 
  Usage:
    get_packages.sh system package [package repo]
  
  Support system:
    centos6 centos7 centos8 debian8 debian9 debian10

  Example:
    get_packages.sh centos7 ansible
    get_packages.sh centos7 "python36 python36-devel"
    get_packages.sh centos7 ceph /root/ceph.repo
```

> Usage: 使用的格式，package repo 需要指定**绝对路径**
>
> Support system:  支持的系统
>
> Example:  示例

### centos7-ansible

下载 **centos7** 系统上  **ansible** 的相关依赖包

> 因为要在 centos 7 系列系统上安装 ansible

```bash
[root@node130 ~]# bash get_packages.sh centos7 ansible
[Docker] start container
Unable to find image 'centos:7' locally
7: Pulling from library/centos
ab5ef0e58194: Pull complete 
Digest: sha256:4a701376d03f6b39b8c2a8f4a8e499441b0d567f9ab9d58e4991de4472fb813c
Status: Downloaded newer image for centos:7
05f802d89c660f0bb67c41a5e3f9ffa15363c9e5a300976098e8ed8df2f32ce2

[Docker] update repo cache
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
Metadata Cache Created

[Docker] download package
Loaded plugins: fastestmirror, ovl
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package ansible.noarch 0:2.9.3-1.el7 will be installed
.......
---> Package python-ply.noarch 0:3.4-11.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                              Arch    Version             Repository
                                                                           Size
================================================================================
Installing:
 ansible                              noarch  2.9.3-1.el7         epel     17 M
Installing for dependencies:
 PyYAML                               x86_64  3.10-11.el7         base    153 k
.......
 sshpass                              x86_64  1.06-2.el7          extras   21 k

Transaction Summary
================================================================================
Install  1 Package (+23 Dependent packages)

Total download size: 22 M
Installed size: 125 M
Background downloading packages, then exiting:
warning: /tmp/package/ansible-2.9.3-1.el7.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY
Public key for ansible-2.9.3-1.el7.noarch.rpm is not installed
--------------------------------------------------------------------------------
Total                                              1.8 MB/s |  22 MB  00:12     
exiting because "Download Only" specified

[Docker] stop container
package

[Local] show file
Path: /root/package_centos7_ansible

总用量 22988
drwxr-xr-x  2 root root     4096 3月  10 21:32 .
dr-xr-x---. 4 root root      240 3月  10 21:31 ..
-rw-r--r--  1 root root 18180603 1月  21 06:13 ansible-2.9.3-1.el7.noarch.rpm
.......
-rw-r--r--  1 root root    21896 9月   8 2017 sshpass-1.06-2.el7.x86_64.rpm
```

这个脚本会先下载docker镜像 `centos:7`, 然后更新**仓库缓存**，接着下载**软件包及依赖**。最后**退出容器**打印软件包列表，软件存储在脚本运行目录的`package_centos7_ansible`目录中，拿着这些包就可以去无网的机器上安装了(`yum localinstall *.rpm`)。

### centos7-python36

下载 **centos7** 系统上  **python36** 的相关依赖包

> 因为要在 centos 7 系列系统上安装   python36 python36-devel

```bash
[root@node130 ~]# bash -x get_packages.sh centos7 "python36 python36-devel"
[Docker] start container
124acdfcaf36bd5cb75b6823b770bc90079817ec7022420f589a89ce288c0f92

[Docker] update repo cache
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
Metadata Cache Created

[Docker] download package
+ docker exec package yum install -y --downloadonly --downloaddir=/tmp/package python36 python36-devel
Loaded plugins: fastestmirror, ovl
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package python3.x86_64 0:3.6.8-10.el7 will be installed
--> Processing Dependency: python3-libs(x86-64) = 3.6.8-10.el7 for package: python3-3.6.8-10.el7.x86_64
......
---> Package perl-parent.noarch 1:0.225-244.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                    Arch       Version                   Repository
                                                                           Size
================================================================================
Installing:
 python3                    x86_64     3.6.8-10.el7              base      69 k
 python3-devel              x86_64     3.6.8-10.el7              base     215 k
......
 zip                        x86_64     3.0-11.el7                base     260 k

Transaction Summary
================================================================================
Install  2 Packages (+40 Dependent packages)

Total download size: 22 M
Installed size: 89 M
Background downloading packages, then exiting:
--------------------------------------------------------------------------------
Total                                              2.0 MB/s |  22 MB  00:11     
exiting because "Download Only" specified


[Docker] stop container
package

[Local] show file
Path: /root/package_centos7_python36

总用量 23116
drwxr-xr-x  2 root root    4096 3月  11 11:16 .
dr-xr-x---. 8 root root    4096 3月  11 11:14 ..
-rw-r--r--  1 root root  101080 7月   4 2014 dwz-0.11-3.el7.x86_64.rpm
.....
-rw-r--r--  1 root root   70736 8月  23 2019 python3-3.6.8-10.el7.x86_64.rpm
-rw-r--r--  1 root root  220244 8月  23 2019 python3-devel-3.6.8-10.el7.x86_64.rpm
```



### debian7-ceph

下载 **centos7** 系统上  **ceph** 的相关依赖包

> 因为要在 centos 7 系列系统上安装 ceph

`ceph` 安装包并不在系统包仓库中，所以我们要给出仓库文件

```bash
[root@node130 ~]# cat >> ceph.repo << EOF
[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-luminous/el7/x86_64/
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.aliyun.com/ceph/keys/release.asc
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-luminous/el7/noarch/
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.aliyun.com/ceph/keys/release.asc
[ceph-source]
name=ceph-source
baseurl=http://mirrors.aliyun.com/ceph/rpm-luminous/el7/SRPMS/
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=http://mirrors.aliyun.com/ceph/keys/release.asc
EOF
```

运行脚本

> repo 需要指定绝对路径

```bash
[root@node130 ~]# bash get_packages.sh centos7 ceph /root/ceph.repo
[Docker] start container
5d9625fd5206c8199ba0c0d75d4ef52edb488c78be37b6f5f269b3e95d13b1ce

[Docker] update repo cache
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
Metadata Cache Created

[Docker] download package
Loaded plugins: fastestmirror, ovl
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
....
---> Package libnfnetlink.x86_64 0:1.0.1-4.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                             Arch   Version               Repository
                                                                           Size
================================================================================
Installing:
 ceph                                x86_64 2:12.2.13-0.el7       ceph    3.0 k
Installing for dependencies:
 audit-libs-python                   x86_64 2.8.5-4.el7           base     76 k
......
 xfsprogs                            x86_64 4.5.0-20.el7          base    896 k
Updating for dependencies:
 device-mapper                       x86_64 7:1.02.158-2.el7_7.2  updates 294 k
 device-mapper-libs                  x86_64 7:1.02.158-2.el7_7.2  updates 322 k

Transaction Summary
================================================================================
Install  1 Package  (+117 Dependent packages)
Upgrade             (   2 Dependent packages)

Total download size: 93 M
Background downloading packages, then exiting:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
warning: /tmp/package/ceph-12.2.13-0.el7.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 460f3994: NOKEY
Public key for ceph-12.2.13-0.el7.x86_64.rpm is not installed
warning: /tmp/package/leveldb-1.12.0-11.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY
Public key for leveldb-1.12.0-11.el7.x86_64.rpm is not installed
--------------------------------------------------------------------------------
Total                                              2.4 MB/s |  93 MB  00:38     
exiting because "Download Only" specified

[Docker] stop container
package

[Local] show file
Path: /root/package_centos7_ceph

总用量 95428
drwxr-xr-x  2 root root     8192 3月  10 21:44 .
dr-xr-x---. 5 root root      285 3月  10 21:42 ..
-rw-r--r--  1 root root    78256 8月  23 2019 audit-libs-python-2.8.5-4.el7.x86_64.rpm
.......
-rw-r--r--  1 root root   917404 8月  23 2019 xfsprogs-4.5.0-20.el7.x86_64.rpm
```



### debian9-ansible

下载 **debian9** 系统上  **ansible** 的相关依赖包

> 因为要在 debian 9 系列系统上安装 ansible

```bash
[root@node130 ~]#  bash get_packages.sh debian9 ansible
[Docker] start container
Unable to find image 'debian:9' locally
9: Pulling from library/debian
c0c53f743a40: Pull complete 
Digest: sha256:ddb131307ad9c70ebf8c7962ba73c20101f68c7a511915aea3ad3b7ad47b9d20
Status: Downloaded newer image for debian:9
6c338a63c0baa639ff1c72fb6671d32efd0a08849ea3f17e8d1b69d123ecdf2a

[Docker] update repo cache
Ign:1 http://mirrors.aliyun.com/debian stretch InRelease
Get:2 http://mirrors.aliyun.com/debian-security stretch/updates InRelease [94.3 kB]
Get:3 http://mirrors.aliyun.com/debian stretch-updates InRelease [91.0 kB]
Get:4 http://mirrors.aliyun.com/debian stretch Release [118 kB]
Get:5 http://mirrors.aliyun.com/debian-security stretch/updates/main amd64 Packages [520 kB]
Get:6 http://mirrors.aliyun.com/debian stretch-updates/main amd64 Packages [27.9 kB]
Get:7 http://mirrors.aliyun.com/debian stretch Release.gpg [2410 B]
Get:8 http://mirrors.aliyun.com/debian stretch/main amd64 Packages [7083 kB]
Fetched 7937 kB in 7s (1025 kB/s)
Reading package lists...

[Docker] download package
Reading package lists...
Building dependency tree...
Reading state information...
The following additional packages will be installed:
  bzip2 ca-certificates file ieee-data krb5-locales libexpat1 libffi6 libgmp10
  ......
  python-yaml python2.7 python2.7-minimal readline-common wget xz-utils
Suggested packages:
  cowsay sshpass bzip2-doc gnutls-bin krb5-doc krb5-user python-doc python-tk
  ......
  python2.7-doc binutils binfmt-support readline-doc
Recommended packages:
  python-winrm
The following NEW packages will be installed:
  ansible bzip2 ca-certificates file ieee-data krb5-locales libexpat1 libffi6
  ......
  python-yaml python2.7 python2.7-minimal readline-common wget xz-utils
0 upgraded, 59 newly installed, 0 to remove and 0 not upgraded.
Need to get 16.2 MB of archives.
After this operation, 69.9 MB of additional disk space will be used.
Get:1 http://mirrors.aliyun.com/debian stretch/main amd64 libpython2.7-minimal amd64 2.7.13-2+deb9u3 [389 kB]
  ......
Get:59 http://mirrors.aliyun.com/debian stretch/main amd64 publicsuffix all 20190415.1030-0+deb9u1 [108 kB]
Fetched 16.2 MB in 9s (1789 kB/s)
Download complete and in download only mode

[Docker] stop container
package

[Local] show file
Path: /root/package_debian9_ansible

总用量 15988
drwxr-xr-x  2 root root    4096 3月  10 21:50 .
dr-xr-x---. 7 root root    4096 3月  10 21:50 ..
-rw-r--r--  1 root root 1675354 3月  10 21:50 ansible_2.2.1.0-2+deb9u1_all.deb
......
-rw-r--r--  1 root root  265858 3月  10 21:50 xz-utils_5.2.2-1.2+b1_amd64.deb
```

使用 `dpkg -i *.deb` 就能安装吧。