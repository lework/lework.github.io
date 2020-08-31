---
layout: post
title: "初创型公司-持续部署系列（七）创建回滚任务"
date: "2016-12-10 19:09:53"
categories: 初创型公司运维专题
tags: jenkins
excerpt: "从上面的代码可以看到，我们在每次发布代码的同时也会备份线上的代码到D:\\Packages\\pre目录，回滚操作也就是把这个目录得代码拷贝到站点目..."
auth: lework
---
* content
{:toc}

> 从上面的代码可以看到，我们在每次发布代码的同时也会备份线上的代码到D:\Packages\pre目录，回滚操作也就是把这个目录得代码拷贝到站点目录，重启服务即可。

回滚流程图：

![Paste_Image.png](/assets/images/ops/3629406-f74e39ad88814565.png)

### 在jenkins上创建Deploy_rollback_demo任务

![Paste_Image.png](/assets/images/ops/3629406-eb52be105df6d870.png)

添加构建

![Paste_Image.png](/assets/images/ops/3629406-9639dfa9815bf561.png)

```powershell
[string]$xmldocpath = "D:\scriptes\config.xml"
$xmlDoc = New-Object "system.xml.xmldocument"
$xmlDoc.Load($xmldocpath)
$nodeList=$xmlDoc.GetElementsByTagName("Server");
$Script = {D:\scripts\deploy-rollback.bat}

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
}
```


D:\scriptes\deploy-rollback.bat 内容

```
@echo off
setlocal ENABLEDELAYEDEXPANSION

echo [-] 服务器从负载均衡移除

for /f "usebackq delims= " %%i in (`D:\tools\curl.exe -s -X PUT -d "{\"weight\":2, \"max_fails\":2, \"fail_timeout\":10, \"down\":1}" http://192.168.77.129:8500/v1/kv/upstreams/test/192.168.77.140:80`) do (
set status=%%i)


if "%status%" == "true" (
echo [-] Script Run Success.


echo [-] 关闭iis站点
%windir%\system32\inetsrv\appcmd.exe stop site "demo"

echo [-] 关闭应用程序池
%windir%\system32\inetsrv\appcmd.exe stop apppool "demo"

echo [-] copy iis站点文件到temp目录
rd /S /Q D:\Packages\temp\ && mkdir  D:\Packages\temp\

xcopy /s /e /h D:\iis_sites\demo\* D:\Packages\temp\

echo [-] 清空站点目录文件
rd /S /Q D:\iis_sites\demo\  && mkdir D:\iis_sites\demo

echo [-] copy pre目录文件到站点目录
xcopy /s /e /h D:\Packages\pre\* D:\iis_sites\demo\

echo [-] 开启应用程序池
%windir%\system32\inetsrv\appcmd.exe start apppool "demo"

echo [-] 开启iis站点
%windir%\system32\inetsrv\appcmd.exe start site "demo"

D:\tools\curl.exe -I -s http://192.168.77.140 | findstr "200 OK"
if not "%errorlevel%" == "0" (
echo [-] 检测站点出现错误
exit %errorlevel%
)

for /f "usebackq delims= " %%i in (`D:\tools\curl.exe -s -X PUT -d "{\"weight\":2, \"max_fails\":2, \"fail_timeout\":10, \"down\":0}" http://192.168.77.129:8500/v1/kv/upstreams/test/192.168.77.140:80`) do (
set result=%%i)

echo "!result!"

if "!result!" == "true" (
echo [-] 服务器成功添加到负载均衡
)else (
echo [-] 添加服务器,注册中心返回失败
)

) else (
echo [-] 删除服务器,注册中心返回失败
)
```

