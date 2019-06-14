---
layout: post
title: "Tomcat进程意外退出的问题分析"
date: "2018-05-10 16:01:25"
categories: 杂项
excerpt: "感谢同事宏江投递本稿。 节前某个部门的测试环境反馈tomcat会意外退出，我们到实际环境排查后发现不是jvm crash，日志里有进程销毁的记录..."
auth: lework
---
* content
{:toc}

感谢同事**宏江**投递本稿。

节前某个部门的测试环境反馈tomcat会意外退出，我们到实际环境排查后发现不是jvm crash，日志里有进程销毁的记录，从pause到destory的整个过程：

```
org.apache.coyote.AbstractProtocol pause
Pausing ProtocolHandler
org.apache.catalina.core.StandardService stopInternal
Stopping service Catalina
org.apache.coyote.AbstractProtocol stop
Stopping ProtocolHandler
org.apache.coyote.AbstractProtocol destroy
Destroying ProtocolHandler

```

从上面日志来可以判断：

##### 1) tomcat不是通过脚本正常关闭(viaport: 即通过8005端口发送shutdown指令)

因为正常关闭(viaport)的话会在 pause 之前有这样的一句warn日志：

```
    org.apache.catalina.core.StandardServer await
    A valid shutdown command was received via the shutdown port. Stopping the Server instance.
    然后才是 pause -> stop -> destory 

```

##### 2) tomcat的shutdownhook被触发，执行了销毁逻辑

而这又有两种情况，一是应用代码里有地方用`System.exit`来退出jvm，二是系统发的信号(`kill -9`除外，SIGKILL信号JVM不会有机会执行shutdownhook)

先通过排查代码，应用方和中间件团队都排查了`System.exit`在这个应用中使用的可能。那就只剩下Signal的情况了；经过一番排查后，发现每次tomcat意外退出的时间与ssh会话结束的时间正好吻合。

有了这个线索之后，银时同学立刻看了一下对方测试环境的脚本，简化后如下：

```
$ cat test.sh
#!/bin/bash
cd /data/server/tomcat/bin/
./catalina.sh start
tail -f /data/server/tomcat/logs/catalina.out

```

tomcat启动为后，当前shell进程并没有退出，而是挂住在tail进程，往终端输出日志内容。这种情况下，如果用户直接关闭ssh终端的窗口(用鼠标或快捷键)，则java进程也会退出。而如果先`ctrl-c`终止**test.sh**进程，然后再关闭ssh终端的话，则java进程不会退出。

这是一个有趣的现象，`catalina.sh start`方式启动的tomcat会把java进程挂到`init`(进程id为1)的父进程下，已经与当前`test.sh`进程脱离了父子关系，也与ssh进程没有关系，为什么关闭ssh终端窗口会导致java进程退出？

我们的推测是ssh窗口在关闭时，对当前交互的shell以及正在运行的test.sh等子进程发送某个退出的Signal，找了一台装有systemtap的机器来验证，所用的stap脚本是从涧泉同学那里copy的：

```
function time_str: string () {
    return ctime(gettimeofday_s() + 8 * 60 * 60);
}

probe begin {
    printdln(" ", time_str(), "BEGIN");
}

probe end {
    printdln(" ", time_str(), "END");
}

probe signal.send {
    if (sig_name == "SIGHUP" || sig_name == "SIGQUIT" || 
        sig_name=="SIGINT" || sig_name=="SIGKILL" || sig_name=="SIGABRT") {
        printd(" ", time_str(), sig_name, "[", uid(), pid(), cmdline_str(), 
                "] -> [", task_uid(task), sig_pid, pid_name, "], ");
        task = pid2task(pid());
        while (task_pid(task) > 0) {
            printd(" ", "[", task_uid(task), task_pid(task), task_execname(task), "]");
            task = task_parent(task);
        }
        println("");
    }
}

```

模拟时的进程层级(pstree)大致如下，tomcat启动后java进程已经脱离test.sh，挂在init下：

```
|-sshd(1622)-+-sshd(11681)---sshd(11699)---bash(11700)---test.sh(13285)---tail(13299)

```

经过内核组伯俞的协助，我们发现

##### a) 用 ctrl-c 终止当前test.sh进程时，系统events进程向 java 和 tail 两个进程发送了`SIGINT` 信号

```
SIGINT [ 0 11  ] -> [ 0 20629 tail ] 
SIGINT [ 0 11  ] -> [ 0 20628 java ] 
SIGINT [ 0 11  ] -> [ 0 20615 test.sh ] 

注pid 11是events进程

```

##### b) 关闭ssh终端窗口时，sshd向下游进程发送`SIGHUP`, 为何java进程也会收到？

```
SIGHUP [ 0 11681 sshd: hongjiang.wanghj [priv] ] -> [ 57316 11700 bash ] 
SIGHUP [ 57316 11700 -bash ] -> [ 57316 11700 bash ]
SIGHUP [ 57316 11700 ] -> [ 0 13299 tail ] 
SIGHUP [ 57316 11700 ] -> [ 0 13298 java ] 
SIGHUP [ 57316 11700 ] -> [ 0 13285 test.sh ] 

```

不过伯俞很忙没有继续协助分析这个问题(他给出了一些猜测，但后来证明并不是那样)。

确定了是由signal引起的之后，我的疑惑变成了：

##### 1) 为什么`SIGINT` (kill -2) 不会让tomcat进程退出？

##### 2) 为什么`SIGHUP` (kill -1) 会让tomcat进程退出?

我第一反应可能是jvm在某些参数下(或因为某些jni)对os的信号处理会不同，看了一下应用的jvm参数，没有看出问题，也排除了tomcat使用apr/tcnative的情况。

我们看一下默认情况下，jvm进程对`SIGINT`和`SIGHUP`是怎么处理的，用scala的repl模拟一下：

```
scala> Runtime.getRuntime().addShutdownHook(
            new Thread() { override def run() { println("ok") } })

```

对这个java进程分别用`kill -2`和`kill -1`发现都会导致jvm进程退出，并且也触发`shutdownhook`。这也符合oracle对hotspot虚拟机处理Signal的说明，参考[这里](http://www.oracle.com/technetwork/java/javase/signals-139944.html#gbzbl)，`SIGTERM`,`SIGINT`,`SIGHUP`三种信号都会触发`shutdownhook`

看来并不是jvm的事，继续猜测是否与进程的状态有关？catalina.sh脚本里并没有使用`start-stop-daemon`之类的方式启动java进程，start参数的执行方式简化后脚本相当于：

```
eval '"/pathofjdk/bin/java"' 'params' org.apache.catalina.startup.Bootstrap start '&'

```

就是简单的把java放到后台执行。当catalina.sh自身进程退出后，java进程的ppid变成了1

花了很多的时间猜测可能是OS层面的原因，后来发现并没有关系。春节后回来让少明和涧泉也一起分析这个问题，因为他们有c的背景，对系统底层知道的多一些，用了大半天时间，不断猜测和验证，最后确认了是Shell的原因。

#### `SIGINT` (kill -2) 不会让后台java进程退出的原因

为了简便，我们用sleep来模拟进程，当我们在交互模式下：

```
$ sleep 1000 & 

$ ps -opid,pgid,ppid,stat,cmd -C sleep
  PID  PGID  PPID STAT CMD
 9897  9897  9813 S    sleep 1000   

```

注意，进程`sleep 1000`的pid与pgid(进程组)是相同的，这时我们用`kill -2`是可以杀掉`sleep 1000`进程的。

现在我们把sleep进程放到一个脚本里后台执行：

```
$ cat a.sh
#!/bin/sh
sleep 4400 &
echo "shell exit"

```

运行a.sh脚本之后，`sleep 4400`进程的pid与pgid是不同的，pgid是其父进程的id，即已经退出了的a.sh进程

```
$ ps -opid,pgid,ppid,comm -p 63376
  PID  PGID  PPID COMM
63376 63375     1 sleep

```

这时我们用`kill -2`是杀不掉`sleep 4400`进程的。

到了这一步，已经非常接近原因了，一定是shell对后台进程`signal_handler`做了什么手脚。少明实现了一个自定handler的命令看看是否对`kill -2`有效：

```
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>

void my_handler(int sig) {
    printf("handler aaa\n");
    exit(0);
}

int main() {
    signal(SIGINT, my_handler);
    for(;;) { }
    return 0;
}

```

我们把编译后的a.out命令在脚本里以后台方式运行：

```
$ cat a.sh
#!/bin/sh
/tmp/a.out &

```

这次再尝试用`kill -2`去杀a.out进程，是可以的。这说明shell对`signal_handler`做手脚是在执行用户逻辑之前，也就是脚本在fork出子进程的时候就设置了。按照这个线索我们google后了解到: shell在**非交互模式**下对后台进程处理`SIGINT`信号时设置的是`IGNORE`。

###### 交互模式与非交互模式对作业控制(job control)默认方式不同

为什么在交互模式下shell不会对后台进程处理`SIGINT`信号设置为忽略，而非交互模式下会设置为忽略呢？还是比较好理解的，举例来说，我们先某个前台进程运行时间太长，可以`ctrl-z`中止一下，然后通过`bg %n`把这个进程放入后台，同样也可以把一个`cmd &`方式启动的后台进程，通过`fg %n`放回前台，然后在`ctrl-c`停止它，当然不能忽略`SIGINT`。

为何交互模式下的后台进程会设置一个自己的进程组ID呢？因为默认如果采用父进程的进程组ID，父进程会把收到的键盘事件比如`ctrl-c`之类的`SIGINT`传播给进程组中的每个成员，假设后台进程也是父进程组的成员，因为作业控制的需要不能忽略`SIGINT`，你在终端随意`ctrl-c`就可能导致所有的后台进程退出，显然这样是不合理的；所以为了避免这种干扰后台进程设置为自己的pgid。

而非交互模式下，通常是不需要作业控制的，所以作业控制在非交互模式下默认也是关闭的（当然也可以在脚本里通过选项`set -m`打开作业控制选项）。不开启作业控制的话，脚本里的后台进程可以通过设置忽略`SIGINT`信号来避免父进程对组中成员的传播，因为对它来说这个信号已经没有意义。

回到tomcat的例子，catalina.sh脚本通过start参数启动的时候，就是以非交互方式后台启动，java进程也被shell设置了忽略`SIGINT`信号，因此在`ctrl-c`结束test.sh进程时，系统发送的`SIGINT`对java没有影响。

#### `SIGHUP` (kill -1) 让tomcat进程退出的原因

在非交互模式下，shell对java进程设置了`SIGINT`，`SIGQUIT`信号设置了忽略，但并没有对`SIGHUP`信号设为忽略。再看一下当时的进程层级：

```
|-sshd(1622)-+-sshd(11681)---sshd(11699)---bash(11700)---test.sh(13285)---tail(13299)

```

sshd把`SIGHUP`传递给bash进程后，bash会把`SIGHUP`传递给它的子进程，并且对于其子进程test.sh，bash还会对test.sh的进程组里的成员都传播一遍`SIGHUP`。因为java后台进程从父进程catalina.sh(又是从其父进程test.sh)继承的pgid，所以java进程仍属于test.sh进程组里的成员，收到`SIGHUP`后退出。

如果我们在test.sh里设置开启作业控制的话，就不会让java进程退出了

```
#!/bin/bash
set -m  
cd /home/admin/tt/tomcat/bin/
./catalina.sh start
tail -f /home/admin/tt/tomcat/logs/catalina.out

```

此时java后台进程继承父进程catalina.sh的pgid，而catalina.sh不再使用test.sh的进程组，而是自己的pid作为pgid，catalina.sh进程在执行完退出后，java进程挂到了init下，java与test.sh进程就完全脱离关系了，bash也不会再向它发送信号。

**原创文章，转载请注明：** 转载自[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Tomcat进程意外退出的问题分析](http://ifeve.com/why-kill-2-cannot-stop-tomcat/)
