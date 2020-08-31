---
layout: post
title: "初创型公司-持续部署系列（四）部署jenkins服务器"
date: "2016-12-10 19:08:48"
categories: 初创型公司运维专题
tags: jenkins
excerpt: "本次使用的操作系统： Windows Server 2008 R2 Enterprisevs版本： vs 2015jenkins： 2.19.4..."
auth: lework
---
* content
{:toc}

本次使用的操作系统： Windows Server 2008 R2 Enterprise
vs版本： vs 2015
jenkins： 2.19.4

### 下载jenkins
http://mirror.xmission.com/jenkins/windows-stable/jenkins-2.19.4.zip

### 安装jenkins

![Paste_Image.png](/assets/images/ops/3629406-31fffbffe145796b.png)

都默认设置就可以。

### 设置jenkins服务的启动权限
进入服务-右键Jenkins-属性-登录

![Paste_Image.png](/assets/images/ops/3629406-29aadd85a8dcf332.png)

需要重启服务

### 访问8080端口，输入密钥，解锁jenkins


![Paste_Image.png](/assets/images/ops/3629406-b3e86b8abc5128e4.png)

### 选择默认安装方式
> 如果网络差的话，可以自定义把插件都取消掉，登陆系统后在选择安装，如果安装还是失败，可以先去官网现在插件，通过jenkins后台上传插件即可。



![](/assets/images/ops/3629406-c46b49ab7923cda6.png)

 
### 完成后，再次安装插件

安装的插件如下：
- Git client plugin
- Git plugin
- MSBuild Plugin
- PowerShell plugin

###  安装git客户端

安装文件：Git-2.8.3-64-bit.exe
采用默认安装。

git执行文件路径：`C:\Program Files\Git\bin\git.exe`

### 安装vs2015
> 注意下，版本号最好要与开发用的版本一致。

安装过程跳过，一切默认即可。

msbuild路径：`C:\Program Files (x86)\MSBuild\14.0\Bin\MSBuild.exe`

### 配置git，msbuild插件的路径

![Paste_Image.png](/assets/images/ops/3629406-a0753c00de674af1.png)

![Paste_Image.png](/assets/images/ops/3629406-4da0c09d64e103db.png)
