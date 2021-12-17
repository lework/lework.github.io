---
layout: post
title: 'GreenPlum 集群安装'
date: '2021-08-05 20:02'
category: GreenPlum
tags: linux GreenPlum
author: lework 
---
* content
{:toc}

GreenPlum 是一个基于 PostgreSQL 的 数据仓库

## 安装说明

GreenPlum 6.X 目前支持以下版本操作系统:

- Red Hat Enterprise Linux 64-bit 7.x (See the following [Note](https://docs.greenplum.org/6-16/install_guide/platform-requirements.html#topic13__7x-issue).)
- Red Hat Enterprise Linux 64-bit 6.x
- CentOS 64-bit 7.x
- CentOS 64-bit 6.x
- Ubuntu 18.04 LTS
- Oracle Linux 64-bit 7, using the Red Hat Compatible Kernel (RHCK)



本次安装使用 1个 master 节点，2个 segment 节点. 操作系统为 CentOS 7.8

| 主机名    | ip地址         | 操作系统   | 硬件配置 | 角色             |
| --------- | -------------- | ---------- | -------- | ---------------- |
| gp-master | 192.168.77.130 | CentOS 7.8 | 2C4G     | Master           |
| gp-node1  | 192.168.77.131 | CentOS 7.8 | 2C8G     | seg1,mirror seg2 |
| gp-node2  | 192.168.77.132 | CentOS 7.8 | 2C8G     | seg2,mirror seg1 |





## 初始化所有节点

> 在集群所有节点上执行下面的操作
> **注意**：以下操作有些存在过度优化，请根据自身情况择选。

### 仓库源调整

镜像源调整

```bash
sed -e 's!^#baseurl=!baseurl=!g' \
       -e  's!^mirrorlist=!#mirrorlist=!g' \
       -e 's!mirror.centos.org!mirrors.aliyun.com!g' \
       -i  /etc/yum.repos.d/CentOS-Base.repo

yum install -y epel-release

sed -e 's!^mirrorlist=!#mirrorlist=!g' \
    -e 's!^metalink=!#metalink=!g' \
    -e 's!^#baseurl=!baseurl=!g' \
    -e 's!//download\.fedoraproject\.org/pub!//mirrors.aliyun.com!g' \
    -e 's!http://mirrors\.aliyun!https://mirrors.aliyun!g' \
    -i /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel-testing.repo
```

### 关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

### 关闭 selinux
```bash
setenforce 0
sed -i "s#=enforcing#=disabled#g" /etc/selinux/config
```

### 关闭 swap
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
cat << EOF >  /etc/sysctl.d/99-greenplum.conf
kernel.shmmax = $(expr $(getconf _PHYS_PAGES) / 2 \* $(getconf PAGE_SIZE))           #注意这里查看说明Shared Memory Pages
kernel.shmmni = 4096
kernel.shmall = $(expr $(getconf _PHYS_PAGES) / 2)

vm.overcommit_memory = 2        # 注意这里查看说明Segment Host Memory
vm.overcommit_ratio = 95         # 注意这里查看说明Segment Host Memory

net.ipv4.ip_local_port_range = 10000 65535     # See Port Settings
kernel.sem = 500 2048000 200 4096
kernel.sysrq = 1
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.msgmni = 2048
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.conf.all.arp_filter = 1
net.core.netdev_max_backlog = 10000
net.core.rmem_max = 2097152
net.core.wmem_max = 2097152
vm.swappiness = 10
vm.zone_reclaim_mode = 0
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100
vm.dirty_background_ratio = 3      # 注意这里查看说明System Memory
vm.dirty_ratio = 10
#vm.dirty_background_bytes = 1610612736
#vm.dirty_bytes = 4294967296 
EOF
sysctl --system
```

### history 数据格式和 ps1

```bash
cat << EOF >> /etc/bashrc 
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

### 时间同步

在master节点配置数据中心的时间,

```bash
yum install -y ntpdate ntp
ntpdate 0.cn.pool.ntp.org
hwclock --systohc
cat << EOF >> /etc/ntp.conf
driftfile /var/lib/ntp/drift
server 0.cn.pool.ntp.org
server 1.cn.pool.ntp.org
server 2.cn.pool.ntp.org
server 3.cn.pool.ntp.org
EOF

systemctl enable --now ntpd
ntpq -p
```

在数据节点配置为master节点的IP为优先

```bash
yum install -y ntpdate ntp
ntpdate 0.cn.pool.ntp.org
hwclock --systohc
cat << EOF >> /etc/ntp.conf
server gp-master prefer
driftfile /var/lib/ntp/drift
server 0.cn.pool.ntp.org
server 1.cn.pool.ntp.org
server 2.cn.pool.ntp.org
server 3.cn.pool.ntp.org
EOF

systemctl enable --now ntpd
ntpq -p
```

### 关闭透明大页(THP)

```bash
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

### 配置host地址

```bash
cat << EOF >> /etc/hosts
192.168.77.130 gp-master
192.168.77.131 gp-node1
192.168.77.132 gp-node2
EOF
```

各个节点也要修改主机名称

```
hostnamectl set-hostname gp-master
```

### 创建 gp 用户

需要创建一个特定的用户运行 gp,一般为 gpadmin:

```bash
groupadd gpadmin
useradd -g gpadmin gpadmin
echo "gpadmin" | passwd --stdin gpadmin
```

### 安装依赖包

```bash
yum install -y apr apr-util bash bzip2 curl krb5 libcurl \
               libevent libevent2 libxml2 libyaml zlib openldap \
               openssh openssl openssl-libs perl readline rsync \
               R sed tar zip apr apr-util libyaml libevent java
```

### 创建数据目录

> 注意:
> The only file system supported for running Greenplum Database is the XFS file system. All other file systems are explicitly not supported by Pivotal.
> 官网提示, gp 只支持 xfs 文件系统,不支持其它文件系统.

我们这里使用的磁盘为 /dev/sdb 大小为50GB,挂载目录为 /data

```bash
pvcreate /dev/sdb
vgcreate datavg /dev/sdb
lvcreate -L +49.9G -n datalv datavg
mkfs.xfs  /dev/datavg/datalv
mkdir /data
echo "/dev/datavg/datalv      /data                   xfs     nodev,noatime,nobarrier,inode64  0 0" >> /etc/fstab
mount -a
chown -R gpadmin:gpadmin /data
```

### SSH 连接

```bash
echo "
MaxStartups 10:30:200
MaxSessions 200
" >> /etc/ssh/sshd_config
```

### IPC对象删除

> 禁用RHEL 7.2或CentOS 7.2或Ubuntu的IPC对象删除。默认值 系统的 设置 RemoveIPC =YES 非系统用户帐户注销时删除IPC连接。这将导致Greenplum数据库实用程序 gpinitsystem因信号错误而失败。

```
echo "RemoveIPC=no" >>/etc/systemd/logind.conf
systemctl restart systemd-logind
```

完成后，重启所有节点。

## 安装 Greenplum

Greenplum数据库软件

```bash
wget https://github.com/greenplum-db/gpdb/releases/download/6.17.0/open-source-greenplum-db-6.17.0-rhel7-x86_64.rpm
rpm -ivh open-source-greenplum-db-6.17.0-rhel7-x86_64.rpm 
```

将所有者和已安装文件的组更改为 gpadmin：

```bash
chown -R gpadmin:gpadmin /usr/local/greenplum*
chgrp -R gpadmin /usr/local/greenplum*
```

## 配置 SSH 免密

在 master 节点上免密登录 其他节点

```bash
yum install -y sshpass
for NODE in gp-master gp-node1 gp-node2; do
  echo "--- $NODE ---"
  sshpass -p gpadmin ssh gpadmin@${NODE} "ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
"
  sshpass -p gpadmin ssh-copy-id -o "StrictHostKeyChecking no" -i /home/gpadmin/.ssh/id_rsa.pub gpadmin@${NODE}
done
```

创建 hostfile-exkeys 文件

```bash
su - gpadmin
echo "source/usr/local/greenplum-db-6.17.0/greenplum_path.sh >> ~/.bash_profile"
. ~/.bash_profile
cat << EOF >  ~/hostfile_exkeys
gp-node1
gp-node2
EOF
```

**使用gpssh-exkeys启用无密码访问**

```
[gpadmin@gp-master ~]$ gpssh-exkeys -f ~/hostfile_exkeys 
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] retrieving credentials from remote hosts
  ... send to gp-node1
  ... send to gp-node2

[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with gp-node1
  ... finished key exchange with gp-node2

[INFO] completed successfully
```

**使用gpssh确认免密码登录配置成功**

```
[gpadmin@gp-master ~]$ gpssh -f ~/hostfile_exkeys -e 'ls -l /usr/local/greenplum-db'
[gp-node2] ls -l /usr/local/greenplum-db
[gp-node2] lrwxrwxrwx 1 gpadmin gpadmin 30 Jul 20 15:17 /usr/local/greenplum-db -> /usr/local/greenplum-db-6.17.0
[gp-node1] ls -l /usr/local/greenplum-db
[gp-node1] lrwxrwxrwx 1 gpadmin gpadmin 30 Jul 20 15:17 /usr/local/greenplum-db -> /usr/local/greenplum-db-6.17.0
```

使用gpadmin用户,在其它所有的节点执行以下命令:

```
su - gpadmin 
echo "source/usr/local/greenplum-db-6.17.0/greenplum_path.sh >> ~/.bash_profile"
. ~/.bash_profile

cat << EOF >  ~/hostfile_exkeys
gp-node1
gp-node2
EOF
gpssh-exkeys -f ~/hostfile_exkeys 
gpssh -f ~/hostfile_exkeys -e 'ls -l /usr/local/greenplum-db'
```

## 创建数据存储区域

### 在master上创建存储区域

```
mkdir -p /data/master
chown -R gpadmin:gpadmin /data/master
```

### 在semgents实例创建存储区域

```
su - gpadmin 
gpssh -f ~/hostfile_exkeys -e 'mkdir -p /data/primary;mkdir -p /data/mirror;chown -R gpadmin /data/*'
```

## 测试性能

### 验证网络性能

```
[gpadmin@gp-node1 ~]$  gpcheckperf -f ~/hostfile_exkeys -r N -d /tmp
/usr/local/greenplum-db-6.17.0/bin/gpcheckperf -f /home/gpadmin/hostfile_exkeys -r N -d /tmp

-------------------
--  NETPERF TEST
-------------------
NOTICE: -t is deprecated, and has no effect
NOTICE: -f is deprecated, and has no effect
NOTICE: -t is deprecated, and has no effect
NOTICE: -f is deprecated, and has no effect

====================
==  RESULT 2021-07-20T15:54:43.188993
====================
Netperf bisection bandwidth test
gp-node1 -> gp-node2 = 332.800000
gp-node2 -> gp-node1 = 325.560000

Summary:
sum = 658.36 MB/sec
min = 325.56 MB/sec
max = 332.80 MB/sec
avg = 329.18 MB/sec
median = 332.80 MB/sec
```
### 验证磁盘IO和内存带宽

```
[gpadmin@gp-node1 ~]$ gpcheckperf -f ~/hostfile_exkeys -r ds -D -d /data/primary -d  /data/mirror 
/usr/local/greenplum-db-6.17.0/bin/gpcheckperf -f /home/gpadmin/hostfile_exkeys -r ds -D -d /data/primary -d /data/mirror

--------------------
--  DISK WRITE TEST
--------------------

--------------------
--  DISK READ TEST
--------------------

--------------------
--  STREAM TEST
--------------------

====================
==  RESULT 2021-07-20T15:59:11.754871
====================

 disk write avg time (sec): 41.56
 disk write tot bytes: 65454604288
 disk write tot bandwidth (MB/s): 1501.99
 disk write min bandwidth (MB/s): 749.73 [gp-node2]
 disk write max bandwidth (MB/s): 752.26 [gp-node1]
 -- per host bandwidth --
    disk write bandwidth (MB/s): 749.73 [gp-node2]
    disk write bandwidth (MB/s): 752.26 [gp-node1]


 disk read avg time (sec): 14.71
 disk read tot bytes: 65454604288
 disk read tot bandwidth (MB/s): 4245.00
 disk read min bandwidth (MB/s): 2117.45 [gp-node2]
 disk read max bandwidth (MB/s): 2127.55 [gp-node1]
 -- per host bandwidth --
    disk read bandwidth (MB/s): 2117.45 [gp-node2]
    disk read bandwidth (MB/s): 2127.55 [gp-node1]


 stream tot bandwidth (MB/s): 23051.70
 stream min bandwidth (MB/s): 9154.40 [gp-node1]
 stream max bandwidth (MB/s): 13897.30 [gp-node2]
 -- per host bandwidth --
    stream bandwidth (MB/s): 13897.30 [gp-node2]
    stream bandwidth (MB/s): 9154.40 [gp-node1]
```

## 初始化数据库

初始化Greenplum数据库步骤
1.确保已完成配置系统和安装Greenplum数据库软件中描述的所有安装任务 。
2.创建一个包含文件segment主机地址的主机文件 。例如前面配置~/hostfile_gpssh_segonly。
3.创建您的Greenplum数据库系统配置文件。请参阅创建Greenplum数据库配置文件。
4.默认情况下，Greenplum数据库将使用master主机系统的语言环境进行初始化。确保这是您要使用的正确语言环境，因为某些语言环境选项在初始化后无法更改。有关更多信息，请参见配置时区和本地化设置。
5.在主控主机上运行Greenplum数据库初始化实用程序。请参阅 运行初始化实用程序。
6.设置Greenplum数据库时区。请参阅设置Greenplum数据库时区。
7.为Greenplum数据库用户设置环境变量。请参阅设置Greenplum环境变量。

```
su - gpadmin
mkdir -p ~/gpconfigs
cat <<EOF > ~/gpconfigs/host_seg
gp-node1
gp-node2
EOF
```

## 创建gp配置文件

```
cp $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config \
     /home/gpadmin/gpconfigs/gpinitsystem_config
     
     
grep -v ^# gpconfigs/gpinitsystem_config | grep -v '^$'
ARRAY_NAME="Greenplum Data Platform"
SEG_PREFIX=gpseg
PORT_BASE=6000
declare -a DATA_DIRECTORY=(/data/primary /data/primary)
MASTER_HOSTNAME=gp-master
MASTER_DIRECTORY=/data/master
MASTER_PORT=5432
TRUSTED_SHELL=ssh
CHECK_POINT_SEGMENTS=8
ENCODING=UNICODE
MIRROR_PORT_BASE=7000
declare -a MIRROR_DATA_DIRECTORY=(/data/mirror /data/mirror)
```

保存配置

```
[gpadmin@gp-master ~]$ gpinitsystem -c gpconfigs/gpinitsystem_config -h gpconfigs/host_seg -O gpconfigs/init_config_template   
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Reading Greenplum configuration file gpconfigs/gpinitsystem_config
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Locale has not been set in gpconfigs/gpinitsystem_config, will set to default value
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Locale set to en_US.utf8
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-No DATABASE_NAME set, will exit following template1 updates
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-MASTER_MAX_CONNECT not set, will set to default value 250
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking configuration parameters, Completed
20210720:16:29:51:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
..
20210720:16:29:52:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Configuring build for standard array
20210720:16:29:52:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20210720:16:29:52:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Building primary segment instance array, please wait...
....
20210720:16:29:54:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Building group mirror array type , please wait...
....
20210720:16:29:55:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking Master host
20210720:16:29:55:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking new segment hosts, please wait...
........
20210720:16:30:01:014276 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking new segment hosts, Completed
```

初始化

```
[gpadmin@gp-master ~]$ gpinitsystem -c gpconfigs/gpinitsystem_config -h gpconfigs/host_seg
20210720:16:30:50:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking configuration parameters, please wait...
20210720:16:30:50:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Reading Greenplum configuration file gpconfigs/gpinitsystem_config
20210720:16:30:50:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Locale has not been set in gpconfigs/gpinitsystem_config, will set to default value
20210720:16:30:50:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Locale set to en_US.utf8
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-No DATABASE_NAME set, will exit following template1 updates
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-MASTER_MAX_CONNECT not set, will set to default value 250
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking configuration parameters, Completed
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
..
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Configuring build for standard array
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20210720:16:30:51:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Building primary segment instance array, please wait...
....
20210720:16:30:52:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Building group mirror array type , please wait...
....
20210720:16:30:54:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking Master host
20210720:16:30:54:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking new segment hosts, please wait...
........
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Checking new segment hosts, Completed
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Greenplum Database Creation Parameters
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:---------------------------------------
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master Configuration
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:---------------------------------------
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master instance name       = Greenplum Data Platform
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master hostname            = gp-master
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master port                = 5432
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master instance dir        = /data/master/gpseg-1
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master LOCALE              = en_US.utf8
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Greenplum segment prefix   = gpseg
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master Database            = 
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master connections         = 250
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master buffers             = 128000kB
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Segment connections        = 750
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Segment buffers            = 128000kB
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Checkpoint segments        = 8
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Encoding                   = UNICODE
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Postgres param file        = Off
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Initdb to be used          = /usr/local/greenplum-db-6.17.0/bin/initdb
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-GP_LIBRARY_PATH is         = /usr/local/greenplum-db-6.17.0/lib
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-HEAP_CHECKSUM is           = on
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-HBA_HOSTNAMES is           = 0
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Ulimit check               = Passed
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Array host connect type    = Single hostname per node
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master IP address [1]      = ::1
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master IP address [2]      = 172.17.0.1
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master IP address [3]      = 192.168.77.130
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Master IP address [4]      = fe80::20c:29ff:fe50:241
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Standby Master             = Not Configured
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Number of primary segments = 2
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total Database segments    = 4
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Trusted shell              = ssh
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Number segment hosts       = 2
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Mirror port base           = 7000
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Number of mirror segments  = 2
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Mirroring config           = ON
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Mirroring type             = Group
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:----------------------------------------
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Greenplum Primary Segment Configuration
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:----------------------------------------
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node1        6000    gp-node1        /data/primary/gpseg0    2
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node1        6001    gp-node1        /data/primary/gpseg1    3
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node2        6000    gp-node2        /data/primary/gpseg2    4
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node2        6001    gp-node2        /data/primary/gpseg3    5
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:---------------------------------------
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Greenplum Mirror Segment Configuration
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:---------------------------------------
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node2        7000    gp-node2        /data/mirror/gpseg0     6
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node2        7001    gp-node2        /data/mirror/gpseg1     7
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node1        7000    gp-node1        /data/mirror/gpseg2     8
20210720:16:31:01:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-gp-node1        7001    gp-node1        /data/mirror/gpseg3     9

Continue with Greenplum creation Yy|Nn (default=N):
> y
20210720:16:31:04:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Building the Master instance database, please wait...
20210720:16:31:08:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Starting the Master in admin mode
20210720:16:31:09:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Commencing parallel build of primary segment instances
20210720:16:31:09:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
....
20210720:16:31:10:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
.............
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:------------------------------------------------
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Parallel process exit status
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:------------------------------------------------
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total processes marked as completed           = 4
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total processes marked as killed              = 0
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total processes marked as failed              = 0
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:------------------------------------------------
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Deleting distributed backout files
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Removing back out file
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-No errors generated from parallel processes
20210720:16:31:23:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Restarting the Greenplum instance in production mode
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Starting gpstop with args: -a -l /home/gpadmin/gpAdminLogs -m -d /data/master/gpseg-1
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Gathering information and validating the environment...
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Obtaining Segment details from master...
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Greenplum Version: 'postgres (Greenplum Database) 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source'
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Commencing Master instance shutdown with mode='smart'
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Master segment instance directory=/data/master/gpseg-1
20210720:16:31:23:018519 gpstop:gp-master:gpadmin-[INFO]:-Stopping master segment and waiting for user connections to finish ...
server shutting down
20210720:16:31:24:018519 gpstop:gp-master:gpadmin-[INFO]:-Attempting forceful termination of any leftover master process
20210720:16:31:24:018519 gpstop:gp-master:gpadmin-[INFO]:-Terminating processes for segment /data/master/gpseg-1
20210720:16:31:24:018543 gpstart:gp-master:gpadmin-[INFO]:-Starting gpstart with args: -a -l /home/gpadmin/gpAdminLogs -d /data/master/gpseg-1
20210720:16:31:24:018543 gpstart:gp-master:gpadmin-[INFO]:-Gathering information and validating the environment...
20210720:16:31:24:018543 gpstart:gp-master:gpadmin-[INFO]:-Greenplum Binary Version: 'postgres (Greenplum Database) 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source'
20210720:16:31:24:018543 gpstart:gp-master:gpadmin-[INFO]:-Greenplum Catalog Version: '301908232'
20210720:16:31:24:018543 gpstart:gp-master:gpadmin-[INFO]:-Starting Master instance in admin mode
20210720:16:31:25:018543 gpstart:gp-master:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20210720:16:31:25:018543 gpstart:gp-master:gpadmin-[INFO]:-Obtaining Segment details from master...
20210720:16:31:25:018543 gpstart:gp-master:gpadmin-[INFO]:-Setting new master era
20210720:16:31:25:018543 gpstart:gp-master:gpadmin-[INFO]:-Master Started...
20210720:16:31:25:018543 gpstart:gp-master:gpadmin-[INFO]:-Shutting down master
20210720:16:31:25:018543 gpstart:gp-master:gpadmin-[INFO]:-Commencing parallel segment instance startup, please wait...
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-Process results...
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-   Successful segment starts                                            = 4
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-   Failed segment starts                                                = 0
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-   Skipped segment starts (segments are marked down in configuration)   = 0
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-Successfully started 4 of 4 segment instances 
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-Starting Master instance gp-master directory /data/master/gpseg-1 
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-Command pg_ctl reports Master gp-master instance active
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-Connecting to dbname='template1' connect_timeout=15
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-No standby master configured.  skipping...
20210720:16:31:26:018543 gpstart:gp-master:gpadmin-[INFO]:-Database successfully started
20210720:16:31:26:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Completed restart of Greenplum instance in production mode
20210720:16:31:26:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Commencing parallel build of mirror segment instances
20210720:16:31:26:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
....
20210720:16:31:26:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
.........
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:------------------------------------------------
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Parallel process exit status
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:------------------------------------------------
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total processes marked as completed           = 4
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total processes marked as killed              = 0
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Total processes marked as failed              = 0
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:------------------------------------------------
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Scanning utility log file for any warning messages
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Log file scan check passed
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Greenplum Database instance successfully created
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-------------------------------------------------------
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-To complete the environment configuration, please 
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-update gpadmin .bashrc file with the following
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-1. Ensure that the greenplum_path.sh file is sourced
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-2. Add "export MASTER_DATA_DIRECTORY=/data/master/gpseg-1"
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-   to access the Greenplum scripts for this instance:
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-   or, use -d /data/master/gpseg-1 option for the Greenplum scripts
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-   Example gpstate -d /data/master/gpseg-1
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Script log file = /home/gpadmin/gpAdminLogs/gpinitsystem_20210720.log
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-To remove instance, run gpdeletesystem utility
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-To initialize a Standby Master Segment for this Greenplum instance
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Review options for gpinitstandby
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-------------------------------------------------------
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-The Master /data/master/gpseg-1/pg_hba.conf post gpinitsystem
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-has been configured to allow all hosts within this new
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-array to intercommunicate. Any hosts external to this
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-new array must be explicitly added to this file
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-Refer to the Greenplum Admin support guide which is
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-located in the /usr/local/greenplum-db-6.17.0/docs directory
20210720:16:31:36:015437 gpinitsystem:gp-master:gpadmin-[INFO]:-------------------------------------------------------
```

登录数据库

```bash
[gpadmin@gp-master ~]$ psql -d postgres
psql (9.4.24)
Type "help" for help.

postgres=# \l
                               List of databases
   Name    |  Owner  | Encoding |  Collate   |   Ctype    |  Access privileges  
-----------+---------+----------+------------+------------+---------------------
 postgres  | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | =c/gpadmin         +
           |         |          |            |            | gpadmin=CTc/gpadmin
 template1 | gpadmin | UTF8     | en_US.utf8 | en_US.utf8 | =c/gpadmin         +
           |         |          |            |            | gpadmin=CTc/gpadmin
(3 rows)

postgres=# select version();
                                                                                                      version                                                    
                                                  
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------
 PostgreSQL 9.4.24 (Greenplum Database 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (GC
C) 6.4.0, 64-bit compiled on Jul 12 2021 10:13:35
(1 row)
```

修改密码

```bash
postgres=# \du
                                                          List of roles
 Role name |                                                Attributes                                                | Member of 
-----------+----------------------------------------------------------------------------------------------------------+-----------
 gpadmin   | Superuser, Create role, Create DB, Ext gpfdist Table, Wri Ext gpfdist Table, Ext http Table, Replication | {}

postgres=# alter user gpadmin with password 'gpadmin';
ALTER ROLE
```

远程登录

```bash
[gpadmin@gp-master ~]$ echo 'host     all         all             0.0.0.0/0     md5' >> /data/master/gpseg-1/pg_hba.conf 
[gpadmin@gp-master ~]$ gpstop -u
```

## 增加节点

### standby 节点

完成下面两步初始化

1. 完成初始化节点
2. 安装greenplum

**集群其他节点增加主机名解析**

```
echo '192.168.77.133 gp-standby' >> /etc/hosts
```

**免密**

```
for NODE in gp-standby; do
  echo "--- $NODE ---"
  sshpass -p gpadmin ssh gpadmin@${NODE} "ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
"
  sshpass -p gpadmin ssh-copy-id -o "StrictHostKeyChecking no" -i /home/gpadmin/.ssh/id_rsa.pub gpadmin@${NODE}
done

gpssh-exkeys  -h gp-standby
```

**设置环境变量**

```
su - gpadmin 
echo "source/usr/local/greenplum-db-6.17.0/greenplum_path.sh >> ~/.bash_profile"
. ~/.bash_profile
```

**在standby上创建存储区域**

```
mkdir -p /data/master
chown -R gpadmin:gpadmin /data/master
```

**在Master节点添加Standby**

```bash
$ gpinitstandby -s gp-standby
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Validating environment and parameters for standby initialization...
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Checking for data directory /data/master/gpseg-1 on gp-standby
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:------------------------------------------------------
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum standby master initialization parameters
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:------------------------------------------------------
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum master hostname               = gp-master
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum master data directory         = /data/master/gpseg-1
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum master port                   = 5432
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum standby master hostname       = gp-standby
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum standby master port           = 5432
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum standby master data directory = /data/master/gpseg-1
20210804:16:55:38:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Greenplum update system catalog         = On
Do you want to continue with standby master initialization? Yy|Nn (default=N):
> y
20210804:16:55:41:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Syncing Greenplum Database extensions to standby
20210804:16:55:42:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-The packages on gp-standby are consistent.
20210804:16:55:42:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Adding standby master to catalog...
20210804:16:55:42:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Database catalog updated successfully.
20210804:16:55:42:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Updating pg_hba.conf file...
20210804:16:55:43:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-pg_hba.conf files updated successfully.
20210804:16:56:17:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Starting standby master
20210804:16:56:17:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Checking if standby master is running on host: gp-standby  in directory: /data/master/gpseg-1
20210804:16:56:19:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Cleaning up pg_hba.conf backup files...
20210804:16:56:19:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Backup files of pg_hba.conf cleaned up successfully.
20210804:16:56:19:014679 gpinitstandby:gp-master:gpadmin-[INFO]:-Successfully created standby master on gp-standby
```

**查看状态**

```
[gpadmin@gp-master ~]$ gpstate -s
20210804:16:57:13:014832 gpstate:gp-master:gpadmin-[INFO]:-Starting gpstate with args: -s
20210804:16:57:13:014832 gpstate:gp-master:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source'
20210804:16:57:13:014832 gpstate:gp-master:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 9.4.24 (Greenplum Database 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (GCC) 6.4.0, 64-bit compiled on Jul 12 2021 10:13:35'
20210804:16:57:13:014832 gpstate:gp-master:gpadmin-[INFO]:-Obtaining Segment details from master...
20210804:16:57:13:014832 gpstate:gp-master:gpadmin-[INFO]:-Gathering data from segments...
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:--Master Configuration & Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Master host                    = gp-master
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Master postgres process ID     = 10227
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Master data directory          = /data/master/gpseg-1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Master port                    = 5432
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Master current role            = dispatch
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Greenplum initsystem version   = 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Greenplum current version      = PostgreSQL 9.4.24 (Greenplum Database 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (GCC) 6.4.0, 64-bit compiled on Jul 12 2021 10:13:35
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Postgres version               = 9.4.24
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Master standby                 = gp-standby
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Standby master state           = Standby host passive
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-Segment Instance Status Report
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/mirror/gpseg0
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 7000
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Synchronized
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 10104
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Database status                   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/primary/gpseg0
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 6000
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Streaming
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Replication Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Sent Location                 = 1/5CDDB530
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Flush Location                = 1/5CDDB530
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Replay Location               = 1/5CDDB530
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 22493
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Segment status                    = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/mirror/gpseg1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 7001
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Synchronized
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 10103
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Database status                   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/primary/gpseg1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 6001
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Streaming
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Replication Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Sent Location                 = 1/63479A90
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Flush Location                = 1/63479A90
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Replay Location               = 1/63479A90
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 22492
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Segment status                    = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/mirror/gpseg2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 7000
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Synchronized
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 10115
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Database status                   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/primary/gpseg2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 6000
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Streaming
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Replication Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Sent Location                 = 1/5CEFDAF8
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Flush Location                = 1/5CEFDAF8
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Replay Location               = 1/5CEFDAF8
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 22278
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Segment status                    = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node1
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/mirror/gpseg3
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 7001
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Synchronized
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 10116
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Database status                   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-----------------------------------------------------
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Segment Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Hostname                          = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Address                           = gp-node2
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Datadir                           = /data/primary/gpseg3
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Port                              = 6001
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Mirroring Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Current role                      = Mirror
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Preferred role                    = Primary
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Mirror status                     = Streaming
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Replication Info
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Sent Location                 = 1/5F5F6198
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Flush Location                = 1/5F5F6198
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      WAL Replay Location               = 1/5F5F6198
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-   Status
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      PID                               = 22279
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Configuration reports status as   = Up
20210804:16:57:14:014832 gpstate:gp-master:gpadmin-[INFO]:-      Segment status                    = Up
[gpadmin@gp-master ~]$ 
```



```bash
[gpadmin@gp-master ~]$ psql postgres
psql (9.4.24)
Type "help" for help.

postgres=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port |  hostname  |  address   |       datadir        
------+---------+------+----------------+------+--------+------+------------+------------+----------------------
    1 |      -1 | p    | p              | n    | u      | 5432 | gp-master  | gp-master  | /data/master/gpseg-1
    8 |       2 | p    | m              | s    | u      | 7000 | gp-node1   | gp-node1   | /data/mirror/gpseg2
    4 |       2 | m    | p              | s    | u      | 6000 | gp-node2   | gp-node2   | /data/primary/gpseg2
    9 |       3 | p    | m              | s    | u      | 7001 | gp-node1   | gp-node1   | /data/mirror/gpseg3
    5 |       3 | m    | p              | s    | u      | 6001 | gp-node2   | gp-node2   | /data/primary/gpseg3
    6 |       0 | p    | m              | s    | u      | 7000 | gp-node2   | gp-node2   | /data/mirror/gpseg0
    2 |       0 | m    | p              | s    | u      | 6000 | gp-node1   | gp-node1   | /data/primary/gpseg0
    7 |       1 | p    | m              | s    | u      | 7001 | gp-node2   | gp-node2   | /data/mirror/gpseg1
    3 |       1 | m    | p              | s    | u      | 6001 | gp-node1   | gp-node1   | /data/primary/gpseg1
   10 |      -1 | m    | m              | s    | u      | 5432 | gp-standby | gp-standby | /data/master/gpseg-1
(10 rows)
```

### semgent 节点

完成下面两步初始化

1. 完成初始化节点
2. 安装greenplum

**集群其他节点增加主机名解析**

```
echo -e '192.168.77.134 gp-node3\n192.168.77.135 gp-node4' >> /etc/hosts
```

**免密**

```
for NODE in gp-node3 gp-node4; do
  echo "--- $NODE ---"
  sshpass -p gpadmin ssh gpadmin@${NODE} "ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
"
  sshpass -p gpadmin ssh-copy-id -o "StrictHostKeyChecking no" -i /home/gpadmin/.ssh/id_rsa.pub gpadmin@${NODE}
done

gpssh-exkeys -h gp-node3 -h gp-node4
```

**设置环境变量**

```
su - gpadmin 
echo "source/usr/local/greenplum-db-6.17.0/greenplum_path.sh >> ~/.bash_profile"
. ~/.bash_profile
```

**在节点上创建存储区域**

```
mkdir -p /data/{primary,mirror}
chown -R gpadmin:gpadmin /data/
```

**在Master节点添加Semgent**

```
echo -e "gp-node3\ngp-node4" >> expend_seg_hosts
```

执行命令，生成参数文件

```bash
[gpadmin@gp-master ~]$ gpexpand -f expend_seg_hosts
20210805:15:12:59:080109 gpexpand:gp-master:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source'
20210805:15:12:59:080109 gpexpand:gp-master:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 9.4.24 (Greenplum Database 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (GCC) 6.4.0, 64-bit compiled on Jul 12 2021 10:13:35'
20210805:15:12:59:080109 gpexpand:gp-master:gpadmin-[INFO]:-Querying gpexpand schema for current expansion state

System Expansion is used to add segments to an existing GPDB array.
gpexpand did not detect a System Expansion that is in progress.

Before initiating a System Expansion, you need to provision and burn-in
the new hardware.  Please be sure to run gpcheckperf to make sure the
new hardware is working properly.

Please refer to the Admin Guide for more information.

Would you like to initiate a new System Expansion Yy|Nn (default=N):
> y

You must now specify a mirroring strategy for the new hosts.  Spread mirroring places
a given hosts mirrored segments each on a separate host.  You must be 
adding more hosts than the number of segments per host to use this. 
Grouped mirroring places all of a given hosts segments on a single 
mirrored host.  You must be adding at least 2 hosts in order to use this.



What type of mirroring strategy would you like?
 spread|grouped (default=grouped):
> 

    By default, new hosts are configured with the same number of primary
    segments as existing hosts.  Optionally, you can increase the number
    of segments per host.

    For example, if existing hosts have two primary segments, entering a value
    of 2 will initialize two additional segments on existing hosts, and four
    segments on new hosts.  In addition, mirror segments will be added for
    these new primary segments if mirroring is enabled.
    

How many new primary segments per host do you want to add? (default=0):
> 1
Enter new primary data directory 1:
> /data/primary
Enter new mirror data directory 1:
> /data/mirror

Generating configuration file...

20210805:15:13:36:080109 gpexpand:gp-master:gpadmin-[INFO]:-Generating input file...

Input configuration file was written to 'gpexpand_inputfile_20210805_151336'.

Please review the file and make sure that it is correct then re-run
with: gpexpand -i gpexpand_inputfile_20210805_151336
                
20210805:15:13:36:080109 gpexpand:gp-master:gpadmin-[INFO]:-Exiting...
```

参数内容如下

```bash
[gpadmin@gp-master ~]$  cat gpexpand_inputfile_20210805_151336
gp-node3|gp-node3|7000|/data/primary/gpseg4|11|4|p
gp-node4|gp-node4|6000|/data/mirror/gpseg4|17|4|m
gp-node3|gp-node3|7001|/data/primary/gpseg5|12|5|p
gp-node4|gp-node4|6001|/data/mirror/gpseg5|18|5|m
gp-node4|gp-node4|7000|/data/primary/gpseg6|13|6|p
gp-node3|gp-node3|6000|/data/mirror/gpseg6|15|6|m
gp-node4|gp-node4|7001|/data/primary/gpseg7|14|7|p
gp-node3|gp-node3|6001|/data/mirror/gpseg7|16|7|m
gp-node1|gp-node1|7002|/data/primary/gpseg8|19|8|p
gp-node3|gp-node3|6002|/data/mirror/gpseg8|25|8|m
gp-node2|gp-node2|7002|/data/primary/gpseg9|20|9|p
gp-node1|gp-node1|6002|/data/mirror/gpseg9|23|9|m
gp-node3|gp-node3|7002|/data/primary/gpseg10|21|10|p
gp-node4|gp-node4|6002|/data/mirror/gpseg10|26|10|m
gp-node4|gp-node4|7002|/data/primary/gpseg11|22|11|p
gp-node2|gp-node2|6002|/data/mirror/gpseg11|24|11|m
```

如果存放的目录有所改变，可以手动去修改此文件，将该计算节点存放在自己想要放在的位置

利用参数文件执行拓展命令

```bash
gpexpand -i gpexpand_inputfile_20210805_151336
20210805:15:18:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source'
20210805:15:18:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 9.4.24 (Greenplum Database 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (GCC) 6.4.0, 64-bit compiled on Jul 12 2021 10:13:35'
20210805:15:18:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-Querying gpexpand schema for current expansion state
20210805:15:18:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-Heap checksum setting consistent across cluster
20210805:15:18:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-Syncing Greenplum Database extensions
20210805:15:18:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-The packages on gp-node2 are consistent.
20210805:15:18:40:080742 gpexpand:gp-master:gpadmin-[INFO]:-The packages on gp-node3 are consistent.
20210805:15:18:40:080742 gpexpand:gp-master:gpadmin-[INFO]:-The packages on gp-node1 are consistent.
20210805:15:18:41:080742 gpexpand:gp-master:gpadmin-[INFO]:-The packages on gp-node4 are consistent.
20210805:15:18:41:080742 gpexpand:gp-master:gpadmin-[INFO]:-Locking catalog
20210805:15:18:41:080742 gpexpand:gp-master:gpadmin-[INFO]:-Locked catalog
20210805:15:18:42:080742 gpexpand:gp-master:gpadmin-[INFO]:-Creating segment template
20210805:15:18:44:080742 gpexpand:gp-master:gpadmin-[INFO]:-Copying postgresql.conf from existing segment into template
20210805:15:18:44:080742 gpexpand:gp-master:gpadmin-[INFO]:-Copying pg_hba.conf from existing segment into template
20210805:15:18:45:080742 gpexpand:gp-master:gpadmin-[INFO]:-Creating schema tar file
20210805:15:19:04:080742 gpexpand:gp-master:gpadmin-[INFO]:-Distributing template tar file to new hosts
20210805:15:19:27:080742 gpexpand:gp-master:gpadmin-[INFO]:-Configuring new segments (primary)
20210805:15:19:27:080742 gpexpand:gp-master:gpadmin-[INFO]:-{'gp-node2': '/data/primary/gpseg9:7002:true:false:20:9::-1:', 'gp-node3': '/data/primary/gpseg4:7000:true:false:11:4::-1:,/data/primary/gpseg5:7001:true:false:12:5::-1:,/data/primary/gpseg10:7002:true:false:21:10::-1:', 'gp-node1': '/data/primary/gpseg8:7002:true:false:19:8::-1:', 'gp-node4': '/data/primary/gpseg6:7000:true:false:13:6::-1:,/data/primary/gpseg7:7001:true:false:14:7::-1:,/data/primary/gpseg11:7002:true:false:22:11::-1:'}
20210805:15:22:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-Cleaning up temporary template files
20210805:15:22:39:080742 gpexpand:gp-master:gpadmin-[INFO]:-Cleaning up databases in new segments.
20210805:15:22:43:080742 gpexpand:gp-master:gpadmin-[INFO]:-Unlocking catalog
20210805:15:22:43:080742 gpexpand:gp-master:gpadmin-[INFO]:-Unlocked catalog
20210805:15:22:43:080742 gpexpand:gp-master:gpadmin-[INFO]:-Creating expansion schema
...
```

查看新添加的状态
此时去数据库里查看相关节点信息，可以看到已经增加了两个节点了。

```bash
[gpadmin@gp-master ~]$ psql postgres
psql (9.4.24)
Type "help" for help.

postgres=# 
postgres=# SELECT * from gp_segment_configuration ;
 dbid | content | role | preferred_role | mode | status | port |  hostname  |  address   |        datadir        
------+---------+------+----------------+------+--------+------+------------+------------+-----------------------
    1 |      -1 | p    | p              | n    | u      | 5432 | gp-master  | gp-master  | /data/master/gpseg-1
   10 |      -1 | m    | m              | s    | u      | 5432 | gp-standby | gp-standby | /data/master/gpseg-1
   22 |      11 | p    | p              | s    | u      | 7002 | gp-node4   | gp-node4   | /data/primary/gpseg11
   24 |      11 | m    | m              | s    | u      | 6002 | gp-node2   | gp-node2   | /data/mirror/gpseg11
    2 |       0 | p    | p              | s    | u      | 6000 | gp-node1   | gp-node1   | /data/primary/gpseg0
    6 |       0 | m    | m              | s    | u      | 7000 | gp-node2   | gp-node2   | /data/mirror/gpseg0
   13 |       6 | p    | p              | s    | u      | 7000 | gp-node4   | gp-node4   | /data/primary/gpseg6
   15 |       6 | m    | m              | s    | u      | 6000 | gp-node3   | gp-node3   | /data/mirror/gpseg6
   14 |       7 | p    | p              | s    | u      | 7001 | gp-node4   | gp-node4   | /data/primary/gpseg7
   16 |       7 | m    | m              | s    | u      | 6001 | gp-node3   | gp-node3   | /data/mirror/gpseg7
    3 |       1 | p    | p              | s    | u      | 6001 | gp-node1   | gp-node1   | /data/primary/gpseg1
    7 |       1 | m    | m              | s    | u      | 7001 | gp-node2   | gp-node2   | /data/mirror/gpseg1
   19 |       8 | p    | p              | s    | u      | 7002 | gp-node1   | gp-node1   | /data/primary/gpseg8
   25 |       8 | m    | m              | s    | u      | 6002 | gp-node3   | gp-node3   | /data/mirror/gpseg8
   20 |       9 | p    | p              | s    | u      | 7002 | gp-node2   | gp-node2   | /data/primary/gpseg9
   23 |       9 | m    | m              | s    | u      | 6002 | gp-node1   | gp-node1   | /data/mirror/gpseg9
    4 |       2 | p    | p              | s    | u      | 6000 | gp-node2   | gp-node2   | /data/primary/gpseg2
    8 |       2 | m    | m              | s    | u      | 7000 | gp-node1   | gp-node1   | /data/mirror/gpseg2
    5 |       3 | p    | p              | s    | u      | 6001 | gp-node2   | gp-node2   | /data/primary/gpseg3
    9 |       3 | m    | m              | s    | u      | 7001 | gp-node1   | gp-node1   | /data/mirror/gpseg3
   21 |      10 | p    | p              | s    | u      | 7002 | gp-node3   | gp-node3   | /data/primary/gpseg10
   26 |      10 | m    | m              | s    | u      | 6002 | gp-node4   | gp-node4   | /data/mirror/gpseg10
   12 |       5 | p    | p              | s    | u      | 7001 | gp-node3   | gp-node3   | /data/primary/gpseg5
   18 |       5 | m    | m              | s    | u      | 6001 | gp-node4   | gp-node4   | /data/mirror/gpseg5
   11 |       4 | p    | p              | s    | u      | 7000 | gp-node3   | gp-node3   | /data/primary/gpseg4
   17 |       4 | m    | m              | s    | u      | 6000 | gp-node4   | gp-node4   | /data/mirror/gpseg4
(26 rows)
```

执行重分布命令将数据重分布

```bash
 gpexpand -a -d 60:00:00
20210805:16:48:13:089727 gpexpand:gp-master:gpadmin-[INFO]:-local Greenplum Version: 'postgres (Greenplum Database) 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source'
20210805:16:48:13:089727 gpexpand:gp-master:gpadmin-[INFO]:-master Greenplum Version: 'PostgreSQL 9.4.24 (Greenplum Database 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source) on x86_64-unknown-linux-gnu, compiled by gcc (GCC) 6.4.0, 64-bit compiled on Jul 12 2021 10:13:35'
20210805:16:48:13:089727 gpexpand:gp-master:gpadmin-[WARNING]:-The last gpexpand setup did not complete successfully.
20210805:16:48:13:089727 gpexpand:gp-master:gpadmin-[WARNING]:-But you can not rollback to the original state, for new segments have been online.
20210805:16:48:13:089727 gpexpand:gp-master:gpadmin-[WARNING]:-So retry the failing work again in gpexpand setup.
20210805:16:48:14:089727 gpexpand:gp-master:gpadmin-[INFO]:-Creating expansion schema
20210805:16:48:16:089727 gpexpand:gp-master:gpadmin-[INFO]:-Populating gpexpand.status_detail with data from database template1
20210805:16:48:17:089727 gpexpand:gp-master:gpadmin-[INFO]:-Populating gpexpand.status_detail with data from database postgres
20210805:16:48:18:089727 gpexpand:gp-master:gpadmin-[INFO]:-Populating gpexpand.status_detail with data from database sample
20210805:16:48:18:089727 gpexpand:gp-master:gpadmin-[INFO]:-Populating gpexpand.status_detail with data from database gpperfmon
20210805:16:48:20:089727 gpexpand:gp-master:gpadmin-[INFO]:-Starting new mirror segment synchronization
20210805:16:48:21:089727 gpexpand:gp-master:gpadmin-[INFO]:-************************************************
20210805:16:48:21:089727 gpexpand:gp-master:gpadmin-[INFO]:-Initialization of the system expansion complete.
20210805:16:48:21:089727 gpexpand:gp-master:gpadmin-[INFO]:-To begin table expansion onto the new segments
20210805:16:48:21:089727 gpexpand:gp-master:gpadmin-[INFO]:-rerun gpexpand
20210805:16:48:21:089727 gpexpand:gp-master:gpadmin-[INFO]:-************************************************
20210805:16:48:21:089727 gpexpand:gp-master:gpadmin-[INFO]:-Exiting...
```

## 移除节点

备份数据库

```bash
wget https://github.com/greenplum-db/gpbackup/releases/download/1.20.4/gpbackup
chmod +x gpbackup
[gpadmin@gp-master ~]$ ./gpbackup --dbname sample --with-stats --backup-dir /home/gpadmin/backup
20210805:17:12:21 gpbackup:gpadmin:gp-master:091051-[INFO]:-gpbackup version = 1.20.4
20210805:17:12:22 gpbackup:gpadmin:gp-master:091051-[INFO]:-Greenplum Database Version = 6.17.0 build commit:9b887d27cef94c03ce3a3e63e4f6eefb9204631b Open Source
20210805:17:12:22 gpbackup:gpadmin:gp-master:091051-[INFO]:-Starting backup of database sample
20210805:17:12:23 gpbackup:gpadmin:gp-master:091051-[INFO]:-Backup Timestamp = 20210805171222
20210805:17:12:23 gpbackup:gpadmin:gp-master:091051-[INFO]:-Backup Database = sample
20210805:17:12:23 gpbackup:gpadmin:gp-master:091051-[INFO]:-Gathering table state information
20210805:17:12:23 gpbackup:gpadmin:gp-master:091051-[INFO]:-Acquiring ACCESS SHARE locks on tables
Locks acquired:  1 / 1 [================================================================] 100.00% 0s
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Gathering additional table metadata
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Getting partition definitions
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Getting storage information
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Getting child partitions with altered schema
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Metadata will be written to /home/gpadmin/backup/gpseg-1/backups/20210805/20210805171222/gpbackup_20210805171222_metadata.sql
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Writing global database metadata
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Global database metadata backup complete
20210805:17:12:24 gpbackup:gpadmin:gp-master:091051-[INFO]:-Writing pre-data metadata
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Pre-data metadata metadata backup complete
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Writing post-data metadata
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Post-data metadata backup complete
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Writing data to file
Tables backed up:  1 / 1 [==============================================================] 100.00% 0s
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Data backup complete
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Writing query planner statistics to /home/gpadmin/backup/gpseg-1/backups/20210805/20210805171222/gpbackup_20210805171222_statistics.sql
20210805:17:12:25 gpbackup:gpadmin:gp-master:091051-[INFO]:-Query planner statistics backup complete
20210805:17:12:26 gpbackup:gpadmin:gp-master:091051-[INFO]:-Found neither /usr/local/greenplum-db-6.17.0/bin/gp_email_contacts.yaml nor /home/gpadmin/gp_email_contacts.yaml
20210805:17:12:26 gpbackup:gpadmin:gp-master:091051-[INFO]:-Email containing gpbackup report /home/gpadmin/backup/gpseg-1/backups/20210805/20210805171222/gpbackup_20210805171222_report will not be sent
20210805:17:12:26 gpbackup:gpadmin:gp-master:091051-[INFO]:-Backup completed successfully
```

关闭数据库

```
gpstop -a
```

启动数据库到master模式，进入管理模式手动删除节点信息：

```
gpstart -m
PGOPTIONS="-c gp_session_role=utility" psql -d postgres
```

```
postgres=# select * from gp_segment_configuration where hostname in ('gp-node3', 'gp-node4') order by dbid;    
 dbid | content | role | preferred_role | mode | status | port | hostname | address  |        datadir        
------+---------+------+----------------+------+--------+------+----------+----------+-----------------------
   11 |       4 | p    | p              | s    | u      | 7000 | gp-node3 | gp-node3 | /data/primary/gpseg4
   12 |       5 | p    | p              | s    | u      | 7001 | gp-node3 | gp-node3 | /data/primary/gpseg5
   13 |       6 | p    | p              | s    | u      | 7000 | gp-node4 | gp-node4 | /data/primary/gpseg6
   14 |       7 | p    | p              | s    | u      | 7001 | gp-node4 | gp-node4 | /data/primary/gpseg7
   15 |       6 | m    | m              | s    | u      | 6000 | gp-node3 | gp-node3 | /data/mirror/gpseg6
   16 |       7 | m    | m              | s    | u      | 6001 | gp-node3 | gp-node3 | /data/mirror/gpseg7
   17 |       4 | m    | m              | s    | u      | 6000 | gp-node4 | gp-node4 | /data/mirror/gpseg4
   18 |       5 | m    | m              | s    | u      | 6001 | gp-node4 | gp-node4 | /data/mirror/gpseg5
   21 |      10 | p    | p              | s    | u      | 7002 | gp-node3 | gp-node3 | /data/primary/gpseg10
   22 |      11 | p    | p              | s    | u      | 7002 | gp-node4 | gp-node4 | /data/primary/gpseg11
   25 |       8 | m    | m              | s    | u      | 6002 | gp-node3 | gp-node3 | /data/mirror/gpseg8
   26 |      10 | m    | m              | s    | u      | 6002 | gp-node4 | gp-node4 | /data/mirror/gpseg10
   
   postgres=# select * from gp_id;
  gpname   | numsegments | dbid | content 
-----------+-------------+------+---------
 Greenplum |          -1 |   -1 |      -1
(1 row)
```

```
postgres=# set allow_system_table_mods=true; 
SET
postgres=# delete from gp_segment_configuration where content in(select dbid from gp_segment_configuration where hostname in ('gp-node3', 'gp-node4') order by dbid);
DELETE 12

postgres=# select * from gp_segment_configuration order by dbid;           
 dbid | content | role | preferred_role | mode | status | port |  hostname  |  address   |       datadir        
------+---------+------+----------------+------+--------+------+------------+------------+----------------------
    1 |      -1 | p    | p              | n    | u      | 5432 | gp-master  | gp-master  | /data/master/gpseg-1
    2 |       0 | p    | p              | s    | u      | 6000 | gp-node1   | gp-node1   | /data/primary/gpseg0
    3 |       1 | p    | p              | s    | u      | 6001 | gp-node1   | gp-node1   | /data/primary/gpseg1
    4 |       2 | p    | p              | s    | u      | 6000 | gp-node2   | gp-node2   | /data/primary/gpseg2
    5 |       3 | p    | p              | s    | u      | 6001 | gp-node2   | gp-node2   | /data/primary/gpseg3
    6 |       0 | m    | m              | s    | u      | 7000 | gp-node2   | gp-node2   | /data/mirror/gpseg0
    7 |       1 | m    | m              | s    | u      | 7001 | gp-node2   | gp-node2   | /data/mirror/gpseg1
    8 |       2 | m    | m              | s    | u      | 7000 | gp-node1   | gp-node1   | /data/mirror/gpseg2
    9 |       3 | m    | m              | s    | u      | 7001 | gp-node1   | gp-node1   | /data/mirror/gpseg3
   10 |      -1 | m    | m              | s    | u      | 5432 | gp-standby | gp-standby | /data/master/gpseg-1
   20 |       9 | p    | p              | s    | u      | 7002 | gp-node2   | gp-node2   | /data/primary/gpseg9
   23 |       9 | m    | m              | s    | u      | 6002 | gp-node1   | gp-node1   | /data/mirror/gpseg9
(12 rows)
```

## 其他操作

**时区**

```
gpconfig -s TimeZon
gpconfig -c TimeZone -v 'Asia/Shanghai'
```

**环境变量**

```
PGAPPNAME:此环境变量默认为psql,表示连接的app名称,将会在activity 视图中显示
PGDATABASE:默认连接的数据库名称 
PGHOST:GP Master主机名
PGHOSTADDR:GP Master IP地址,设置此可以不需要进行DNS解析
PGPASSWORD: PG的默认密码,可以指定此环境变量,这样就不需要输入密码
PGPASSFILE: PG登录的默认密码文件,默认为~/.pgpass 配置此文件,可以不输入密码登录数据库
PGOPTIONS: 为master设置额外的配置参数
PGPORT:默认为5432 数据库端口号
PGUSER:GP连接用户名
PGDATESTYLE:为会话设置默认的时间格式,等同于SET datestype To...
PGTZ:为会话设置默认的时区,等同于SET timezone TO ...
PGCLIENTENCODING:为会话设置默认的客户端字符集,等同于SET client_encoding TO...
```

**安装扩展模块**

```bash
postgres=# CREATE EXTENSION dblink;
```

**简单测试**

```bash
postgres=# select datname,datdba,encoding,datacl from pg_database; 
  datname  | datdba | encoding |              datacl              
-----------+--------+----------+----------------------------------
 template1 |     10 |        6 | {=c/gpadmin,gpadmin=CTc/gpadmin}
 template0 |     10 |        6 | {=c/gpadmin,gpadmin=CTc/gpadmin}
 postgres  |     10 |        6 | 
(3 rows)

postgres=# create database sample;
CREATE DATABASE
postgres=# \c sample;
Password for user gpadmin: 
You are now connected to database "sample" as user "gpadmin".
sample=# select datname,datdba,encoding,datacl from pg_database;
  datname  | datdba | encoding |              datacl              
-----------+--------+----------+----------------------------------
 template1 |     10 |        6 | {=c/gpadmin,gpadmin=CTc/gpadmin}
 template0 |     10 |        6 | {=c/gpadmin,gpadmin=CTc/gpadmin}
 postgres  |     10 |        6 | 
 sample    |     10 |        6 | 
(4 rows)

sample=# create table t1(c1 int) distributed by (c1);
CREATE TABLE
sample=# insert into t1 select * from generate_series(1,100000);
INSERT 0 100000
sample=# select count(*) from t1;
 count  
--------
 100000
(1 row)

# 键值分布
sample=# SELECT gp_segment_id, count(*)  FROM t1 GROUP BY gp_segment_id;
 gp_segment_id | count 
---------------+-------
             1 | 25018
             0 | 25111
             3 | 24936
             2 | 24935
```

**常用命令**

```bash
# 启动
gpstart

# 停止
gpstop
gpstop -M fast

# 重启
gpstop -r

# 重新加载配置文件
gpstop -u

# 以维护模式启动master
# 只启动Master来执行维护或者管理任务而不影响Segment上的数据。
# 例如，可以用维护模式连接到一个只在Master实例上的数据库并且编辑系统目录设置。 更多有关系统目录表的信息请见Greenplum Database Reference Guide
gpstart -m #维护模式打开数据库
PGOPTIONS=’-c gp_session_role=utility’ psql postgres #以维护模式连接到master
gpstop -mr #停止维护模式,以正常模式重启数据库

# 查看gp配置参数
psql -c 'show all'
# 或者
gpconfig -s <参数名>

# 修改gp参数
# 设置max_connections,在master上设置为10, 在segment上设置为100
# -v指定所有节点,包括master,standby,segment
# -m指定master节点
# –master-only指定只修改master节点
# -r移除参数的配置
# -l显示所有可配置的参数
gpconfig -c max_connections -v 100 -m 10

# 查看gp状态
gpstate #查看简要信息
gpstate -s #查看详细信息
gpstate -m #查看镜像配置
gpstate -c #查看镜像的映射关系
gpstate -f #查看standby状态

# 恢复
gprecoverseg # 恢复失败的 segment 实例
gprecoverseg -F -v # 全量恢复
gprecoverseg -r # 还原所有Segment的角色

关闭集群
gpstop -a -M fast

以restricted方式启动数据库
gpstart -a -R 

开始修复故障节点
gprecoverseg -a

查看修复状态 
gpstate -m

重启greenplum集群
gpstop -a -r
```

**sql语句**
```sql
# 查看所有segment节点信息
select * from gp_segment_configuration;

# 查看节点主机磁盘空闲空间
SELECT * FROM gp_toolkit.gp_disk_free ORDER BY dfsegment;

# 查看数据库的使用空间
SELECT * FROM gp_toolkit.gp_size_of_database ORDER BY sodddatname;

# 查看表的磁盘使用空间
SELECT relname AS name, sotdsize AS size, sotdtoastsize 
   AS toast, sotdadditionalsize AS other 
   FROM gp_toolkit.gp_size_of_table_disk as sotd, pg_class 
   WHERE sotd.sotdoid=pg_class.oid ORDER BY relname;
   
# 查看索引的使用空间
SELECT soisize, relname as indexname
   FROM pg_class, gp_toolkit.gp_size_of_index
   WHERE pg_class.oid=gp_size_of_index.soioid 
   AND pg_class.relkind='i';
   
# 查看表的分布键
\d+ table_name

# 查看表的数据分布
SELECT gp_segment_id, count(*) 
   FROM table_name GROUP BY gp_segment_id;
   
# 查看当前正在运行的会话
select pid,usename,datname,waiting_reason,query,client_addr from pg_stat_activity where state='active';

# 查看当前等待的会话
select pid,usename,datname,query,client_addr,waiting_reason from pg_stat_activity where waiting ;

# 杀死会话

select pg_cancel_backedn(111);

select pg_terminate_backend(pid) from pg_stat_activity where ;   -- 必须要接条件,而且最好先查出你需要的杀的会话.
select 'select pg_terminate_backend('||pid||');' from pg_stat_activity where    ;   --必须要接条件
select 'select pg_cancel_backend('||pid||');' from pg_stat_activity where    ;   --
```

```sql
-- 查看资源队列的配置
select * from pg_resqueue_attributes;

-- 查看资源队列的使用情况
select * from pg_resqueue_status;

-- 资源队列的视图
postgres=# \dv gp_toolkit.gp_resq*
                    List of relations
   Schema   |            Name            | Type |  Owner  
------------+----------------------------+------+---------
 gp_toolkit | gp_resq_activity           | view | gpadmin
 gp_toolkit | gp_resq_activity_by_queue  | view | gpadmin
 gp_toolkit | gp_resq_priority_backend   | view | gpadmin
 gp_toolkit | gp_resq_priority_statement | view | gpadmin
 gp_toolkit | gp_resq_role               | view | gpadmin
 gp_toolkit | gp_resqueue_status         | view | gpadmin
(6 rows)

```

```
在psql中使用\h command可以获取具体命令的语法
\h create view
```

**数据目录结构**

```
$ tree -L 1  /data/master/gpseg-1/  
/data/master/gpseg-1/
├── base
├── global
├── gpmetrics
├── gpperfmon
├── gpsegconfig_dump
├── gpssh.conf
├── internal.auto.conf
├── pg_clog
├── pg_distributedlog
├── pg_dynshmem
├── pg_hba_archive
├── pg_hba.conf
├── pg_hba.conf.old
├── pg_ident.conf
├── pg_log
├── pg_logical
├── pg_multixact
├── pg_notify
├── pg_replslot
├── pg_serial
├── pg_snapshots
├── pg_stat
├── pg_stat_tmp
├── pg_subtrans
├── pg_tblspc
├── pg_twophase
├── pg_utilitymodedtmredo
├── PG_VERSION
├── pg_xlog
├── postgresql.auto.conf
├── postgresql.conf
├── postmaster.opts
└── postmaster.pid

22 directories, 11 files
```

- base是数据目录，每个数据库在这个目录下，会有一个对应的文件夹。
- global是每一个数据库公用的数据目录。
- gpperfmon监控数据库性能时，存放监控数据的地方。
- pg_changetracking是Segment之间主备同步用到的一些原数据信息保存的地方。
- pg_clog是记录数据库事务信息的地方，保存了每一个事务id的状态，这个非常重要，不能丢失，一旦丢失，整个数据库就基本上不可用了。
- pg_log是数据库的日志信息。
- pg_twophase是二阶段提交的事务信息（关于二阶段提交的内容可参阅第7章中的介绍）
- pg_xlog是数据库重写日志保存的地方，其中每个文件固定大小为64MB，并不断重复使用。
- gp_dbid记录这个数据库的dbid以及它对应的mirror节点的dbid。


## 参考
> https://docs.greenplum.org/6-16/install_guide/platform-requirements.html
> https://gp-docs-cn.github.io/docs/
