---
layout: post
title: "Ansible 小手册系列 二十一（管理windows系统）"
date: "2017-04-09 13:46:10"
categories: Ansible
excerpt: "windows 系统要求 PowerShell 3 或更高 不支持Cygwin 在Windows 7和Server 2008 R2计算机上，由于..."
auth: lework
---
* content
{:toc}

## windows 系统要求
- PowerShell 3 或更高
- 不支持Cygwin

>在Windows 7和Server 2008 R2计算机上，由于Windows Management Framework 3.0中的错误，可能需要安装此修补程序 http://support.microsoft.com/kb/2842230 以避免收到内存和堆栈溢出异常。已知新安装的Server 2008 R2系统与Windows更新不兼容，具有此问题。
Windows 8.1和Server 2012 R2不受Windows Management Framework 4.0附带的此问题的影响。

## inventory 示例文件
```yml
[windows]
winserver1.example.com
winserver2.example.com

[windows:vars]
ansible_user=Administrator
ansible_password=123456
ansible_port=5986
ansible_connection=winrm
# The following is necessary for Python 2.7.9+ (or any older Python that has backported SSLContext, eg, Python 2.7.5 on RHEL7) when using default WinRM self-signed certificates:
ansible_winrm_server_cert_validation=ignore
```

## inventory 可用变量
- ansible_winrm_scheme：指定用于WinRM连接的连接方案（http或https）。https默认情况下可以使用，除非端口是5985。
- ansible_winrm_path：指定WinRM端点的备用路径。/wsman默认情况下可以使用。
- ansible_winrm_realm：指定用于Kerberos身份验证的领域。如果用户名包含@，Ansible将@默认使用部分用户名。
- ansible_winrm_transport：以一个逗号分隔的列表指定一个或多个传输。默认情况下，kerberos,plaintext如果已kerberos安装模块并定义了一个域，则“ 可用”将会使用，否则plaintext。
- ansible_winrm_server_cert_validation：指定服务器证书验证模式（ignore或validate）。validatePython 2.7.9及更高版本的可选默认值将导致Windows自签名证书的证书验证错误。除非在WinRM侦听器上配置了可验证的证书，否则应将其设置为ignore
- ansible_winrm_kerberos_delegation：设置为true在远程主机上使用Kerberos时启用命令委派。
- ansible_winrm_*：winrm.Protocol可以提供支持的任何其他关键字参数。

## 有什么模块可以用的？
	• add_host
	• assert
	• async
	• debug
	• fail
	• fetch
	• group_by
	• include_vars
	• meta
	• pause
	• raw
	• script
	• set_fact
	• setup
	• slurp
	• template (also: win_template)

专门用作windows系统的模块
http://docs.ansible.com/ansible/list_of_windows_modules.html

## 获取 Windows Facts
```
ansible winhost.example.com -m setup
```
## Windows Playbook示例
```
执行powershell脚本
- name: test script module
  hosts: windows
  tasks:
    - name: run test script
      script: files/test_script.ps1

获取ip地址信息
- name: test raw module
  hosts: windows
  tasks:
    - name: run ipconfig
      win_command: ipconfig
      register: ipconfig
    - debug: var=ipconfig

使用dos命令，移动文件
- name: another raw module example
  hosts: windows
  tasks:
     - name: Move file on remote Windows Server from one location to another
       win_command: CMD /C "MOVE /Y C:\teststuff\myfile.conf C:\builds\smtp.conf"

使用powershell命令，移动文件
- name: another raw module example demonstrating powershell one liner
  hosts: windows
  tasks:
     - name: Move file on remote Windows Server from one location to another
       win_command: Powershell.exe "Move-Item C:\teststuff\myfile.conf C:\builds\smtp.conf"

查看文件状态
- name: test stat module
  hosts: windows
  tasks:
    - name: test stat module on file
      win_stat: path="C:/Windows/win.ini"
      register: stat_file

    - debug: var=stat_file

    - name: check stat_file result
      assert:
          that:
             - "stat_file.stat.exists"
             - "not stat_file.stat.isdir"
             - "stat_file.stat.size > 0"
             - "stat_file.stat.md5"
```
---

## 本次实验环境
windows os：`Microsoft Windows Server 2008 R2 Enterprise with sp1 x64`
ansible manager：`centos 6.7 X64`
ansible version： `2.2.1.0`
python version：`Python 2.6.6`


##  windows 侧配置
1. 以管理员身份打开powershell
![Paste_Image.png](/assets/images/Ansible/3629406-4b35fd357296eb59.png)
2. 查看当前ps版本
```$PSVersionTable```
![Paste_Image.png](/assets/images/Ansible/3629406-252c48591e7bfbfb.png)

3. 系统自带的powershell版本是2.0，需要更新至powershell 3 以上版本
  a. 下载安装Microsoft .NET Framework 4
https://www.microsoft.com/en-us/download/details.aspx?id=17851
	b. 下载安装Windows Management Framework 3.0
https://www.microsoft.com/en-us/download/details.aspx?id=34595
选择 Windows6.1-KB2506143-x64.msu
c.安装完，重启服务器,查看powershell版本
```$PSVersionTable```
![Paste_Image.png](/assets/images/Ansible/3629406-d5a6c422988546c8.png)
4. 配置winrm
```
mkdir C:\work
cd C:\work
Invoke-WebRequest -Uri https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1 -OutFile ConfigureRemotingForAnsible.ps1
powershell -ExecutionPolicy RemoteSigned .\ConfigureRemotingForAnsible.ps1 -SkipNetworkProfileCheck
```
![Paste_Image.png](/assets/images/Ansible/3629406-eec5f63a38527eab.png)

## ansible manager 配置
1. 安装ansible和pywinrm
```
yum -y install ansible
curl -sL https://bootstrap.pypa.io/get-pip.py | python
pip install pywinrm
```
2. 配置hosts
```
[root@master ~]# cd /etc/ansible/
[root@master ansible]# cat win_hosts 
[windows]
192.168.77.137
[windows:vars]
ansible_user=Administrator
ansible_password=Q123.123
ansible_port=5986
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
```
3. 测试通信
```ansible -i win_hosts windows -m win_ping```
![Paste_Image.png](/assets/images/Ansible/3629406-bdaefc8e274ffb66.png)
4. 查看ip地址
```ansible -i win_hosts windows -m win_command -a "ipconfig"```
![Paste_Image.png](/assets/images/Ansible/3629406-bfbb4bdc55bec3a0.png)
```ansible -i win_hosts windows -m raw -a "ipconfig"```
![Paste_Image.png](/assets/images/Ansible/3629406-d47767a177bd86e0.png)

修改上面的中文问题：
对命令输出的信息进行utf-8编码，修改winrm模块的protocol.py
```
sed -i "s#tdout_buffer.append(stdout)#tdout_buffer.append(stdout.decode('gbk').encode('utf-8'))#g" /usr/lib/python2.6/site-packages/winrm/protocol.py
sed -i "s#stderr_buffer.append(stderr)#stderr_buffer.append(stderr.decode('gbk').encode('utf-8'))#g" /usr/lib/python2.6/site-packages/winrm/protocol.py
```
![Paste_Image.png](/assets/images/Ansible/3629406-f45899120e0c7463.png)
修改完之后，重新运行命令，中文已正常显示。
![Paste_Image.png](/assets/images/Ansible/3629406-1ceab5296c7d5b20.png)
