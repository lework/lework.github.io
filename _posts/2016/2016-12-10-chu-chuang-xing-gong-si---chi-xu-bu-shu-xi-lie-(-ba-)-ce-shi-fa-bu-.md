---
layout: post
title: "初创型公司-持续部署系列（八）测试发布"
date: "2016-12-10 19:10:09"
categories: 初创型公司运维专题
tags: jenkins
excerpt: "执行编译任务 jenkins任务信息 在后端服务器可以看到D:\\Packages\\online目录下已经存在了一个压缩文件 执行发布任务 在li..."
auth: lework
---
* content
{:toc}

### 执行编译任务
jenkins任务信息

```
	Started by user admin
Building in workspace C:\Program Files (x86)\Jenkins\workspace\Build_demo
 > C:\Program Files\Git\bin\git.exe rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > C:\Program Files\Git\bin\git.exe config remote.origin.url https://git.oschina.net/lework/Webdemo.git # timeout=10
Fetching upstream changes from https://git.oschina.net/lework/Webdemo.git
 > C:\Program Files\Git\bin\git.exe --version # timeout=10
using GIT_ASKPASS to set credentials 
 > C:\Program Files\Git\bin\git.exe fetch --tags --progress https://git.oschina.net/lework/Webdemo.git +refs/heads/*:refs/remotes/origin/*
 > C:\Program Files\Git\bin\git.exe rev-parse "refs/remotes/origin/master^{commit}" # timeout=10
 > C:\Program Files\Git\bin\git.exe rev-parse "refs/remotes/origin/origin/master^{commit}" # timeout=10
.....(省略)
Checking out Revision 7a1ffd4d8688273f537ef556d4a2c761181475a6 (refs/remotes/origin/master)
 > C:\Program Files\Git\bin\git.exe config core.sparsecheckout # timeout=10
 > C:\Program Files\Git\bin\git.exe checkout -f 7a1ffd4d8688273f537ef556d4a2c761181475a6
 > C:\Program Files\Git\bin\git.exe rev-list 7a1ffd4d8688273f537ef556d4a2c761181475a6 # timeout=10
Path To MSBuild.exe: C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe
Executing the command cmd.exe /C " "C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe" /t:clean /t:rebuild /p:Configuration=release /p:WebProjectOutputDir=d:\build_code\demo /p:OutputPath=d:\build_code\demo\bin Webdemo.sln " && exit %%ERRORLEVEL%% from C:\Program Files (x86)\Jenkins\workspace\Build_demo
[Build_demo] $ cmd.exe /C " "C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe" /t:clean /t:rebuild /p:Configuration=release /p:WebProjectOutputDir=d:\build_code\demo /p:OutputPath=d:\build_code\demo\bin Webdemo.sln " && exit %%ERRORLEVEL%%
Microsoft (R) 生成引擎版本 14.0.23107.0
版权所有(C) Microsoft Corporation。保留所有权利。
	在此解决方案中一次生成一个项目。若要启用并行生成，请添加“/m”开关。
生成启动时间为 2016/12/10 12:16:00。
项目“C:\Program Files (x86)\Jenkins\workspace\Build_demo\Webdemo.sln”在节点 1 上(clean;rebuild 个目标)。
	…….(省略编译信息)
	已成功生成。
	    0 个警告
	    0 个错误
	
	已用时间 00:00:13.79
	[Build_demo] $ powershell.exe -NonInteractive -ExecutionPolicy ByPass "& 'C:\Users\ADMINI~1\AppData\Local\Temp\hudson6389577925705337950.ps1'"
	7-Zip [64] 16.04 : Copyright (c) 1999-2016 Igor Pavlov : 2016-10-04
	
	Scanning the drive:
	15 folders, 161 files, 43667500 bytes (42 MiB)
	
	Creating archive: D:\Packages\upload\demo-20161210121614.7z
	
	Items to compress: 176
	
	
	Files read from disk: 161
	Archive size: 6849177 bytes (6689 KiB)
	Everything is Ok
	[Build_demo] $ powershell.exe -NonInteractive -ExecutionPolicy ByPass "& 'C:\Users\ADMINI~1\AppData\Local\Temp\hudson4508233314385919964.ps1'"
	192.168.77.140 主机 【执行完毕】
	192.168.77.142 主机 【执行完毕】
	Finished: SUCCESS
```	

在后端服务器可以看到D:\Packages\online目录下已经存在了一个压缩文件

![Paste_Image.png](/assets/images/ops/3629406-a82bbcfc3a655b70.png)
	
### 执行发布任务
	
> 在linux下持续访问nginx后，再去执行发布任务
	
jenkins任务信息
```
	Started by user admin
Building in workspace C:\Program Files (x86)\Jenkins\workspace\Deploy_demo
[Deploy_demo] $ powershell.exe -NonInteractive -ExecutionPolicy ByPass "& 'C:\Users\ADMINI~1\AppData\Local\Temp\hudson5325306686045764609.ps1'"
	[-] 服务器从负载均衡移除
[-] Script Run Success.
[-] 暂停5秒
[-] 关闭iis站点
“demo”已成功停止
[-] 关闭应用程序池
“demo”已成功停止
[-] copy iis站点文件
……..(省略)
	复制了 161 个文件
[-] 清空站点目录文件
[-] 解压文件到站点目录
	7-Zip [64] 16.04 : Copyright (c) 1999-2016 Igor Pavlov : 2016-10-04
	Scanning the drive for archives:
1 file, 6849177 bytes (6689 KiB)
	Extracting archive: D:\Packages\online\demo-20161210121614.7z
--
Path = D:\Packages\online\demo-20161210121614.7z
Type = 7z
Physical Size = 6849177
Headers Size = 2643
Method = LZMA2:24 BCJ
Solid = +
Blocks = 2
	Everything is Ok
	Folders: 15
Files: 161
Size:       43667500
Compressed: 6849177
[-] 开启应用程序池
“demo”已成功启动。
[-] 开启iis站点
“demo”已成功启动。
[-] 移动发布包
D:\Packages\online\demo-20161210121614.7z
移动了         1 个文件。
HTTP/1.1 200 OK
	[-] 服务器成功添加到负载均衡
192.168.77.140 主机 【执行完毕】
--------------------------------------------------------------------------
	[-] 服务器从负载均衡移除
[-] Script Run Success.
[-] 暂停5秒
[-] 关闭iis站点
“demo”已成功停止
[-] 关闭应用程序池
“demo”已成功停止
[-] copy iis站点文件
	……..(省略)
	复制了 161 个文件
[-] 清空站点目录文件
[-] 解压文件到站点目录
	7-Zip [64] 16.04 : Copyright (c) 1999-2016 Igor Pavlov : 2016-10-04
	Scanning the drive for archives:
1 file, 6849177 bytes (6689 KiB)
	Extracting archive: D:\Packages\online\demo-20161210121614.7z
--
Path = D:\Packages\online\demo-20161210121614.7z
Type = 7z
Physical Size = 6849177
Headers Size = 2643
Method = LZMA2:24 BCJ
Solid = +
Blocks = 2
	Everything is Ok
	Folders: 15
Files: 161
Size:       43667500
Compressed: 6849177
[-] 开启应用程序池
“demo”已成功启动。
[-] 开启iis站点
“demo”已成功启动。
[-] 移动发布包
D:\Packages\online\demo-20161210121614.7z
移动了         1 个文件。
HTTP/1.1 200 OK
[-] 服务器成功添加到负载均衡
192.168.77.142 主机 【执行完毕】
--------------------------------------------------------------------------
	Finished: SUCCESS
```
	
linux下持续访问效果

![Paste_Image.png](/assets/images/ops/3629406-f71dac6e911ed02e.png)

