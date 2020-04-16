---
layout: post
title: "使用源码包更新openssl"
date: "2020-01-03 19:00"
category: openssl
tags: openssl
author: lework
---
* content
{:toc}

在计算机网络上，OpenSSL是一个开放源代码的软件库包，应用程序可以使用这个包来进行安全通信，避免窃听，同时确认另一端连线者的身份。这个包广泛被应用在互联网的网页服务器上。 其主要库是以C语言所写成，实现了基本的加密功能，实现了SSL与TLS协议。

低版本有些漏洞，并且openssh，nginx 的 http2 都需要新版本，所以要升级版本。




## centos 7.4 更新

```bash
# 当前版本
# openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017

# 删除旧版本包
yum remove openssl

# 安装依赖
yum install gcc gcc-c++ autoconf automake zlib zlib-devel pcre-devel 

# 下载包
wget https://www.openssl.org/source/openssl-1.1.1d.tar.gz
tar -zxf openssl-1.1.1d.tar.gz 

# 编译
cd ./openssl-1.1.1d/
./config --prefix=/usr/local/openssl shared zlib
make
make install

# 创建软链接
ln -sv /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -sv /usr/local/openssl/include/openssl /usr/include/openssl

# 设置动态库地址
echo '/usr/local/openssl/lib' > /etc/ld.so.conf.d/openssl-ld.conf
ldconfig -v

# 查看版本
openssl version
OpenSSL 1.1.1d  10 Sep 2019
```

升级完成， 最好重启下服务器。


## debian 9 更新

```bash
# 当前版本
# openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017

# 删除旧版本包
apt-get --purge remove openssl

# 安装依赖
apt-get update
apt-get install build-essential checkinstall zlib1g-dev -y

# 下载包
wget https://www.openssl.org/source/openssl-1.1.1d.tar.gz
tar -zxf openssl-1.1.1d.tar.gz 

# 编译
cd ./openssl-1.1.1d/
./config --prefix=/usr/local/openssl shared zlib
make
make install

# 创建软链接
ln -sv /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -sv /usr/local/openssl/include/openssl /usr/include/openssl

# 设置动态库地址
echo '/usr/local/openssl/lib' > /etc/ld.so.conf.d/openssl-ld.conf
ldconfig -v

# 查看版本
openssl version
OpenSSL 1.1.1d  10 Sep 2019
```

升级完成， 最好重启下服务器。

## 自动部署

作者也提供了自动化安装openssl的脚本，详情请看[openssl](https://github.com/lework/Ansible-roles/tree/master/openssl)
