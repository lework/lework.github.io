---
layout: post
title:  "SoftEther VPN Client 使用方法"
categories: vpn
tags: vpn vpngate
excerpt: 本文档描述了如何使用 SoftEther VPN Client 连接到 VPN Gate 的一个 VPN 中继服务器。通过使用 SoftEther VPN Client，您可以轻松、舒适、快速地建立一个 VPN 连接。需要注意的是 SoftEther VPN Client 仅能在 Windows 上运行。Mac, iPhone / iPad 和安卓用户不得不选择其他方法。
auth: lework
---
* content
{:toc}

## SoftEther VPN Client 使用方法

> 本文档描述了如何使用 SoftEther VPN Client 连接到 VPN Gate 的一个 VPN 中继服务器。通过使用 SoftEther VPN Client，您可以轻松、舒适、快速地建立一个 VPN 连接。需要注意的是 SoftEther VPN Client 仅能在 Windows 上运行。Mac, iPhone / iPad 和安卓用户不得不选择其他方法。


### NO.1 安装带有 VPN Gate Client 插件的 SoftEther VPN Client (只需在第一次时安装一次)
---

下载带有 “VPN Gate Client 插件” 的 SoftEther VPN Client 的特殊版本。

> 可能需要翻墙访问

- [官方下载页面](http://www.vpngate.net/cn/download.aspx)


	![vpngate001](/assets/images/vpn/vpngate001.png)

	解压缩下载的 ZIP 文件内容到一个文件夹中。如上图，安装程序和一些 DLL 文件被提取。

	执行以 “vpngate-client” 开头的文件名的安装程序，并继续进行安装。

	![vpngate002](/assets/images/vpn/vpngate002.png)
	![vpngate003](/assets/images/vpn/vpngate003.png)

	上述安装程序将启动。你必须在 “选择软件组件安装” 屏幕中选择 “SoftEther VPN Client” ，然后选择安装网络协议

	![vpngate004](/assets/images/vpn/vpngate004.png)

	安装完成后，将在桌面上创建 SoftEther VPN Client 的图标。

### NO.2 运行 VPN Gate Client 插件并连接到 VPN Gate 服务器
---

在桌面上双击 SoftEther VPN Client 图标。

![vpngate005](/assets/images/vpn/vpngate005.png)

如上图， “VPN Gate 公共 VPN 中继服务器” 图标会显示在窗口中。双击该图标。

如果有通知显示，继续按屏幕描述的进行。


![vpngate010](/assets/images/vpn/vpngate010.png)

“SoftEther VPN Client 的 VPN Gate 学术实验项目插件” 启动。

在此屏幕上，你可以看到当前正在运行的 VPN Gate 公共 VPN 服务器的列表。此屏幕上的列表与 顶页的列表 是相同的。从列表中选择一个连接，然后单击 “连接到 VPN 服务器” 按钮。

![vpngate006](/assets/images/vpn/vpngate006.png)

如果选定的 VPN Gate 服务器同时支持 TCP 和 UDP 协议，上面的屏幕将会出现。在屏幕上选择 TCP 或 UDP。

第一次安装会创建一个虚拟网卡

![vpngate007](/assets/images/vpn/vpngate007.png)

如果一个 VPN 连接建立成功了，上面的消息将出现。这个窗口将在 5 秒后自动消失。如果您无法连接到指定的 VPN 服务器，再试一次。

![vpngate008](/assets/images/vpn/vpngate008.png)

### NO.3 通过 VPN 中继享受互联网
---

虽然建立了 VPN 连接，在 Windows 上将创建一个虚拟网络适配器，该适配器将被分配一个以 “10.211” 开始的 IP 地址。默认网关地址将被指定在虚拟网络适配器上。您可以在 Windows 命令提示下运行 "ipconfig / all" 命令，确认这些网络配置。

![vpngate009](/assets/images/vpn/vpngate009.png)

当 VPN 建立时，所有到互联网的通讯将通过 VPN 服务器转发。您可以在 Windows 命令提示中使用"tracert 8.8.8.8"命令验证。

![vpngate011](/assets/images/vpn/vpngate011.png)

如上图，如果数据包路径是通过 “10.211.254.254” ，你的通信现在就是通过 VPN Gate 公共 VPN 服务器中的一个转发的。您还可以访问 VPN Gate 顶部页面 来查看当前的全球 IP 地址。如果你连接到一个位于海外国家的 VPN 服务器，您可以看到您的来源国或地区已更改为其他的。


![vpngate012](/assets/images/vpn/vpngate012.jpg)