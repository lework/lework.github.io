---
layout: post
title: "初创型公司-持续部署系列（六）创建发布任务"
date: "2016-12-10 19:09:36"
categories: 初创型公司运维专题
tags: jenkins
excerpt: "发布任务很简单，就一个powershell脚本，用来执行后端服务器的发布bat 编译打包的流程图如下： 在jenkins上创建Deploy_de..."
auth: lework
---
* content
{:toc}

> 发布任务很简单，就一个powershell脚本，用来执行后端服务器的发布bat

编译打包的流程图如下：


![Paste_Image.png](/assets/images/ops/3629406-dd300b4ec40dc30d.png)



### 在jenkins上创建Deploy_demo任务

![Paste_Image.png](/assets/images/ops/3629406-25113df66686bce4.png)

添加构建

![Paste_Image.png](/assets/images/ops/3629406-d11aa77cfbaf9b07.png)

```powershell
[string]$xmldocpath = "D:\scripts\config.xml"
$xmlDoc = New-Object "system.xml.xmldocument"
$xmlDoc.Load($xmldocpath)
$nodeList=$xmlDoc.GetElementsByTagName("Server");
$Script = {D:\scripts\deploy.bat}

foreach($node in $nodeList){
    $childNodes = $node.ChildNodes
	
	#Predefine necessary information
	$ComputerName = $childNodes.Item(0).InnerXml.ToString()     # 远端地址
	$Username = $childNodes.Item(1).InnerXml.ToString()         # 用户名
	$Password = $childNodes.Item(2).InnerXml.ToString()         # 密码

	#Create credential object
	$SecurePassWord = ConvertTo-SecureString -AsPlainText $Password -Force
	$Cred = New-Object -TypeName "System.Management.Automation.PSCredential" -ArgumentList $Username, $SecurePassWord

	#Create session object with this
	$Session = New-PSSession -ComputerName $ComputerName -credential $Cred

	#Invoke-Command
	$Job = Invoke-Command -Session $Session -Scriptblock $Script
	echo $Job

	#Close Session
	Remove-PSSession -Session $Session
	
	Write-Host $ComputerName  '主机 【执行完毕】' 
        Write-Host --------------------------------------------------------------------------
        Start-Sleep -s 5
}
```

D:\scriptes\config.xml  内容
```xml
<?xml version="1.0" encoding="utf-8"?>
<Servers>
   <Server>
      <RemoteHost>192.168.77.140</RemoteHost>
      <RemoteLoginName>administrator</RemoteLoginName>
      <RemotePwd>123456</RemotePwd>
   </Server>
   <Server>
      <RemoteHost>192.168.77.142</RemoteHost>
      <RemoteLoginName>administrator</RemoteLoginName>
      <RemotePwd>123456</RemotePwd>
   </Server>
</Servers>
```

### 后端服务器配置
> 在每台服务器上都要执行下列操作。

配置winrm执行权限
在powershell窗口执行下列命令

允许192.168.77.* 网段的主机远程执行winrm
```
Set-Item wsman:\localhost\Client\TrustedHosts -value 192.168.77.*
winrm quickconfig
```

![Paste_Image.png](/assets/images/ops/3629406-85fc74c84d486680.png)



### 创建目录

- D:\Packages\online   用来存放jenkins发来的压缩文件
- D:\Packages\old         发布的历史包
- D:\Packages\pre         用来存放上次发布的代码
- D:\Packages\temp     临时目录
- D:\scripts                     用于存放脚本文件
- D:\tools                           用于存放工具


在D:\scripts 目录下创建脚本文件deploy.bat

```
@echo off
setlocal ENABLEDELAYEDEXPANSION

if not exist  "D:\Packages\online\*.7z" (
echo [-] 没有代码包
exit 1
)


echo [-] 服务器从负载均衡移除

for /f "usebackq delims= " %%i in (`D:\tools\curl.exe -s -X PUT -d "{\"weight\":2, \"max_fails\":2, \"fail_timeout\":10, \"down\":1}" http://192.168.77.129:8500/v1/kv/upstreams/test/192.168.77.140:80`) do (
set status=%%i)


if "%status%" == "true" (
echo [-] Script Run Success.

echo [-] 暂停5秒
ping -n 5 127.0.0.1 > nul

echo [-] 关闭iis站点
%windir%\system32\inetsrv\appcmd.exe stop site "demo"

echo [-] 关闭应用程序池
%windir%\system32\inetsrv\appcmd.exe stop apppool "demo"

echo [-] copy iis站点文件
rd  /S /Q D:\Packages\pre\ && mkdir  D:\Packages\pre\

xcopy /s /e /h D:\iis_sites\demo\* D:\Packages\pre\

echo [-] 清空站点目录文件
rd /S /Q D:\iis_sites\demo\  && mkdir D:\iis_sites\demo

echo [-] 解压文件到站点目录
cd /d D:\iis_sites\demo\ && "D:\tools\7z.exe" x D:\Packages\online\*.7z

echo [-] 开启应用程序池
%windir%\system32\inetsrv\appcmd.exe start apppool "demo"

echo [-] 开启iis站点
%windir%\system32\inetsrv\appcmd.exe start site "demo"

echo [-] 移动发布包
move D:\Packages\online\*.7z D:\Packages\old

for /f "usebackq delims= " %%i in (`D:\tools\curl.exe -s -X PUT -d "{\"weight\":2, \"max_fails\":2, \"fail_timeout\":10, \"down\":0}" http://192.168.77.129:8500/v1/kv/upstreams/test/192.168.77.140:80`) do (
set result=%%i)

D:\tools\curl.exe -I -s http://192.168.77.140 | findstr "200 OK"
if not "%errorlevel%" == "0" (
echo [-] 检测站点出现错误
exit %errorlevel%
)

if "!result!" == "true" (
echo [-] 服务器成功添加到负载均衡
)else (
echo [-] 添加服务器,注册中心返回失败
)

) else (
echo [-] 删除服务器,注册中心返回失败
)
```

这里只给出一台服务器的配置，其他服务器按照这个步骤来，更改相应的ip地址即可。

注：远程执行命令，还可以使用微软得pstool工具。
