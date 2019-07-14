---
layout: post
title: "初创型公司-持续部署系列（九）测试环境的持续交付"
date: "2016-12-17 12:59:28"
categories: 初创型公司运维专题
excerpt: "测试环境没有像生产环境的流程那么多，步骤那么严谨。讲究的是发布速度快，快速呈现给开发/测试一个环境出来。 测试环境持续交付的流程： 本实例用得是..."
auth: lework
---
* content
{:toc}

> 测试环境没有像生产环境的流程那么多，步骤那么严谨。讲究的是发布速度快，快速呈现给开发/测试一个环境出来。

测试环境持续交付的流程：

![Paste_Image.png](/assets/images/ops/3629406-8b02efea67659148.png)




本实例用得是192.168.77.140作为测试服务器，后端数据库不做考虑，iis站点配置不做说明。

[git仓库](https://git.oschina.net/lework/Webdemo.git)目前存在3个分支：
- master
- dev-pages
- release-1.0


### 测试服务器配置

在测试服务器上创建目录

- D:\test\Packages   用来存放jenkins发来的压缩文件
- D:\tools                           用于存放工具


允许192.168.77.* 网段的主机远程执行winrm
```
Set-Item wsman:\localhost\Client\TrustedHosts -value 192.168.77.*
winrm quickconfig
```
### jenkins配置

** 安装插件**
- Git Parameter Plug-In   动态获取git仓库中得分支目录

** 在jenkins服务器上创建目录** 

- D:\build_code\demo，用于存放编译后的代码
- D:\test\Packages，用于存放代码压缩文件
- D:\tools，用于存放工具

** 创建一个名为192.168.77.140-test 视图** 

![Paste_Image.png](/assets/images/ops/3629406-23ddab796b818d7a.png)

> 目的是为了区分多个测试环境的任务。一个测试环境的发布任务都放在一个视图中。



** 在192.168.77.140-test 视图下创建一个名为Build-deploy_demo_140的任务** 



![Paste_Image.png](/assets/images/ops/3629406-65d5b626df2e1467.png)



** 配置参数化构建** 

选择参数化构建，点击添加参数，选择Git Parameter

![Paste_Image.png](/assets/images/ops/3629406-31e0a858f52d067d.png)

![Paste_Image.png](/assets/images/ops/3629406-a19310a1762da690.png)

** 配置源码管理** 

![Paste_Image.png](/assets/images/ops/3629406-19b8b8c3fd830dec.png)

** 配置编译脚本** 

![Paste_Image.png](/assets/images/ops/3629406-7db0c6503a2169b2.png)

```
/t:clean /t:rebuild /p:Configuration=release  /p:WebProjectOutputDir=d:\test\build_code\demo /p:OutputPath=d:\test\build_code\demo\bin
```

** 配置编译发布脚本** 

![Paste_Image.png](/assets/images/ops/3629406-0dbd1fe5b8fb6abf.png)

```powershell
function GetRequest($url)
{
	$request = [System.Net.HttpWebRequest]::Create($url)
	$response = [System.Net.HttpWebResponse]$request.GetResponse()
	$code = [System.Int32]$response.StatusCode
	$response.Close()
	Write-Host '[-] HTTP 状态吗： ' $code
}

$datetime=Get-Date -Format 'yyyyMMddHHmmss'
#Predefine necessary information
$ComputerName = "192.168.77.140"     # 远端地址
$Username = "administrator"         # 用户名
$Password = "123456"         # 密码

Write-Host '[-] 清空上传目录'
Remove-Item D:\test\Packages\*  -force -recurse

Write-Host '[-] 打包文件'
D:\tools\7z.exe a D:\test\Packages\demo-$datetime.7z D:\test\build_code\demo\*

#Create credential object
$SecurePassWord = ConvertTo-SecureString -AsPlainText $Password -Force
$Cred = New-Object -TypeName "System.Management.Automation.PSCredential" -ArgumentList $Username, $SecurePassWord

#Create session object with this
$Session = New-PSSession -ComputerName $ComputerName -credential $Cred

Write-Host '[-] 清空部署服务器的上传目录'
Invoke-Command -Session $Session -ScriptBlock {Remove-Item D:\test\Packages\*  -force -recurse}

Copy-Item "D:\test\Packages\demo-*.7z" -Destination "\\$ComputerName\d$\test\Packages\"
Write-Host $ComputerName  '文件 【传送完毕】' 

Write-Host '[-] 关闭iis站点'
Invoke-Command -Session $Session -ScriptBlock {cmd /c "%windir%\system32\inetsrv\appcmd.exe stop site demo"}

Write-Host '[-] 关闭应用程序池'
Invoke-Command -Session $Session -ScriptBlock {cmd /c "%windir%\system32\inetsrv\appcmd.exe stop site demo"}

Write-Host '[-] 关闭iis站点'
Invoke-Command -Session $Session -ScriptBlock {cmd /c "%windir%\system32\inetsrv\appcmd.exe stop apppool demo"}

Write-Host '[-] 清空部署服务器的iis站点目录'
Invoke-Command -Session $Session -ScriptBlock {Remove-Item D:\iis_sites\demo\*  -force -recurse}

Write-Host '[-] 解压文件至iis站点目录’
Invoke-Command -Session $Session -ScriptBlock {cmd /c "cd /d D:\iis_sites\demo\ && D:\tools\7z.exe x D:\test\Packages\*.7z"}

Write-Host '[-] 开启应用程序池'
Invoke-Command -Session $Session -ScriptBlock {cmd /c "%windir%\system32\inetsrv\appcmd.exe start apppool demo"}

Write-Host '[-] 开启iis站点'
Invoke-Command -Session $Session -ScriptBlock {cmd /c "%windir%\system32\inetsrv\appcmd.exe start site demo"}

#Close Session
Remove-PSSession -Session $Session

GetRequest 'http://192.168.77.140'
```

### 测试构建


 点击任务的Build with Parametes, 选择dev-pages分支编译发布


![Paste_Image.png](/assets/images/ops/3629406-959207a9f90f3a3d.png)



### 构建日志
```
Started by user admin
Building in workspace C:\Program Files (x86)\Jenkins\workspace\Build-deploy_demo_140
Cloning the remote Git repository
Cloning repository https://git.oschina.net/lework/Webdemo.git
 > C:\Program Files\Git\bin\git.exe init C:\Program Files (x86)\Jenkins\workspace\Build-deploy_demo_140 # timeout=10
Fetching upstream changes from https://git.oschina.net/lework/Webdemo.git
 > C:\Program Files\Git\bin\git.exe --version # timeout=10
using GIT_ASKPASS to set credentials 
 > C:\Program Files\Git\bin\git.exe fetch --tags --progress https://git.oschina.net/lework/Webdemo.git +refs/heads/*:refs/remotes/origin/*
 > C:\Program Files\Git\bin\git.exe config remote.origin.url https://git.oschina.net/lework/Webdemo.git # timeout=10
 > C:\Program Files\Git\bin\git.exe config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
 > C:\Program Files\Git\bin\git.exe config remote.origin.url https://git.oschina.net/lework/Webdemo.git # timeout=10
Fetching upstream changes from https://git.oschina.net/lework/Webdemo.git
using GIT_ASKPASS to set credentials 
 > C:\Program Files\Git\bin\git.exe fetch --tags --progress https://git.oschina.net/lework/Webdemo.git +refs/heads/*:refs/remotes/origin/*
 > C:\Program Files\Git\bin\git.exe rev-parse "refs/remotes/origin/dev-pages^{commit}" # timeout=10
 > C:\Program Files\Git\bin\git.exe rev-parse "refs/remotes/origin/origin/dev-pages^{commit}" # timeout=10
Checking out Revision 7a1ffd4d8688273f537ef556d4a2c761181475a6 (refs/remotes/origin/dev-pages)
 > C:\Program Files\Git\bin\git.exe config core.sparsecheckout # timeout=10
 > C:\Program Files\Git\bin\git.exe checkout -f 7a1ffd4d8688273f537ef556d4a2c761181475a6
 > C:\Program Files\Git\bin\git.exe rev-list 7a1ffd4d8688273f537ef556d4a2c761181475a6 # timeout=10
Path To MSBuild.exe: C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe
Executing the command cmd.exe /C " "C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe" /t:clean /t:rebuild /p:Configuration=release /p:WebProjectOutputDir=d:\test\build_code\demo /p:OutputPath=d:\test\build_code\demo\bin /p:version=origin/dev-pages Webdemo.sln " && exit %%ERRORLEVEL%% from C:\Program Files (x86)\Jenkins\workspace\Build-deploy_demo_140
[Build-deploy_demo_140] $ cmd.exe /C " "C:\Program Files (x86)\MSBuild\14.0\Bin\msbuild.exe" /t:clean /t:rebuild /p:Configuration=release /p:WebProjectOutputDir=d:\test\build_code\demo /p:OutputPath=d:\test\build_code\demo\bin /p:version=origin/dev-pages Webdemo.sln " && exit %%ERRORLEVEL%%
Microsoft (R) 生成引擎版本 14.0.23107.0

....(编译日志)

已成功生成。
    0 个警告
    0 个错误

已用时间 00:00:37.51
[Build-deploy_demo_140] $ powershell.exe -NonInteractive -ExecutionPolicy ByPass "& 'C:\Users\ADMINI~1\AppData\Local\Temp\hudson1881669104991386343.ps1'"
[-] 清空上传目录
[-] 打包文件
7-Zip [64] 16.04 : Copyright (c) 1999-2016 Igor Pavlov : 2016-10-04

Scanning the drive:
15 folders, 161 files, 43669548 bytes (42 MiB)

Creating archive: D:\test\Packages\demo-20161217123320.7z

Items to compress: 176


Files read from disk: 161
Archive size: 6849624 bytes (6690 KiB)
Everything is Ok
[-] 清空部署服务器的上传目录
192.168.77.140 文件 【传送完毕】
[-] 关闭iis站点
“demo”已成功停止
[-] 关闭应用程序池
“demo”已成功停止
[-] 关闭iis站点
“demo”已成功停止
[-] 清空部署服务器的iis站点目录
[-] 解压文件至iis站点目录

7-Zip [64] 16.04 : Copyright (c) 1999-2016 Igor Pavlov : 2016-10-04

Scanning the drive for archives:
1 file, 6849624 bytes (6690 KiB)

Extracting archive: D:\test\Packages\demo-20161217123320.7z
--
Path = D:\test\Packages\demo-20161217123320.7z
Type = 7z
Physical Size = 6849624
Headers Size = 2698
Method = LZMA2:24 BCJ
Solid = +
Blocks = 2

Everything is Ok

Folders: 15
Files: 161
Size:       43669548
Compressed: 6849624
[-] 开启应用程序池
“demo”已成功启动。
[-] 开启iis站点
“demo”已成功启动。
[-] HTTP 状态吗：  200
Finished: SUCCESS
```
