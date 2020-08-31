---
layout: post
title: "初创型公司-持续部署系列（二）部署web站点"
date: "2016-12-10 19:06:37"
categories: 初创型公司运维专题
tags: jenkins
excerpt: "本次使用的操作系统：Windows Server 2008 R2 Enterprise本次使用的实例代码：https://git.oschina..."
auth: lework
---
* content
{:toc}

本次使用的操作系统：`Windows Server 2008 R2 Enterprise`
本次使用的实例代码：https://git.oschina.net/lework/Webdemo

## 安装iis

打开服务器管理器，选择添加角色
![Paste_Image.png](/assets/images/ops/3629406-e225bea01cf4881f.png)

选择web服务器（IIS）

![Paste_Image.png](/assets/images/ops/3629406-1d606535ea4c8a53.png)


选择为iis安装的角色服务

![Paste_Image.png](/assets/images/ops/3629406-bfd462079f84e026.png)

![Paste_Image.png](/assets/images/ops/3629406-eb483ed23e0dcb27.png)

![Paste_Image.png](/assets/images/ops/3629406-4467e577b0c043fd.png)


![Paste_Image.png](/assets/images/ops/3629406-79ed83acf812326f.png)




## 安装 .net 4.5.2
> .net 版本要与代码使用的.net版本一致


![Paste_Image.png](/assets/images/ops/3629406-745ca9b0be6e89bb.png)

采用默认即可。



## 放置c#代码
把网站代码放在D:\iis_sites\demo目录下

![Paste_Image.png](/assets/images/ops/3629406-bfd296bbd9fc3a1a.png)


## 配置iis站点
打开iis管理器
> 开始==》所有程序==》管理工具


![Paste_Image.png](/assets/images/ops/3629406-62a54bb1255bde70.png)


停止默认站点

![Paste_Image.png](/assets/images/ops/3629406-b6b6562a9b56d37f.png)


添加demo应用池

![Paste_Image.png](/assets/images/ops/3629406-f6b4d403c9adf6e5.png)

![Paste_Image.png](/assets/images/ops/3629406-8949485ddf26173e.png)


添加demo网站

![Paste_Image.png](/assets/images/ops/3629406-5dac39d0218e9d62.png)


![Paste_Image.png](/assets/images/ops/3629406-a5ceb4955690cd1d.png)



打开ISAPI和CGI限制

![Paste_Image.png](/assets/images/ops/3629406-6e236867c11b2fa5.png)

设置为允许
> 如果没有.net 4，就是你先装.net 4 再装iis的原因，重新装下.net4就可以了。

![Paste_Image.png](/assets/images/ops/3629406-bdaa382b3399b62b.png)


## 浏览网站

看到下面页面，说明网站正常运行了。


![Paste_Image.png](/assets/images/ops/3629406-8d3f43c72fec44fc.png)




> 在其中一台服务器上配置一个用作显示维护的页面。这里配置在192.168.77.140:8080站点上

站点代码目录`D:\iis_sites\error`


![Paste_Image.png](/assets/images/ops/3629406-f145521f647b2c0e.png)


## 添加error站点


![Paste_Image.png](/assets/images/ops/3629406-8effd1e83bc8ec34.png)

## 访问error站点

![Paste_Image.png](/assets/images/ops/3629406-011a74f79945bd02.png)



确保防火墙允许外面访问8080端口

![Paste_Image.png](/assets/images/ops/3629406-386cf5221111c3e1.png)
