---
layout: post
title: "使用lsc同步openldap和ad之间的用户和组"
date: "2019-07-25 11:50"
category: LDAP
tags: LDAP AD
author: lework
---
* content
{:toc}

使用[lsc](http://ltb-project.org)项目同步openldap与AD之间的用户信息

什么是LSC？
Ldap同步连接器通过在源和目标引用之间读取，转换和比较这些数据来同步来自任何数据源（包括数据库，LDAP目录或文件）的数据。然后，可以使用这些连接器将数据源连续同步到目录，进行单次导入或仅通过输出CSV或LDIF格式报告来比较差异。

LSC提供了一个基于脚本语言的强大转换引擎，可以轻松地动态处理数据。

包含 各种身份管理功能以用于特定于目录的兼容性 - 最明显的是Active Directory（更改密码，帐户状态，上次登录等）。

LSC是一个用Java编写的开源项目，可以在[BSD许可](http://www.opensource.org/licenses/bsd-license.php)下获得。




## 系统环境

OS： `centos 7.4`

OpenLDAP: 请按照[在CentOS 7上安装OpenLDAP服务器]({{ post.url }}/2019/07/18/ldap-install/)文章进行安装部署，并添加2个测试用户和组

AD： 请按照[Windows Server 2016安装AD并开启SSL]({{ post.url }}/2019/07/24/ad-install/)文章进行安装部署，并添加2个测试用户

## 安装lsc

```bash
cat > /etc/yum.repos.d/lsc-project.repo << EOF
[lsc-project]
name=LSC project packages
baseurl=http://lsc-project.org/rpm/noarch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-LTB-project
EOF

rpm --import http://ltb-project.org/wiki/lib/RPM-GPG-KEY-LTB-project
yum install -y lsc java
```

## 配置lsc

### OpenLDAP同步到AD

LSC会将OpenLDAP中的数据源同步到Active Directory，即lsc.xml的内容：

```bash
echo '192.168.77.135 WIN-V5SBNPSNFOM.lework.com' >> /etc/hosts
mkdir /etc/lsc/openldap2ad/
cp /etc/lsc/logback.xml /etc/lsc/openldap2ad

cat /etc/lsc/openldap2ad/lsc.xml
<?xml version="1.0" encoding="utf-8"?>

<lsc xmlns="http://lsc-project.org/XSD/lsc-core-2.1.xsd" revision="0">  
  <connections>
    <ldapConnection>
      <name>ad</name>  
      <url>ldaps://WIN-V5SBNPSNFOM.lework.com:636/dc=lework,dc=com</url>  
      <username>administrator@lework.com</username>  
      <password>Admin123</password>  
      <authentication>SIMPLE</authentication>  
      <pageSize>1000</pageSize>
    </ldapConnection>  
    <ldapConnection>
      <name>openldap</name>  
      <url>ldap://127.0.0.1:389/dc=lework,dc=com</url>  
      <username>cn=Manager,dc=lework,dc=com</username>  
      <password>123456</password>  
      <authentication>SIMPLE</authentication>
    </ldapConnection>
  </connections>  
  <tasks>
    <task>
      <name>a-PeopleSyncTask</name>  
      <bean>org.lsc.beans.SimpleBean</bean>  
      <ldapSourceService>
        <name>openldap-source-service</name>  
        <connection reference="openldap"/>  
        <baseDn>ou=people,dc=lework,dc=com</baseDn>  
        <pivotAttributes>
          <string>uid</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>description</string>  
          <string>cn</string>  
          <string>sn</string>  
          <string>givenName</string>  
          <string>displayName</string>  
          <string>userPassword</string>  
          <string>objectClass</string>  
          <string>uid</string>  
          <string>mail</string>  
          <string>title</string>
        </fetchedAttributes>  
        <getAllFilter><![CDATA[(objectClass=inetOrgPerson)]]></getAllFilter>  
        <getOneFilter><![CDATA[(&(objectClass=inetOrgPerson)(uid={uid}))]]></getOneFilter>  
        <cleanFilter><![CDATA[(&(objectClass=inetOrgPerson)(uid={sAMAccountName}))]]></cleanFilter>
      </ldapSourceService>  
      <ldapDestinationService>
        <name>ad-dst-service</name>  
        <connection reference="ad"/>  
        <baseDn>CN=Users,DC=lework,DC=com</baseDn>  
        <pivotAttributes>
          <string>samAccountName</string>
        </pivotAttributes>
        <fetchedAttributes>
          <string>description</string>  
          <string>cn</string>  
          <string>sn</string>  
          <string>givenName</string>  
          <string>objectClass</string>  
          <string>title</string>  
          <string>displayName</string>  
          <string>samAccountName</string>  
          <string>userPrincipalName</string>  
          <string>unicodePwd</string>  
          <string>pwdLastSet</string>  
          <string>userAccountControl</string>
        </fetchedAttributes>  
        <getAllFilter><![CDATA[(objectClass=user)]]></getAllFilter>  
        <getOneFilter><![CDATA[(&(objectClass=user)(sAMAccountName={uid}))]]></getOneFilter>
      </ldapDestinationService>  
      <propertiesBasedSyncOptions>
        <mainIdentifier>"CN=" + srcBean.getDatasetFirstValueById("cn") + ", CN=Users,DC=lework,DC=com"</mainIdentifier>  
        <defaultDelimiter>;</defaultDelimiter>  
        <defaultPolicy>FORCE</defaultPolicy>  
        <conditions>
          <create>true</create>  
          <update>true</update>  
          <delete>false</delete>  
          <changeId>true</changeId>
        </conditions>
        <dataset>
          <name>description</name>  
          <policy>FORCE</policy>  
          <forceValues>
            <string>js:srcBean.getDatasetFirstValueById("sn").toUpperCase() + " (" + srcBean.getDatasetFirstValueById("mail") + ")"</string>
          </forceValues>
        </dataset>  
        <dataset>
          <name>samAccountName</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>js:srcBean.getDatasetFirstValueById("uid")</string>
          </createValues>
        </dataset>  
        <dataset>
          <name>userPrincipalName</name>  
          <policy>FORCE</policy>  
          <forceValues>
            <string>js:srcBean.getDatasetFirstValueById("uid") + "@lework.com"</string>
          </forceValues>
        </dataset>  
        <dataset>
          <name>objectClass</name>  
          <policy>FORCE</policy>  
          <createValues> 
            <string>"user"</string>  
            <string>"organizationalPerson"</string>  
            <string>"person"</string>  
            <string>"top"</string>
          </createValues>
        </dataset>  
        <dataset>
          <name>userAccountControl</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>AD.userAccountControlSet( "0", [AD.UAC_SET_NORMAL_ACCOUNT])</string>
          </createValues>
        </dataset>  
        <dataset>
          <!-- unicodePwd = "changeit" at creation (requires SSL connection to AD) -->  
          <name>unicodePwd</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>AD.getUnicodePwd("Admin123")</string>
          </createValues>
        </dataset>  
        <dataset>
          <!-- pwdLastSet = 0 to force user to change password on next connection -->  
          <name>pwdLastSet</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>"0"</string>
          </createValues>
        </dataset>
      </propertiesBasedSyncOptions>
    </task>
    <task>
      <name>b-GroupSyncTask</name>  
      <bean>org.lsc.beans.SimpleBean</bean>  
      <asyncLdapSourceService>
        <name>group-source-service</name>  
        <connection reference="openldap"/>  
        <baseDn>ou=Group,dc=lework,dc=com</baseDn>  
        <pivotAttributes>
          <string>cn</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>cn</string>  
          <string>description</string>  
          <string>memberUid</string>
        </fetchedAttributes>  
        <getAllFilter><![CDATA[(objectClass=posixGroup)]]></getAllFilter>  
        <getOneFilter><![CDATA[(&(objectClass=posixGroup)(cn={cn}))]]></getOneFilter>  
        <cleanFilter><![CDATA[(&(objectClass=posixGroup)(cn={cn}))]]></cleanFilter>  
        <serverType>OpenLDAP</serverType>
      </asyncLdapSourceService>  
      <ldapDestinationService>
        <name>group-dst-service</name>  
        <connection reference="ad"/>  
        <baseDn>CN=Users,DC=lework,DC=com</baseDn>  
        <pivotAttributes>
          <string>cn</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>cn</string>  
          <string>description</string>  
          <string>member</string>  
          <string>objectClass</string>
        </fetchedAttributes>  
        <getAllFilter><![CDATA[(objectClass=group)]]></getAllFilter>  
        <getOneFilter><![CDATA[(&(objectClass=group)(cn={cn}))]]></getOneFilter>
      </ldapDestinationService>  
      <propertiesBasedSyncOptions>
        <mainIdentifier>js:"cn=" + javax.naming.ldap.Rdn.escapeValue(srcBean.getDatasetFirstValueById("cn")) + ",CN=Users,DC=lework,DC=com"</mainIdentifier>  
        <defaultDelimiter>;</defaultDelimiter>  
        <defaultPolicy>FORCE</defaultPolicy>  
        <conditions>
          <create>true</create>  
          <update>true</update>  
          <delete>false</delete>  
          <changeId>true</changeId>
        </conditions>  
        <dataset>
          <name>objectclass</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>"group"</string>  
            <string>"top"</string>
          </createValues>
        </dataset>
        <dataset>
          <name>member</name>  
          <policy>FORCE</policy>  
          <forceValues>
            <string> <![CDATA[
              rdjs:
                var membersSrcDn = srcBean.getAttributeValuesById("memberUid").toArray();
                var membersDstDn = [];

                for  (var i=0; i<membersSrcDn.length; i++) {
                  try {
                     membersSrcDn[i] = ldap.attribute(ldap.list("CN=Users","(sAMAccountName=" + (membersSrcDn[i]) + ")").get(0), 'distinguishedName').get(0)
                  } catch (e) {
                     membersSrcDn[i]=null;
                  }
                }
                var j=0;
                for (var i=0; i<membersSrcDn.length; i++) {
                  if (membersSrcDn[i]!=null) membersDstDn[j++]=membersSrcDn[i];
                }
                membersDstDn;
             ]]> </string>
          </forceValues>
        </dataset>
      </propertiesBasedSyncOptions>
    </task>
  </tasks>
</lsc>
```

lsc文件主要配置

```bash
<connections></connections>  配置ldap的连接信息
<pageSize></pageSize>  指定了数目
<tasks></tasks>  配置的是同步任务

task 具体配置
<name>a-People</name>   指定task名称
<ldapSourceService></ldapSourceService>  指定task的源ldap配置
<ldapDestinationService></ldapDestinationService> 指定task的目的ldap配置
<propertiesBasedSyncOptions>
</propertiesBasedSyncOptions> 同步属性操作
<mainIdentifier></mainIdentifier> 指定同步目的的dn位置

AD.getUnicodePwd("Admin123")  指定openldap同步到ad域中的用户密码

<conditions>    这个是指定对对象的操作
  <create>true</create>    可以创建
  <update>true</update>    可以更新
  <delete>false</delete>   可以删除，这个选项可以保持src和dest ldap的数据一致
  <changeId>true</changeId> 可以修改
</conditions>

```

针对`member` 属性js脚本的说明

* 我们在源代码中获得DN的成员`membersSrcDn`
* 对于每个值,在源LDAP(ad)中进行搜索以获取其`distinguishedName`,将获取的值赋值给membersSrcDn数组，如果没找到则为`null`.
* 将不是`null`的成员复制给`membersDstDn`
* 该`membersDstDn`数组返回给LSC

简而言之，就是从AD传来的用户组里的用户是否在OpenLDAP中存在，如果存在，就在OpenLDAP中设置用户组关系，不存在就不设置。


**设置ad域的证书**
> 因为需要更改ad域中的用户密码，必须使用ldaps连接才能更改。

可以通过在Active Directory服务器上执行此命令来导出证书

```bash
certutil -ca.cert client.crt
```

将到处的证书放在`/root/openldap/client.crt`

导入证书

```bash
cd "$(dirname $(dirname `readlink -f /etc/alternatives/java`))/lib/security/"

../../bin/keytool -import -file /root/openldap/client.crt -keystore jssecacerts
输入密钥库口令:
所有者: CN=lework-WIN-V5SBNPSNFOM-CA, DC=lework, DC=com
发布者: CN=lework-WIN-V5SBNPSNFOM-CA, DC=lework, DC=com
序列号: 29be7c115db30cb0406ef332495fefa0
有效期为 Wed Jul 24 18:21:06 CST 2019 至 Tue Jul 24 18:31:06 CST 2029
证书指纹:
         MD5:  16:07:17:D6:A1:CE:F1:1C:45:90:DF:5E:5F:5C:D9:93
         SHA1: D5:3A:9C:95:FF:5E:A9:62:38:AA:DF:8B:9E:DA:6A:EC:84:4C:8E:76
         SHA256: 6D:97:5C:5D:5A:F6:92:5C:4E:0B:60:2A:FC:E5:4C:4C:21:16:6F:E5:C0:96:C3:35:4F:D3:F8:36:59:58:41:46
签名算法名称: SHA256withRSA
主体公共密钥算法: 2048 位 RSA 密钥
版本: 3

扩展:

#1: ObjectId: 1.3.6.1.4.1.311.21.1 Criticality=false
0000: 02 01 00                                           ...


#2: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:true
  PathLen:2147483647
]

#3: ObjectId: 2.5.29.15 Criticality=false
KeyUsage [
  DigitalSignature
  Key_CertSign
  Crl_Sign
]

#4: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: D3 14 63 41 56 1F CB C4   E6 1B 2B E5 90 FD 86 B1  ..cAV.....+.....
0010: 85 E3 AA D0                                        ....
]
]

是否信任此证书? [否]: 是
证书已添加到密钥库中
```

> 口令为域密码

**执行同步**

```bash
# 测试任务
/usr/bin/lsc -f /etc/lsc/openldap2ad -s all -c all -n
......
七月 25 11:35:21 - INFO  - Starting sync for a-PeopleSyncTask
七月 25 11:35:23 - INFO  - All entries: 2, to modify entries: 2, successfully modified entries: 0, errors: 0
七月 25 11:35:23 - INFO  - Starting clean for a-PeopleSyncTask
七月 25 11:35:23 - INFO  - All entries: 6, to modify entries: 0, successfully modified entries: 0, errors: 0
七月 25 11:35:23 - INFO  - Starting sync for b-GroupSyncTask
七月 25 11:35:23 - INFO  - All entries: 2, to modify entries: 2, successfully modified entries: 0, errors: 0
七月 25 11:35:23 - INFO  - Starting clean for b-GroupSyncTask
七月 25 11:35:23 - INFO  - All entries: 20, to modify entries: 0, successfully modified entries: 0, errors: 0

# 执行同步
/usr/bin/lsc -f /etc/lsc/openldap2ad -s all -c all
.....
七月 25 11:36:15 - INFO  - Starting sync for a-PeopleSyncTask
七月 25 11:36:17 - INFO  - # Adding new object CN=ldapuser1, CN=Users,DC=lework,DC=com for a-PeopleSyncTask
# Thu Jul 25 11:36:17 CST 2019
dn: CN=ldapuser1, CN=Users,DC=lework,DC=com
changetype: add
unicodePwd:: IgBBAGQAbQBpAG4AMQAyADMAIgA=
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top
objectClass: shadowAccount
samAccountName: ldapuser1
description: LDAPUSER1 ()
cn: ldapuser1
sn: ldapuser1
userPrincipalName: ldapuser1@lework.com
userAccountControl: 512
pwdLastSet: 0

七月 25 11:36:17 - INFO  - # Adding new object CN=ldapuser2, CN=Users,DC=lework,DC=com for a-PeopleSyncTask
# Thu Jul 25 11:36:17 CST 2019
dn: CN=ldapuser2, CN=Users,DC=lework,DC=com
changetype: add
unicodePwd:: IgBBAGQAbQBpAG4AMQAyADMAIgA=
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: top
objectClass: shadowAccount
samAccountName: ldapuser2
description: LDAPUSER2 ()
cn: ldapuser2
sn: ldapuser2
userPrincipalName: ldapuser2@lework.com
userAccountControl: 512
pwdLastSet: 0

七月 25 11:36:17 - INFO  - All entries: 2, to modify entries: 2, successfully modified entries: 2, errors: 0
七月 25 11:36:17 - INFO  - Starting clean for a-People
七月 25 11:36:17 - INFO  - All entries: 8, to modify entries: 0, successfully modified entries: 0, errors: 0
七月 25 11:36:17 - INFO  - Starting sync for b-GroupSyncTask
七月 25 11:36:17 - INFO  - # Adding new object cn=ldapgroup2,CN=Users,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:36:17 CST 2019
dn: cn=ldapgroup2,CN=Users,DC=lework,DC=com
changetype: add
member: CN=ldapuser2,CN=Users,DC=lework,DC=com
objectClass: group
objectClass: top
cn: ldapgroup2

七月 25 11:36:17 - INFO  - # Adding new object cn=ldapgroup1,CN=Users,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:36:17 CST 2019
dn: cn=ldapgroup1,CN=Users,DC=lework,DC=com
changetype: add
member: CN=ldapuser1,CN=Users,DC=lework,DC=com
objectClass: group
objectClass: top
cn: ldapgroup1

七月 25 11:36:17 - INFO  - All entries: 2, to modify entries: 2, successfully modified entries: 2, errors: 0
七月 25 11:36:17 - INFO  - Starting clean for b-GroupSyncTask
七月 25 11:36:17 - INFO  - All entries: 22, to modify entries: 0, successfully modified entries: 0, errors: 0
```

可以看到同步了2个用户和2个组，没有清理任务数据

AD中也能看到同步的用户和组，注意，同步的用户(密码：Admin123)首次登陆需要更改密码
![openldap-to-ad1.png](/assets/images/ldap/openldap-to-ad1.png)

### AD同步到OpenLDAP

LSC会将Active Directory中的数据源同步到OpenLDAP，即lsc.xml的内容：

```bash
mkdir /etc/lsc/ad2openldap/
cp /etc/lsc/logback.xml /etc/lsc/openldap2ad

cat /etc/lsc/ad2openldap/lsc.xml
<?xml version="1.0" encoding="utf-8"?>

<lsc xmlns="http://lsc-project.org/XSD/lsc-core-2.1.xsd" revision="0">  
  <connections>
    <ldapConnection>
      <name>ad</name>  
      <url>ldap://WIN-V5SBNPSNFOM.lework.com:389/dc=lework,dc=com</url>  
      <username>CN=Administrator,CN=Users,DC=lework,DC=com</username>  
      <password>Admin123</password>  
      <authentication>SIMPLE</authentication>  
      <referral>IGNORE</referral>  
      <derefAliases>NEVER</derefAliases>  
      <version>VERSION_3</version>  
      <pageSize>1000</pageSize>  
      <factory>com.sun.jndi.ldap.LdapCtxFactory</factory>  
      <tlsActivated>false</tlsActivated>
    </ldapConnection>  
    <ldapConnection>
      <name>openldap</name>  
      <url>ldap://127.0.0.1:389/dc=lework,dc=com</url>  
      <username>cn=Manager,dc=lework,dc=com</username>  
      <password>123456</password>  
      <authentication>SIMPLE</authentication>  
      <referral>THROW</referral>  
      <derefAliases>NEVER</derefAliases>  
      <version>VERSION_3</version>  
      <pageSize>-1</pageSize>  
      <factory>com.sun.jndi.ldap.LdapCtxFactory</factory>  
      <tlsActivated>false</tlsActivated>
    </ldapConnection>
  </connections>  
  <tasks>
    <task>
      <name>a-PeopleSyncTask</name>  
      <bean>org.lsc.beans.SimpleBean</bean>  
      <ldapSourceService>
        <name>ad-source-service</name>  
        <connection reference="ad"/>  
        <baseDn>CN=Users,DC=lework,DC=com</baseDn>  
        <pivotAttributes>
          <string>samAccountName</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>description</string>  
          <string>cn</string>  
          <string>sn</string>  
          <string>givenName</string>  
          <string>mail</string>  
          <string>samAccountName</string>  
          <string>userPrincipalName</string>
        </fetchedAttributes>  
        <getAllFilter>(&amp;(objectClass=user)(!(cn=Administrator))(!(cn=Guest))(!(cn=DefaultAccount))(!(cn=krbtgt)))</getAllFilter>  
        <getOneFilter>(&amp;(objectClass=user)(samAccountName={samAccountName}))</getOneFilter>  
        <cleanFilter>(&amp;(objectClass=user)(samAccountName={uid}))</cleanFilter>
      </ldapSourceService>  
      <ldapDestinationService>
        <name>opends-dst-service</name>  
        <connection reference="openldap"/>  
        <baseDn>ou=People,dc=lework,dc=com</baseDn>  
        <pivotAttributes>
          <string>uid</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>description</string>  
          <string>cn</string>  
          <string>sn</string>  
          <string>userPassword</string>  
          <string>objectClass</string>  
          <string>uid</string>  
          <string>mail</string>
        </fetchedAttributes>  
        <getAllFilter>(objectClass=inetorgperson)</getAllFilter>  
        <getOneFilter>(&amp;(objectClass=inetorgperson)(uid={samAccountName}))</getOneFilter>
      </ldapDestinationService>  
      <propertiesBasedSyncOptions>
        <mainIdentifier>"uid=" + srcBean.getDatasetFirstValueById("samAccountName") + ",ou=People,dc=lework,dc=com"</mainIdentifier>  
        <defaultDelimiter>;</defaultDelimiter>  
        <defaultPolicy>FORCE</defaultPolicy>  
        <conditions>
          <create>true</create>  
          <update>true</update>  
          <delete>false</delete>  
          <changeId>true</changeId>
        </conditions>  
        <dataset>
          <name>objectClass</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>"organizationalPerson"</string> 
            <string>"inetOrgPerson"</string>  
            <string>"person"</string>
            <string>"top"</string>
          </createValues>
        </dataset>  
        <dataset>
          <name>description</name>  
          <policy>FORCE</policy>  
          <forceValues>
            <string>js:srcBean.getDatasetFirstValueById("cn").toUpperCase() + " (" + srcBean.getDatasetFirstValueById("userPrincipalName") + ")"</string>
          </forceValues>
        </dataset>
        <dataset>
          <name>userPassword</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>js:"{SASL}" + srcBean.getDatasetFirstValueById("userPrincipalName")</string>
          </createValues>
        </dataset>  
        <dataset>
          <name>sn</name>  
          <policy>FORCE</policy>  
          <createValues>
            <string>js:srcBean.getDatasetFirstValueById("cn")</string>
          </createValues>
        </dataset>  
        <dataset>
          <name>uid</name>  
          <policy>KEEP</policy>  
          <createValues>
            <string>js:srcBean.getDatasetFirstValueById("cn")</string>
          </createValues>
        </dataset>
        <dataset>
          <name>mail</name>
          <policy>FORCE</policy>
          <forceValues>
            <string>js:srcBean.getDatasetFirstValueById("mail")</string>
          </forceValues>
        </dataset>
      </propertiesBasedSyncOptions>
    </task>
    <task>
      <name>b-GroupSyncTask</name>  
      <bean>org.lsc.beans.SimpleBean</bean>  
      <ldapSourceService>
        <name>ad-src-service</name>  
        <connection reference="ad"/>  
        <baseDn>CN=Users,DC=lework,DC=com</baseDn>  
        <pivotAttributes>
          <string>cn</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>cn</string>  
          <string>member</string>
          <string>description</string>
        </fetchedAttributes>  
        <getAllFilter>(&amp;(objectClass=group)(member=*))</getAllFilter>  
        <getOneFilter>(&amp;(objectClass=group)(cn={cn}))</getOneFilter>  
        <cleanFilter>(&amp;(objectClass=group)(cn={cn}))</cleanFilter>  
        <interval>100</interval>
      </ldapSourceService>  
      <ldapDestinationService>
        <name>openldap-dst-service</name>  
        <connection reference="openldap"/>  
        <baseDn>ou=Group,DC=lework,DC=com</baseDn>  
        <pivotAttributes>
          <string>cn</string>
        </pivotAttributes>  
        <fetchedAttributes>
          <string>cn</string>  
          <string>member</string>
          <string>objectClass</string>
          <string>description</string>
        </fetchedAttributes>  
        <getAllFilter>(objectClass=groupOfNames)</getAllFilter>  
        <getOneFilter>(&amp;(objectClass=groupOfNames)(cn={cn}))</getOneFilter>
      </ldapDestinationService>  
      <propertiesBasedSyncOptions>
        <mainIdentifier>"cn=" + javax.naming.ldap.Rdn.escapeValue(srcBean.getDatasetFirstValueById("cn")) + ",ou=Group,DC=lework,DC=com"</mainIdentifier>  
        <defaultDelimiter>;</defaultDelimiter>  
        <defaultPolicy>FORCE</defaultPolicy>  
        <conditions>
          <create>true</create>  
          <update>true</update>  
          <delete>false</delete>  
          <changeId>true</changeId>
        </conditions>
        <dataset>
          <name>objectClass</name>  
          <policy>FORCE</policy>  
          <forceValues>
            <string>"groupOfNames"</string>  
            <string>"top"</string>
          </forceValues>  
          <delimiter>$</delimiter>
        </dataset>  
        <dataset>
          <name>default</name>  
          <policy>FORCE</policy>
        </dataset>
      </propertiesBasedSyncOptions>
    </task>
  </tasks>
</lsc>
```

> 这里可以使用非ssl方式连接AD

**执行同步**

```bash
# 测试任务
/usr/bin/lsc -f /etc/lsc/ad2openldap -s all -c all -n
......
七月 25 11:43:20 - INFO  - Starting sync for a-PeopleSyncTask
七月 25 11:43:22 - INFO  - All entries: 4, to modify entries: 2, successfully modified entries: 0, errors: 0
七月 25 11:43:22 - INFO  - Starting clean for a-PeopleSyncTask
七月 25 11:43:22 - INFO  - All entries: 2, to modify entries: 0, successfully modified entries: 0, errors: 0
七月 25 11:43:22 - INFO  - Starting sync for b-GroupSyncTask
七月 25 11:43:23 - INFO  - All entries: 8, to modify entries: 8, successfully modified entries: 0, errors: 0
七月 25 11:43:23 - INFO  - Starting clean for b-GroupSyncTask
七月 25 11:43:23 - ERROR - Empty or non existant destination (no IDs found)

# 执行同步
/usr/bin/lsc -f /etc/lsc/ad2openldap -s all -c all
......

七月 25 11:43:36 - INFO  - Starting sync for a-PeopleSyncTask
七月 25 11:43:38 - INFO  - # Adding new object uid=ad_test2,ou=People,dc=lework,dc=com for a-PeopleSyncTask
# Thu Jul 25 11:43:38 CST 2019
dn: uid=ad_test2,ou=People,dc=lework,dc=com
changetype: add
uid: ad_test2
userPassword: {SASL}ad_test2@lework.com
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: person
objectClass: top
description: AD_TEST2 (ad_test2@lework.com)
cn: ad_test2
sn: ad_test2

七月 25 11:43:38 - INFO  - # Adding new object uid=ad_test1,ou=People,dc=lework,dc=com for a-PeopleSyncTask
# Thu Jul 25 11:43:38 CST 2019
dn: uid=ad_test1,ou=People,dc=lework,dc=com
changetype: add
uid: ad_test1
userPassword: {SASL}ad_test1@lework.com
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: person
objectClass: top
description: AD_TEST1 (ad_test1@lework.com)
cn: ad_test1
sn: ad_test1

七月 25 11:43:38 - INFO  - All entries: 4, to modify entries: 2, successfully modified entries: 2, errors: 0
七月 25 11:43:38 - INFO  - Starting clean for a-PeopleSyncTask
七月 25 11:43:38 - INFO  - All entries: 4, to modify entries: 0, successfully modified entries: 0, errors: 0
七月 25 11:43:38 - INFO  - Starting sync for b-GroupSyncTask
七月 25 11:43:39 - ERROR - Error while adding entry cn=ldapgroup2,ou=Group,DC=lework,DC=com in directory :javax.naming.NameAlreadyBoundException: [LDAP: error code 68 - Entry Already Exists]; remaining name 'cn=ldapgroup2,ou=Group'
七月 25 11:43:39 - ERROR - Error while synchronizing ID cn=ldapgroup2,ou=Group,DC=lework,DC=com: java.lang.Exception: Technical problem while applying modifications to the destination
# Thu Jul 25 11:43:39 CST 2019
dn: cn=ldapgroup2,ou=Group,DC=lework,DC=com
changetype: add
member: CN=ldapuser2,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
cn: ldapgroup2

七月 25 11:43:39 - INFO  - # Adding new object cn=Denied RODC Password Replication Group,ou=Group,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:43:39 CST 2019
dn: cn=Denied RODC Password Replication Group,ou=Group,DC=lework,DC=com
changetype: add
member: CN=Read-only Domain Controllers,CN=Users,DC=lework,DC=com
member: CN=Group Policy Creator Owners,CN=Users,DC=lework,DC=com
member: CN=Domain Admins,CN=Users,DC=lework,DC=com
member: CN=Cert Publishers,CN=Users,DC=lework,DC=com
member: CN=Enterprise Admins,CN=Users,DC=lework,DC=com
member: CN=Schema Admins,CN=Users,DC=lework,DC=com
member: CN=Domain Controllers,CN=Users,DC=lework,DC=com
member: CN=krbtgt,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
description:: 5LiN5YWB6K645bCG5q2k57uE5Lit5oiQ5ZGY55qE5a+G56CB5aSN5Yi25Yiw5Z+f5Lit55qE5omA5pyJ5Y+q6K+75Z+f5o6n5Yi25Zmo
cn: Denied RODC Password Replication Group

七月 25 11:43:39 - INFO  - # Adding new object cn=Schema Admins,ou=Group,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:43:39 CST 2019
dn: cn=Schema Admins,ou=Group,DC=lework,DC=com
changetype: add
member: CN=Administrator,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
description:: 5p625p6E55qE5oyH5a6a57O757uf566h55CG5ZGY
cn: Schema Admins

七月 25 11:43:39 - INFO  - # Adding new object cn=Domain Admins,ou=Group,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:43:39 CST 2019
dn: cn=Domain Admins,ou=Group,DC=lework,DC=com
changetype: add
member: CN=Administrator,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
description:: 5oyH5a6a55qE5Z+f566h55CG5ZGY
cn: Domain Admins

七月 25 11:43:39 - INFO  - # Adding new object cn=Enterprise Admins,ou=Group,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:43:39 CST 2019
dn: cn=Enterprise Admins,ou=Group,DC=lework,DC=com
changetype: add
member: CN=Administrator,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
description:: 5LyB5Lia55qE5oyH5a6a57O757uf566h55CG5ZGY
cn: Enterprise Admins

七月 25 11:43:39 - INFO  - # Adding new object cn=Cert Publishers,ou=Group,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:43:39 CST 2019
dn: cn=Cert Publishers,ou=Group,DC=lework,DC=com
changetype: add
member: CN=WIN-V5SBNPSNFOM,OU=Domain Controllers,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
description:: 5q2k57uE55qE5oiQ5ZGY6KKr5YWB6K645Y+R5biD6K+B5Lmm5Yiw55uu5b2V
cn: Cert Publishers

七月 25 11:43:39 - ERROR - Error while adding entry cn=ldapgroup1,ou=Group,DC=lework,DC=com in directory :javax.naming.NameAlreadyBoundException: [LDAP: error code 68 - Entry Already Exists]; remaining name 'cn=ldapgroup1,ou=Group'
七月 25 11:43:39 - ERROR - Error while synchronizing ID cn=ldapgroup1,ou=Group,DC=lework,DC=com: java.lang.Exception: Technical problem while applying modifications to the destination
# Thu Jul 25 11:43:39 CST 2019
dn: cn=ldapgroup1,ou=Group,DC=lework,DC=com
changetype: add
member: CN=ldapuser1,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
cn: ldapgroup1

七月 25 11:43:39 - INFO  - # Adding new object cn=Group Policy Creator Owners,ou=Group,DC=lework,DC=com for b-GroupSyncTask
# Thu Jul 25 11:43:39 CST 2019
dn: cn=Group Policy Creator Owners,ou=Group,DC=lework,DC=com
changetype: add
member: CN=Administrator,CN=Users,DC=lework,DC=com
objectClass: groupOfNames
objectClass: top
description:: 6L+Z5Liq57uE5Lit55qE5oiQ5ZGY5Y+v5Lul5L+u5pS55Z+f55qE57uE562W55Wl
cn: Group Policy Creator Owners

七月 25 11:43:39 - ERROR - All entries: 8, to modify entries: 8, successfully modified entries: 6, errors: 2
七月 25 11:43:39 - INFO  - Starting clean for b-GroupSyncTask
七月 25 11:43:39 - INFO  - All entries: 6, to modify entries: 0, successfully modified entries: 0, errors: 0
```

可以看到有2个用户，6个组成功同步到openldap中

![openldap-to-ad2.png](/assets/images/ldap/openldap-to-ad2.png)

## 参考

* [https://lsc-project.org/documentation/tutorial/synchronizegroups](https://lsc-project.org/documentation/tutorial/synchronizegroups)
* [https://github.com/lsc-project/lsc/tree/master/sample/ad](https://github.com/lsc-project/lsc/tree/master/sample/ad)
* [https://confluence.atlassian.com/crowd/configuring-an-ssl-certificate-for-microsoft-active-directory-63504388.html](https://confluence.atlassian.com/crowd/configuring-an-ssl-certificate-for-microsoft-active-directory-63504388.html)
* [https://florinloghiade.ro/2016/06/migrating-openldap-to-active-directory/](https://florinloghiade.ro/2016/06/migrating-openldap-to-active-directory/)
* [https://camratus.com/2017/01/24/openldap-lsc-active-directory-sync-and-login-pass-through/](https://camratus.com/2017/01/24/openldap-lsc-active-directory-sync-and-login-pass-through/)
