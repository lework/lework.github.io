---
layout: post
title: 'rabbitmq集群安装'
date: '2019-12-22 20:00:00'
category: rabbitmq
tags: rabbitmq
author: lework
---
* content
{:toc}

## 软件简介

RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

AMQP，即Advanced message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。

AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。

官网： https://www.rabbitmq.com




## 集群概述

通过 Erlang 的分布式特性（通过 magic cookie 认证节点）进行 RabbitMQ 集群，各 RabbitMQ 服务为对等节点，即每个节点都提供服务给客户端连接，进行消息发送与接收。

这些节点通过 RabbitMQ HA 队列（镜像队列）进行消息队列结构复制。本方案中搭建 3 个节点，并且都是磁盘节点（所有节点状态保持一致，节点完全对等），只要有任何一个节点能够工作，RabbitMQ 集群对外就能提供服务。


## 环境信息

**软件版本**

| 软件     | 版本   |
| -------- | ------ |
| rabbitmq | 3.8.2  |
| erlang   | 22.1.8 |

**宿主机**
宿主机托管在vmware workstation 中的虚拟机

| System OS          | IP Address     | Kernel       | Hostname | Cpu  | Memory | Application |
| ------------------ | -------------- | ------------ | -------- | ---- | ------ | ----------- |
| CentOS    7.4.1708 | 192.168.77.130 | 5.1.11-1.el7 | rmq0     | 1C   | 1G     | rabbitmq    |
| CentOS    7.4.1708 | 192.168.77.131 | 5.1.11-1.el7 | rmq1     | 1C   | 1G     | rabbitmq    |
| CentOS    7.4.1708 | 192.168.77.132 | 5.1.11-1.el7 | rmq2     | 1C   | 1G     | rabbitmq    |


## 网络信息

集群网络: 192.168.77.0/24

集群DNS: 192.168.77.2

## 节点初始化

> 在所有节点上操作

**关闭防火墙**
```bash
systemctl stop firewalld && systemctl disable firewalld
```

**关闭网络服务**
```bash
systemctl stop NetworkManager && systemctl disable NetworkManager
```
**关闭selinux**

```bash
setenforce 0
sed -i "s#=enforcing#=disabled#g" /etc/selinux/config
```
**关闭swap**

```bash
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```
**启动同步时间**

```bash
yum install-y chrony
cp /etc/chrony.conf{,.bak}
cat > /etc/chrony.conf  <<EOF
server 0.cn.pool.ntp.org iburst
server 1.cn.pool.ntp.org iburst
server 0.asia.pool.ntp.org iburst
server 1.asia.pool.ntp.org iburst
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF

systemctl enable --now chronyd
chronyc sourcestats
```
**系统参数调整**

```bash
cat << EOF > /etc/sysctl.d/basic.conf
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_max_tw_buckets = 60000
net.ipv4.ip_local_port_range = 10240 65535
net.netfilter.nf_conntrack_max = 2310720
net.core.somaxconn = 262144
fs.file-max = 52706963
fs.nr_open = 52706963
vm.swappiness = 0
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF
sysctl --system
```

**修改系统限制**

```bash
cp /etc/security/limits.conf{,.bak}
cat >> /etc/security/limits.conf <<EOF
* - nofile 165535
* soft nofile 165535
* hard nofile 165535
root - nofile 165535
root soft nofile 165535
root hard nofile 165535
EOF
```
**设置主机名解析**

```bash
cat >> /etc/hosts <<EOF
192.168.77.130 rmq0
192.168.77.131 rmq1
192.168.77.132 rmq2
EOF
```
**添加依赖仓库**

```bash
yum install -y epel-release
sed -e 's!^mirrorlist=!#mirrorlist=!g' \
    -e 's!^#baseurl=!baseurl=!g' \
    -e 's!^metalink!#metalink!g' \
    -e 's!//download\.fedoraproject\.org/pub!//mirrors.ustc.edu.cn!g' \
    -e 's!http://mirrors\.ustc!https://mirrors.ustc!g' \
    -i /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel-testing.repo
```
## 安装部署
> 在所有集群节点上执行

**设置主机名**
各节点按照上述的节点列表的主机名称设置

```bash
hostnamectl set-hostname rmq0
```
修改完主机名称后，需退出当前shell再次进入

**安装依赖包**

```bash
yum -y install socat bash-completion
```
**安装erlang**

```bash
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v22.1.8/erlang-22.1.8-1.el7.x86_64.rpm
rpm -i erlang-22.1.8-1.el7.x86_64.rpm
```
**安装rabbitmq**

```bash
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.2/rabbitmq-server-3.8.2-1.el7.noarch.rpm
rpm -i rabbitmq-server-3.8.2-1.el7.noarch.rpm
```
**安装rabbitmqadmin**

```bash
wget https://raw.githubusercontent.com/rabbitmq/rabbitmq-management/v3.8.2/bin/rabbitmqadmin  -O /usr/sbin/rabbitmqadmin
chmod +x /usr/sbin/rabbitmqadmin
rabbitmqadmin --bash-completion > /etc/bash_completion.d/rabbitmqadmin
```
## 配置

> 下列操作在所有节点上操作

**设置cookie**

```bash
echo "rabbitmq-cluster-cookie" > /var/lib/rabbitmq/.erlang.cookie
chmod 600 /var/lib/rabbitmq/.erlang.cookie 
chown rabbitmq.rabbitmq /var/lib/rabbitmq/.erlang.cookie 
```
**设置配置文件**

```bash
cat > /etc/rabbitmq/rabbitmq-env.conf <<EOF
RABBITMQ_MNESIA_BASE=/var/lib/rabbitmq/mnesia
CONFIG_FILE=/etc/rabbitmq/rabbitmq.conf
EOF

cat > /etc/rabbitmq/rabbitmq.conf <<EOF
listeners.tcp.default = 5672

management.tcp.port = 15672
management.tcp.ip   = 0.0.0.0
management.http_log_dir = /var/log/rabbitmq/management_access
#management.load_definitions = /etc/rabbitmq/rabbitmq-definitions.json

vm_memory_high_watermark.absolute = 512MiB
vm_memory_high_watermark_paging_ratio = 0.2

cluster_name = rabbitmq-cluster

cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rmq0
cluster_formation.classic_config.nodes.2 = rabbit@rmq1
cluster_formation.classic_config.nodes.3 = rabbit@rmq2

collect_statistics_interval = 5000
log.dir = /var/log/rabbitmq
loopback_users.guest = true
EOF
```
> 本次使用了集群静态配置，不需要手动join进集群

**开启插件**

```bash
rabbitmq-plugins enable rabbitmq_management
```
**设置开启自启动并启动服务**

```bash
systemctl enable --now rabbitmq-server
```
**查看监听的端口**

```bash
ss -natup | grep LISTEN | grep beam.smp
```
## 验证

**查看rabbitmq 状态**

```bash
rabbitmqctl status

Status of node rabbit@rmq0 ...
Runtime

OS PID: 62802
OS: Linux
Uptime (seconds): 2193
RabbitMQ version: 3.8.2
Node name: rabbit@rmq0
Erlang configuration: Erlang/OTP 22 [erts-10.5.6] [source] [64-bit] [smp:1:1] [ds:1:1:10] [async-threads:64] [hipe]
Erlang processes: 419 used, 1048576 limit
Scheduler run queue: 1
Cluster heartbeat timeout (net_ticktime): 60

Plugins

Enabled plugin file: /etc/rabbitmq/enabled_plugins
Enabled plugins:

 * rabbitmq_management
 * rabbitmq_management_agent
 * rabbitmq_web_dispatch
 * amqp_client
 * cowboy
 * cowlib

Data directory

Node data directory: /var/lib/rabbitmq/mnesia/rabbit@rmq0

Config files

 * /etc/rabbitmq/rabbitmq.conf

Log file(s)

 * /var/log/rabbitmq/rabbit@rmq0.log
 * /var/log/rabbitmq/rabbit@rmq0_upgrade.log

Alarms

(none)

Memory

Calculation strategy: rss
Memory high watermark setting: 0.5369 gb, computed to: 0.5369 gb
code: 0.0255 gb (30.14 %)
other_proc: 0.0219 gb (25.92 %)
allocated_unused: 0.0193 gb (22.82 %)
other_system: 0.0115 gb (13.55 %)
other_ets: 0.0031 gb (3.64 %)
atom: 0.0015 gb (1.8 %)
plugins: 0.0011 gb (1.31 %)
mgmt_db: 0.0002 gb (0.25 %)
metrics: 0.0002 gb (0.25 %)
binary: 0.0001 gb (0.14 %)
mnesia: 0.0001 gb (0.1 %)
quorum_ets: 0.0 gb (0.05 %)
msg_index: 0.0 gb (0.03 %)
connection_other: 0.0 gb (0.0 %)
connection_channels: 0.0 gb (0.0 %)
connection_readers: 0.0 gb (0.0 %)
connection_writers: 0.0 gb (0.0 %)
queue_procs: 0.0 gb (0.0 %)
queue_slave_procs: 0.0 gb (0.0 %)
quorum_queue_procs: 0.0 gb (0.0 %)
reserved_unallocated: 0.0 gb (0.0 %)

File Descriptors

Total: 2, limit: 32671
Sockets: 0, limit: 29401

Free Disk Space

Low free disk space watermark: 0.05 gb
Free disk space: 29.5692 gb

Totals

Connection count: 0
Queue count: 0
Virtual host count: 1

Listeners

Interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Interface: [::], port: 15672, protocol: http, purpose: HTTP API
```

查看rabbitmq cluster状态

```bash
rabbitmqctl cluster_status

Cluster status of node rabbit@rmq0 ...
Basics

Cluster name: rabbitmq-cluster

Disk Nodes

rabbit@rmq0
rabbit@rmq1
rabbit@rmq2

Running Nodes

rabbit@rmq0
rabbit@rmq1
rabbit@rmq2

Versions

rabbit@rmq0: RabbitMQ 3.8.2 on Erlang 22.1.8
rabbit@rmq1: RabbitMQ 3.8.2 on Erlang 22.1.8
rabbit@rmq2: RabbitMQ 3.8.2 on Erlang 22.1.8

Alarms

(none)

Network Partitions

(none)

Listeners

Node: rabbit@rmq0, interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Node: rabbit@rmq0, interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Node: rabbit@rmq0, interface: [::], port: 15672, protocol: http, purpose: HTTP API
Node: rabbit@rmq1, interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Node: rabbit@rmq1, interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Node: rabbit@rmq1, interface: [::], port: 15672, protocol: http, purpose: HTTP API
Node: rabbit@rmq2, interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Node: rabbit@rmq2, interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Node: rabbit@rmq2, interface: [::], port: 15672, protocol: http, purpose: HTTP API

Feature flags

Flag: drop_unroutable_metric, state: enabled
Flag: empty_basic_get_metric, state: enabled
Flag: implicit_default_bindings, state: enabled
Flag: quorum_queue, state: enabled
Flag: virtual_host_metadata, state: enabled
```
集群节点都正常

**重启集群**

关闭rmq0

```bash
rabbitmqctl stop_app
rabbitmqctl shutdown
```


关闭rmq1

```bash
rabbitmqctl stop_app
rabbitmqctl shutdown
```

关闭rmq2

```bash
rabbitmqctl stop_app
rabbitmqctl shutdown
```

**重启节点**

> 启动节点时，一定要先启动最后关闭的节点

在rmq2上执行

```bash
reboot
```

等待重启完之后，在执行其他节点重启
在rmq1上执行

```bash
reboot
```

等待重启完之后，在执行其他节点重启
在rmq0上执行

```bash
reboot
```

在rmq0节点上查看集群状态
```bash
rabbitmqctl cluster_status

Cluster status of node rabbit@rmq0 ...
Basics

Cluster name: rabbitmq-cluster

Disk Nodes

rabbit@rmq0
rabbit@rmq1
rabbit@rmq2

Running Nodes

rabbit@rmq0
rabbit@rmq1
rabbit@rmq2

Versions

rabbit@rmq0: RabbitMQ 3.8.2 on Erlang 22.1.8
rabbit@rmq1: RabbitMQ 3.8.2 on Erlang 22.1.8
rabbit@rmq2: RabbitMQ 3.8.2 on Erlang 22.1.8

Alarms

(none)

Network Partitions

(none)

Listeners

Node: rabbit@rmq0, interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Node: rabbit@rmq0, interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Node: rabbit@rmq0, interface: [::], port: 15672, protocol: http, purpose: HTTP API
Node: rabbit@rmq1, interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Node: rabbit@rmq1, interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Node: rabbit@rmq1, interface: [::], port: 15672, protocol: http, purpose: HTTP API
Node: rabbit@rmq2, interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Node: rabbit@rmq2, interface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0
Node: rabbit@rmq2, interface: [::], port: 15672, protocol: http, purpose: HTTP API

Feature flags

Flag: drop_unroutable_metric, state: enabled
Flag: empty_basic_get_metric, state: enabled
Flag: implicit_default_bindings, state: enabled
Flag: quorum_queue, state: enabled
Flag: virtual_host_metadata, state: enabled


rabbitmqadmin list nodes
+-------------+------+----------+
|    name     | type | mem_used |
+-------------+------+----------+
| rabbit@rmq0 | disc | 81215488 |
| rabbit@rmq1 | disc | 81145856 |
| rabbit@rmq2 | disc | 82006016 |
```
集群节点都正常启动

**登录web页面**

http://192.168.77.130:15672  用户名admin，密码admin



## 常用操作

**添加用户**

```bash
rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin '.*' '.*' '.*'
rabbitmqctl list_users
```
**设置策略**

```bash
rabbitmqctl set_policy ha3 \
    "^ha3.*" '{"ha-mode":"exactly", "ha-params":3}' \
    --priority 0 \
    --apply-to queues \
    --vhost /
Setting policy "ha3" for pattern "^ha3.*" to "{"ha-mode":"exactly", "ha-params":3}" with priority "0" for vhost "/" …
```
**创建 Queue**

```bash
rabbitmqadmin declare queue --vhost=/ name=ha3.queue durable=true
rabbitmqctl list_queues -p /

Timeout: 60.0 seconds …
Listing queues for vhost / ...
name    messages
ha3.queue       0
```

**创建 exchange**

```bash
rabbitmqadmin declare exchange --vhost=/ name=ha.exchange type=direct durable=true
rabbitmqctl list_exchanges -p /
```
**将exchange 与queue绑定**

```bash
rabbitmqadmin declare binding --vhost=/ source=ha.exchange destination=ha3.queue routing_key=ha3.queue
rabbitmqctl list_bindings -p /
```
**使用队列发送消息**

```bash
rabbitmqadmin publish routing_key=ha3.queue payload="just for queue"
```
**消费队列消息**

```bash
rabbitmqadmin get queue=ha3.queue ackmode=ack_requeue_false
+-------------+----------+---------------+----------------+---------------+------------------+------------+-------------+
| routing_key | exchange | message_count |    payload     | payload_bytes | payload_encoding | properties | redelivered |
+-------------+----------+---------------+----------------+---------------+------------------+------------+-------------+
| ha3.queue   |          | 0             | just for queue | 14            | string           |            | False       |
+-------------+----------+---------------+----------------+---------------+------------------+------------+-------------+

rabbitmqadmin get queue=ha3.queue
No items
```

**使用路由发送消息**

```bash
rabbitmqadmin publish exchange=ha.exchange routing_key="ha3.queue" payload="hello, world"
rabbitmqadmin get queue="ha3.queue"
+-------------+-------------+---------------+--------------+---------------+------------------+------------+-------------+
| routing_key |  exchange   | message_count |   payload    | payload_bytes | payload_encoding | properties | redelivered |
+-------------+-------------+---------------+--------------+---------------+------------------+------------+-------------+
| ha3.queue   | ha.exchange | 0             | hello, world | 12            | string           |            | False       |
+-------------+-------------+---------------+--------------+---------------+------------------+------------+-------------+
```

**消费消息**

```bash
rabbitmqadmin get queue=ha3.queue ackmode=ack_requeue_false
+-------------+-------------+---------------+--------------+---------------+------------------+------------+-------------+
| routing_key |  exchange   | message_count |   payload    | payload_bytes | payload_encoding | properties | redelivered |
+-------------+-------------+---------------+--------------+---------------+------------------+------------+-------------+
| ha3.queue   | ha.exchange | 0             | hello, world | 12            | string           |            | True        |
+-------------+-------------+---------------+--------------+---------------+------------------+------------+-------------+
rabbitmqadmin get queue=ha3.queue
```
