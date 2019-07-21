---
layout: post
title: "在CentOS 7上配置OpenLDAP双主复制模式"
date: "2019-07-21 18:23:00"
category: LDAP
tags: LDAP
author: lework
---
* content
{:toc}

在Multi-Master复制中，两个或多个服务器充当主服务器，所有这些服务器对LDAP目录中的任何更改都具有权威性。来自客户端的查询将在replication的帮助下分布在多个服务器上。

![master-master.png](/assets/images/ldap/master-master.png)




## 节点信息

| IP             | hostname     | role            |
| -------------- | ------------ | --------------- |
| 192.168.77.130 | ldap-master1 | OpenLDAP Master |
| 192.168.77.131 | ldap-master2 | OpenLDAP Master |


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
hostnamectl set-hostname ldap-master1
# 192.168.77.131
hostnamectl set-hostname ldap-master2
```

## 安装和配置OpenLDAP

> 所有节点都需要运行

### 安装OpenLDAP

```bash
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel

systemctl enable --now slapd
```

### 配置ldap日志

```bash
echo "local4.* /var/log/ldap.log" >> /etc/rsyslog.conf
systemctl restart rsyslog
```

### 配置openldap的多主复制

**配置数据库**

```bash
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/*
```

**开启同步模块**

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

**设置olcServerID**

> master1为1，master2为2

```bash
# ldap-master1
cat > olcserverid.ldif <<EOF
dn: cn=config
changetype: modify
add: olcServerID
olcServerID: 1
EOF

# ldap-master2
cat > olcserverid.ldif <<EOF
dn: cn=config
changetype: modify
add: olcServerID
olcServerID: 2
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f olcserverid.ldif
```

**生成ldap的密码**

```bash
slappasswd -h {SSHA} -s 123456
{SSHA}pzFz1tZBBiqb7KfU4N6NMXjl9uLBBEgB
```

**配置数据库的密码**

```bash
cat > olcdatabase.ldif <<EOF
dn: olcDatabase={0}config,cn=config
add: olcRootPW
olcRootPW: {SSHA}pzFz1tZBBiqb7KfU4N6NMXjl9uLBBEgB
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f olcdatabase.ldif
```

**配置复制**

```bash
cat > configrep.ldif <<EOF
### Update Server ID with LDAP URL ###

dn: cn=config
changetype: modify
replace: olcServerID
olcServerID: 1 ldap://192.168.77.130
olcServerID: 2 ldap://192.168.77.131

### Enable Config Replication###

dn: olcOverlay=syncprov,olcDatabase={0}config,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov

### Adding config details for confDB replication ###

dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcSyncRepl
olcSyncRepl: rid=001 provider=ldap://192.168.77.130 binddn="cn=config"
  bindmethod=simple credentials=123456 searchbase="cn=config"
  type=refreshAndPersist retry="5 5 300 5" timeout=1
olcSyncRepl: rid=002 provider=ldap://192.168.77.131 binddn="cn=config"
  bindmethod=simple credentials=123456 searchbase="cn=config"
  type=refreshAndPersist retry="5 5 300 5" timeout=1
-
add: olcMirrorMode
olcMirrorMode: TRUE
EOF
```

将配置发送给LDAP服务器

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f configrep.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "cn=config"

adding new entry "olcOverlay=syncprov,olcDatabase={0}config,cn=config"

modifying entry "olcDatabase={0}config,cn=config"
```


### 开启数据库复制

> 由于其他节点都处于复制状态，因此下面的操作在任意一个节点上执行就行了。

**为hdb数据库启用syncprov**

```bash
cat > syncprov.ldif <<EOF
dn: olcOverlay=syncprov,olcDatabase={2}hdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcSyncProvConfig
olcOverlay: syncprov
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f syncprov.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "olcOverlay=syncprov,olcDatabase={2}hdb,cn=config"
```

**为hdb数据库设置复制**

```bash
cat > olcdatabasehdb.ldif <<EOF
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=lework,dc=com
-
replace: olcRootDN
olcRootDN: cn=Manager,dc=lework,dc=com
-
replace: olcRootPW
olcRootPW: {SSHA}pzFz1tZBBiqb7KfU4N6NMXjl9uLBBEgB
-
add: olcSyncRepl
olcSyncRepl: rid=004 provider=ldap://192.168.77.130 binddn="cn=Manager,dc=lework,dc=com" bindmethod=simple
  credentials=123456 searchbase="dc=lework,dc=com" type=refreshOnly
  interval=00:00:00:10 retry="5 5 300 5" timeout=1
olcSyncRepl: rid=005 provider=ldap://192.168.77.131 binddn="cn=Manager,dc=lework,dc=com" bindmethod=simple
  credentials=123456 searchbase="dc=lework,dc=com" type=refreshOnly
  interval=00:00:00:10 retry="5 5 300 5" timeout=1
-
add: olcDbIndex
olcDbIndex: entryUUID  eq
-
add: olcDbIndex
olcDbIndex: entryCSN  eq
-
add: olcMirrorMode
olcMirrorMode: TRUE
EOF

ldapmodify -Y EXTERNAL  -H ldapi:/// -f olcdatabasehdb.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"
```

对olcDatabase={1}monitor.ldif文件进行更改

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

**添加基本schema**

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

**生成base.ldif**

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

## 测试LDAP复制

在130节点上创建账号

```bash
cat > ldaptest.ldif <<EOF
dn: uid=ldaptest,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaptest
uid: ldaptest
uidNumber: 9988
gidNumber: 100
homeDirectory: /home/ldaptest
loginShell: /bin/bash
gecos: LDAP Test User
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
EOF

ldapadd -x -w 123456 -D "cn=Manager,dc=lework,dc=com" -f ldaptest.ldif
```

搜索ldapetest用户

```bash
ldapsearch -x cn=ldaptest -b dc=lework,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=lework,dc=com> with scope subtree
# filter: cn=ldaptest
# requesting: ALL
#

# ldaptest, People, lework.com
dn: uid=ldaptest,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaptest
uid: ldaptest
uidNumber: 9988
gidNumber: 100
homeDirectory: /home/ldaptest
loginShell: /bin/bash
gecos: LDAP Test User
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

在131节点上搜索

```bash
ldapsearch -x cn=ldaptest -b dc=lework,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=lework,dc=com> with scope subtree
# filter: cn=ldaptest
# requesting: ALL
#

# ldaptest, People, lework.com
dn: uid=ldaptest,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaptest
uid: ldaptest
uidNumber: 9988
gidNumber: 100
homeDirectory: /home/ldaptest
loginShell: /bin/bash
gecos: LDAP Test User
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

在131节点上添加用户

```bash
cat > ldaptest2.ldif <<EOF
dn: uid=ldaptest2,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaptest2
uid: ldaptest2
uidNumber: 9989
gidNumber: 100
homeDirectory: /home/ldaptest2
loginShell: /bin/bash
gecos: LDAP Test User
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
EOF

ldapadd -x -w 123456 -D "cn=Manager,dc=lework,dc=com" -f ldaptest2.ldif
```

搜索用户

```bash
ldapsearch -x cn=ldaptest2 -b dc=lework,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=lework,dc=com> with scope subtree
# filter: cn=ldaptest2
# requesting: ALL
#

# ldaptest2, People, lework.com
dn: uid=ldaptest2,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaptest2
uid: ldaptest2
uidNumber: 9989
gidNumber: 100
homeDirectory: /home/ldaptest2
loginShell: /bin/bash
gecos: LDAP Test User
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

在130节点上搜索用户ldaptest2 

```bash
ldapsearch -x cn=ldaptest2 -b dc=lework,dc=com
# extended LDIF
#
# LDAPv3
# base <dc=lework,dc=com> with scope subtree
# filter: cn=ldaptest2
# requesting: ALL
#

# ldaptest2, People, lework.com
dn: uid=ldaptest2,ou=People,dc=lework,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: ldaptest2
uid: ldaptest2
uidNumber: 9989
gidNumber: 100
homeDirectory: /home/ldaptest2
loginShell: /bin/bash
gecos: LDAP Test User
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

可以看到2个节点都可以新增用户，并且同步到其他节点上。

使用ldappasswd更改用户的密码

```bash
ldappasswd -s password123 -W -D "cn=Manager,dc=lework,dc=com" -x "uid=ldaptest,ou=People,dc=lework,dc=com"
```

* -x 指定用户的dn
* -s 指定用户的密码
* -D 指定认证LDAP的dn


配置LDAP客户端也绑定到新的主服务器。

```bash
authconfig --enableldap --enableldapauth --ldapserver=192.168.77.130,192.168.77.131 --ldapbasedn="dc=lework,dc=com" --enablemkhomedir --update
```