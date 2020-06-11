---
layout: post
title: "限制docker容器的系统资源使用"
date: "2020-01-10 19:00"
category: docker
tags: docker systemd
author: lework
---
* content
{:toc}

我们知道，在启动docker容器的时候，可以通过添加`--cpu-quota`, `--memory` 来限制容器的cpu和内存使用，如果对每个容器都要设置限制，那就需要为在启动时添加这个。这样很是麻烦，有时候不需要限制每个容器，但需要确保所有在主机上运行的容器不能超过宿主机资源的80%, 这样做可以避免因为容器进程导致宿主机资源不够使用从而产生宕机事件。




## 解决方案

为 docker 添加 systemd 的 cgroup 资源限制 

**创建slice**

```bash
cat << EOF > /etc/systemd/system/limit-docker.slice
[Unit]
Description=Slice with MemoryLimit and CPUQuota for docker
Before=slices.target

[Slice]
MemoryAccounting=true
MemoryLimit=4G
CPUQuota=50%
EOF
```

Slice的参数说明

新版本配置

- CPUAccounting=：是否开启该unit的CPU使用统计，BOOL型，true或者false。
- CPUWeight=weight, StartupCPUWeight=weight：用于设置cgroup v2的cpu.weight参数。取值范围1-1000，默认值100。StartupCPUWeight应用于系统启动阶段，CPUWeight应用于正常运行时。这两个配置取代了旧版本的CPUShares=和StartupCPUShares=。
- CPUQuota=：用于设置cgroup v2的cpu.max参数或者cgroup v1的cpu.cfs_quota_us参数。表示可以占用的CPU时间配额百分比。如：20%表示最大可以使用单个CPU核的20%。可以超过100%，比如200%表示可以使用2个CPU核。
- MemoryAccounting=：是否开启该unit的memory使用统计，BOOL型，true或者false。
- MemoryLow=bytes：用于设置cgroup v2的memory.low参数，不支持cgroup v1。当unit使用的内存低于该值时将被保护，其内存不会被回收。可以设置不同的后缀：K,M,G或者T表示不同的单位。
- MemoryHigh=bytes：用于设置cgroup v2的memory.high参数，不支持cgroup v1。内存使用超过该值时，进程将被降低运行时间，并快速回收其占用的内存。同样可以设置不同的后缀：K,M,G或者T（单位1024）。也可以设置为infinity表示没有限制。
- MemoryMax=bytes：用于设置cgroup v2的memory.max参数，如果进程的内存超过该限制，则会触发out-of-memory将其kill掉。同样可以设置不同的后缀：K,M,G或者T（单位1024），以及设置为infinity。该参数去掉旧版本的MemoryLimit=。
- MemorySwapMax=bytes：用于设置cgroup v2的memory.swap.max"参数。和MemoryMax类似，不同的是用于控制Swap的使用上限。
- TasksAccounting=：是否开启unit的task个数统计，BOOL型，ture或者false。
- TasksMax=N：用于设置cgroup的pids.max参数。控制unit可以创建的最大tasks个数。
- IOAccounting：是否开启Block IO的统计，BOOL型，true或者false。对应旧版本的BlockIOAccounting=参数。
- IOWeight=weight, StartupIOWeight=weight：设置cgroup v2的io.weight参数，控制IO的权重。取值范围0-1000，默认100。该设置取代了旧版本的BlockIOWeight=和StartupBlockIOWeight=。
- IODeviceWeight=device weight：控制单个设备的IO权重，同样设置在cgroup v2的io.weight参数里，如“/dev/sda 1000”。取值范围0-1000，默认100。该设置取代了旧版本的BlockIODeviceWeight=。
- IOReadBandwidthMax=device bytes, IOWriteBandwidthMax=device bytes：设置磁盘IO读写带宽上限，对应cgroup v2的io.max参数。该参数格式为“path bandwidth”，path为具体设备名或者文件系统路径（最终限制的是文件系统对应的设备名）。数值bandwidth支持以K,M,G,T后缀（单位1000）。可以设置多行以限制对多个设备的IO带宽。该参数取代了旧版本的BlockIOReadBandwidth=和BlockIOWriteBandwidth=。
- IOReadIOPSMax=device IOPS, IOWriteIOPSMax=device IOPS：设置磁盘IO读写的IOPS上限，对应cgroup v2的io.max参数。格式和上面带宽限制的格式一样一样的。
- IPAccounting=：BOOL型，如果为true，则开启ipv4/ipv6的监听和已连接的socket网络收发包统计。
- IPAddressAllow=ADDRESS[/PREFIXLENGTH]…, IPAddressDeny=ADDRESS[/PREFIXLENGTH]…：开启AF_INET和AF_INET6 sockets的网络包过滤功能。参数格式为IPv4或IPv6的地址列表，IP地址后面支持地址匹配前缀（以'/'分隔），如”10.10.10.10/24“。需要注意，该功能仅在开启“eBPF”模块的系统上才支持。
- DeviceAllow=：用于控制对指定的设备节点的访问限制。格式为“设备名 权限”，设备名以"/dev/"开头或者"char-"、“block-”开头。权限为'r','w','m'的组合，分别代表可读、可写和可以通过mknode创建指定的设备节点。对应cgroup的"devices.allow"和"devices.deny"参数。
- DevicePolicy=auto\|closed\|strict：控制设备访问的策略。strict表示：只允许明确指定的访问类型；closed表示：此外，还允许访问包含/dev/null,/dev/zero,/dev/full,/dev/random,/dev/urandom等标准伪设备。auto表示：此外，如果没有明确的DeviceAllow=存在，则允许访问所有设备。auto是默认设置。
- Slice=：存放unit的slice目录，默认为system.slice。
- Delegate=：默认关闭，开启后将更多的资源控制交给进程自己管理。开启后unit可以在单其cgroup下创建和管理其自己的cgroup的私人子层级，systemd将不在维护其cgoup以及将其进程从unit的cgroup里移走。开启方法：“Delegate=yes”。所以通过设置Delegate选项，可以解决上面的问题。

旧版本配置

> 这些是旧版本的选项，新版本已经弃用。列出来是因为centos 7里的systemd是旧版本，所以要使用这些配置。

- CPUShares=weight, StartupCPUShares=weight：进程获取CPU运行时间的权重值，对应cgroup的"cpu.shares"参数，取值范围2-262144，默认值1024。
- MemoryLimit=bytes：进程内存使用上限，对应cgroup的"memory.limit_in_bytes"参数。支持K,M,G,T（单位1024）以及infinity。默认值-1表示不限制。
- BlockIOAccounting=：开启磁盘IO统计选项，同上面的IOAccounting=。
- BlockIOWeight=weight, StartupBlockIOWeight=weight：磁盘IO的权重，对应cgroup的"blkio.weight"参数。取值范围10-1000，默认值500。
- BlockIODeviceWeight=device weight：指定磁盘的IO权重，对应cgroup的"blkio.weight_device"参数。取值范围1-1000，默认值500。
- BlockIOReadBandwidth=device bytes, BlockIOWriteBandwidth=device bytes：磁盘IO带宽的上限配置，对应cgroup的"blkio.throttle.read_bps_device"和 "blkio.throttle.write_bps_device"参数。支持K,M,G,T后缀（单位1000）。



查看 `slice` 的详细说明 [resource-control](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html)

**配置docker**

向 `/etc/docker/daemon.json` 配置文件中添加

```
{
    "cgroup-parent": "limit-docker.slice"
}
```

查看 `cgroup-parent` 参数的说明 [documentation](https://docs.docker.com/engine/reference/commandline/dockerd/#miscellaneous-options)

**重载systemctl及重启docker**

```bash
systemctl daemon-reload
systemctl restart docker
```

**测试**

宿主机配置：

- os: `centos 7.4 x64`
- cpu: `2c`
- mem: `8g`

**测试 cpu 限制**

```bash
# 启动2个进程，每个进程内存分配1g
# docker run -it --rm lorel/docker-stress-ng --vm 2 --vm-bytes 1g
```

资源使用情况

```bash
# docker stats

CONTAINER ID  NAME          CPU %     MEM USAGE / LIMIT     MEM %      NET I/O    BLOCK I/O  PIDS
146f10358399  vibrant_boyd  49.7%    2.000GiB / 7.801GiB   14.32%     758B / 0B  0B / 0B    5


# systemd-cgtop
Control Group                                                                                                   Tasks   %CPU   Memory  Input/s Output/s
/                                                                                                                   -   52.7     3.0G        -        -
/limit.slice                                                                                                        5   49.7     2.0G        -        -
/limit.slice/limit-docker.slice                                                                                     5   49.7     2.0G        -        -
/limit.slice/limit-docker.slice/docker-dc26c8b78c98e3a6bf66ab88ed5486cd7c4f1ba47b989d2ef731ad49472ecdb0.scope       5   49.7     2.0G
```

> 可以看到cpu只用到了49.7%, 内存使用了2g。 cpu已经被限制在50%。

**测试 mem 限制**

```bash
# docker run -it --rm lorel/docker-stress-ng --vm 2 --vm-bytes 2g
Value 2147483648 is out of range for vm-bytes, allowed: 4096 .. 1073741824
```

因为限制内存在4g, 创建2个进程，每个进程2g的任务就会失败。


**测试多个容器**

```bash
# 启动2个进程，每个进程内存分配1g
# docker run -itd --rm lorel/docker-stress-ng --vm 2 --vm-bytes 1g
# docker run -itd --rm lorel/docker-stress-ng --vm 2 --vm-bytes 1g
```

资源使用情况

```bash
# docker stats

CONTAINER ID  NAME            CPU %   MEM USAGE / LIMIT     MEM %   NET I/O       BLOCK I/O     PIDS
dc26c8b78c98  loving_spence   23.95%  2.005GiB / 7.801GiB   25.70%  648B / 0B     0B / 0B       5
b5652d8b4445  festive_lamport 26.40%  1.389GiB / 7.801GiB   17.81%  828B / 0B     0B / 0B       5


# systemd-cgtop
Control Group                                                                                                   Tasks   %CPU   Memory  Input/s Output/s
/                                                                                                                   -   51.9     5.0G        -        -
/limit.slice                                                                                                       10   49.6     3.9G        -        -
/limit.slice/limit-docker.slice                                                                                    10   49.6     3.9G        -        -
/limit.slice/limit-docker.slice/docker-dc26c8b78c98e3a6bf66ab88ed5486cd7c4f1ba47b989d2ef731ad49472ecdb0.scope       5   25.0     2.0G        -        -
/limit.slice/limit-docker.slice/docker-b5652d8b44452cf3445bef440b7acf601f5c64292e443914c6a5819d407aa44b.scope       5   24.6     1.9G 
```

> 从数据上可以看出, 使用docker启动的所有容器资源使用之和不能超过systemd资源的限制， 这样可以减少因为容器进程资源占用过高导致宿主机宕机的风险