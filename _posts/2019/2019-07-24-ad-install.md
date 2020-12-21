---
layout: post
title: "Windows Server 2016安装AD并开启SSL"
date: "2019-07-24 18:50"
category: AD
tags: LDAP AD
author: lework
---
* content
{:toc}

[AD](https://docs.microsoft.com/zh-cn/windows-server/identity/ad-ds/active-directory-domain-services)是Active Directory的简写，中文称活动目录。活动目录(Active Directory)主要提供以下功能：

* 1、服务器及客户端计算机管理
* 2、用户服务
* 3、资源管理
* 4、桌面配置
* 5、应用系统支撑等；

更多AD DS概述请查看[微软技术文档](https://docs.microsoft.com/zh-cn/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview),本文详细介绍AD DS的部署。



## 森林模型

![c50a3c6a-b0e4-43ec-ad62-f05d05f0bbd2.png](/assets/images/ldap/c50a3c6a-b0e4-43ec-ad62-f05d05f0bbd2.png)

这里不描述系统安装过程

## AD域角色安装

在需要安装AD域控制器的电脑上打开服务器管理器，点击**添加角色和功能**

![1563962585628](/assets/images/ldap/1563962585628.png)

打开**添加角色和功能向导**，点击**下一步**

![1563962606688](/assets/images/ldap/1563962606688.png)

安装类型选择**基于角色或基于功能的安装**，点击**下一步**

![1563962625329](/assets/images/ldap/1563962625329.png)



服务器选择**从服务器池中选择服务器**，再选中池中的本地服务器，点击**下一步**

![1563962640569](/assets/images/ldap/1563962640569.png)

服务器角色选择**Active Directory域服务**，会弹出**添加Active Directory域服务所需的功能？**，点击**添加功能**

![1563962664796](/assets/images/ldap/1563962664796.png)

点击**下一步**， 这里不需要选择

![1563962687136](/assets/images/ldap/1563962687136.png)

点击**下一步**

![1563962706693](/assets/images/ldap/1563962706693.png)

确认这里勾选**如果需要，自动重新启动目标服务器**，点击**安装**

![1563962726163](/assets/images/ldap/1563962726163.png)

Active Directory域服务角色安装完成，点**关闭**

![1563962784046](/assets/images/ldap/1563962784046.png)



## 运行部署向导

运行AD DS（Active Directory域服务的简称）部署向导，打开本地服务器的服务器管理器，点击**通知**-**将此服务器提升为域控制器**

![1563962810531](/assets/images/ldap/1563962810531.png)

打开AD DS的部署向导，由于我们这里是部署新的AD控制器，所以部署配置选择**添加新林**，把**根域名**设置成**lework.com**，点击**下一步**

![1563962844122](/assets/images/ldap/1563962844122.png)

解释：

* 将域控制器添加到现有域：在现有的域控制器中添加新的域控制器
* 将新域添加到现有林：在现有的林中新建域，与林中现有的域不同
* 添加新林：在没有林的情况下新建林

设置域密码，点击下一步

![1563962964073](/assets/images/ldap/1563962964073.png)

域控制器选项：

* 林功能级别（包含Windows Server 2008到Windows Server 2016级别都有）：Windows Server 2016
* 域功能级别（只包含Windows Server 2016域功能）：Windows Server 2016
指定域控制器功能：默认

点击**下一步**

![1563962976933](/assets/images/ldap/1563962976933.png)

点击**下一步**

![1563963009466](/assets/images/ldap/1563963009466.png)

设置AD DS的数据库、日志文件和SYSVOL的位置，点击**下一步**

![1563963055372](/assets/images/ldap/1563963055372.png)

点击**下一步**

![1563963068970](/assets/images/ldap/1563963068970.png)

先决条件检查通过，点击**安装**，如果不通过请根据提示查看原因

![1563963127740](/assets/images/ldap/1563963127740.png)

正在进行自动部署，部署完成后会自动重启服务器

AD域控制器部署完成，打开**服务器管理器**-**工具**-**Active Directory用户和计算机**

![1563963502244](/assets/images/ldap/1563963502244.png)

就可以看到我们刚才部署好的域，这样一个完整的域就部署完成了

## 启用LDAPS

### 创建证书颁发机构

添加**Active Directory 证书服务** 角色

![1563963533190](/assets/images/ldap/1563963533190.png)

选择**证书颁发机构**

![1563963547423](/assets/images/ldap/1563963547423.png)

点击**下一步**进行安装

![1563963589516](/assets/images/ldap/1563963589516.png)

### 配置域证书

点击**通知**-**配置目标服务器上的Active Directory 证书服务**

![1563963618106](/assets/images/ldap/1563963618106.png)

点击**下一步**

![1563963701273](/assets/images/ldap/1563963701273.png)

勾选**证书颁发机构**,点击**下一步**

![1563963712630](/assets/images/ldap/1563963712630.png)

选择**企业CA**,点击**下一步**

![1563963785147](/assets/images/ldap/1563963785147.png)

选择**根CA**,点击**下一步**

![1563963754110](/assets/images/ldap/1563963754110.png)

选择**创建新的私钥**,点击**下一步**

![1563964040955](/assets/images/ldap/1563964040955.png)

指定CA的加密，默认即可.点击**下一步**

![1563964055082](/assets/images/ldap/1563964055082.png)

指定CA名称,点击**下一步**

![1563964106388](/assets/images/ldap/1563964106388.png)

指定有效期，这里设置为10年,点击**下一步**

![1563964236799](/assets/images/ldap/1563964236799.png)

指定CA数据库的位置，默认即可.点击**下一步**

![1563964255519](/assets/images/ldap/1563964255519.png)

确认证书的配置，点击**配置**.点击**下一步**

![1563964268482](/assets/images/ldap/1563964268482.png)

配置完成后，点击**关闭**页面

![1563964289350](/assets/images/ldap/1563964289350.png)

配置完成后，重启下服务器

在证书颁发机构中可以看到给域控颁发的证书

![1563964545925](/assets/images/ldap/1563964545925.png)

### 连接AD

运行-->ldp.exe

```
host: WIN-V5SBNPSNFOM.lework.com port: 389
conn：LDAP:\\WIN-V5SBNPSNFOM.lework.com:389
```

![1563964600770](/assets/images/ldap/1563964600770.png)

```
host: WIN-V5SBNPSNFOM.lework.com port: 636
conn：LDAPS:\\WIN-V5SBNPSNFOM.lework.com:636
```
SSL 勾选上

![1563964651346](/assets/images/ldap/1563964651346.png)

在Active Directory服务器上执行以下命令来导出证书,供客户端连接使用


```bash
C:\Users\Administrator>certutil -ca.cert client.crt
CA 证书[0]: 3 -- 有效
CA 证书[0]:
-----BEGIN CERTIFICATE-----
MIIDfTCCAmWgAwIBAgIQKb58EV2zDLBAbvMySV/voDANBgkqhkiG9w0BAQsFADBR
MRMwEQYKCZImiZPyLGQBGRYDY29tMRYwFAYKCZImiZPyLGQBGRYGbGV3b3JrMSIw
IAYDVQQDExlsZXdvcmstV0lOLVY1U0JOUFNORk9NLUNBMB4XDTE5MDcyNDEwMjEw
NloXDTI5MDcyNDEwMzEwNlowUTETMBEGCgmSJomT8ixkARkWA2NvbTEWMBQGCgmS
JomT8ixkARkWBmxld29yazEiMCAGA1UEAxMZbGV3b3JrLVdJTi1WNVNCTlBTTkZP
TS1DQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANbilH1FFCE8UH/Y
ye8E0jMeDqjIEtAowvmu2yKbU+adKUJHo+gMYaL/3dXjFFI5Tr++WC/QROIbVAub
RzCZudGFQ2OKbr/yJ3mt6adB/VmdmGljX+2c1hmRHZcnuMyjnx7J/xqwkBlWxMpp
uP58VoaSJxQtUr/aO9dR53NsAa3pDcKYKfgWtNpCCa43YtY2x2pznpe4OOmQ1ufs
JENjJwA1e73Uq+TxKRKRRsE92SVxefbgSsOzO8Pg4Hyk1B2pIx267eYQFMngHlq2
ojd003HsMBtGU68F3IZRpyX+njpb28PANOL1MgVIRCT5HpddtV6R0Uvj84mBp0q0
6CwPWA8CAwEAAaNRME8wCwYDVR0PBAQDAgGGMA8GA1UdEwEB/wQFMAMBAf8wHQYD
VR0OBBYEFNMUY0FWH8vE5hsr5ZD9hrGF46rQMBAGCSsGAQQBgjcVAQQDAgEAMA0G
CSqGSIb3DQEBCwUAA4IBAQCj3EgCh4O7AutmMZE0/3UjOpz2o+GVIpym9V9JJGQw
z3rmmKtFO7G//YjjEN+bBmiDTUrmXTzar7RK8Vu2mLs+XqZipEE/GmcmdraZjQQD
2u3QZjKWFnLom1IIArbeIw9Mq6ZEr2cxsKI+biIg5YTpGjggyRrAHdFIdOInFYol
Zj50okNMZ+D7NJ83GupFCfFT7p4Glh2zL89a9u5qae9WE95y1G8fU30linQbCed2
ddCWWwU1+Jn5eEm0cAX5ogrY+UwqiYYBegWYLcxpndl/xLTGBYx7o7Sk2VMpHFO4
mPfPzpZ22rgS+Cvd7+S3nAvb22ygg1L+jMF63z8SFIP/
-----END CERTIFICATE-----

CertUtil: -ca.cert 命令成功完成。
```
