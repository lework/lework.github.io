---
layout: post
title: "Ansible 安全 之【命令审计】"
date: "2017-07-08 18:15:30"
categories: Ansible
excerpt: "服务器记录命令 实现该功能要求如下:1.接受审计的登录用户默认shell必须为bash2.bash版本至少3.00或以上 需要该要求的原因是实现..."
auth: lework
---
* content
{:toc}

## 服务器记录命令

实现该功能要求如下:
1.接受审计的登录用户默认shell必须为bash
2.bash版本至少3.00或以上

需要该要求的原因是实现功能的方法需要用到history命令的HISTTIMEFORMAT变量和PROMPT_COMMAND变量.
对于其他的shell我并未测试.如果其他shell可以实现这两个变量的功能那么理论上也可以使用.

实现方法如下:
使用root用户 操作
```
1.创建一个审计日志文件 
mkdir /var/log/shell_audit
touch /var/log/shell_audit/audit.log
2.将日志文件所有者赋予一个最低权限的用户  
chown nobody:nobody /var/log/shell_audit/audit.log
3.给该日志文件赋予所有人的写权限  
chmod 002 /var/log/shell_audit/audit.log
4.设置文件权限,使所有用户对该文件只有追加权限  
chattr +a /var/log/shell_audit/audit.log

5.编辑/etc/profile
在末尾添加下面内容
HISTSIZE=2048
HISTTIMEFORMAT="%Y/%m/%d %T   ";export HISTTIMEFORMAT
export HISTORY_FILE=/var/log/shell_audit/audit.log
export PROMPT_COMMAND='{ code=$?;thisHistID=`history 1|awk "{print \\$1}"`;lastCommand=`history 1| awk "{\\$1=\"\" ;print}"`;user=`id -un`;whoStr=(`who -u am i`);realUser=${whoStr[0]};logDay=${whoStr[2]};logTime=${whoStr[3]};pid=${whoStr[5]};ip=${whoStr[6]};if [ ${thisHistID}x != ${lastHistID}x ];then echo -E `date "+%Y/%m/%d %H:%M:%S"` $user\($realUser\)@$ip[PID:$pid][LOGIN:$logDay $logTime] --- [$PWD]$lastCommand [$code];lastHistID=$thisHistID;fi; } >> $HISTORY_FILE' 
 

重新登录后,即可看到/var/log/shell_audit/audit.log刷新的实时日志
2017/07/08 16:12:00 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:12:00 whoami [0]
2017/07/08 16:17:41 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:17:41 logrotate -vf /etc/logrotate.d/shell_audit [0]
2017/07/08 16:19:18 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:19:17 last [0]
2017/07/08 16:19:19 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:19:19 list [127]
2017/07/08 16:19:21 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:19:21 what [127]
2017/07/08 16:19:32 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:19:29 top [0]
2017/07/08 16:19:35 root(root)@(192.168.77.1)[PID:127876][LOGIN:2017-07-08 15:55] --- [/root] 2017/07/08 16:19:35 ps aux [0]


2017/07/08 16:12:00	记录时间/命令执行完成时间
root(root)	执行命令的用户(最初登录的用户) 
@(192.168.77.1)	登录的IP
[PID:127876]	最初登录时的LOGIN产生的PID
[LOGIN:2017-07-08 15:55]	命令执行开始的时间
[/root]	命令执行的当前目录
2017/07/08 16:12:00	命令执行开始的时间
whoami	执行的命令
[0]	命令返回的状态码

6. 设置audit.log的日志轮换
~]# cat /etc/logrotate.d/shell_audit 
/var/log/shell_audit/audit.log { 
    weekly  
    missingok 
    dateext 
    rotate 100
    sharedscripts 
    prerotate 
    /usr/bin/chattr -a /var/log/shell_audit/audit.log 
    endscript 
    sharedscripts 
    postrotate 
      /bin/touch /var/log/shell_audit/audit.log
      /bin/chmod 002 /var/log/shell_audit/audit.log
      /bin/chown nobody:nobody /var/log/shell_audit/audit.log
      /usr/bin/chattr +a /var/log/shell_audit/audit.log
    endscript 
}

可以测试一下！刚跑过了一次。
~]# logrotate -vf /etc/logrotate.d/shell_audit 
reading config file /etc/logrotate.d/shell_audit
reading config info for /var/log/shell_audit/audit.log 

Handling 1 logs

rotating pattern: /var/log/shell_audit/audit.log  forced from command line (100 rotations)
empty log files are rotated, old logs are removed
considering log /var/log/shell_audit/audit.log
  log needs rotating
rotating log /var/log/shell_audit/audit.log, log->rotateCount is 100
dateext suffix '-20170708'
glob pattern '-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
glob finding old rotated logs failed
running prerotate script
renaming /var/log/shell_audit/audit.log to /var/log/shell_audit/audit.log-20170708
running postrotate script
```


## 客户端记录日志
我们在使用xshell的时候，可以设置日志记录。

![image.png](/assets/images/Ansible/3629406-cb021cdb4baea1a7.png)
重新连接，在xshell窗口输入命令，该该窗口的所有信息都会记录到日志文件中。

![image.png](/assets/images/Ansible/3629406-b27f6265b4b4d5f8.png)
