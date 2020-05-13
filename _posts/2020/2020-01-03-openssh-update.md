---
layout: post
title: '使用源码包更新openssh'
date: '2020-01-03 20:00'
category: openssh
tags: openssl openssh
author: lework
---
* content
{:toc}

OpenSSH 是使用 SSH 透过计算机网络加密通信的实现。它是取代由 SSH Communications Security 所提供的商用版本的开放源代码方案。当前 OpenSSH 是 OpenBSD 的子项目。

- 官方网站：https://www.openssh.com
- GitHub：https://github.com/openssl/openssl

低版本有些漏洞，所以要升级。




## 升级 openssl

详情见[使用源码包更新 openssl](/2020/01/03/openssl-update/)

## centos 7.4 更新

```bash
# 当前版本
ssh -V
OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017

# 安装依赖
yum -y install pam-devel libselinux-devel

# 备份ssh配置
cp -rf /etc/ssh /etc/ssh.bak

# 设置文件权限
chmod 600 /etc/ssh/ssh_host_rsa_key
chmod 600 /etc/ssh/ssh_host_ecdsa_key
chmod 600 /etc/ssh/ssh_host_ed25519_key

# 配置sshd配置
sed -i 's/^#PermitRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^GSSAPIAuthentication/#&/' /etc/ssh/sshd_config
sed -i 's/^GSSAPICleanupCredentials/#&/' /etc/ssh/sshd_config
sed -i 's/^UsePAM/#&/' /etc/ssh/sshd_config

# 配置service, 取消notify
sed -i 's/^Type/#&/' /usr/lib/systemd/system/sshd.service

# 下载包
wget https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-8.1p1.tar.gz
tar zxf openssh-8.1p1.tar.gz

# 编译安装
cd openssh-8.1p1
./configure --prefix=/usr --with-privsep-path=/var/empty/sshd/ \
       --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/openssl/ \
       --with-default-path=/usr/local/bin:/bin:/usr/bin \
       --with-superuser-path=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin \
       --with-pam --with-selinux --disable-strip --with-md5-passwords
make
make install

# 重启服务
systemctl daemon-reload
systemctl restart sshd

# 现在版本
ssh -V
OpenSSH_8.1p1, OpenSSL 1.1.1d  10 Sep 2019

sshd -V
unknown option -- V
OpenSSH_8.1p1, OpenSSL 1.1.1d  10 Sep 2019
usage: sshd [-46DdeiqTt] [-C connection_spec] [-c host_cert_file]
            [-E log_file] [-f config_file] [-g login_grace_time]
            [-h host_key_file] [-o option] [-p port] [-u len]
```

升级完成， 最好重启下服务器。

## debian 9 更新

```bash
# 当前版本
/usr/bin/ssh -V
OpenSSH_7.4p1 Debian-10+deb9u6, OpenSSL 1.0.2r  26 Feb 2019

# 安装依赖
apt-get install libpam0g-dev libselinux1-dev

# 备份ssh配置
cp -rf /etc/ssh /etc/ssh.bak

# 设置文件权限
chmod 600 /etc/ssh/ssh_host_rsa_key
chmod 600 /etc/ssh/ssh_host_ecdsa_key
chmod 600 /etc/ssh/ssh_host_ed25519_key

# 配置sshd配置
sed -i 's/^#PermitRootLogin yes/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/^GSSAPIAuthentication/#&/' /etc/ssh/sshd_config
sed -i 's/^GSSAPICleanupCredentials/#&/' /etc/ssh/sshd_config
sed -i 's/^UsePAM/#&/' /etc/ssh/sshd_config

# 配置service, 取消notify
sed -i 's/^Type/#&/' /lib/systemd/system/ssh.service

# 下载包
wget https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-8.1p1.tar.gz
tar zxf openssh-8.1p1.tar.gz

# 编译安装
cd openssh-8.1p1
./configure --prefix=/usr --with-privsep-path=/var/empty/sshd/ \
       --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/openssl/ \
       --with-default-path=/usr/local/bin:/bin:/usr/bin \
       --with-superuser-path=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin \
       --with-pam --with-selinux --disable-strip --with-md5-passwords
make
make install

# 重启服务
systemctl daemon-reload
systemctl restart sshd

# 现在版本
ssh -V
OpenSSH_8.1p1, OpenSSL 1.1.1d  10 Sep 2019

sshd -V
unknown option -- V
OpenSSH_8.1p1, OpenSSL 1.1.1d  10 Sep 2019
usage: sshd [-46DdeiqTt] [-C connection_spec] [-c host_cert_file]
            [-E log_file] [-f config_file] [-g login_grace_time]
            [-h host_key_file] [-o option] [-p port] [-u len]
```

升级完成， 最好重启下服务器。

## 自动部署

作者也提供了自动化安装 openssh 的脚本，详情请看[openssh](https://github.com/lework/Ansible-roles/tree/master/openssh)
