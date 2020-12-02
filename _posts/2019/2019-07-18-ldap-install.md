---
layout: post
title: "在CentOS 7上安装OpenLDAP服务器"
date: "2019-07-18 23:17:48"
category:  LDAP
tags: LDAP openldap
author: lework
---
* content
{:toc}

[OpenLDAP](http://www.openldap.org/)是一款轻量级目录访问协议（Lightweight Directory Access Protocol，LDAP），属于开源集中账号管理架构的实现，且支持众多系统版本，被广大互联网公司所采用。openldap-server的数据必须用原配的Berkeley DB，不能使用mysql作为后端数据库。

[phpldapadmin](http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page)是使用php语言开发的web端管理ldap的应用。

[Self Service Password](https://ltb-project.org/documentation/self-service-password)是一个Web应用，可以让用户自行更新、修改和重置LDAP中的用户密码。支持标准的LDAPv3目录服务，包括：OpenLDAP,Active Directory,OpenDS,ApacheDS等。





## 系统环境

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

systemctl enable ntpd  && systemctl start ntpd
ntpq -p
```

## 安装OpenLDAP

```bash
yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel migrationtools
```

查看OpenLDAP版本
```
slapd -VV
@(#) $OpenLDAP: slapd 2.4.44 (Jan 29 2019 17:42:45) $
	mockbuild@x86-01.bsys.centos.org:/builddir/build/BUILD/openldap-2.4.44/openldap-2.4.44/servers/slapd
```

OpenLDAP的相关配置文件信息

* /etc/openldap/slapd.conf：OpenLDAP的主配置文件，记录根域信息，管理员名称，密码，日志，权限等
* /etc/openldap/slapd.d/*：这下面是/etc/openldap/slapd.conf配置信息生成的文件，每修改一次配置信息，这里的东西就要重新生成
* /etc/openldap/schema/*：OpenLDAP的schema存放的地方
* /var/lib/ldap/*：OpenLDAP的数据文件
* /usr/share/openldap-servers/DB_CONFIG.example 模板数据库配置文件

OpenLDAP监听的端口：

* 默认监听端口：389（明文数据传输）
* 加密监听端口：636（密文数据传输）


## 配置OpenLDAP

### 配置OpenLDAP数据库

OpenLDAP默认使用的数据库是BerkeleyDB，现在来开始配置OpenLDAP数据库，使用如下命令：

```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
```

### 修改配置文件

**生成ldap管理员密码**

```
slappasswd -s 123456
{SSHA}DyaM9C7Q4cmcq0DNNsry5ObaeShv8hIa
```

**修改olcDatabase={2}hdb.ldif文件**

```bash
vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{2\}hdb.ldif

olcSuffix: dc=lework,dc=com
olcRootDN: cn=Manager,dc=lework,dc=com
olcRootPW: {SSHA}DyaM9C7Q4cmcq0DNNsry5ObaeShv8hIa

```
> 注意：其中cn=Manager中的Manager表示OpenLDAP管理员的用户名，dc是我们的组织信息，而olcRootPW表示OpenLDAP管理员的密码,用刚才我们生成的密码。

**修改olcDatabase={1}monitor.ldif文件**

```bash
vim /etc/openldap/slapd.d/cn\=config/olcDatabase\=\{1\}monitor.ldif

olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=extern
 al,cn=auth" read by dn.base="cn=Manager,dc=lework,dc=com" read by * none
```
> 注意：该修改中的dn.base是修改OpenLDAP的管理员的相关信息的。

**验证OpenLDAP的基本配置**

```bash
slaptest -u
5d30769d ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif"
5d30769d ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldif"
config file testing succeeded

```

> 校验error请忽略


**修改ldap文件权限**

```bash
chown -R ldap:ldap /var/lib/ldap/
chown -R ldap:ldap /etc/openldap/
```

### 启动OpenLDAP

```bash
systemctl enable --now slapd
systemctl status slapd.service
● slapd.service - OpenLDAP Server Daemon
   Loaded: loaded (/usr/lib/systemd/system/slapd.service; enabled; vendor preset: disabled)
   Active: active (running) since 四 2019-07-18 21:44:29 CST; 12s ago
     Docs: man:slapd
           man:slapd-config
           man:slapd-hdb
           man:slapd-mdb
           file:///usr/share/doc/openldap-servers/guide.html
  Process: 1770 ExecStart=/usr/sbin/slapd -u ldap -h ${SLAPD_URLS} $SLAPD_OPTIONS (code=exited, status=0/SUCCESS)
  Process: 1748 ExecStartPre=/usr/libexec/openldap/check-config.sh (code=exited, status=0/SUCCESS)
 Main PID: 1773 (slapd)
   Memory: 14.7M
   CGroup: /system.slice/slapd.service
           └─1773 /usr/sbin/slapd -u ldap -h ldapi:/// ldap:///

ss -natup | grep 389
tcp    LISTEN     0      128       *:389                   *:*                   users:(("slapd",pid=1773,fd=8))
tcp    LISTEN     0      128      :::389                  :::*                   users:(("slapd",pid=1773,fd=9))

```

### 导入Schema

导入基本Schema

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

**修改migrate_common.ph文件**

migrate_common.ph文件主要是用于生成ldif文件使用，修改migrate_common.ph文件，如下：

```bash
vim /usr/share/migrationtools/migrate_common.ph +71
# Default DNS domain
$DEFAULT_MAIL_DOMAIN = "lework.com";

# Default base 
$DEFAULT_BASE = "dc=lework,dc=com";

$EXTENDED_SCHEMA = 1;
```

**生成base.ldif**

```bash
mkdir /root/openldap
/usr/share/migrationtools/migrate_base.pl >/root/openldap/base.ldif

ldapadd -x -D "cn=Manager,dc=lework,dc=com" -w 123456 -f /root/openldap/base.ldif
```

> `-D`指定绑定dn `-w`指定管理员密码 `-f`指定文件


**添加用户及用户组**

默认情况下OpenLDAP是没有普通用户的，但是有一个管理员用户。管理用户就是前面我们刚刚配置的root。

现在我们把系统中的用户，添加到OpenLDAP中。为了进行区分，我们现在新加两个用户ldapuser1和ldapuser2，和两个用户组ldapgroup1和ldapgroup2.

添加用户组，使用如下命令：
```bash
groupadd ldapgroup1
groupadd ldapgroup2
```

添加用户并设置密码

```bash
useradd -g ldapgroup1 ldapuser1
useradd -g ldapgroup2 ldapuser2
echo '123456' | passwd --stdin ldapuser1
echo '123456' | passwd --stdin ldapuser2

```

把刚刚添加的用户和用户组提取出来，这包括该用户的密码和其他相关属性，如下

```bash
grep ":10[0-9][0-9]" /etc/passwd > /root/openldap/users
grep ":10[0-9][0-9]" /etc/group > /root/openldap/groups
```

根据上述生成的用户和用户组属性，使用migrate_passwd.pl文件生成要添加用户和用户组的ldif，如下：

```bash
/usr/share/migrationtools/migrate_group.pl /root/openldap/groups > /root/openldap/groups.ldif
/usr/share/migrationtools/migrate_passwd.pl /root/openldap/users > /root/openldap/users.ldif
```

**导入用户及用户组到OpenLDAP数据库**

```
ldapadd -x -w "123456" -D "cn=Manager,dc=lework,dc=com" -f /root/openldap/groups.ldif
ldapadd -x -w "123456" -D "cn=Manager,dc=lework,dc=com" -f /root/openldap/users.ldif 

```

**把OpenLDAP用户加入到用户组**

尽管我们已经把用户和用户组信息，导入到OpenLDAP数据库中了。但实际上目前OpenLDAP用户和用户组之间是没有任何关联的。

如果我们要把OpenLDAP数据库中的用户和用户组关联起来的话，我们还需要做另外单独的配置。

现在我们要把ldapuser1用户加入到ldapgroup1用户组，需要新建添加用户到用户组的ldif文件，如下

```bash
cat > /root/openldap/add_user_to_groups.ldif << "EOF"
dn: cn=ldapgroup1,ou=Group,dc=lework,dc=com
changetype: modify
add: memberuid
memberuid: ldapuser1

dn: cn=ldapgroup2,ou=Group,dc=lework,dc=com
changetype: modify
add: memberuid
memberuid: ldapuser2
EOF

ldapadd -x -w "123456" -D "cn=Manager,dc=lework,dc=com" -f /root/openldap/add_user_to_groups.ldif

```

查询添加的OpenLDAP用户组信息，如下：

```bash
ldapsearch -LLL -x -D "cn=Manager,dc=lework,dc=com" -w "123456" -b "dc=lework,dc=com" "cn=ldapgroup1"
dn: cn=ldapgroup1,ou=Group,dc=lework,dc=com
objectClass: posixGroup
objectClass: top
cn: ldapgroup1
userPassword:: e2NyeXB0fXg=
gidNumber: 1000
memberUid: ldapuser1
```

### 开启OpenLDAP日志访问功能

默认情况下OpenLDAP是没有启用日志记录功能的，但是在实际使用过程中，我们为了定位问题需要使用到OpenLDAP日志。

新建日志配置ldif文件，如下：

```bash
cat > /root/openldap/loglevel.ldif << EOF
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f /root/openldap/loglevel.ldif

```

修改rsyslog配置文件，并重启rsyslog服务，如下：

```bash
cat >> /etc/rsyslog.conf << EOF
local4.* /var/log/slapd.log
EOF

```

重启服务

```bash
systemctl restart rsyslog
systemctl restart slapd

```

使用ldapuser1认证下

```bash
ldapwhoami -x -D uid=ldapuser1,ou=People,dc=lework,dc=com -w 123456
dn:uid=ldapuser1,ou=People,dc=lework,dc=com
```

查看OpenLDAP日志，如下：

```bash
tail  /var/log/slapd.log

Jul 18 22:07:49 node130 slapd[2019]: conn=1000 fd=11 ACCEPT from IP=[::1]:54500 (IP=[::]:389)
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=0 BIND dn="uid=ldapuser1,ou=People,dc=lework,dc=com" method=128
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=0 BIND dn="uid=ldapuser1,ou=People,dc=lework,dc=com" mech=SIMPLE ssf=0
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=0 RESULT tag=97 err=0 text=
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=1 EXT oid=1.3.6.1.4.1.4203.1.11.3
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=1 WHOAMI
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=1 RESULT oid= err=0 text=
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 op=2 UNBIND
Jul 18 22:07:49 node130 slapd[2019]: conn=1000 fd=11 closed
```

## 安装phpldapadmin

[phpldapadmin](http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page)是使用php语言开发的web端管理ldap的应用。

```bash
yum --enablerepo=epel -y install phpldapadmin
```

修改配置文件

```bash
vim /etc/phpldapadmin/config.php +397

#397行取消注释，398行添加注释

$servers->setValue('login','attr','dn');
//$servers->setValue('login','attr','uid');


vim /etc/httpd/conf.d/phpldapadmin.conf +11

<IfModule mod_authz_core.c>
# Apache 2.4
Require local
#添加一行内容，指定可访问的ip段
Require ip 192.168.77.0/24
</IfModule>
```

启动httpd

```bash
systemctl enable --now httpd
```

浏览器访问phpldapadmin：

* url: http://服务器地址/phpldapadmin/
* 用户名：cn=Manager,dc=lework,dc=com
* 密码：设定的管理员密码


![phpldapadmin.png](/assets/images/ldap/phpldapadmin.png)

## 安装Self Service Password

为了解放管理员的工作，让OpenLDAP用户可以自行进行密码的修改和重置，就需要我们来搭建一套自助修改密码系统。

在此我们使用的是开源的基于php语言开发的ldap自助修改密码系统[Self Service Password](https://ltb-project.org/documentation/self-service-password)。

Self Service Password是一个Web应用，可以让用户自行更新、修改和重置LDAP中的用户密码。支持标准的LDAPv3目录服务，包括：OpenLDAP,Active Directory,OpenDS,ApacheDS等。

源码地址： https://github.com/ltb-project/self-service-password

### 安装服务

```

cat > /etc/yum.repos.d/ltb-project-noarch.repo << EOF
[ltb-project-noarch]
name=LTB project packages
baseurl=https://ltb-project.org/rpm/\$releasever/noarch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-LTB-project
EOF

rpm --import https://ltb-project.org/lib/RPM-GPG-KEY-LTB-project

yum -y install self-service-password

```

现在我们来查看下Self Service Password安装的文件，如下：

```
rpm -ql self-service-password
```

比较重要的文件
```
/etc/httpd/conf.d/self-service-password.conf    apache配置文件
/usr/share/self-service-password/conf/config.inc.php   self-service-password配置文件
```
### 修改配置文件

如果不用虚拟主机的形式，可以修改apache配置文件

```
cp /etc/httpd/conf.d/self-service-password.conf{,.bak}
cat > /etc/httpd/conf.d/self-service-password.conf << EOF
Alias /ssp /usr/local/self-service-password
 
<Directory /usr/local/self-service-password>
        AllowOverride None
        <IfVersion >= 2.3>
            Require all granted
        </IfVersion>
        <IfVersion < 2.3>
            Order Deny,Allow
            Allow from all
        </IfVersion>
        DirectoryIndex index.php
        AddDefaultCharset UTF-8
</Directory>
EOF
```

修改Self Service Password的配置文件

修改ldap连接和email

```
vim  /usr/share/self-service-password/conf/config.inc.php +37
$ldap_url = "ldap://localhost";
$ldap_starttls = false;
$ldap_binddn = "cn=Manager,dc=lework,dc=com";
$ldap_bindpw = "123456";
$ldap_base = "ou=People,dc=lework,dc=com";
$ldap_login_attribute = "uid";
$ldap_fullname_attribute = "cn";
$ldap_filter = "(&(objectClass=person)($ldap_login_attribute={login}))";


$keyphrase = "leworkRandmon";

$hash = "SSHA";

$who_change_password = "manager";


$mail_from = "admin@example.com";
$mail_from_name = "Self Service Password";
$mail_signature = "";
# Notify users anytime their password is changed
$notify_on_change = false;
# PHPMailer configuration (see https://github.com/PHPMailer/PHPMailer)
$mail_sendmailpath = '/usr/sbin/sendmail';
$mail_protocol = 'smtp';
$mail_smtp_debug = 0;
$mail_debug_format = 'error_log';
$mail_smtp_host = 'localhost';
$mail_smtp_auth = false;
$mail_smtp_user = '';
$mail_smtp_pass = '';
$mail_smtp_port = 25;
$mail_smtp_timeout = 30;
$mail_smtp_keepalive = false;
$mail_smtp_secure = 'tls';
$mail_smtp_autotls = true;
$mail_contenttype = 'text/plain';
$mail_wordwrap = 0;
$mail_charset = 'utf-8';
$mail_priority = 3;
$mail_newline = PHP_EOL;
```

> 修改ldap的连接信息, `$keyphrase` 设定一个随机字符串




重启Apache
```
systemctl restart httpd
```

访问页面

`http://服务器地址/index.php`


修改密码
![ssp.png](/assets/images/ldap/ssp.png)


在终端验证修改的密码
```
ldapwhoami -x -D uid=ldapuser1,ou=People,dc=lework,dc=com -w 123456
ldap_bind: Invalid credentials (49)
ldapwhoami -x -D uid=ldapuser1,ou=People,dc=lework,dc=com -w 12345678
dn:uid=ldapuser1,ou=People,dc=lework,dc=com
```

也可以使用邮件找回，或者短信找回。