---
layout: post
title: "初创型公司-持续部署系列（五）创建编译任务"
date: "2016-12-10 19:09:18"
categories: 初创型公司运维专题
tags: jenkins
excerpt: "编译打包的流程图如下： 在jenkins服务器上创建目录 D:\\build_code\\demo目录，用于存放编译后的代码 D:\\Packages..."
auth: lework
---
* content
{:toc}

编译打包的流程图如下：


![Paste_Image.png](/assets/images/ops/3629406-a12d7f92220300b1.png)


### 在jenkins服务器上创建目录

- D:\build_code\demo目录，用于存放编译后的代码
- D:\Packages\upload目录，用于存放代码压缩文件
- D:\Packages\old目录，用于存放历史代码压缩文件
- D:\scripts目录，用于存放脚本文件
- D:\tools目录，用于存放工具

### 在jenkins上创建Build_demo任务

![Paste_Image.png](/assets/images/ops/3629406-bdc3b8ffe8da98e1.png)

配置源码管理

![Paste_Image.png](/assets/images/ops/3629406-43f40df52c2bf351.png)

配置构建

![Paste_Image.png](/assets/images/ops/3629406-0368c72f3f7b73d5.png)


cmd命令：
```bat
"C:\Program Files (x86)\MSBuild\14.0\Bin\MSBuild.exe" webdemo.sln /t:clean /t:rebuild /p:Configuration=release  /p:WebProjectOutputDir=d:\build_code\demo /p:OutputPath=d:\build_code\demo\bin
```

配置打包程序的脚本

![Paste_Image.png](/assets/images/ops/3629406-f61ff4ff0d6f39a8.png)

```powershell
$datetime=Get-Date -Format 'yyyyMMddHHmmss'  #定义日期
Move-Item D:\Packages\upload\* D:\Packages\old\   #备份上次发布的文件
D:\tools\7z.exe a D:\Packages\upload\demo-$datetime.7z D:\build_code\demo\*    压缩编译代码
```

配置发送压缩文件到后端服务器的脚本

![Paste_Image.png](/assets/images/ops/3629406-fd2dd9e1a09fcbba.png)

```powershell
[string]$xmldocpath = "D:\scripts\config.xml"         #读取xml配置文件
 
$xmlDoc = New-Object "system.xml.xmldocument"
$xmlDoc.Load($xmldocpath)
$nodeList=$xmlDoc.GetElementsByTagName("Server");


foreach($node in $nodeList){
	$childNodes = $node.ChildNodes
	#Predefine necessary information
	$ComputerName = $childNodes.Item(0).InnerXml.ToString()     # 远端地址
	$Username = $childNodes.Item(1).InnerXml.ToString()         # 用户名
	$Password = $childNodes.Item(2).InnerXml.ToString()         # 密码

	#创建用户认证
	$SecurePassWord = ConvertTo-SecureString -AsPlainText $Password -Force
	$Cred = New-Object -TypeName "System.Management.Automation.PSCredential" -ArgumentList $Username, $SecurePassWord

	#创建session
	$Session = New-PSSession -ComputerName $ComputerName -credential $Cred
	
	#拷贝文件到后端服务器
	Copy-Item "D:\Packages\upload\demo-*.7z" -Destination "\\$ComputerName\d$\Packages\online"

	#关闭session
	Remove-PSSession -Session $Session
	
	Write-Host $ComputerName  '主机 【执行完毕】' 
}
```
D:\scriptes\config.xml 内容
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

如果想要发送完成后邮件通知，可以配置


![Paste_Image.png](/assets/images/ops/3629406-185269f6a5caa18e.png)



### msbuild参数解释
`webdemo.sln:`    工程文件路径
`/t:clean: ` 编译前清理上次编译的文件
`/t:rebuild： `采用重新编译
`/p:Configuration=release：`  release模式编译，还有debug模式
`/p:WebProjectOutputDir： `web项目文件输出目录
`/p:OutputPath：` dll文件输出目录

一些其他得参数：
`PublishProfile： `在用VS执行发布操作时，会生成这个文件。里面指定了发布操作的各种配置参数，比如发布路径，基于X86/X64平台等
`SolutionDir： `解决方案文件夹
`VisualStudioVersion： `安装VisualStudio时会安装一些SDK，这个参数告诉MSBuild该去哪里找这些SDK。由于我安装的是VisualStudio 2013，因此此处填写了14.0。如果不想每次都填写这个参数，也可以在csproj里面搜索这个参数名称，填写一个默认值。
`Platform=x86：`编译程序所使用得
`TargetFrameworkVersion=v3.5： `指定编译的.net版本号

官方文档：https://msdn.microsoft.com/zh-cn/library/0k6kkbsd.aspx

