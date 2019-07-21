---
layout: post
title: "在CentOS 7上配置OpenLDAP主从复制模式"
date: "2019-07-21 17:58:00"
category: LDAP
tags: LDAP
author: lework
---
* content
{:toc}

使用Syncrepl同步模式

由于syncrepl为拉取模式（到master拉数据），所以配置文件配置slave端的slapd.conf文件即可。初始化操作2种：

* 1）通过配置文件，当开启syncrepl引擎后会到master拉数据；
* 2）从主服务器备份数据，复制到slave。当从备份数据初始化的时候，不必担心数据老，因为syncrepl会自动进行校验，然后进行相应的修改、同步。

![master-slave.png](/assets/images/ldap/master-slave.png)




## 节点信息

| IP             | hostname    | role            |
| -------------- | ----------- | --------------- |
| 192.168.77.130 | ldap-master | OpenLDAP Master |
| 192.168.77.131 | ldap-slave  | OpenLDAP Slave  |

## 初始化配置

> 所有节点都需要运行

os: `CentOS Linux release 7.4.1708`

### 关闭selinux和防火墙

```bash
setenforce 0
sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
systemctl disable firewalld.service && systemctl stop firewalld.service
systemctl stop NetworkManager && systemctl disable NetworkManager
```

### 更换系统源
```bash
sed -e 's!^#baseurl=!baseurl=!g' \
       -e  's!^mirrorlist=!#mirrorlist=!g' \
       -e 's!mirror.centos.org!mirrors.ustc.edu.cn!g' \
       -i  /etc/yum.repos.d/CentOS-Base.repo

yum install -y epel-release
sed -e 's!^mirrorlist=!#mirrorlist=!g' \
	-e 's!^#baseurl=!baseurl=!g' \
	-e 's!^metalink!#metalink!g' \
	-e 's!//download\.fedoraproject\.org/pub!//mirrors.ustc.edu.cn!g' \
	-e 's!http://mirrors\.ustc!https://mirrors.ustc!g' \
	-i /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel-testing.repo
```

### 同步时间

```bash
yum install -y ntpdate ntp
ntpdate 0.cn.pool.ntp.org
hwclock --systohc

cat <<EOF>> /etc/ntp.conf
driftfile /var/lib/ntp/drift
server 0.cn.pool.ntp.org
server 1.cn.pool.ntp.org
server 2.cn.pool.ntp.org
server 3.cn.pool.ntp.org
EOF

systemctl enable --now ntpd
ntpq -p
```

### 更改hostname

```bash
# 192.168.77.130
hostnamectl set-hostname ldap-master
# 192.168.77.131
hostnamectl set-hostname ldap-slave
```

## 安装和配置OpenLDAP

> 所有节点都需要运行

### 安装OpenLDAP

```bash
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel

systemctl enable --now slapd

slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Jan 29 2019 17:42:45) $
	mockbuild@x86-01.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```

### 配置OpenLDAP

**生成`LDAP`管理员密码**

```bash
slappasswd -h {SSHA} -s 123456
{SSHA}SBMUfCI9kzumIDGbUEpR7yZC1lqSzZD4
```

**设定数据库**

```bash
cat > db.ldif <<EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=lework,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=lework,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}SBMUfCI9kzumIDGbUEpR7yZC1lqSzZD4
EOF

ldapmodify -Y EXTERNAL  -H ldapi:/// -f db.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"
```
> olcRootPW 使用上面生成的密码

修改`/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif`, 不需要手动修改文件，使用更新配置的方式更改。

```bash
cat > monitor.ldif <<EOF
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=Manager,dc=lework,dc=com" read by * none
EOF

ldapmodify -Y EXTERNAL  -H ldapi:/// -f monitor.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"
```

**配置ldap数据库**

```bash
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/*
```

**导入基础schema**

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

**配置openldap基础的数据库**

```bash
cat > base.ldif <<EOF
dn: dc=lework,dc=com
dc: lework
objectClass: top
objectClass: domain

dn: cn=Manager,dc=lework,dc=com
objectClass: organizationalRole
cn: Manager
description: LDAP Manager

dn: ou=People,dc=lework,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=lework,dc=com
objectClass: organizationalUnit
ou: Group
EOF

ldapadd -x -w 123456 -D "cn=Manager,dc=lework,dc=com" -f base.ldif
```

**开启日志**

```bash
cat > loglevel.ldif << EOF
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f loglevel.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

echo 'local4.* /var/log/slapd.log' >> /etc/rsyslog.conf

systemctl restart rsyslog
systemctl restart slapd
```


## 配置master

> 在master节点上执行

创建一个对所有LDAP对象具有读访问权限的用户,用作`slave`访问`master`

```bash
cat > rpuser.ldif <<EOF
dn: uid=rpuser,dc=lework,dc=com
objectClass: simpleSecurityObject
objectclass: account
uid: rpuser
description: Replication User
userPassword: root1234
EOF

ldapadd -x -w 123456 -D "cn=Manager,dc=lework,dc=com" -f rpuser.ldif
```

开启syncprov module

```bash
cat >syncprov_mod.ldif <<EOF
dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulePath: /usr/lib64/openldap
olcModuleLoad: syncprov.la
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov_mod.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=module,cn=config"
```

为每个目录开启syncprov 

```bash
cat >syncprov.ldif <<EOF
dn: olcOverlay=syncprov,olcDatabase={2}hdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
olcSpSessionLog: 100
EOF

ldapadd -Y EXTERNAL -H ldapi:/// -f syncprov.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "olcOverlay=syncprov,olcDatabase={2}hdb,cn=config"
```

## 配置slave

> 在slave节点上执行

配置同步

```bash
cat >syncrepl.ldif <<EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=001
  provider=ldap://192.168.77.130:389/
  bindmethod=simple
  binddn="uid=rpuser,dc=lework,dc=com"
  credentials=root1234
  searchbase="dc=lework,dc=com"
  scope=sub
  schemachecking=on
  type=refreshAndPersist
  retry="30 5 300 3"
  interval=00:00:05:00
EOF

ldapmodify -Y EXTERNAL  -H ldapi:/// -f syncrepl.ldif
```

>  rid = xxx是每个服务器唯一的三位数字。

## 测试LDAP的复制

在master上添加测试账号

```bash
cat > ldaprptest.ldif <<EOF
dn: uid=ldaprptest,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaprptest
uid: ldaprptest
uidNumber: 9988
gidNumber: 100
homeDirectory: /home/ldaprptest
loginShell: /bin/bash
gecos: LDAP Replication Test User
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
EOF

ldapadd -x -w 123456 -D "cn=Manager,dc=lework,dc=com" -f ldaprptest.ldif
adding new entry "uid=ldaprptest,ou=People,dc=lework,dc=com"
```

在slave中搜索用户

```bash
ldapsearch -x cn=ldaprptest -b dc=lework,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=lework,dc=com> with scope subtree
# filter: cn=ldaprptest
# requesting: ALL
#

# ldaprptest, People, lework.com
dn: uid=ldaprptest,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaprptest
uid: ldaprptest
uidNumber: 9988
gidNumber: 100
homeDirectory: /home/ldaprptest
loginShell: /bin/bash
gecos: LDAP Replication Test User
userPassword:: e2NyeXB0fXg=
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

客户端绑定slave

```bash
authconfig --enableldap --enableldapauth --ldapserver=192.168.77.130,192.168.77.131 --ldapbasedn="dc=lework,dc=com" --enablemkhomedir --update
```