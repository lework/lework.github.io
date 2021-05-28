---
layout: post
title: "在 debian 10 上以 kubeadm 方式安装 Kubernetes v1.20.5 ha集群"
date: "2021-04-03 22:10:00"
category: kubernetes
tags: kubernetes k8s-install
author: lework
---
* content
{:toc}

本次使用全手工的方式在 `debian 10` 系统上以 `kubeadm` 形式部署 `kubernetes` 的 `ha`集群，`ha` 方式选择 `node` 节点代理 `apiserver` 的方式。

![k8s-node-ha](/assets/images/kubernetes/k8s-node-ha.png)




{% raw %}
## 环境信息

| System OS   |   IP Address   | Docker  |     Kernel      |     Hostname      | Cpu  | Memory | Role   |
| ----------- | :------------: | :-----: | :-------------: | :---------------: | ---- | :----: | ------ |
| Debian 10.9 | 192.168.77.140 | 20.10.5 | 4.19.0-16-amd64 | k8s--master-node1 | 2C   |   4G   | master |
| Debian 10.9 | 192.168.77.141 | 20.10.5 | 4.19.0-16-amd64 | k8s-master-node2  | 2C   |   4G   | master |
| Debian 10.9 | 192.168.77.142 | 20.10.5 | 4.19.0-16-amd64 | k8s-master-node3  | 2C   |   4G   | master |
| Debian 10.9 | 192.168.77.143 | 20.10.5 | 4.19.0-16-amd64 | k8s-worker-node1  | 2C   |   4G   | worker |
| Debian 10.9 | 192.168.77.144 | 20.10.5 | 4.19.0-16-amd64 | k8s-worker-node2  | 2C   |   4G   | worker |
| Debian 10.9 | 192.168.77.144 | 20.10.5 | 4.19.0-16-amd64 | k8s-worker-node3  | 2C   |   4G   | worker |

## 版本信息

kubeadm: `v1.20.5`

Kubernetes: `v1.20.5`

etcd: `v3.4.13`

Docker CE: `20.10.5`

Flannel :` v0.13.0`

## 网络信息

- Cluster IP CIDR: `10.244.0.0/16`
- Service Cluster IP CIDR: `10.96.0.0/12`
- Service DNS IP: `10.96.0.10`
- DNS DN: `cluster.local`
- Kubernetes API:  `apiserver.k8s.local:6443`

## 初始化所有节点

> 在集群所有节点上执行下面的操作
> **注意**：以下操作有些存在过度优化，请根据自身情况择选。

### APT调整

镜像源调整

```bash
mv  /etc/apt/sources.list{,.bak}
cat > /etc/apt/sources.list <<EOF
deb http://mirrors.aliyun.com/debian/ buster main contrib non-free
deb-src http://mirrors.aliyun.com/debian/ buster main contrib non-free

deb http://mirrors.aliyun.com/debian/ buster-updates main contrib non-free
deb-src http://mirrors.aliyun.com/debian/ buster-updates main contrib non-free

deb http://mirrors.aliyun.com/debian-security/ buster/updates main contrib non-free
deb-src http://mirrors.aliyun.com/debian-security/ buster/updates main contrib non-free
EOF
apt-get update
```

取消安装服务自启动

```bash
echo -e '#!/bin/sh\nexit 101' | install -m 755 /dev/stdin /usr/sbin/policy-rc.d
```

取消自动更新包

```bash
systemctl mask apt-daily.service apt-daily-upgrade.service
systemctl stop apt-daily.timer apt-daily-upgrade.timer
systemctl disable apt-daily.timer apt-daily-upgrade.timer
systemctl kill --kill-who=all apt-daily.service

cat > /etc/apt/apt.conf.d/10cloudinit-disable << __EOF
APT::Periodic::Enable "0";
// undo what's in 20auto-upgrade
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
__EOF
```

### 关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

### 关闭selinux
```bash
setenforce 0
sed -i "s#=enforcing#=disabled#g" /etc/selinux/config
```

### 关闭swap
```bash
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

### limit 限制

```bash
[ ! -f /etc/security/limits.conf_bak ] && cp /etc/security/limits.conf{,_bak}
cat << EOF >> /etc/security/limits.conf
root soft nofile 655360
root hard nofile 655360
root soft nproc 655360
root hard nproc 655360
root soft core unlimited
root hard core unlimited

* soft nofile 655360
* hard nofile 655360
* soft nproc 655360
* hard nproc 655360
* soft core unlimited
* hard core unlimited
EOF

[ ! -f /etc/systemd/system.conf_bak ] && cp /etc/systemd/system.conf.conf{,_bak}
cat << EOF >> /etc/systemd/system.conf
DefaultLimitCORE=infinity
DefaultLimitNOFILE=655360
DefaultLimitNPROC=655360
EOF
```

### 系统参数
```bash
cat << EOF >  /etc/sysctl.d/99-kube.conf
# https://www.kernel.org/doc/Documentation/sysctl/
#############################################################################################
# 调整虚拟内存
#############################################################################################

# Default: 30
# 0 - 任何情况下都不使用swap。
# 1 - 除非内存不足（OOM），否则不使用swap。
vm.swappiness = 0

# 内存分配策略
#0 - 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。
#1 - 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
#2 - 表示内核允许分配超过所有物理内存和交换空间总和的内存
vm.overcommit_memory=1

# OOM时处理
# 1关闭，等于0时，表示当内存耗尽时，内核会触发OOM killer杀掉最耗内存的进程。
vm.panic_on_oom=0

# vm.dirty_background_ratio 用于调整内核如何处理必须刷新到磁盘的脏页。
# Default value is 10.
# 该值是系统内存总量的百分比，在许多情况下将此值设置为5是合适的。
# 此设置不应设置为零。
vm.dirty_background_ratio = 5

# 内核强制同步操作将其刷新到磁盘之前允许的脏页总数
# 也可以通过更改 vm.dirty_ratio 的值（将其增加到默认值30以上（也占系统内存的百分比））来增加
# 推荐 vm.dirty_ratio 的值在60到80之间。
vm.dirty_ratio = 60

# vm.max_map_count 计算当前的内存映射文件数。
# mmap 限制（vm.max_map_count）的最小值是打开文件的ulimit数量（cat /proc/sys/fs/file-max）。
# 每128KB系统内存 map_count应该大约为1。 因此，在32GB系统上，max_map_count为262144。
# Default: 65530
vm.max_map_count = 2097152

#############################################################################################
# 调整文件
#############################################################################################

fs.may_detach_mounts = 1

# 增加文件句柄和inode缓存的大小，并限制核心转储。
fs.file-max = 2097152
fs.nr_open = 2097152
fs.suid_dumpable = 0

# 文件监控
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=524288
fs.inotify.max_queued_events=16384

#############################################################################################
# 调整网络设置
#############################################################################################

# 为每个套接字的发送和接收缓冲区分配的默认内存量。
net.core.wmem_default = 25165824
net.core.rmem_default = 25165824

# 为每个套接字的发送和接收缓冲区分配的最大内存量。
net.core.wmem_max = 25165824
net.core.rmem_max = 25165824

# 除了套接字设置外，发送和接收缓冲区的大小
# 必须使用net.ipv4.tcp_wmem和net.ipv4.tcp_rmem参数分别设置TCP套接字。
# 使用三个以空格分隔的整数设置这些整数，分别指定最小，默认和最大大小。
# 最大大小不能大于使用net.core.wmem_max和net.core.rmem_max为所有套接字指定的值。
# 合理的设置是最小4KiB，默认64KiB和最大2MiB缓冲区。
net.ipv4.tcp_wmem = 20480 12582912 25165824
net.ipv4.tcp_rmem = 20480 12582912 25165824

# 增加最大可分配的总缓冲区空间
# 以页为单位（4096字节）进行度量
net.ipv4.tcp_mem = 65536 25165824 262144
net.ipv4.udp_mem = 65536 25165824 262144

# 为每个套接字的发送和接收缓冲区分配的最小内存量。
net.ipv4.udp_wmem_min = 16384
net.ipv4.udp_rmem_min = 16384

# 启用TCP窗口缩放，客户端可以更有效地传输数据，并允许在代理方缓冲该数据。
net.ipv4.tcp_window_scaling = 1

# 提高同时接受连接数。
net.ipv4.tcp_max_syn_backlog = 10240

# 将net.core.netdev_max_backlog的值增加到大于默认值1000
# 可以帮助突发网络流量，特别是在使用数千兆位网络连接速度时，
# 通过允许更多的数据包排队等待内核处理它们。
net.core.netdev_max_backlog = 65536

# 增加选项内存缓冲区的最大数量
net.core.optmem_max = 25165824

# 被动TCP连接的SYNACK次数。
net.ipv4.tcp_synack_retries = 2

# 允许的本地端口范围。
net.ipv4.ip_local_port_range = 2048 65535

# 防止TCP时间等待
# Default: net.ipv4.tcp_rfc1337 = 0
net.ipv4.tcp_rfc1337 = 1

# 减少tcp_fin_timeout连接的时间默认值
net.ipv4.tcp_fin_timeout = 15

# 积压套接字的最大数量。
# Default is 128.
net.core.somaxconn = 32768

# 打开syncookies以进行SYN洪水攻击保护。
net.ipv4.tcp_syncookies = 1

# 避免Smurf攻击
# 发送伪装的ICMP数据包，目的地址设为某个网络的广播地址，源地址设为要攻击的目的主机，
# 使所有收到此ICMP数据包的主机都将对目的主机发出一个回应，使被攻击主机在某一段时间内收到成千上万的数据包
net.ipv4.icmp_echo_ignore_broadcasts = 1

# 为icmp错误消息打开保护
net.ipv4.icmp_ignore_bogus_error_responses = 1

# 启用自动缩放窗口。
# 如果延迟证明合理，这将允许TCP缓冲区超过其通常的最大值64K。
net.ipv4.tcp_window_scaling = 1

# 打开并记录欺骗，源路由和重定向数据包
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# 告诉内核有多少个未附加的TCP套接字维护用户文件句柄。 万一超过这个数字，
# 孤立的连接会立即重置，并显示警告。
# Default: net.ipv4.tcp_max_orphans = 65536
net.ipv4.tcp_max_orphans = 65536

# 不要在关闭连接时缓存指标
net.ipv4.tcp_no_metrics_save = 1

# 启用RFC1323中定义的时间戳记：
# Default: net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_timestamps = 1

# 启用选择确认。
# Default: net.ipv4.tcp_sack = 1
net.ipv4.tcp_sack = 1

# 增加 tcp-time-wait 存储桶池大小，以防止简单的DOS攻击。
# net.ipv4.tcp_tw_recycle 已从Linux 4.12中删除。请改用net.ipv4.tcp_tw_reuse。
net.ipv4.tcp_max_tw_buckets = 14400
net.ipv4.tcp_tw_reuse = 1

# accept_source_route 选项使网络接口接受设置了严格源路由（SSR）或松散源路由（LSR）选项的数据包。
# 以下设置将丢弃设置了SSR或LSR选项的数据包。
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# 打开反向路径过滤
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 禁用ICMP重定向接受
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# 禁止发送所有IPv4 ICMP重定向数据包。
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# 开启IP转发.
net.ipv4.ip_forward = 1

# 禁止IPv6
net.ipv6.conf.lo.disable_ipv6=1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# 要求iptables不对bridge的数据进行处理
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1

# arp缓存
# 存在于 ARP 高速缓存中的最少层数，如果少于这个数，垃圾收集器将不会运行。缺省值是 128
net.ipv4.neigh.default.gc_thresh1=2048
# 保存在 ARP 高速缓存中的最多的记录软限制。垃圾收集器在开始收集前，允许记录数超过这个数字 5 秒。缺省值是 512
net.ipv4.neigh.default.gc_thresh2=4096
# 保存在 ARP 高速缓存中的最多记录的硬限制，一旦高速缓存中的数目高于此，垃圾收集器将马上运行。缺省值是 1024
net.ipv4.neigh.default.gc_thresh3=8192

# 持久连接
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10

# conntrack表
net.nf_conntrack_max=1048576
net.netfilter.nf_conntrack_max=1048576
net.netfilter.nf_conntrack_buckets=262144
net.netfilter.nf_conntrack_tcp_timeout_fin_wait=30
net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
net.netfilter.nf_conntrack_tcp_timeout_close_wait=15
net.netfilter.nf_conntrack_tcp_timeout_established=300

#############################################################################################
# 调整内核参数
#############################################################################################

# 地址空间布局随机化（ASLR）是一种用于操作系统的内存保护过程，可防止缓冲区溢出攻击。
# 这有助于确保与系统上正在运行的进程相关联的内存地址不可预测，
# 因此，与这些流程相关的缺陷或漏洞将更加难以利用。
# Accepted values: 0 = 关闭, 1 = 保守随机化, 2 = 完全随机化
kernel.randomize_va_space = 2

# 调高 PID 数量
kernel.pid_max = 65536
kernel.threads-max=30938

# coredump
kernel.core_pattern=core

# 决定了检测到soft lockup时是否自动panic，缺省值是0
kernel.softlockup_all_cpu_backtrace=1
kernel.softlockup_panic=1
EOF
sysctl --system
```

### history 数据格式和 ps1

```bash
cat << EOF >> /etc/bash.bashrc
# history actions record，include action time, user, login ip
HISTFILESIZE=5000
HISTSIZE=5000
USER_IP=\$(who -u am i 2>/dev/null | awk '{print \$NF}' | sed -e 's/[()]//g')
if [ -z \$USER_IP ]
then
  USER_IP=\$(hostname -i)
fi
HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S \$USER_IP:\$(whoami) "
export HISTFILESIZE HISTSIZE HISTTIMEFORMAT

# PS1
PS1='\[\033[0m\]\[\033[1;36m\][\u\[\033[0m\]@\[\033[1;32m\]\h\[\033[0m\] \[\033[1;31m\]\w\[\033[0m\]\[\033[1;36m\]]\[\033[33;1m\]\\$ \[\033[0m\]'
EOF
```

### journal 日志

```bash
mkdir -p /var/log/journal /etc/systemd/journald.conf.d
cat << EOF > /etc/systemd/journald.conf.d/99-prophet.conf
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=10G
# 单日志文件最大 200M
SystemMaxFileSize=200M
# 日志保存时间 3 周
MaxRetentionSec=3week
# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
```

### ssh登录信息

```bash
cat << EOF > /etc/profile.d/zz-ssh-login-info.sh
#!/bin/sh
#
# @Time    : 2020-02-04
# @Author  : lework
# @Desc    : ssh login banner

export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
shopt -q login_shell && : || return 0
echo -e "\033[0;32m
 ██╗  ██╗ █████╗ ███████╗
 ██║ ██╔╝██╔══██╗██╔════╝
 █████╔╝ ╚█████╔╝███████╗
 ██╔═██╗ ██╔══██╗╚════██║
 ██║  ██╗╚█████╔╝███████║
 ╚═╝  ╚═╝ ╚════╝ ╚══════ by lework\033[0m"

# os
upSeconds="\$(cut -d. -f1 /proc/uptime)"
secs=\$((\${upSeconds}%60))
mins=\$((\${upSeconds}/60%60))
hours=\$((\${upSeconds}/3600%24))
days=\$((\${upSeconds}/86400))
UPTIME_INFO=\$(printf "%d days, %02dh %02dm %02ds" "\$days" "\$hours" "\$mins" "\$secs")

if [ -f /etc/redhat-release ] ; then
    PRETTY_NAME=\$(< /etc/redhat-release)

elif [ -f /etc/debian_version ]; then
   DIST_VER=\$(</etc/debian_version)
   PRETTY_NAME="\$(grep PRETTY_NAME /etc/os-release | sed -e 's/PRETTY_NAME=//g' -e  's/"//g') (\$DIST_VER)"

else
    PRETTY_NAME=\$(cat /etc/*-release | grep "PRETTY_NAME" | sed -e 's/PRETTY_NAME=//g' -e 's/"//g')
fi

if [[ -d "/system/app/" && -d "/system/priv-app" ]]; then
    model="\$(getprop ro.product.brand) \$(getprop ro.product.model)"

elif [[ -f /sys/devices/virtual/dmi/id/product_name ||
        -f /sys/devices/virtual/dmi/id/product_version ]]; then
    model="\$(< /sys/devices/virtual/dmi/id/product_name)"
    model+=" \$(< /sys/devices/virtual/dmi/id/product_version)"

elif [[ -f /sys/firmware/devicetree/base/model ]]; then
    model="\$(< /sys/firmware/devicetree/base/model)"

elif [[ -f /tmp/sysinfo/model ]]; then
    model="\$(< /tmp/sysinfo/model)"
fi

MODEL_INFO=\${model}
KERNEL=\$(uname -srmo)
USER_NUM=\$(who -u | wc -l)
RUNNING=\$(ps ax | wc -l | tr -d " ")

# disk
totaldisk=\$(df -h -x devtmpfs -x tmpfs -x debugfs -x aufs -x overlay --total 2>/dev/null | tail -1)
disktotal=\$(awk '{print \$2}' <<< "\${totaldisk}")
diskused=\$(awk '{print \$3}' <<< "\${totaldisk}")
diskusedper=\$(awk '{print \$5}' <<< "\${totaldisk}")
DISK_INFO="\033[0;33m\${diskused}\033[0m of \033[1;34m\${disktotal}\033[0m disk space used (\033[0;33m\${diskusedper}\033[0m)"

# cpu
cpu=\$(awk -F':' '/^model name/ {print \$2}' /proc/cpuinfo | uniq | sed -e 's/^[ \t]*//')
cpun=\$(grep -c '^processor' /proc/cpuinfo)
cpuc=\$(grep '^cpu cores' /proc/cpuinfo | tail -1 | awk '{print \$4}')
cpup=\$(grep '^physical id' /proc/cpuinfo | wc -l)
CPU_INFO="\${cpu} \${cpup}P \${cpuc}C \${cpun}L"

# get the load averages
read one five fifteen rest < /proc/loadavg
LOADAVG_INFO="\033[0;33m\${one}\033[0m / \${five} / \${fifteen} with \033[1;34m\$(( cpun*cpuc ))\033[0m core(s) at \033[1;34m\$(grep '^cpu MHz' /proc/cpuinfo | tail -1 | awk '{print \$4}')\033 MHz"

# mem
MEM_INFO="\$(cat /proc/meminfo | awk '/MemTotal:/{total=\$2/1024/1024;next} /MemAvailable:/{use=total-\$2/1024/1024; printf("\033[0;33m%.2fGiB\033[0m of \033[1;34m%.2fGiB\033[0m RAM used (\033[0;33m%.2f%%\033[0m)",use,total,(use/total)*100);}')"

# network
# extranet_ip=" and \$(curl -s ip.cip.cc)"
IP_INFO="\$(ip a | grep glo | awk '{print \$2}' | head -1 | cut -f1 -d/)\${extranet_ip:-}"

# Container info
CONTAINER_INFO="\$(sudo /usr/bin/crictl ps -a -o yaml 2> /dev/null | awk '/^  state: /{gsub("CONTAINER_", "", \$NF) ++S[\$NF]}END{for(m in S) printf "%s%s:%s ",substr(m,1,1),tolower(substr(m,2)),S[m]}')Images:\$(sudo /usr/bin/crictl images -q 2> /dev/null | wc -l)"

# info
echo -e "
 Information as of: \033[1;34m\$(date +"%Y-%m-%d %T")\033[0m
 
 \033[0;1;31mProduct\033[0m............: \${MODEL_INFO}
 \033[0;1;31mOS\033[0m.................: \${PRETTY_NAME}
 \033[0;1;31mKernel\033[0m.............: \${KERNEL}
 \033[0;1;31mCPU\033[0m................: \${CPU_INFO}

 \033[0;1;31mHostname\033[0m...........: \033[1;34m\$(hostname)\033[0m
 \033[0;1;31mIP Addresses\033[0m.......: \033[1;34m\${IP_INFO}\033[0m

 \033[0;1;31mUptime\033[0m.............: \033[0;33m\${UPTIME_INFO}\033[0m
 \033[0;1;31mMemory\033[0m.............: \${MEM_INFO}
 \033[0;1;31mLoad Averages\033[0m......: \${LOADAVG_INFO}
 \033[0;1;31mDisk Usage\033[0m.........: \${DISK_INFO} 

 \033[0;1;31mUsers online\033[0m.......: \033[1;34m\${USER_NUM}\033[0m
 \033[0;1;31mRunning Processes\033[0m..: \033[1;34m\${RUNNING}\033[0m
 \033[0;1;31mContainer Info\033[0m.....: \${CONTAINER_INFO}
"
EOF

chmod +x /etc/profile.d/zz-ssh-login-info.sh
echo 'ALL ALL=NOPASSWD: /usr/bin/crictl info' > /etc/sudoers.d/crictl
```

### 时间同步

```bash
ntpd --version > /dev/null 2>1 && apt-get remove -y ntp
apt-get install -y chrony
[ ! -f /etc/chrony.conf_bak ] && cp /etc/chrony.conf{,_bak} #备份默认配置
cat << EOF > /etc/chrony.conf
server ntp.aliyun.com iburst
server cn.ntp.org.cn iburst
server ntp.shu.edu.cn iburst
server 0.cn.pool.ntp.org iburst
server 1.cn.pool.ntp.org iburst
server 2.cn.pool.ntp.org iburst
server 3.cn.pool.ntp.org iburst

driftfile /var/lib/chrony/drift
makestep 1.0 3
logdir /var/log/chrony
EOF

timedatectl set-timezone Asia/Shanghai
chronyd -q -t 1 'server cn.pool.ntp.org iburst maxsamples 1'
systemctl enable chronyd
systemctl start chronyd
chronyc sources -v
chronyc sourcestats
```

### 启用ipvs

```
apt-get install -y ipvsadm ipset sysstat conntrack libseccomp2
```

开机自启动加载ipvs内核
```
:> /etc/modules-load.d/ipvs.conf
module=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
br_netfilter
)
for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
# systemctl enable --now systemd-modules-load.service

ipvsadm --clear
```

### 系统审计

```bash
apt-get install -y auditd audispd-plugins
cat << EOF > /etc/audit/rules.d/audit.rules

# Remove any existing rules
-D

# Buffer Size
-b 8192

# Failure Mode
-f 1

# Ignore errors
-i 

# docker
-w /usr/bin/dockerd -k docker
-w /var/lib/docker -k docker
-w /etc/docker -k docker
-w /usr/lib/systemd/system/docker.service -k docker
-w /etc/systemd/system/docker.service -k docker
-w /usr/lib/systemd/system/docker.socket -k docker
-w /etc/default/docker -k docker
-w /etc/sysconfig/docker -k docker
-w /etc/docker/daemon.json -k docker

# containerd
-w /usr/bin/containerd -k containerd
-w /var/lib/containerd -k containerd
-w /usr/lib/systemd/system/containerd.service -k containerd
-w /etc/containerd/config.toml -k containerd

# runc 
-w /usr/bin/runc -k runc

# kube
-w /usr/bin/kubeadm -k kubeadm
-w /usr/bin/kubelet -k kubelet
-w /usr/bin/kubectl -k kubectl
-w /var/lib/kubelet -k kubelet
-w /etc/kubernetes -k kubernetes
EOF
chmod 600 /etc/audit/rules.d/audit.rules
sed -i 's#max_log_file =.*#max_log_file = 80#g' /etc/audit/auditd.conf 

systemctl stop auditd && systemctl start auditd
systemctl enable auditd
```

### dns 选项

```bash
grep single-request-reopen /etc/resolv.conf || sed -i '1ioptions timeout:2 attempts:3 rotate single-request-reopen' /etc/resolv.conf
```

### 升级内核

> 注意：这里是可选项。 


debian 10 默认关闭了 cgroup hugetlb 。可以通过更新内核开启。

```bash
root@debian:~# cat /etc/debian_version
10.9
root@debian:~# uname -r
4.19.0-16-amd64
root@debian:~# grep HUGETLB /boot/config-$(uname -r)    
# CONFIG_CGROUP_HUGETLB is not set
CONFIG_ARCH_WANT_GENERAL_HUGETLB=y
CONFIG_HUGETLBFS=y
CONFIG_HUGETLB_PAGE=y
```

更新内核

```bash
# 添加 backports 源
$ sudo echo "deb   http://mirrors.aliyun.com/debian buster-backports main" > /etc/apt/sources.list.d/backports.list

# 更新来源
$ sudo apt update

# 安装 Linux 内核映像 
$ sudo apt -t buster-backports  install linux-image-amd64

# 安装 Linux 内核标头（可选）
$ sudo apt -t buster-backports install linux-headers-amd64
```

重启之后，再次查看 cgroup hugetlb

```bash
root@debian:~# cat /etc/debian_version 
10.9
root@debian:~# uname -r
5.10.0-0.bpo.4-amd64
root@debian:~# grep HUGETLB /boot/config-$(uname -r)                   
CONFIG_CGROUP_HUGETLB=y
CONFIG_ARCH_WANT_GENERAL_HUGETLB=y
CONFIG_HUGETLBFS=y
CONFIG_HUGETLB_PAGE=y
```

### 安装 docker-ce

```bash
apt-get install -y apt-transport-https ca-certificates curl gnupg2 lsb-release bash-completion
    
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo apt-key add -
echo "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/debian $(lsb_release -cs)   stable" > /etc/apt/sources.list.d/docker-ce.list
sudo apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io
apt-mark hold docker-ce docker-ce-cli containerd.io

cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
mkdir  /etc/docker
cat >> /etc/docker/daemon.json <<EOF
{
  "data-root": "/var/lib/docker",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "200m",
    "max-file": "5"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 655360,
      "Soft": 655360
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 655360,
      "Soft": 655360
    }
  },
  "live-restore": true,
  "oom-score-adjust": -1000,
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 10,
  "storage-driver": "overlay2",
  "storage-opts": ["overlay2.override_kernel_check=true"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "https://yssx4sxy.mirror.aliyuncs.com/"
  ]
}
EOF

systemctl enable --now docker

sed -i 's|#oom_score = 0|oom_score = -999|' /etc/containerd/config.toml
cat << EOF > /etc/crictl.yaml
runtime-endpoint: unix:///var/run/dockershim.sock
image-endpoint: unix:///var/run/dockershim.sock
timeout: 2
debug: false
pull-image-on-create: true
disable-pull-on-run: false
EOF

systemctl enable --now containerd
```


###  主机名配置
```
hostnamectl set-hostname k8s-master-node1
```
> 根据每个节点名称进行设置


### 节点主机名解析

```bash
cat << EOF >> /etc/hosts
192.168.77.140 k8s-master-node1
192.168.77.141 k8s-master-node2
192.168.77.142 k8s-master-node3
192.168.77.143 k8s-worker-node1
192.168.77.144 k8s-worker-node2
192.168.77.145 k8s-worker-node3
EOF
```




## 部署master节点

### 安装 kube 组件

> 在所有的 k8s-master 节点上安装 `kubeadm` , `kubelet`, `kubectl`

``` bash
echo 'deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
apt-get update

export KUBE_VERSION="1.20.5"
apt-get install -y kubeadm=$KUBE_VERSION-00 kubelet=$KUBE_VERSION-00 kubectl=$KUBE_VERSION-00
sudo apt-mark hold kubelet kubeadm kubectl

[ -d /etc/bash_completion.d ] && \
    { kubectl completion bash > /etc/bash_completion.d/kubectl; \
      kubeadm completion bash > /etc/bash_completion.d/kubadm; }
      
[ ! -d /usr/lib/systemd/system/kubelet.service.d ] && mkdir -p /usr/lib/systemd/system/kubelet.service.d
cat << EOF > /usr/lib/systemd/system/kubelet.service.d/11-cgroup.conf
[Service]
CPUAccounting=true
MemoryAccounting=true
BlockIOAccounting=true
ExecStartPre=/usr/bin/bash -c '/usr/bin/mkdir -p /sys/fs/cgroup/{cpuset,memory,systemd,pids,"cpu,cpuacct"}/{system,kube,kubepods}.slice'
Slice=kube.slice
EOF
systemctl daemon-reload
 
systemctl enable kubelet.service
```


> 下面的操作在 k8s-master-node1 节点上进行

### 设置 apiserver 解析

```bash
echo '127.0.0.1 apiserver.cluster.local' >> /etc/hosts
```

### 配置 kubeadm

建立`kubeadm-config.yaml`

```bash
[ ! -d etc/kubernetes ] && mkdir -p /etc/kubernetes

cat <<EOF > /etc/kubernetes/kubeadm-config.yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  # cri 配置
  criSocket: /var/run/dockershim.sock
  kubeletExtraArgs:
    runtime-cgroups: /system.slice/docker.service

---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  # ipvs 配置
  minSyncPeriod: 5s
  syncPeriod: 5s
  scheduler: wrr

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
maxPods: 200
cgroupDriver: systemd
runtimeRequestTimeout: 5m
# 此配置保证了 kubelet 能在 swap 开启的情况下启动
failSwapOn: false
nodeStatusUpdateFrequency: 5s
rotateCertificates: true
imageGCLowThresholdPercent: 70
imageGCHighThresholdPercent: 80
# 软驱逐阀值
evictionSoft:
  imagefs.available: 15%
  memory.available: 512Mi
  nodefs.available: 15%
  nodefs.inodesFree: 10%
# 达到软阈值之后，持续时间超过多久才进行驱逐
evictionSoftGracePeriod:
  imagefs.available: 3m
  memory.available: 1m
  nodefs.available: 3m
  nodefs.inodesFree: 1m
# 硬驱逐阀值
evictionHard:
  imagefs.available: 10%
  memory.available: 256Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionMaxPodGracePeriod: 30
# 节点资源预留
kubeReserved:
  cpu: 200m
  ephemeral-storage: 1Gi
systemReserved:
  cpu: 300m
  ephemeral-storage: 1Gi
kubeReservedCgroup: /kube.slice
systemReservedCgroup: /system.slice
enforceNodeAllocatable: 
- pods
- kube-reserved
- system-reserved

---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: 1.20.5
controlPlaneEndpoint: apiserver.cluster.local:6443
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/16
imageRepository: registry.cn-hangzhou.aliyuncs.com/kainstall
apiServer:
  certSANs:
  - 127.0.0.1
  - apiserver.cluster.local
  - 192.168.77.140
  - 192.168.77.141
  - 192.168.77.142
  extraArgs:
    event-ttl: 720h
    service-node-port-range: 30000-50000
    # 审计日志相关配置
    audit-log-maxage: '20'
    audit-log-maxbackup: '10'
    audit-log-maxsize: '100'
    audit-log-path: /var/log/kube-audit/audit.log
    audit-policy-file: /etc/kubernetes/audit-policy.yaml
  extraVolumes:
  - name: audit-config
    hostPath: /etc/kubernetes/audit-policy.yaml
    mountPath: /etc/kubernetes/audit-policy.yaml
    readOnly: true
    pathType: File
  - name: audit-log
    hostPath: /var/log/kube-audit
    mountPath: /var/log/kube-audit
    pathType: DirectoryOrCreate
  - name: localtime
    hostPath: /etc/localtime
    mountPath: /etc/localtime
    readOnly: true
    pathType: File
controllerManager:
  extraArgs:
    bind-address: 0.0.0.0
    node-cidr-mask-size: '24'
    deployment-controller-sync-period: '10s'
    pod-eviction-timeout: 2m
    terminated-pod-gc-threshold: '30'
    experimental-cluster-signing-duration: 87600h
    feature-gates: RotateKubeletServerCertificate=true
    horizontal-pod-autoscaler-use-rest-clients: 'true'
    horizontal-pod-autoscaler-sync-period: 10s
    node-monitor-grace-period: 10s
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
    pathType: File
scheduler:
  extraArgs:
    bind-address: 0.0.0.0
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
    pathType: File
EOF
```

>  以上是对kubeadm的配置信息

### 配置审计

```
cat << EOF > /etc/kubernetes/audit-policy.yaml
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF
```

### 初始化节点

使用`kubeadm`初始化`control plane`


``` bash
kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml --upload-certs

…
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join apiserver.cluster.local:6443 --token nj9wia.n8pmj18em5cfzg7e \
    --discovery-token-ca-cert-hash sha256:64ddf85513a7d0321cc19bad82209c83783276daa4cd8ebcb114f63804e649be \
    --control-plane --certificate-key 7bd6f9874e65df89af33f8f235494ec2a3544e58be89f96b5118be9e8bbc7a15

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join apiserver.cluster.local:6443 --token nj9wia.n8pmj18em5cfzg7e \
    --discovery-token-ca-cert-hash sha256:64ddf85513a7d0321cc19bad82209c83783276daa4cd8ebcb114f63804e649be 
```

>  记录下join信息，后面node节点加入时使用。

**注意**：这里如果出现 提示找不到 hugetlb，是因为 debian 10 默认关闭了 cgroup hugetlb 关闭了。可以通过更新内核开启。

使用netstat -ntlp查看服务是否正常启动

``` bash
[root@k8s-master-node1 /home/test]# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 192.168.77.140:2379     0.0.0.0:*               LISTEN      7336/etcd           
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      7336/etcd           
tcp        0      0 192.168.77.140:2380     0.0.0.0:*               LISTEN      7336/etcd           
tcp        0      0 127.0.0.1:2381          0.0.0.0:*               LISTEN      7336/etcd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      483/sshd            
tcp        0      0 127.0.0.1:29159         0.0.0.0:*               LISTEN      7621/kubelet        
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      7621/kubelet        
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      8042/kube-proxy     
tcp6       0      0 :::6443                 :::*                    LISTEN      7511/kube-apiserver 
tcp6       0      0 :::10256                :::*                    LISTEN      8042/kube-proxy     
tcp6       0      0 :::10257                :::*                    LISTEN      6993/kube-controlle 
tcp6       0      0 :::10259                :::*                    LISTEN      7164/kube-scheduler 
tcp6       0      0 :::22                   :::*                    LISTEN      483/sshd            
tcp6       0      0 :::10250                :::*                    LISTEN      7621/kubelet      
```


设置kubeconfig
``` bash
mkdir -p $HOME/.kube
cp -rp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

通过kubectl检查kubernetes运行情况

``` bash
[root@k8s-master-node1 /home/test]# kubectl get no
NAME               STATUS     ROLES                  AGE   VERSION
k8s-master-node1   NotReady   control-plane,master   94s   v1.20.5
```

> 因为没安装 cni，所以现在还是 NotReady

###  其他 Master 加入集群

> 在其他 master 上执行

将其他两个master节点加入进集群

**配置审计策略**

```
cat << EOF > /etc/kubernetes/audit-policy.yaml
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF
```

**加入集群**

``` bash
# api域名先指向k8s-m1
echo '192.168.77.140 apiserver.cluster.local' >> /etc/hosts

kubeadm join apiserver.cluster.local:6443 --token nj9wia.n8pmj18em5cfzg7e \
  --discovery-token-ca-cert-hash sha256:64ddf85513a7d0321cc19bad82209c83783276daa4cd8ebcb114f63804e649be \
  --control-plane --certificate-key 7bd6f9874e65df89af33f8f235494ec2a3544e58be89f96b5118be9e8bbc7a15

mkdir -p $HOME/.kube
cp -rp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 将api域名指向本地
sed -i 's#192.168.77.140 apiserver.k8s.local#127.0.0.1 apiserver.k8s.local#g' /etc/hosts
```

master 节点完成后，此时的集群列表

```bash
# kubectl get nodes
NAME               STATUS     ROLES                  AGE     VERSION
k8s-master-node1   NotReady   control-plane,master   2m      v1.20.5
k8s-master-node2   NotReady   control-plane,master   2m22s   v1.20.5
k8s-master-node3   NotReady   control-plane,master   2m42s   v1.20.5
```



## 部署worker节点

### 安装 kube 组件

安装`kubeadm`和`kubelet`

``` bash
echo 'deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
apt-get update

export KUBE_VERSION="1.20.5"
apt-get install -y kubeadm=$KUBE_VERSION-00 kubelet=$KUBE_VERSION-00
apt-mark hold kubelet kubeadm

[ -d /etc/bash_completion.d ] && kubeadm completion bash > /etc/bash_completion.d/kubadm
      
[ ! -d /usr/lib/systemd/system/kubelet.service.d ] && mkdir -p /usr/lib/systemd/system/kubelet.service.d
cat << EOF > /usr/lib/systemd/system/kubelet.service.d/11-cgroup.conf
[Service]
CPUAccounting=true
MemoryAccounting=true
BlockIOAccounting=true
ExecStartPre=/usr/bin/bash -c '/usr/bin/mkdir -p /sys/fs/cgroup/{cpuset,memory,systemd,pids,"cpu,cpuacct"}/{system,kube,kubepods}.slice'
Slice=kube.slice
EOF
systemctl daemon-reload
 
systemctl enable kubelet.service
```

### 配置 haproxy

> 使用 haproxy 来提供 Kubernetes API Server 的负载均衡

``` bash
apt-ge install -y haproxy

cat <<EOF > /etc/haproxy/haproxy.cfg
global
  log 127.0.0.1 local0
  log 127.0.0.1 local1 notice
  tune.ssl.default-dh-param 2048

defaults
  log global
  mode http
  option dontlognull
  timeout connect 5000ms
  timeout client 600000ms
  timeout server 600000ms

listen stats
  bind :9090
  mode http
  balance
  stats uri /haproxy_stats
  stats auth admin:admin123
  stats admin if TRUE

frontend kube-apiserver-https
   mode tcp
   bind :6443
   default_backend kube-apiserver-backend

backend kube-apiserver-backend
  mode tcp
  balance roundrobin
  stick-table type ip size 200k expire 30m
  stick on src
  server apiserver1 192.168.77.140:6443 check
  server apiserver2 192.168.77.141:6443 check
  server apiserver3 192.168.77.142:6443 check
EOF
```
启动haproxy
``` bash
systemctl start haproxy
systemctl enable --now haproxy
```

### 加入集群

``` bash
# echo '127.0.0.1 apiserver.cluster.local' >> /etc/hosts

# kubeadm join apiserver.cluster.local:6443 --token nj9wia.n8pmj18em5cfzg7e \
    --discovery-token-ca-cert-hash sha256:64ddf85513a7d0321cc19bad82209c83783276daa4cd8ebcb114f63804e649be 
```

## 配置集群

### 配置 worker role

将 worker 节点的 role 标签设置为 `worker`

```bash
# kubectl get node --selector='!node-role.kubernetes.io/master' | grep '<none>' | awk '{print "kubectl label node " $1 " node-role.kubernetes.io/worker= --overwrite" }' | bash

# kubectl get node
NAME               STATUS     ROLES                  AGE   VERSION
k8s-master-node1   NotReady   control-plane,master   40m   v1.20.5
k8s-master-node2   NotReady   control-plane,master   20m   v1.20.5
k8s-master-node3   NotReady   control-plane,master   17m   v1.20.5
k8s-worker-node1   NotReady   worker                 85s   v1.20.5
k8s-worker-node2   NotReady   worker                 81s   v1.20.5
k8s-worker-node3   NotReady   worker                 79s   v1.20.5
```

### 配置 网络

[Flannel](https://github.com/flannel-io/flannel) 是一种简单易行的方式来配置为Kubernetes设计的第三层网络结构。本次选用 flannel 组件作为集群网络组件。

```bash
# 下载 flannel
wget https://cdn.jsdelivr.net/gh/coreos/flannel@v0.13.0/Documentation/kube-flannel.yml

# 配置 pod 网络
sed -i 's#10.244.0.0/16#10.244.0.0/16#g' kube-flannel.yml

# 配置直接路由
sed -i 's#"Type": "vxlan"#"Type": "vxlan", "DirectRouting": true#g' kube-flannel.yml

# 应用 flannel
kubectl apply -f kube-flannel.yml

# 等待 flannel pod 正常启动
kubectl wait --namespace kube-system --for=condition=ready pods --selector=app=flannel --timeout=60s
```

flannel 启动正常时，节点状态也就ready了

```bash
# kubectl  get nodes
NAME               STATUS   ROLES                  AGE    VERSION
k8s-master-node1   Ready    control-plane,master   41m   v1.20.5
k8s-master-node2   Ready    control-plane,master   22m   v1.20.5
k8s-master-node3   Ready    control-plane,master   18m   v1.20.5
k8s-worker-node1   Ready    worker                 2m    v1.20.5
k8s-worker-node2   Ready    worker                 2m    v1.20.5
k8s-worker-node3   Ready    worker                 2m    v1.20.5
```

### 配置 metrics-server

[Metrics Server](https://github.com/kubernetes-sigs/metrics-server) 是实现了Metrics API的元件，其目标是取代Heapster作为Pod与Node提供资源的Usage metrics，该组件会从每个Kubernetes节点上的Kubelet所公开的Summary API中收集Metrics。

在任意master节点上执行kubectl top命令

```bash
# kubectl top node
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)
```

发现top指令无法取得Metrics，这表示Kubernetes 丛集没有安装Heapster或是Metrics Server 来提供Metrics API给top指令取得资源使用量。

部署metric-server组件

```bash
# wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml -O metrics-server.yaml

# sed -i -e 's#k8s.gcr.io/metrics-server#registry.cn-hangzhou.aliyuncs.com/kainstall#g' \
       -e '/--kubelet-preferred-address-types=.*/d' \
	   -e 's/\(.*\)- --secure-port=4443/\1- --secure-port=4443\n\1- --kubelet-insecure-tls\n\1- --kubelet-preferred-address-types=InternalIP,InternalDNS,ExternalIP,ExternalDNS,Hostname/g' \
	   metrics-server.yaml
             
# kubectl apply -f metrics-server.yaml
```

查看聚合的api

```bash
# kubectl get apiservices.apiregistration.k8s.io  | grep v1beta1.metrics.k8s.io
v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        55M

# kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
{"kind":"NodeMetricsList","apiVersion":"metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/metrics.k8s.io/v1beta1/nodes"},"items":[{"metadata":{"name":"k8s-master-node1","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-master-node1","creationTimestamp":"2021-04-03T13:56:08Z"},"timestamp":"2021-04-03T13:55:41Z","window":"30s","usage":{"cpu":"224628621n","memory":"1255272Ki"}},{"metadata":{"name":"k8s-master-node2","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-master-node2","creationTimestamp":"2021-04-03T13:56:08Z"},"timestamp":"2021-04-03T13:55:45Z","window":"30s","usage":{"cpu":"207660844n","memory":"984580Ki"}},{"metadata":{"name":"k8s-master-node3","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-master-node3","creationTimestamp":"2021-04-03T13:56:08Z"},"timestamp":"2021-04-03T13:55:39Z","window":"30s","usage":{"cpu":"217148133n","memory":"1053340Ki"}},{"metadata":{"name":"k8s-worker-node1","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-worker-node1","creationTimestamp":"2021-04-03T13:56:08Z"},"timestamp":"2021-04-03T13:55:42Z","window":"30s","usage":{"cpu":"87641005n","memory":"565416Ki"}},{"metadata":{"name":"k8s-worker-node2","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-worker-node2","creationTimestamp":"2021-04-03T13:56:08Z"},"timestamp":"2021-04-03T13:55:38Z","window":"30s","usage":{"cpu":"85071223n","memory":"454372Ki"}},{"metadata":{"name":"k8s-worker-node3","selfLink":"/apis/metrics.k8s.io/v1beta1/nodes/k8s-worker-node3","creationTimestamp":"2021-04-03T13:56:08Z"},"timestamp":"2021-04-03T13:55:37Z","window":"30s","usage":{"cpu":"47689888n","memory":"408796Ki"}}]}

```

完成后，等待一段时间收集Metrics，再次执行`kubectl top`

```bash
# kubectl top node
NAME               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master-node1   238m         15%    1272Mi          74%       
k8s-master-node2   220m         14%    956Mi           55%       
k8s-master-node3   208m         13%    1036Mi          60%       
k8s-worker-node1   83m          5%     552Mi           32%       
k8s-worker-node2   86m          5%     443Mi           25%       
k8s-worker-node3   48m          9%     399Mi           23%  
```



### 配置 coredns

kubeadm 中的 coredns，默认没有 **反亲和性** 配置，这样会使 pod 存在同一个节点上，从而增加了风险。

```bash
# kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
NAME                       READY   STATUS    RESTARTS   AGE    IP           NODE               NOMINATED NODE   READINESS GATES
coredns-85bb79f4b4-4r6jw   1/1     Running   0          164m   10.244.2.3   k8s-master-node3   <none>           <none>
coredns-85bb79f4b4-gwptq   1/1     Running   0          164m   10.244.2.2   k8s-master-node3   <none>           <none>
```

给 coredns 添加 **反亲和性**，防止coredns 集中在一个节点上。

```bash
# kubectl -n kube-system patch deployment coredns --patch '{"spec": {"template": {"spec": {"affinity":{"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"weight":100,"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"k8s-app","operator":"In","values":["kube-dns"]}]},"topologyKey":"kubernetes.io/hostname"}}]}}}}}}' --record
```

再次查看 pod 分配，已经分布在不同的节点上了。
```bash
# kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
coredns-8496bbfb78-dvs5r   1/1     Running   0          10m   10.244.3.3   k8s-worker-node1   <none>           <none>
coredns-8496bbfb78-x274f   1/1     Running   0          10m   10.244.4.4   k8s-worker-node2   <none>           <none>
```



### 配置 etcd 定时备份

这里我们通过 CronJob 资源进行定时备份 etcd 数据到节点目录 `/var/lib/etcd/backups` 中, 保存最近 30 个备份。

```bash
master_num=3
etcd_image=$(kubeadm config images list --config=/etc/kubernetes/kubeadm-config.yaml 2>/dev/null | grep etcd:)

cat << EOF | kubectl apply -f -
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: etcd-snapshot
  namespace: kube-system
spec:
  schedule: '0 */6 * * *'
  successfulJobsHistoryLimit: 3
  suspend: false
  concurrencyPolicy: Allow
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 6
      parallelism: ${master_num}
      completions: ${master_num}
      template:
        metadata:
          labels:
            app: etcd-snapshot
        spec:
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values:
                    - etcd-snapshot
                topologyKey: 'kubernetes.io/hostname'
          containers:
          - name: etcd-snapshot
            image: ${etcd_image}
            imagePullPolicy: IfNotPresent
            args:
            - -c
            - etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt
              --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
              snapshot save /backup/etcd-snapshot-\$(date +%Y-%m-%d_%H:%M:%S_%Z).db
              && echo 'delete old backups' && find /backup -type f -mtime +30 -exec rm -fv {} \\; || echo error
            command:
            - /bin/sh
            env:
            - name: ETCDCTL_API
              value: '3'
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
            - name: etc
              mountPath: /etc
            - name: bin
              mountPath: /usr/bin
            - name: lib64
              mountPath: /lib64
            - name: lib
              mountPath: /lib
          dnsPolicy: ClusterFirst
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/master: ''
          tolerations:
          - effect: NoSchedule
            operator: Exists
          restartPolicy: OnFailure
          terminationGracePeriodSeconds: 30
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
              type: DirectoryOrCreate
          - name: backup
            hostPath:
              path: /var/lib/etcd/backups
              type: DirectoryOrCreate
          - name: etc
            hostPath:
              path: /etc
          - name: bin
            hostPath:
              path: /usr/bin
          - name: lib64
            hostPath:
              path: /lib64
          - name: lib
            hostPath:
              path: /lib
EOF

kubectl -n kube-system get cronjob
NAME            SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
etcd-snapshot   0 */6 * * *   False     0        <none>          70s


jobname="etcd-snapshot-$(date +%s)"
kubectl create job --from=cronjob/etcd-snapshot ${jobname} -n kube-system && \
kubectl wait --for=condition=complete job/${jobname} -n kube-system
```



### 配置 Ingress

> [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 是 Kubernetes 中的一个抽象资源，其功能是透过 Web Server 的 Virtual Host 概念以域名(Domain Name)方式转发到内部 Service，这避免了使用 Service 中的 NodePort 与 LoadBalancer 类型所带来的限制(如 Port 数量上限)，而实现 Ingress 功能则是透过 [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers)来达成，它会负责监听 Kubernetes API中的 Ingress 与 Service 资源物件，并在发生资源变化时，依据资源预期的结果来设定 Web Server。另外Ingress Controller 有许多实现可以选择：

- [Ingress NGINX](https://github.com/kubernetes/ingress-nginx): Kubernetes 官方维护的，也是本次安装使用的 Controller。
- [F5 BIG-IP Controller](https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/v1.5/): F5 所开发的 Controller，它能够让管理员透过 CLI 或 API 从 Kubernetes 与 OpenShift 管理 F5 BIG-IP 设备。
- [Ingress Kong](https://konghq.com/blog/kubernetes-ingress-controller-for-kong/): 著名的开源 API Gateway 专案所维护的 Kubernetes Ingress Controller。
- [Træfik](https://github.com/containous/traefik): 是一套开源的 HTTP 反向代理与负载平衡器。
- [Voyager](https://github.com/appscode/voyager): 一套以 HAProxy 为底的 Ingress Controller。

部署 ingress-nginx

```bash
wget https://cdn.jsdelivr.net/gh/kubernetes/ingress-nginx@controller-v0.44.0/deploy/static/provider/baremetal/deploy.yaml -O ingress-nginx.yaml

sed -i -e 's#k8s.gcr.io/ingress-nginx#registry.cn-hangzhou.aliyuncs.com/kainstall#g' \
       -e 's#@sha256:.*$##g' ingress-nginx.yaml

kubectl wait --namespace ingress-nginx --for=condition=ready pods --selector=app.kubernetes.io/component=controller --timeout=60s
pod/ingress-nginx-controller-67848f7b-2gxzb condition met
```

官方默认加上了 `admission` 功能，而我们的 apiserver 使用宿主机的dns，不是coredns，所以连接不上 ingress-nginx Controller 的 service 地址，这里我们把 `admission` 准入钩子去掉，使我们创建 ingress 资源时，不去验证Controller。

> admission webhook 的作用我简单的总结下，当用户的请求到达 k8s apiserver 后，apiserver 根据 `MutatingWebhookConfiguration` 和 `ValidatingWebhookConfiguration` 的配置，先调用 `MutatingWebhookConfiguration` 去修改用户请求的配置文件，最后会调用 `ValidatingWebhookConfiguration` 来验证这个修改后的配置文件是否合法。

```bash
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```



### 配置 Dashboard

[Dashboard ](https://github.com/kubernetes/dashboard)是Kubernetes官方开发的基于Web的仪表板，目的是提升管理Kubernetes集群资源便利性，并以资源视觉化方式，来让人更直觉的看到整个集群资源状态。

部署 dashboard

```bash
wget https://cdn.jsdelivr.net/gh/kubernetes/dashboard@v2.2.0/aio/deploy/recommended.yaml -O dashboard.yaml
kubectl apply -f dashboard.yaml
```

部署 ingress 

```
cat << EOF | kubectl apply -f -
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/secure-backends: 'true'
    nginx.ingress.kubernetes.io/backend-protocol: 'HTTPS'
    nginx.ingress.kubernetes.io/ssl-passthrough: 'true'
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard 
spec:
  tls:
  - hosts:
    - kubernetes-dashboard.cluster.local
    secretName: kubernetes-dashboard-certs
  rules:
  - host: kubernetes-dashboard.cluster.local
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
EOF
```

创建 sa，使用 sa 的  token 进行登录 dashboard

```bash
kubectl create serviceaccount kubernetes-dashboard-admin-sa -n kubernetes-dashboard
kubectl create clusterrolebinding kubernetes-dashboard-admin-sa --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:kubernetes-dashboard-admin-sa -n kubernetes-dashboard

kubectl describe secrets $(kubectl describe sa kubernetes-dashboard-admin-sa -n kubernetes-dashboard | awk '/Tokens/ {print $2}') -n kubernetes-dashboard | awk '/token:/{print $2}'

eyJhbGciOiJSUzI1NiIsImtpZCI6IkFpMWkxemI3cnFlUmNmVzFJbno5a3IzWktpVGxlaXJlaXZna0NxRlRRTWMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi1zYS10b2tlbi05YnY0bCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi1zYSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjI4ODIyOWQ0LTA3NTItNGIyNi05ZjhjLWE1N2ZlYzNjNGYwMyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi1zYSJ9.CEBpIcs8bS5960T-F4Bdo4Di4Y61CBDG9SoRWosjxIPm8RuRD2CwSX_fNBMbXLxPVPZME90EwJkaBTSHBwTb1pYOphety4OvFXStpnqj6tyJl9EWrzLdibfnJnZ1cq0X9cIGyjQ6gcuboqQJQEzxgBJdTuELU3LHHQhXOzQZHH_pMPVjntRIQEQ5wGDvRz50Eig-KMaK3IislFEgA3a8mkdVppkJA3gPprEAnUp6of8bKFa6rrZL1Kcx0u-C9We6sgzZcLYPahx6zKO4l0XkaxHrwiNodgBbCaqJ2C3V78p1HD9u16XMKjoz5rzkakajJmB0zMeRaFdcFHlKXrQ1gQ
```

获取 dashboard 的 ingres 连接地址

```bash
echo https://$(kubectl get node -o jsonpath='{range .items[*]}{ .status.addresses[?(@.type=="InternalIP")].address} {.status.conditions[?(@.status == "True")].status}{"\n"}{end}' | awk '{if($2=="True")a=$1}END{print a}'):$(kubectl get svc --all-namespaces -o go-template="{{range .items}}{{if eq .metadata.name \"ingress-nginx-controller\" }}{{range.spec.ports}}{{if eq .port "443"}}{{.nodePort}}{{end}}{{end}}{{end}}{{end}}")


https://192.168.77.145:34239
```



将 host 绑定后，使用token 进行登录

```bash
192.168.77.145 kubernetes-dashboard.cluster.local

https://kubernetes-dashboard.cluster.local:34239
```



## 测试集群

### 重启集群

1. 将 集群节点 全部重启
2. 获取节点信息

``` bash
# kubectl get node
NAME               STATUS   ROLES                  AGE    VERSION
k8s-master-node1   Ready    control-plane,master   50m   v1.20.5
k8s-master-node2   Ready    control-plane,master   31m   v1.20.5
k8s-master-node3   Ready    control-plane,master   27m   v1.20.5
k8s-worker-node1   Ready    worker                 11m   v1.20.5
k8s-worker-node2   Ready    worker                 11m   v1.20.5
k8s-worker-node3   Ready    worker                 11m   v1.20.5

```

### 部署 whoami app

部署应用

 ``` bash
cat <<EOF | kubectl apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-demo-app
  labels:
    app: ingress-demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ingress-demo-app
  template:
    metadata:
      labels:
        app: ingress-demo-app
    spec:
      containers:
      - name: whoami
        image: traefik/whoami:v1.6.1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-demo-app
spec:
  type: ClusterIP
  selector:
    app: ingress-demo-app
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-demo-app
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: app.demo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: ingress-demo-app
          servicePort: 80
EOF
 ```

获取应用的pods

```bash
kubectl get pods -l app=ingress-demo-app
NAME                                READY   STATUS    RESTARTS   AGE
ingress-demo-app-694bf5d965-8zqlb   1/1     Running   0          41s
ingress-demo-app-694bf5d965-h4bcm   1/1     Running   0          41s

```

通过 ingress 访问

```bash
echo http://$(kubectl get node -o jsonpath='{range .items[*]}{ .status.addresses[?(@.type=="InternalIP")].address} {.status.conditions[?(@.status == "True")].status}{"\n"}{end}' | awk '{if($2=="True")a=$1}END{print a}'):$(kubectl get svc --all-namespaces -o go-template="{{range .items}}{{if eq .metadata.name \"ingress-nginx-controller\" }}{{range.spec.ports}}{{if eq .port "80"}}{{.nodePort}}{{end}}{{end}}{{end}}{{end}}")
http://192.168.77.145:38825

kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE               NOMINATED NODE   READINESS GATES
ingress-nginx-controller-67848f7b-2gxzb   1/1     Running   2          4d    10.244.3.11   k8s-worker-node1   <none>           <none>



curl -H 'Host:app.demo.com' http://192.168.77.145:38825
Hostname: ingress-demo-app-694bf5d965-8zqlb
IP: 127.0.0.1
IP: 10.244.5.2
RemoteAddr: 10.244.3.11:59762
GET / HTTP/1.1
Host: app.demo.com
User-Agent: curl/7.64.0
Accept: */*
X-Forwarded-For: 192.168.77.145
X-Forwarded-Host: app.demo.com
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Real-Ip: 192.168.77.145
X-Request-Id: 1c5c76949fe909222a76d59c631ac82b
X-Scheme: http
```

从 whoami  应用返回单额信息可以看到，我们通过 ingress 访问到了 whomai app。

## 重置集群

安装有问题的时候，可以使用下列命令重置集群

``` bash 
kubeadm reset -f
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm --clear 
systemctl stop kubelet
docker rm -f -v $(docker ps -q)
find /var/lib/kubelet | xargs -n 1 findmnt -n -t tmpfs -o TARGET -T | uniq | xargs -r umount -v
rm -r -f /etc/kubernetes /var/lib/kubelet /var/lib/etcd ~/.kube/config
```

{% endraw %}