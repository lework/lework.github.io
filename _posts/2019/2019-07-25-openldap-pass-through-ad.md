---
layout: post
title: "配置OpenLDAP传递身份验证到Active Directory"
date: "2019-07-25 12:30"
category: LDAP
tags: LDAP AD
author: lework
---
* content
{:toc}

Openldap是开源的目录服务实现，windows AD是微软的目录服务现实。现状是有的场景（应用、客户端）跟openldap结合比较容易，有的场景又是必须要用AD，所以几乎不可能弃用其中的任意一种。但同时维护两套系统意味着维护工作大量增加（不仅仅只是增加一倍，要考虑信息分别维护、同步etc等）、出错几率增加。

其中的一种较成熟、使用比较多的解决方案是：openldap使用windows AD的认证，这样只需要在AD上维护一套用户密码即可。

* 1.LDAP client;这个是实际调用ldap服务的系统，也可以是类似ldapsearch之类的client 程序
* 2.Openldap; 开源服务端，实际进程为slapd
* 3.Saslauthd;简单认证服务层的守护进程，该进程要安装在openldap服务器上
* 4.Active directory;即windows AD

![1563964600770](/assets/images/ldap/openldap-auth-ad.png)




## 系统环境

OS： `centos 7.4`

OepnLDAP: 请按照[在CentOS 7上安装OpenLDAP服务器]({{ post.url }}/2019/07/18/ldap-install/)文章进行安装部署

AD： 请按照[Windows Server 2016安装AD并开启SSL]({{ post.url }}/2019/07/24/ad-install/)文章进行安装部署

AD 需创建用户`ad_test1`,密码`Admin123`用于测试

## 配置SASLAUTHD

**安装saslauthd**

```bash
yum install cyrus-sasl cyrus-sasl-ldap cyrus-sasl-plain -y
```

**启用LDAP机制**

```bash
cat /etc/sysconfig/saslauthd
SOCKETDIR=/run/saslauthd
MECH=ldap
FLAGS="-r"
```

**配置ldap访问参数**

```bash
cat /etc/saslauthd.conf
ldap_servers: ldap://192.168.77.135
ldap_search_base: dc=lework,dc=com
ldap_filter: (|(cn=%u)(userPrincipalName=%u))
ldap_bind_dn: CN=Administrator,cn=Users,dc=lework,dc=com
ldap_password: Admin123
ldap_timeout: 10
ldap_deref: never
ldap_restart: yes
ldap_scope: sub
ldap_use_sasl: no
ldap_start_tls: no
ldap_version: 3
ldap_auth_method: bind
```

**启动saslauthd**

```bash
systemctl enable --now saslauthd  
```

## 测试身份传递验证

在上面的AD示例中，首先检查saslauthd在连接到AD时将使用的DN和密码是否有效：

```bash
ldapsearch -x -H ldap://192.168.77.135/ \
  -D cn=Administrator,cn=Users,DC=lework,DC=com \
  -w Admin123 \
  -b '' \
  -s base
```  
  
接下来检查是否可以找到示例AD用户：

```bash
ldapsearch -x -H ldap://192.168.77.135/ \
  -D cn=Administrator,cn=Users,DC=lework,DC=com \
  -w Admin123 \
  -b cn=Users,DC=lework,DC=com \
  "(userPrincipalName=ad_test1@lework.com)"
```

检查用户是否可以绑定到AD：

```bash
ldapsearch -x -H ldap://192.168.77.135/ \
  -D cn=Administrator,cn=Users,DC=lework,DC=com \
  -w Admin123 \
  -b cn=Users,DC=lework,DC=com \
  -s base \
  "(objectclass=*)"
```

如果一切正常，那么saslauthd应该能够做同样的事情：

```bash
testsaslauthd -u ad_test1@lework.com -p Admin123
0: OK "Success."
```

## sasl设定

```bash
cat /etc/sasl2/slapd.conf
mech_list: plain
pwcheck_method: saslauthd
saslauthd_path: /var/run/saslauthd/mux
```

重启openldap

```bash
systemctl slapd restart
```

绑定ad的用户到ldap中

```bash
cat >> user1.ldif <<EOF
dn: uid=ad_test1,ou=people,dc=lework,dc=com
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: top
uid: ad_test1
cn: ad_test1
sn: ad_test1
userpassword: {SASL}ad_test1@lework.com
givenname: ad_test1
mail: ad_test1@lework.com
EOF
```

> 注意密码的设置,userpassword指定的不是用户密码，如果格式为{SASL}xxx, 则将其转至Active Directory以进行身份验证用户。

向ldap中添加用户

```bash
ldapadd -x -w "123456" -D "cn=Manager,dc=lework,dc=com" -f user1.ldif
```

使用ad密码验证

```bash
ldapwhoami -x -D uid=ad_test1,ou=People,dc=lework,dc=com -w Admin123
dn:uid=ad_test1,ou=People,dc=lework,dc=com
```

可以使用lsc脚本将ad域的账号同步到ldap,具体请看[使用lsc同步openldap和ad之间的用户和组]({{ post.url }}/2019/07/25/openldap-to-ad/)

## 参考

* [https://www.hellovinoth.com/pass-through-openldap-authentication-using-sasl-to-active-directory-on-centos/](https://www.hellovinoth.com/pass-through-openldap-authentication-using-sasl-to-active-directory-on-centos/)
* [https://camratus.com/2017/01/24/openldap-lsc-active-directory-sync-and-login-pass-through/](https://camratus.com/2017/01/24/openldap-lsc-active-directory-sync-and-login-pass-through/)
* [https://blog.sys4.de/cyrus-sasl-saslauthdconf-man-page-en.html](https://blog.sys4.de/cyrus-sasl-saslauthdconf-man-page-en.html)