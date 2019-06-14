---
layout: post
title:  "使用SoftEther VPN建立虚拟专用网络"
categories: vpn
tags: vpn vpngate
excerpt: SoftEther VPN是一个最佳的替代OpenVPN的和微软的VPN服务器。SoftEther VPN有OpenVPN服务器的克隆功能。您可以从OpenVPN的集成到SoftEther VPN顺利。SoftEther VPN比OpenVPN的速度更快。SoftEther VPN还支持微软SSTP VPN适用于Windows Vista / 7/8，不再需要支付昂贵的费用为远程接入VPN功能的Windows Server许可证。
auth: lework
---

* content
{:toc}

## 使用SoftEther VPN建立虚拟专用网络

> SoftEther VPN是一个最佳的替代OpenVPN的和微软的VPN服务器。SoftEther VPN有OpenVPN服务器的克隆功能。您可以从OpenVPN的集成到SoftEther VPN顺利。SoftEther VPN比OpenVPN的速度更快。SoftEther VPN还支持微软SSTP VPN适用于Windows Vista / 7/8，不再需要支付昂贵的费用为远程接入VPN功能的Windows Server许可证。

### 官方
---

> 可能需要翻墙访问

- [官方主站点](http://www.vpngate.net/cn/)
- [官方下载页面](http://www.vpngate.net/cn/)
- [源码](https://github.com/SoftEtherVPN/SoftEtherVPN/)

### 文件下载
---

> 本文中用到的软件

- [客户端官方下载](http://www.softether-download.com/files/softether/v4.12-9514-beta-2014.11.17-tree/Windows/SoftEther_VPN_Client/softether-vpnclient-v4.12-9514-beta-2014.11.17-windows-x86_x64-intel.exe)
- [服务器端官方下载](http://jp.softether-download.com/files/softether/v4.12-9514-beta-2014.11.17-tree/Windows/SoftEther_VPN_Server_and_VPN_Bridge/softether-vpnserver_vpnbridge-v4.12-9514-beta-2014.11.17-windows-x86_x64-intel.exe)
- [百度云下载](http://pan.baidu.com/s/1sjAwEHv) （密码: ml9f）

### 环境介绍
---
- Client

	版本：Windows Server2008 R2 64位 企业版
	
	安装软件：：softether-vpnclient-v4.12-9514-beta-2014.11.17-windows-x86_x64-intel.exe
	
	ip：192.168.11.105
	
	是否联网：否

- Server

	版本：Windows Server2008 R2 64位 企业版
	
	安装软件：softether-vpnserver_vpnbridge-v4.12-9514-beta-2014.11.17-windows-x86_x64-intel.exe
	
	ip：192.168.11.108
	
	是否联网：是

### Server配置
---

1. 安装软件，选择安装 `SoftEther VPN Server`。

	![vpngate-win-server01](/assets/images/vpn/vpngate-win-server01.jpg)

1. 安装完成后，双击 “本地主题（此服务器）” ，输入管理员密码。

	![vpngate-win-server02](/assets/images/vpn/vpngate-win-server02.jpg)

	![vpngate-win-server04](/assets/images/vpn/vpngate-win-server04.jpg)

1. 选择”远程访问 VPN Server”安装方式。


	![vpngate-win-server05](/assets/images/vpn/vpngate-win-server05.jpg)


1. 动态DNS功能，选择退出即可。

	如果服务器在我们的内网，利用动态dns，在外网可以链接到内网中的服务器，只需要在内网出口设置端口映射，如不需要可以禁用。

1. 选择 “启用 L2TP 服务器功能( L2TP over ipsec)”。

	![vpngate-win-server09](/assets/images/vpn/vpngate-win-server09.jpg)

1. 禁用VPN Azure功能。

	别人的云我们就不用了吧。

1. 创建用户。

	![vpngate-win-server12](/assets/images/vpn/vpngate-win-server12.jpg)

1. 设置本地网桥，这里我选择的是我外网网卡。

	![vpngate-win-server13](/assets/images/vpn/vpngate-win-server13.jpg)

1. 设置NAT和DHCP在管理器界面选择”管理虚拟HUB”,选择”虚拟NAT和虚拟DHCP服务器”,选择 “设置DHCP”。

	![vpngate-win-server16](/assets/images/vpn/vpngate-win-server16.jpg)

1. 启用NAT和DHCP。

	至此，服务端已经配置好了。

### Client配置
---

1. 安装VPN Client，选择SoftEther VPN Client。

	![vpngate-win-client01](/assets/images/vpn/vpngate-win-client01.jpg)

1. 选择”添加新的vpn链接”，然后创建一个虚拟网络适配器，输入虚拟适配器的名称。

	![vpngate-win-client04](/assets/images/vpn/vpngate-win-client04.jpg)

1. 设置VPN链接属性。

	![vpngate-win-client05](/assets/images/vpn/vpngate-win-client05.jpg)

1. 完成后，双击test链接，完成链接vpn，下图是分配的ip。

	![vpngate-win-client06](/assets/images/vpn/vpngate-win-client06.jpg)

### 效果检验
---

#### 客户端
	
显示网卡信息
	
	
![vpngate-win-client07](/assets/images/vpn/vpngate-win-client07.jpg)
	
访问百度网页

![vpngate-win-client08](/assets/images/vpn/vpngate-win-client08.jpg)
	
#### Server
	
查看vpn会话
	
![vpngate-win-server17](/assets/images/vpn/vpngate-win-server17.jpg)