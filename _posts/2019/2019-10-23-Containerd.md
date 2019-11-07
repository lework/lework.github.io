---
layout: post
title: 'Docker 组件 Containerd'
date: '2019-10-23 22:00:00'
category: docker
tags: docker
author: lework
---
* content
{:toc}


从Docker 1.11之后，Docker Daemon被分成了多个模块以适应OCI标准。拆分之后，结构分成了以下几个部分。

 ![img](/assets/images/docker/containerd.png) 

从图中可以看出，docker 对容器的管理和操作基本都是通过 containerd 完成的。 那么，containerd 是什么呢？

**[Containerd]( [https://containerd.io](https://containerd.io/) ) 是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性。Containerd 可以在宿主机中管理完整的容器生命周期：容器镜像的传输和存储、容器的执行和管理、存储和网络等。**

详细点说，Containerd 负责干下面这些事情：

- 管理容器的生命周期(从创建容器到销毁容器)
- 拉取/推送容器镜像
- 存储管理(管理镜像及容器数据的存储)
- 调用 `runC` 运行容器(与 `runC` 等容器运行时交互)
- 管理容器网络接口及网络

注意：**Containerd 被设计成嵌入到一个更大的系统中，而不是直接由开发人员或终端用户使用。**




## 为什么需要 containerd

我们可以从下面几点来理解为什么需要独立的 containerd：

- 继续从整体 docker 引擎中分离出的项目(开源项目的思路)
- 抽象出来一层，可以被Kubernets CRI 等项目使用(通用化)
- 为广泛的行业合作打下基础(就像 runC 一样)

重复一遍：**Containerd 被设计成嵌入到一个更大的系统中，而不是直接由开发人员或终端用户使用。**

所以 containerd 具有宏大的愿景：



![img](https://images2018.cnblogs.com/blog/952033/201805/952033-20180520115517655-1607641233.png)

当 containerd 和 runC 成为标准化容器服务的基石后，上层的应用就可以直接建立在 containerd 和 runC 之上。上图中展示的容器平台都已经支持 containerd 和 runC 的组合了，相信接下来会有更多类似的容器平台出现。

## Containerd 的技术方向和目标

- 简洁的基于 gRPC 的 API 和 client library
- 完整的 OCI 支持(runtime 和 image spec)
- 同时具备稳定性和高性能的定义良好的容器核心功能
- 一个解耦的系统(让 image、filesystem、runtime 解耦合)，实现插件式的扩展和重用

下图展示了 containerd 的架构(此图来自互联网)：

![img](https://images2018.cnblogs.com/blog/952033/201805/952033-20180520115610144-588472749.png)

在架构设计和实现方面，核心开发人员在[他们的博客](https://blog.docker.com/2017/12/containerd-ga-features-2/)里提到了通过[反思 graphdriver 的实现](https://blog.mobyproject.org/where-are-containerds-graph-drivers-145fc9b7255)，他们将 containerd 设计成了 snapshotter 的模式，这也使得 containerd 对于 overlay 文件系、snapshot 文件系统的支持比较好。
storage、metadata 和 runtime 的三大块划分非常清晰，通过抽象出 events 的设计，containerd 也得以将网络层面的复杂度交给了上层处理，仅提供 network namespace 相关的一些接口添加和配置 API。这样做的好处无疑是巨大的，保留最小功能集合的纯粹和高效，而将更多的复杂性及灵活性交给了插件及上层系统。



## 安装并运行 containerd

在从概念上对 containerd 有所了解之后，让我们安装最新版的 containerd 并实际把玩一下。

>  本文的演示环境为 Centos 7.4 x64。

> 注意：containerd 需要调用 runC，所以在安装 containerd 之前请先安装 runC。 

**下载containerd**

```
wget https://github.com/containerd/containerd/releases/download/v1.1.8/containerd-1.1.8.linux-amd64.tar.gz
tar zxf containerd-1.1.8.linux-amd64.tar.gz 
mv bin/* /usr/local/sbin/

```

 **生成 containerd 配置文件**

```
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
```

>  Containerd 的配置文件默认为 /etc/containerd/config.toml 

 **配置 containerd 作为服务运行**

```
cat /usr/lib/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/containerd
KillMode=process
Delegate=yes
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity

[Install]
WantedBy=multi-user.target
```

启动服务

```
systemctl daemon-reload
systemctl enable --now  containerd.service
systemctl status containerd.service
```

## 演示 demo

可以使用类似 runC 的方式运行容器，也就是使用现成的客户端工具 ctr，由于用法与 runC 非常相似，所以这里不再赘述。Containerd 还提供了 client package 用于在代码中集成 containerd 客户端，下面的 demo 就采用 golang 和 client package 在代码中访问 containerd 服务来创建并运行容器！

**连接 containerd 服务**
创建 main.go 文件，内容如下：

```
package main

import (
	"log"

	"github.com/containerd/containerd"
)

func main() {
	if err := redisExample(); err != nil {
		log.Fatal(err)
	}
}

func redisExample() error {
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return err
	}
	defer client.Close()
	return nil
}
```

上面代码中使用默认的 containerd 套接字创建了一个客户端对象。因为 containerd daemon 通过 gRPC 协议提供服务，所以我们需要创建一个用于调用客户端方法的上下文。在创建上下文之后，我们还应该为我们的 demo 设置一个 namespace，创建单独的 namespace 可以与用户的资源进行隔离以免发生冲突：

```
ctx := namespaces.WithNamespace(context.Background(), "example")
```

**拉取 redis 镜像**
在创建客户端对象后我们就可以从 dockerhub 上拉取容器镜像了，这里我们拉取一个 redis 镜像：

```
image, err := client.Pull(ctx, "docker.io/library/redis:alpine", containerd.WithPullUnpack)
if err != nil {
	return err
}
```

使用客户端的 Pull 方法从 dockerhub 上拉取 redis 镜像，这个方法支持 Opts 模式，所以我们可以指定containerd.WithPullUnpackso 让下载完成后直接把镜像解压缩为一个 snapshotter 作为即将运行的容器的 rootfs。

**创建 OCI Spec 和容器**
有了 rootfs 还需要运行 OCI 容器所需的 OCI runtime spec，我们通过 NewContainer 方法可以使用默认的 OCI runtime spec 直接创建容器对象。当然，也可以通过 Opts 模式的参数修改默认值：

```
container, err := client.NewContainer(
		ctx,
		"redis-server",
		containerd.WithNewSnapshot("redis-server-snapshot", image),
		containerd.WithNewSpec(oci.WithImageConfig(image)),
        )
if err != nil {
        return err
}
defer container.Delete(ctx, containerd.WithSnapshotCleanup)
```

当我们为容器创建一个 snapshot 时需要提供 snapshot 的 ID及其父镜像。通过提供一个单独的 snapshot ID，而不是容器 ID，我们可以轻松地在不同的容器中重用现有的 snapshot。在完成这个示例之后，我们还添加了 defer container.Delete 调用来删除容器以及它的快照。

**创建运行容器的 task**
一个 container 对象只是包含了运行一个容器所需的资源及配置的数据结构，一个容器真正的运行起来是由 Task 对象实现的：

```
task, err := container.NewTask(ctx, cio.NewCreator(cio.WithStdio))
if err != nil {
	return err
}
defer task.Delete(ctx)
```

此时容器的状态相当于我们在《[RunC 简介](http://www.cnblogs.com/sparkdev/p/9032209.html)》一文中介绍的 "created"。这意味着 namespaces、rootfs 和容器的配置都已经初始化成功了，只是用户进程(这里是 redis-server)还没有启动。在这个时机，我们可以为容器设置网卡，还可以配置工具来对容器进行监控等。

**等待任务**
现在我们有一个处于创建状态的任务，我们需要确保等待该任务退出。

```
exitStatusC, err := task.Wait(ctx)
if err != nil {
	return err
}

if err := task.Start(ctx); err != nil {
	return err
}
```

**让运行中的 task 退出**
当要结束容器的运行时，可以调用 task.Kill 方法。其实就是向容器中运行的进程发送信号：

```
time.Sleep(3 * time.Second)

if err := task.Kill(ctx, syscall.SIGTERM); err != nil {
	return err
}

status := <-exitStatusC
code, exitedAt, err := status.Result()
if err != nil {
	return err
}
fmt.Printf("redis-server exited with status: %d\n", code)
```

向容器发送结束的信号后，代码等待容器结束，并输出返回码。最后我们删除 task 对象：

```
status, err := task.Delete(ctx)
```

完整的 demo 代码请看

```
package main

import (
        "context"
        "fmt"
        "log"
        "syscall"
        "time"

        "github.com/containerd/containerd"
        "github.com/containerd/containerd/cio"
        "github.com/containerd/containerd/oci"
        "github.com/containerd/containerd/namespaces"
)

func main() {
        if err := redisExample(); err != nil {
                log.Fatal(err)
        }
}

func redisExample() error {
        // 创建一个新客户端，该客户端连接到containerd的默认套接字路径
        client, err := containerd.New("/run/containerd/containerd.sock")
        if err != nil {
                return err
        }
        defer client.Close()

        // 创建 "example" namespace
        ctx := namespaces.WithNamespace(context.Background(), "example")

        // 从DockerHub中拉取 redis image
        image, err := client.Pull(ctx, "docker.io/library/redis:alpine", containerd.WithPullUnpack)
        if err != nil {
                return err
        }

        // 创建 container
        container, err := client.NewContainer(
                ctx,
                "redis-server",
                containerd.WithImage(image),
                containerd.WithNewSnapshot("redis-server-snapshot", image),
                containerd.WithNewSpec(oci.WithImageConfig(image)),
        )
        if err != nil {
                return err
        }
        defer container.Delete(ctx, containerd.WithSnapshotCleanup)

        // 为容器创建任务
        task, err := container.NewTask(ctx, cio.NewCreator(cio.WithStdio))
        if err != nil {
                return err
        }
        defer task.Delete(ctx)

        // 确保我们在开始之前等待任务
        exitStatusC, err := task.Wait(ctx)
        if err != nil {
                fmt.Println(err)
        }

        // 在任务上调用启动，以执行Redis服务器
        if err := task.Start(ctx); err != nil {
                return err
        }

        // 等待一会
        time.Sleep(3 * time.Second)

        // 终止进程并获得退出状态
        if err := task.Kill(ctx, syscall.SIGTERM); err != nil {
                return err
        }

        // 等待过程完全退出并打印退出状态
        status := <-exitStatusC
        code, _, err := status.Result()
        if err != nil {
                return err
        }
        fmt.Printf("redis-server exited with status: %d\n", code)

        return nil
}
```

编译 demo 代码并运行：

```
go get -v
go build main.go -o main
./main
1:C 30 Oct 2019 08:05:25.559 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 30 Oct 2019 08:05:25.559 # Redis version=5.0.6, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 30 Oct 2019 08:05:25.559 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 30 Oct 2019 08:05:25.560 # You requested maxclients of 10000 requiring at least 10032 max file descriptors.
1:M 30 Oct 2019 08:05:25.560 # Server can't set maximum open files to 10032 because of OS error: Operation not permitted.
1:M 30 Oct 2019 08:05:25.560 # Current maximum open files is 1024. maxclients has been reduced to 992 to compensate for low ulimit. If you need higher maxclients increase 'ulimit -n'.
1:M 30 Oct 2019 08:05:25.560 * Running mode=standalone, port=6379.
1:M 30 Oct 2019 08:05:25.560 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 30 Oct 2019 08:05:25.560 # Server initialized
1:M 30 Oct 2019 08:05:25.560 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 30 Oct 2019 08:05:25.560 * Ready to accept connections
1:signal-handler (1572422728) Received SIGTERM scheduling shutdown...
1:M 30 Oct 2019 08:05:28.692 # User requested shutdown...
1:M 30 Oct 2019 08:05:28.692 * Saving the final RDB snapshot before exiting.
redis-server exited with status: 0
1:M 30 Oct 2019 08:05:28.692 * DB saved on disk
1:M 30 Oct 2019 08:05:28.692 # Redis is now ready to exit, bye bye...
```