---
layout: post
title: "利用 Supervisor 的 Event & Listener 监控进程并报警"
date: "2019-10-16 22:00:00"
category: supervisor
tags: supervisor
author: lework
---
* content
{:toc}

## 需求

监控supervisor管理的进程状态，在进程异常退出的时候给出报警。

## 实现思路

利用 Supervisor 的 Event & Listener 功能进行订阅异常退出事件，并进行报警处理。

Supervisor 官方对其 Event 机制的描述是：[一个进程的监控/通知框架](http://supervisord.org/events.html#event-listeners-and-event-notifications)。

该机制主要通过一个 event listener 订阅 event 通知实现。当被 Supervisor 管理的进程有特定行为的时候，supervisor 就会自动发出对应类型的 event。即使没有配置 listener，这些 event 也是会发的；如果配置了 listener 并监听该类型的 event，那么这个 listener 就会接收到该 event。
event listener 需要自己实现，并像 program 一样，作为 superviosr 的子进程运行。




下面就简单的介绍下event

### Event Types
Event Type 由 supervisor 官方定义，我们是没法自己添加的，不过官方已经定义了很多类型了，够我们使用的了。
全部的 Event Type 可以参考文档：http://supervisord.org/events.html#event-types

这里以其中一个类型为例 PROCESS_STATE_EXITED，顾名思义，当被管理的子进程退出的时候，就会产生该 event。（关于进程状态请参考 http://supervisord.org/subprocess.html#process-states ）
当 supervisord 发送一个 event 到 listener 时，会先发送一个 “header” 过去，类似这样

```
ver:3.0 server:supervisor serial:21 pool:listener poolserial:10 eventname:PROCESS_COMMUNICATION_STDOUT len:54
```
header 中每项的含义：

| Key        | Description                                                  |
| :--------- | :----------------------------------------------------------- |
| var        | event 协议类型，目前 3.0                                     |
| server     | supervisor 的标识符，对应配置文件中 [supervisord] 块的 identifier |
| serial     | event 的序列号                                               |
| pool       | listener 的 pool 的名字，如果 listener 只启动了一个进程，也就没有 pool 的概念了 |
| poolserial | eventpool 给发送到我这个 pool 过来的 event 编的号，有点绕，只要知道与上边的 serial 不同就行了 |
| eventname  | event 类型名称                                               |
| len        | header 后面的 payload 部分的长度，又称 `PAYLOAD_LENGTH`      |

对于不同的 event type，header 的结构都是一样的，而 payload 的数据结果与类型相关。

PROCESS_STATE_EXITED 的 payload 结构如下：

```
processname:cat groupname:cat from_state:RUNNING expected:0 pid:2766
```
该 payload 中每项的含义：

| Key         | Description                                                  |
| :---------- | :----------------------------------------------------------- |
| processname | 进程名 cat [program:cat]                                     |
| groupname   | 进程组名                                                     |
| from_state  | 进程在退出前是什么状态                                       |
| expected    | 默认情况下 exitcodes=0,2，当退出码为 0 或 2 时，是 expected 的，此时该值为 1；其它的退出码，也就是 unexpected 了，该值为 0 |
| pid         | 退出的进程的 pid                                             |

### Event Listener

eventlistener配置

```
[eventlistener:mylistener]
command=my_custom_listener.py
events=PROCESS_STATE
```

### Event Listener States
一个 listener 进程可能会处于以下三种状态：

| Name         | Description                           |
| :----------- | :------------------------------------ |
| ACKNOWLEDGED | 确认，相当于注册上了这个 listener     |
| READY        | 就绪，event 可以被发送到这个 listener |
| BUSY         | 忙碌，event 不能被发送到这个 listener |

当一个 listener 启动后，会进入 ACKNOWLEDGED 状态。然后它会向自己的 stdout 写一个 READY\n，进入 READY 状态。supervisor 向 READY 状态的 listener 发一个 event 后，listener 进入 BUSY 状态。listener 返回 OK 或 FAIL 后，回到 ACKNOWLEDGED 状态。

### Event Listener Notification Protocol
1. listener 处于 `READY` 时，当 supervisord 产生的 event 在 listener 的配置的可接受的 events 中时，supervisord 就会把该 event 发送给该 listener，并将其状态置为 `BUSY`。
2. listener 先处理 `header` 获取 `len` 并据其读取 `payload`，处理 `payload` 中的数据，这时候如果有相同类型的 event 产生，supersivor 会将该 event 发给相同 listener pool 中的其他 listener。
3. listerner 处理完数据后，要向自己的 stdout 中写一条消息以告诉 supervisor 处理结果，例如 `RESULT 2\nOK` 或 `RESULT 4\nFAIL`
4. supervisor 收到 listener 返回的结果，若为 `OK` 就认为处理成功；若为 `FAIL`，认为 event 处理失败，会把那个 event 再次放入缓存队列并稍后再次发送。不管收到 `OK` 还是 `FAIL`，这个 listener 都会被置为 `ACKNOWLEDGED` 状态。
5. listener 被置为 `ACKNOWLEDGED` 状态后，这个 listener 进程可以退出并稍后自启（配置中 `autorestart=true` 的话），也可以继续运行。如果要继续运行，则它必须立即向自己的 stdout 中发送 `READY` 以让 supervisor 将其状态置为 `READY`。

### Event Listener Demo
先看一个官方的用 python 写的 demo 了解一下流程，

```
import sys

def write_stdout(s):
    # only eventlistener protocol messages may be sent to stdout
    sys.stdout.write(s)
    sys.stdout.flush()

def write_stderr(s):
    sys.stderr.write(s)
    sys.stderr.flush()

def main():
    while 1:
        # transition from ACKNOWLEDGED to READY
        write_stdout('READY\n')

        # read header line and print it to stderr
        line = sys.stdin.readline()
        write_stderr(line)

        # read event payload and print it to stderr
        headers = dict([ x.split(':') for x in line.split() ])
        data = sys.stdin.read(int(headers['len']))
        write_stderr(data)

        # transition from READY to ACKNOWLEDGED
        write_stdout('RESULT 2\nOK')

if __name__ == '__main__':
    main()
```

从理论上来说，supervisor 的 event listener 可以用任何语言实现，但是 python 是实现起来比较容易的，因为有一个名为 `supervisor.childutils` 模块可以方便我们处理 event。另外 [superlance](https://pypi.python.org/pypi/superlance) 这个 package 中，有几个 event listener 的栗子，哦对了，这是个 python 的 package。



## 实现步骤

### 实现脚本

脚本内容

```
#!/usr/bin/python
# -*- coding: utf-8 -*-
#
# @Time    : 2019-10-16
# @Author  : lework
# @Desc    : 一个事件监听器，订阅PROCESS_STATE_CHANGE事件。当supervisor管理的进程意外过渡到EXITED状态时，它将发送邮件。

# [eventlistener:supervisor_event_exited]
# process_name=%(program_name)s
# command=/usr/bin/python /data/scripts/supervisor_event_exited.py
# autostart=true
# autorestart=true
# events=PROCESS_STATE
# log_stdout=true
# log_stderr=true
# stdout_logfile=/var/log/supervisor/supervisor_event_exited-stdout.log
# stdout_logfile_maxbytes=50MB
# stdout_logfile_backups=3
# buffer_size=10
# stderr_logfile=/var/log/supervisor/supervisor_event_exited-stderr.log
# stderr_logfile_maxbytes=50MB
# stderr_logfile_backups=3


import os
import smtplib
import socket
import sys
from supervisor import childutils
from email.header import Header
from email.mime.text import MIMEText


class CrashMail:
    def __init__(self, mail_config, programs):
        self.mail_config = mail_config
        self.programs = programs
        self.stdin = sys.stdin
        self.stdout = sys.stdout
        self.stderr = sys.stderr
        self.time = ''

    def write_stderr(self, s):
        s = s+'\n'
	if self.time:
           s = '[%s] %s' % (self.time, s)
        self.stderr.write(s)
        self.stderr.flush()

    def runforever(self):
        # 死循环, 处理完 event 不退出继续处理下一个
        while 1:
            # 使用 self.stdin, self.stdout, self.stderr 代替 sys.*
            headers, payload = childutils.listener.wait(self.stdin, self.stdout)

            self.time = childutils.get_asctime()
            self.write_stderr('[headers] %s' % str(headers))
            self.write_stderr('[payload] %s' % str(payload))

            # 不处理不是 PROCESS_STATE_EXITED 类型的 event, 直接向 stdout 写入"RESULT\nOK"
            if headers['eventname'] != 'PROCESS_STATE_EXITED':
                childutils.listener.ok(self.stdout)
                continue

            # 解析 payload, 这里我们只用这个 pheaders.
            # pdata 在 PROCESS_LOG_STDERR 和 PROCESS_COMMUNICATION_STDOUT 等类型的 event 中才有
            pheaders, pdata = childutils.eventdata(payload + '\n')

            # 如果在programs中设置，就只处理programs中的，否则全部处理.
            if len(self.programs) !=0 and pheaders['groupname'] not in self.programs:
                childutils.listener.ok(self.stdout)
	        continue

            # 过滤掉 expected 的 event, 仅处理 unexpected 的
            # 当 program 的退出码为对应配置中的 exitcodes 值时, expected=1; 否则为0
            if int(pheaders['expected']):
                childutils.listener.ok(self.stdout)
                continue

            # 获取系统主机名和ip地址
            hostname = socket.gethostname()
            ip = socket.gethostbyname(hostname)

            # 构造报警内容
            msg = "Host: %s(%s)\nProcess: %s\nPID: %s\nEXITED unexpectedly from state: %s" % \
                  (hostname, ip, pheaders['processname'], pheaders['pid'], pheaders['from_state'])

            subject = '[Supervistord] %s crashed at %s' % (pheaders['processname'], self.time)

            self.write_stderr('[INFO] unexpected exit, mailing')

            # 发送邮件
            self.send_mail(subject, msg)

            # 向 stdout 写入"RESULT\nOK"，并进入下一次循环
            childutils.listener.ok(self.stdout)

    def send_mail(self, subject, content):
        """
        :param subject: str
        :param content: str
        :return: bool
        """

        mail_port = self.mail_config.get('mail_port', '')
        mail_host = self.mail_config.get('mail_host', '')
        mail_user = self.mail_config.get('mail_user', '')
        mail_pass = self.mail_config.get('mail_pass', '')
        to_list = self.mail_config.get('to_list', [])

        msg = MIMEText(content, _subtype='plain', _charset='utf-8')
        msg['Subject'] = Header(subject, 'utf-8')
        msg['From'] = mail_user
        msg['to'] = ",".join(to_list)
        try:
            s = smtplib.SMTP_SSL(mail_host, mail_port)
            s.login(mail_user, mail_pass)
            s.sendmail(mail_user, to_list, msg.as_string())
            s.quit()
            self.write_stderr('[mail] ok')
            return True
        except Exception as e:
            self.write_stderr('[mail] error\n\n%s\n' % e)
            return False


def main():
    # listener 必须交由 supervisor 管理, 直接运行是不行的
    if not 'SUPERVISOR_SERVER_URL' in os.environ:
        sys.stderr.write('crashmail must be run as a supervisor event '
                         'listener\n')
        sys.stderr.flush()
        return

    # 设置smtp信息
    mail_config = {
        'mail_host': 'smtp.lework.com',
        'mail_port': '465',
        'mail_user': 'ops@lework.com',
        'mail_pass': '123123123',
        'to_list': ['lework@lework.com']
    }

    # 设置要检测的program,不设置则检测全部
    programs = []
    prog = CrashMail(mail_config, programs)
    prog.runforever()


if __name__ == '__main__':
    main()

```

这里需要配置下`smtp信息`

### 配置supervisor

```
[program:sleep]
process_name=%(program_name)s
command=/usr/bin/sleep 100
events=EVENT
autostart=true
autorestart=true

[program:sleep2]
process_name=%(program_name)s
command=/usr/bin/sleep 1000
events=EVENT
autostart=true
autorestart=true

[eventlistener:supervisor_event_exited]
process_name=%(program_name)s
command=/usr/bin/python /root/scripts/supervisor_event_exited.py
autostart=true
autorestart=true
events=PROCESS_STATE
log_stdout=true
log_stderr=true
stdout_logfile=/var/log/supervisor/supervisor_event_exited-stdout.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=3
buffer_size=10
stderr_logfile=/var/log/supervisor/supervisor_event_exited-stderr.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=3
```

> 注意：eventlistener中的配置如果更改，使用supervisorctl update是无效的，可以使用supervisorctl reload，或者删除eventlistener配置更新后，再次添加并更新。
>

启动后，kill掉sleep的进程，你就能收到邮件啦。



## 参考

- https://windmt.com/2016/02/02/supervisor-event/