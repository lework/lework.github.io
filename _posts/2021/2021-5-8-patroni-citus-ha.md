---
layout: post
title: "基于 Patroni 的 Citus 高可用环境部署"
date: "2021-05-08 22:10:00"
category: PostgreSQL
tags: db PostgreSQL
author: lework
---
* content
{:toc}

基于 Patroni 的 Citus 高可用环境部署

Citus 是一个非常实用的能够使 PostgreSQL 具有进行水平扩展能力的插件，或者说是一款以 PostgreSQL 插件形式部署的基于 PostgreSQL 的分布式 HTAP 数据库。

Patroni 是一个基于ZooKeeper、etcd 或 Consul 的 PostgreSQL HA 实现。

## 集群节点

![Patroni-Citus](/assets/images/ops/Patroni-Citus-HA.png)




| hostname           |           ip | vip                                            | role                   | os         | 配置 |
| ------------------ | -----------: | ---------------------------------------------- | ---------------------- | ---------- | ---- |
| patroni_etcd_node1 | 10.10.10.120 |                                                | etcd                   | CentOS 7.8 | 2C4G |
| patroni_etcd_node2 | 10.10.10.121 |                                                | etcd                   | CentOS 7.8 | 2C4G |
| patroni_etcd_node3 | 10.10.10.122 |                                                | etcd                   | CentOS 7.8 | 2C4G |
| citus_cn1          | 10.10.10.130 | （读写）10.10.10.140<br />（只读）10.10.10.141 | Citus CN               | CentOS 7.8 | 2C4G |
| citus_cn2          | 10.10.10.131 | （读写）10.10.10.140<br />（只读）10.10.10.141 | Citus CN               | CentOS 7.8 | 2C4G |
| citus_worker_g1_1  | 10.10.10.132 | （读写）10.10.10.142<br />（只读）10.10.10.143 | Citus Worker（group1） | CentOS 7.8 | 2C4G |
| citus_worker_g1_2  | 10.10.10.133 | （读写）10.10.10.142<br />（只读）10.10.10.143 | Citus Worker（group1） | CentOS 7.8 | 2C4G |
| citus_worker_g2_1  | 10.10.10.134 | （读写）10.10.10.144<br />（只读）10.10.10.145 | Citus Worker（group2） | CentOS 7.8 | 2C4G |
| citus_worker_g2_2  | 10.10.10.135 | （读写）10.10.10.144<br />（只读）10.10.10.145 | Citus Worker（group2） | CentOS 7.8 | 2C4G |
| citus_worker_g3_1  | 10.10.10.136 | （读写）10.10.10.146<br />（只读）10.10.10.147 | Citus Worker（group3） | CentOS 7.8 | 2C4G |
| citus_worker_g3_2  | 10.10.10.137 | （读写）10.10.10.146<br />（只读）10.10.10.147 | Citus Worker（group3） | CentOS 7.8 | 2C4G |

**主要软件**

- CentOS 7.8
- PostgreSQL `12`
- Citus `10.0.3`
- Patroni `2.0.2`
- etcd `3.3.25`

**相关文档**

- https://patroni.readthedocs.io/
- https://docs.citusdata.com/

## 环境准备

### repo 配置

```bash
[ -f /etc/yum.repos.d/CentOS-Base.repo ] && sed -e 's!^#baseurl=!baseurl=!g' \
    -e 's!^mirrorlist=!#mirrorlist=!g' \
    -e 's!mirror.centos.org!mirrors.aliyun.com!g' \
    -i /etc/yum.repos.d/CentOS-Base.repo
  
yum install -y epel-release
  
[ -f /etc/yum.repos.d/epel.repo ] && sed -e 's!^mirrorlist=!#mirrorlist=!g' \
    -e 's!^metalink=!#metalink=!g' \
    -e 's!^#baseurl=!baseurl=!g' \
    -e 's!//download\.fedoraproject\.org/pub!//mirrors.aliyun.com!g' \
    -e 's!http://mirrors\.aliyun!https://mirrors.aliyun!g' \
    -i /etc/yum.repos.d/epel.repo
```

### 时间同步

所有节点设置时钟同步

```bash
yum install -y chrony 
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

chronyd -q -t 1 'server cn.pool.ntp.org iburst maxsamples 1'
systemctl enable chronyd
systemctl start chronyd
chronyc sources -v
chronyc sourcestats
hwclock -w
```

### 主机名设置

主机名设置

```
hostnamectl set-hostname citus_cn1
```

> 每个节点都要设置对应的名称



所有节点设置主机名解析

```
cat << EOF >> /etc/hosts
10.10.10.120 patroni_etcd_node1
10.10.10.121 patroni_etcd_node2
10.10.10.122 patroni_etcd_node3
10.10.10.130 citus_cn1
10.10.10.131 citus_cn2
10.10.10.132 citus_worker_g1_1
10.10.10.133 citus_worker_g1_2
10.10.10.134 citus_worker_g2_1
10.10.10.135 citus_worker_g2_2
10.10.10.136 citus_worker_g3_1
10.10.10.137 citus_worker_g3_2
EOF
```

### 关闭 selinux

```bash
sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
setenforce 0
```



### 关闭 swap

```bash
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```



### 关闭 firewalld

```bash
systemctl stop firewalld
systemctl disable firewalld
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```



## etcd 集群部署

### 安装

> 在左右的集群节点上

```bash
# 安装需要的包
yum install -y etcd
```

### 配置

编辑etcd配置文件`/etc/etcd/etcd.conf`, 参考配置如下

- 10.10.10.120 节点配置

```bash
cp /etc/etcd/etcd.conf{,_bak}

cat << EOF > /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.120:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.10.10.120:2379"
ETCD_NAME="etcd0"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.120:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.120:2379"
ETCD_INITIAL_CLUSTER="etcd0=http://10.10.10.120:2380,etcd1=http://10.10.10.121:2380,etcd2=http://10.10.10.122:2380"
ETCD_INITIAL_CLUSTER_TOKEN="cluster1"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

- 10.10.10.121 节点配置

```bash
cp /etc/etcd/etcd.conf{,_bak}

cat << EOF > /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.121:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.10.10.121:2379"
ETCD_NAME="etcd1"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.121:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.121:2379"
ETCD_INITIAL_CLUSTER="etcd0=http://10.10.10.120:2380,etcd1=http://10.10.10.121:2380,etcd2=http://10.10.10.122:2380"
ETCD_INITIAL_CLUSTER_TOKEN="cluster1"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

- 10.10.10.122 节点配置

```bash
cp /etc/etcd/etcd.conf{,_bak}

cat << EOF > /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.122:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.10.10.122:2379"
ETCD_NAME="etcd2"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.122:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.122:2379"
ETCD_INITIAL_CLUSTER="etcd0=http://10.10.10.120:2380,etcd1=http://10.10.10.121:2380,etcd2=http://10.10.10.122:2380"
ETCD_INITIAL_CLUSTER_TOKEN="cluster1"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

### 启动

```bash
systemctl enable --now etcd  # 加入自启动，且现在启动
```

### 测试

```bash
# etcdctl cluster-health
member 1395b9df9ce2803c is healthy: got healthy result from http://10.10.10.121:2379
member 38c0514ffb109d35 is healthy: got healthy result from http://10.10.10.122:2379
member bf434997688c0e4f is healthy: got healthy result from http://10.10.10.120:2379
cluster is healthy

# etcdctl   member list
1395b9df9ce2803c: name=etcd1 peerURLs=http://10.10.10.121:2380 clientURLs=http://10.10.10.121:2379 isLeader=false
38c0514ffb109d35: name=etcd2 peerURLs=http://10.10.10.122:2380 clientURLs=http://10.10.10.122:2379 isLeader=false
bf434997688c0e4f: name=etcd0 peerURLs=http://10.10.10.120:2380 clientURLs=http://10.10.10.120:2379 isLeader=true

# etcdctl set test 1
1
# etcdctl get test
1
```

##  PostgreSQL + Citus + Patroni HA部署

在需要运行PostgreSQL的实例上安装相关软件

### 安装PostgreSQL 12和Citus

```bash
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

yum install -y postgresql12-server postgresql12-contrib
yum install -y citus_12
```

### 安装Patroni

```bash
yum install -y gcc python-psycopg2 python-devel python-setuptools

easy_install pip==20.3.4
pip install --upgrade setuptools

pip install wheel
pip install six
pip install patroni[etcd]


```



### 配置Patroni

创建PostgreSQL数据目录

```bash
mkdir -p /pgsql/data
chown postgres:postgres -R /pgsql
chmod -R 700 /pgsql/data
```

创建Partoni的service配置文件`/etc/systemd/system/patroni.service`

```bash
cat << EOF > /etc/systemd/system/patroni.service
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target
 
[Service]
Type=simple
User=postgres
Group=postgres
#StandardOutput=syslog
ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog
ExecStart=/usr/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
 
[Install]
WantedBy=multi-user.target
EOF

```

创建 Patroni 配置文件`/etc/patroni.yml`,以下是 `citus_cn1` 的配置示例

```bash
cat << EOF > /etc/patroni.yml
scope: cn
namespace: /service/
name: pg1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.10.10.130:8008

etcd:
  hosts: 10.10.10.120:2379,10.10.10.121:2379,10.10.10.122:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        listen_addresses: "0.0.0.0"
        port: 5432
        wal_level: logical
        hot_standby: "on"
        wal_keep_segments: 1000
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        max_connections: "100"
        max_prepared_transactions: "100"
        shared_preload_libraries: "citus"
        citus.node_conninfo: "sslmode=prefer"
        citus.replication_model: streaming
        citus.task_assignment_policy: round-robin

  initdb:
  - encoding: UTF8
  - locale: C
  - lc-ctype: zh_CN.UTF-8
  - data-checksums

  pg_hba:
  - host replication repl 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.10.10.130:5432
  data_dir: /pgsql/data
  bin_dir: /usr/pgsql-12/bin

  authentication:
    replication:
      username: repl
      password: "123456"
    superuser:
      username: postgres
      password: "123456"

  basebackup:
    max-rate: 100M
    checkpoint: fast

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
watchdog:
  mode: automatic # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5
EOF

```

其他PG节点的patroni.yml需要相应修改下面4个参数

- `scope`
  - `citus_cn1`，`citus_cn2` 设置为 `cn1`
  - `citus_worker_g1_1`，`citus_worker_g1_2` 设置为 `group1`
  - `citus_worker_g2_1`，`citus_worker_g2_2` 设置为 `group2`
  - `citus_worker_g3_1`，`citus_worker_g3_2` 设置为 `group3`
- `name`
  - 按顺序设置 `pg1`-`pg8`
- `restapi.connect_address`
  - 根据各自节点IP设置
- `postgresql.connect_address`
  - 根据各自节点IP设置



### 启动Patroni

在所有节点上启动 Patroni

```bash
systemctl daemon-reload

systemctl enable --now patroni
```

### Patroni 集群状态

同一个cluster中，第一次启动的Patroni实例会作为leader运行，并初始创建PostgreSQL实例和用户。后续节点初次启动时从leader节点克隆数据

查看cn集群状态

```bash
[root@citus_cn1 ~]# patronictl -c /etc/patroni.yml list
+ Cluster: cn (6948307582909233673) --------+----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg1    | 10.10.10.130 | Leader  | running |  1 |           |
| pg2    | 10.10.10.131 | Replica | running |  1 |       0.0 |
```

查看group1集群状态

```bash
[root@citus_worker_g1_1 ~]# patronictl -c /etc/patroni.yml list
+ Cluster: group1 (6948309235166960069) ----+----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg3    | 10.10.10.132 | Leader  | running |  1 |           |
| pg4    | 10.10.10.133 | Replica | running |  1 |       0.0 |
+--------+--------------+---------+---------+----+-----------+
```


查看group2集群状态
```bash
[root@citus_worker_g2_1 ~]# patronictl -c /etc/patroni.yml list
+ Cluster: group2 (6948309256362359238) ----+----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg5    | 10.10.10.134 | Leader  | running |  1 |           |
| pg6    | 10.10.10.135 | Replica | running |  1 |       0.0 |
+--------+--------------+---------+---------+----+-----------+
```


查看group3集群状态
```bash
[root@citus_worker_g3_1 ~]# patronictl -c /etc/patroni.yml list                            
+ Cluster: group3 (6948309268683684299) ----+----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg7    | 10.10.10.136 | Leader  | running |  1 |           |
| pg8    | 10.10.10.137 | Replica | running |  1 |       0.0 |
```





为了方便日常操作，设置全局环境变量`PATRONICTL_CONFIG_FILE`

```
echo 'export PATRONICTL_CONFIG_FILE=/etc/patroni.yml' >/etc/profile.d/patroni.sh
```

添加以下环境变量到`~postgres/.bash_profile`

```bash
cat << EOF >> ~postgres/.bash_profile
export PGDATA=/pgsql/data
export PATH=/usr/pgsql-12/bin:\$PATH
EOF
```

设置 postgres 用户拥有sudoer权限

```
echo 'postgres        ALL=(ALL)       NOPASSWD: ALL'> /etc/sudoers.d/postgres
```



### 配置 Citus

此次我们通过vip的方案来实现读写分离场景。



#### 配置 Worker 免密

在Worker的主备节点上分别修改`/pgsql/data/pg_hba.conf`配置文件，以下内容添加到其它配置项前面允许CN免密连接Worker。

```bash
host all all 10.10.10.130/32 trust
host all all 10.10.10.131/32 trust
```

> 注意，要放在认证规则之前。

修改后重新加载配置

```bash
sudo -i -u postgres pg_ctl reload
```

#### 配置 VIP

> 这里通过回调脚本来实现vip绑定，也可以通过keepalived

准备加载VIP的回调脚本`/pgsql/loadvip.sh`, 这里给出  cn 的配置

```bash
#!/bin/bash

# 只读vip
R_VIP=10.10.10.141
# 读写vip
RW_VIP=10.10.10.140
# 网关
GATEWAY=10.10.10.254
# 网卡接口
DEV=ens192

action=$1
role=$2
cluster=$3

log()
{
  echo "[loadvip]: $*" | logger
}

load_vip()
{
ip a|grep -w ${DEV}|grep -w ${VIP} >/dev/null
if [ $? -eq 0 ] ;then
  log "vip exists, skip load vip"
else
  sudo ip addr add ${VIP}/32 dev ${DEV} >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to add vip ${VIP} at dev ${DEV} rc=$rc"
    exit 1
  fi

  log "added vip ${VIP} at dev ${DEV}"

  arping -U -I ${DEV} -s ${VIP} ${GATEWAY} -c 5 >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to call arping to gateway ${GATEWAY} rc=$rc"
    exit 1
  fi
  
  log "called arping to gateway ${GATEWAY}"
fi
}

unload_vip()
{
ip a|grep -w ${DEV}|grep -w ${VIP} >/dev/null
if [ $? -eq 0 ] ;then
  sudo ip addr del ${VIP}/32 dev ${DEV} >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to delete vip ${VIP} at dev ${DEV} rc=$rc"
    exit 1
  fi

  log "deleted vip ${VIP} at dev ${DEV}"
else
  log "vip not exists, skip delete vip"
fi
}

log "loadvip start args:'$*'"

case $action in
  on_start|on_restart|on_role_change)
    case $role in
      master)
        [[ "$RW_VIP" != "" ]] && { VIP=$RW_VIP; load_vip; }
        [[ "$RW_VIP" != "$R_VIP" ]] && { VIP=$R_VIP; unload_vip; }
        ;;
      replica)
        [[ "$RW_VIP" != "" ]] && { VIP=$RW_VIP; unload_vip; }
        [[ "$RW_VIP" != "$R_VIP" ]] && { VIP=$R_VIP; load_vip; }
        ;;
      *)
        log "wrong role '$role'"
        exit 1
        ;;
    esac
    ;;
  *)
    log "wrong action '$action'"
    exit 1
    ;;
esac
```

其他 worker 节点的 ` loadvip.sh` 需要相应修改下面2个参数

- R_VIP:  对应的只读 vip 地址
- RW_VIP:  对应的读写 vip 地址



修改节点上的Patroni配置文件`/etc/patroni.yml`，配置回调函数

```yaml
postgresql:
...
  callbacks:
    on_start: /bin/bash /pgsql/loadvip.sh
    on_restart: /bin/bash /pgsql/loadvip.sh
    on_role_change: /bin/bash /pgsql/loadvip.sh
```



依次重启节点 patroni 服务

```bash
systemctl restart patroni.service
```

可以从日志中看到 vip 切换的信息。

```bash
[root@citus_worker1 ~]# cat  /var/log/messages | grep loadvip
cat  /var/log/messages | grep loadvip   
Apr  7 03:35:25 localhost postgres: [loadvip]: loadvip start args:'on_start replica group1'
Apr  7 03:35:25 localhost postgres: [loadvip]: vip not exists, skip delete vip
Apr  7 03:35:25 localhost postgres: [loadvip]: added vip 10.10.10.143 at dev ens192
Apr  7 03:35:29 localhost postgres: [loadvip]: called arping to gateway 10.10.10.254
```



#### 开启 citus 扩展

在所有的 `Leader` 节点上创建 `citus` 扩展

```bash
sudo -i -u postgres psql -c "create extension citus;"
```

这里是在默认的数据库中开启扩展。你也可以新建数据库，在指定的数据库中开启。

```sql
CREATE DATABASE newdb;
\c newdb
CREATE EXTENSION citus;
```



#### 添加节点

在 cn 的  `Leader`  节点上

- 添加 group1的读写VIP(10.10.10.142) 和只读VIP（10.10.10.143），分别作为`primary` worker和`secondary` worker，groupid设置为1。

```bash
sudo -i -u postgres psql -c "SELECT * from citus_add_node('10.10.10.142', 5432, 1, 'primary');"
sudo -i -u postgres psql -c "SELECT * from citus_add_node('10.10.10.143', 5432, 1, 'secondary');"
```

- 添加 group1的读写VIP(10.10.10.144) 和只读VIP（10.10.10.145），分别作为`primary` worker和`secondary` worker，groupid设置为2。

  ```bash
  sudo -i -u postgres psql -c "SELECT * from citus_add_node('10.10.10.144', 5432, 2, 'primary');"
  sudo -i -u postgres psql -c "SELECT * from citus_add_node('10.10.10.145', 5432, 2, 'secondary');"
  ```

- 添加 group1的读写VIP(10.10.10.146) 和只读VIP（10.10.10.147），分别作为`primary` worker和`secondary` worker，groupid设置为3。

  ```bash
  sudo -i -u postgres psql -c "SELECT * from citus_add_node('10.10.10.146', 5432, 3, 'primary');"
  sudo -i -u postgres psql -c "SELECT * from citus_add_node('10.10.10.147', 5432, 3, 'secondary');"
  ```

查看 worker 节点

```bash
[root@citus_cn1 ~]# sudo -i -u postgres psql -c "SELECT * FROM citus_get_active_worker_nodes();"
  node_name   | node_port 
--------------+-----------
 10.10.10.146 |      5432
 10.10.10.142 |      5432
 10.10.10.144 |      5432
```

查看所有节点

```bash
[root@citus_cn1 ~]# sudo -i -u postgres psql -c "select nodeid,nodename,nodeport from pg_dist_node;"
 nodeid |   nodename   | nodeport 
--------+--------------+----------
      1 | 10.10.10.142 |     5432
      2 | 10.10.10.143 |     5432
      3 | 10.10.10.144 |     5432
      4 | 10.10.10.145 |     5432
      5 | 10.10.10.146 |     5432
      6 | 10.10.10.147 |     5432
(6 rows)
```

#### 测试分片表

创建分片表测试验证

```bash
[root@citus_cn1 ~]# sudo -i -u postgres psql
psql (12.6)
Type "help" for help.

# 1. 创建表
postgres=# create table tb1(id int primary key,c1 text);
CREATE TABLE


# 2. 设置分片数量
postgres=# set citus.shard_count = 64;
SET

# 3. 创建分片表
postgres=# select create_distributed_table('tb1','id');
 create_distributed_table 
--------------------------
 
(1 row)

# 4. 插入随机数据
postgres=# insert into tb1 select id,random()*1000 from generate_series(1,10000)id;
INSERT 0 10000

# 5. 查询计划
postgres=# explain select * from tb1;
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=36)
   Task Count: 64
   Tasks Shown: One of 64
   ->  Task
         Node: host=10.10.10.142 port=5432 dbname=postgres
         ->  Seq Scan on tb1_102008 tb1  (cost=0.00..3.58 rows=158 width=22)
(6 rows)

# 6.检查分片位置
postgres=# select * from pg_dist_placement where shardid in (select shardid from pg_dist_shard where logicalrelid='tb1'::regclass);
 placementid | shardid | shardstate | shardlength | groupid 
-------------+---------+------------+-------------+---------
           1 |  102008 |          1 |           0 |       1
           2 |  102009 |          1 |           0 |       2
           3 |  102010 |          1 |           0 |       3
           4 |  102011 |          1 |           0 |       1
           5 |  102012 |          1 |           0 |       2
           6 |  102013 |          1 |           0 |       3
           7 |  102014 |          1 |           0 |       1
           8 |  102015 |          1 |           0 |       2
           9 |  102016 |          1 |           0 |       3
          10 |  102017 |          1 |           0 |       1
          11 |  102018 |          1 |           0 |       2
          12 |  102019 |          1 |           0 |       3
          13 |  102020 |          1 |           0 |       1
          14 |  102021 |          1 |           0 |       2
          15 |  102022 |          1 |           0 |       3
          16 |  102023 |          1 |           0 |       1
          17 |  102024 |          1 |           0 |       2
          18 |  102025 |          1 |           0 |       3
          19 |  102026 |          1 |           0 |       1
          20 |  102027 |          1 |           0 |       2
          21 |  102028 |          1 |           0 |       3
          22 |  102029 |          1 |           0 |       1
          23 |  102030 |          1 |           0 |       2
          24 |  102031 |          1 |           0 |       3
          25 |  102032 |          1 |           0 |       1
          26 |  102033 |          1 |           0 |       2
          27 |  102034 |          1 |           0 |       3
          28 |  102035 |          1 |           0 |       1
          29 |  102036 |          1 |           0 |       2
          30 |  102037 |          1 |           0 |       3
          31 |  102038 |          1 |           0 |       1
          32 |  102039 |          1 |           0 |       2
          33 |  102040 |          1 |           0 |       3
          34 |  102041 |          1 |           0 |       1
          35 |  102042 |          1 |           0 |       2
          36 |  102043 |          1 |           0 |       3
          37 |  102044 |          1 |           0 |       1
          38 |  102045 |          1 |           0 |       2
          39 |  102046 |          1 |           0 |       3
          40 |  102047 |          1 |           0 |       1
          41 |  102048 |          1 |           0 |       2
          42 |  102049 |          1 |           0 |       3
          43 |  102050 |          1 |           0 |       1
          44 |  102051 |          1 |           0 |       2
          45 |  102052 |          1 |           0 |       3
          46 |  102053 |          1 |           0 |       1
          47 |  102054 |          1 |           0 |       2
          48 |  102055 |          1 |           0 |       3
          49 |  102056 |          1 |           0 |       1
          50 |  102057 |          1 |           0 |       2
          51 |  102058 |          1 |           0 |       3
          52 |  102059 |          1 |           0 |       1
          53 |  102060 |          1 |           0 |       2
          54 |  102061 |          1 |           0 |       3
          55 |  102062 |          1 |           0 |       1
          56 |  102063 |          1 |           0 |       2
          57 |  102064 |          1 |           0 |       3
          58 |  102065 |          1 |           0 |       1
          59 |  102066 |          1 |           0 |       2
          60 |  102067 |          1 |           0 |       3
          61 |  102068 |          1 |           0 |       1
          62 |  102069 |          1 |           0 |       2
          63 |  102070 |          1 |           0 |       3
          64 |  102071 |          1 |           0 |       1
(64 rows)

```

#### 配置 读写分离

为了让CN备库连接到secondary的worker，还需要在CN备库上设置以下参数

```bash
sudo -i -u postgres psql -c "alter system set citus.use_secondary_nodes=always;"
sudo -i -u postgres psql -c "select pg_reload_conf();"
```

这个参数的变更只对新创建的会话生效，如果希望立即生效，需要在修改参数后杀掉已有会话。

现在分别到CN主库和备库上执行同一条SQL，可以看到SQL被发往不同的worker。

CN主库（未设置`citus.use_secondary_nodes=always`）：

```bash
[root@citus_cn1 ~]# sudo -i -u postgres psql -c "explain select * from tb1;" 
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=36)
   Task Count: 64
   Tasks Shown: One of 64
   ->  Task
         Node: host=10.10.10.142 port=5432 dbname=postgres
         ->  Seq Scan on tb1_102008 tb1  (cost=0.00..3.58 rows=158 width=22)
(6 rows)

```

CN备库（设置了`citus.use_secondary_nodes=always`）：

```bash
[root@citus_cn2 ~]# sudo -i -u postgres psql -c "explain select * from tb1;"  
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=36)
   Task Count: 64
   Tasks Shown: One of 64
   ->  Task
         Node: host=10.10.10.143 port=5432 dbname=postgres
         ->  Seq Scan on tb1_102008 tb1  (cost=0.00..3.58 rows=158 width=22)
(6 rows)
```



由于CN也会发生主备切换，`citus.use_secondary_nodes` 参数必须动态调节。这可以使用 Patroni 的回调脚本实现

创建动态设置参数的`/pgsql/switch_use_secondary_nodes.sh`

```bash
#!/bin/bash

DBNAME=postgres
KILL_ALL_SQL="select pg_terminate_backend(pid) from pg_stat_activity  where backend_type='client backend' and application_name <> 'Patroni' and pid <> pg_backend_pid()"

action=$1
role=$2
cluster=$3


log()
{
  echo "[switch_use_secondary_nodes]: $*"|logger
}

alter_use_secondary_nodes()
{
  value="$1"
  oldvalue=`psql -d postgres -Atc "show citus.use_secondary_nodes"`
  if [ "$value" = "$oldvalue" ] ; then
    log "old value of use_secondary_nodes already be '${value}', skip change"
	return
  fi

  psql -d ${DBNAME} -c "alter system set citus.use_secondary_nodes=${value}" >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to alter use_secondary_nodes to '${value}' rc=$rc"
    exit 1
  fi

  psql -d ${DBNAME} -c 'select pg_reload_conf()' >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to call pg_reload_conf() rc=$rc"
    exit 1
  fi

  log "changed use_secondary_nodes to '${value}'"

  ## kill all existing connections
  killed_conns=`psql -d ${DBNAME} -Atc "${KILL_ALL_SQL}" | wc -l`
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "failed to kill connections rc=$rc"
    exit 1
  fi
  
  log "killed ${killed_conns} connections"

}

log "switch_use_secondary_nodes start args:'$*'"

case $action in
  on_start|on_restart|on_role_change)
    case $role in
      master)
        alter_use_secondary_nodes never
        ;;
      replica)
        alter_use_secondary_nodes always
        ;;
      *)
        log "wrong role '$role'"
        exit 1
        ;;
    esac
    ;;
  *)
    log "wrong action '$action'"
    exit 1
    ;;
esac
```



修改 `/pgsql/loadvip.sh` 脚本, 在尾部添加

```bash
/bin/bash /pgsql/switch_use_secondary_nodes.sh $@
```



CN上执行switchover后，可以看到`use_secondary_nodes`参数发生了修改

/var/log/messages:

```bash
[root@citus_cn1 ~]# cat /var/log/messages | grep switch_use_secondary_nodes
Apr  7 04:08:06 localhost postgres: [switch_use_secondary_nodes]: switch_use_secondary_nodes start args:'on_start replica cn'
Apr  7 04:08:06 localhost postgres: [switch_use_secondary_nodes]: changed use_secondary_nodes to 'always'
Apr  7 04:08:06 localhost postgres: [switch_use_secondary_nodes]: killed 0 connections
Apr  7 04:08:08 localhost postgres: [switch_use_secondary_nodes]: switch_use_secondary_nodes start args:'on_role_change master cn'
Apr  7 04:08:08 localhost postgres: [switch_use_secondary_nodes]: changed use_secondary_nodes to 'never'
Apr  7 04:08:08 localhost postgres: [switch_use_secondary_nodes]: killed 0 connections
```



### 客户端测试

读写地址

```bash
[root@citus_cn1 ~]# /usr/pgsql-12/bin/psql -h 10.10.10.140 -p 5432 -U postgres -W  postgres
Password: 
psql (12.6)
Type "help" for help.

postgres=# select count(*) from tb1;
 count 
-------
 10000
(1 row)

postgres=# explain select count(*) from tb1;
                                       QUERY PLAN                                       
----------------------------------------------------------------------------------------
 Aggregate  (cost=250.00..250.02 rows=1 width=8)
   ->  Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=8)
         Task Count: 64
         Tasks Shown: One of 64
         ->  Task
               Node: host=10.10.10.142 port=5432 dbname=postgres
               ->  Aggregate  (cost=7.10..7.11 rows=1 width=8)
                     ->  Seq Scan on tb1_102008 tb1  (cost=0.00..6.28 rows=328 width=0)
(8 rows)

postgres=# insert into tb1 select id,random()*1000 from generate_series(10001,20000)id; 
INSERT 0 10000
postgres=# select count(*) from tb1;
 count 
-------
 20000
(1 row)
```

> 可以正常读写数据。

只读地址

```bash
[root@citus_cn1 ~]# /usr/pgsql-12/bin/psql -h 10.10.10.141 -p 5432 -U postgres -W  postgres 
Password: 
psql (12.6)
Type "help" for help.

postgres=# select count(*) from tb1;
 count 
-------
 20000
(1 row)

postgres=# explain select count(*) from tb1;
                                       QUERY PLAN                                       
----------------------------------------------------------------------------------------
 Aggregate  (cost=250.00..250.02 rows=1 width=8)
   ->  Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=8)
         Task Count: 64
         Tasks Shown: One of 64
         ->  Task
               Node: host=10.10.10.143 port=5432 dbname=postgres
               ->  Aggregate  (cost=7.10..7.11 rows=1 width=8)
                     ->  Seq Scan on tb1_102008 tb1  (cost=0.00..6.28 rows=328 width=0)
(8 rows)

postgres=# insert into tb1 select id,random()*1000 from generate_series(10001,20000)id;
ERROR:  writing to worker nodes is not currently allowed
DETAIL:  the database is read-only
postgres=# 
```

> 只能读取数据， 不能写数据


#### 测试普通表

```bash
[root@citus_cn1 ~]# sudo -i -u postgres psql
psql (12.6)
Type "help" for help.

# 1. 创建表
postgres=# create table tb(id int primary key,c1 text);
CREATE TABLE

# 2. 插入随机数据
postgres=# insert into tb1 select id,random()*1000 from generate_series(1,10000)id;
INSERT 0 10000
```



#### 测试分片表

```bash
[root@citus_cn1 ~]# sudo -i -u postgres psql
psql (12.6)
Type "help" for help.

# 1. 创建表
postgres=# create table tb1(id int primary key,c1 text);
CREATE TABLE


# 2. 设置分片数量
postgres=# set citus.shard_count = 64;
SET

# 3. 创建分片表
postgres=# select create_distributed_table('tb1','id');
 create_distributed_table 
--------------------------
 
(1 row)

# 4. 插入随机数据
postgres=# insert into tb1 select id,random()*1000 from generate_series(1,10000)id;
INSERT 0 10000

# 5. 查询计划
postgres=# explain select * from tb1;
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=100000 width=36)
   Task Count: 64
   Tasks Shown: One of 64
   ->  Task
         Node: host=10.10.10.142 port=5432 dbname=postgres
         ->  Seq Scan on tb1_102008 tb1  (cost=0.00..3.58 rows=158 width=22)
(6 rows)

# 6.检查分片位置
postgres=# select * from pg_dist_placement where shardid in (select shardid from pg_dist_shard where logicalrelid='tb1'::regclass);
 placementid | shardid | shardstate | shardlength | groupid 
-------------+---------+------------+-------------+---------
           1 |  102008 |          1 |           0 |       1
           2 |  102009 |          1 |           0 |       2
           3 |  102010 |          1 |           0 |       3
           4 |  102011 |          1 |           0 |       1
           5 |  102012 |          1 |           0 |       2
           6 |  102013 |          1 |           0 |       3
           7 |  102014 |          1 |           0 |       1
           8 |  102015 |          1 |           0 |       2
           9 |  102016 |          1 |           0 |       3
          10 |  102017 |          1 |           0 |       1
          11 |  102018 |          1 |           0 |       2
          12 |  102019 |          1 |           0 |       3
          13 |  102020 |          1 |           0 |       1
          14 |  102021 |          1 |           0 |       2
          15 |  102022 |          1 |           0 |       3
          16 |  102023 |          1 |           0 |       1
          17 |  102024 |          1 |           0 |       2
          18 |  102025 |          1 |           0 |       3
          19 |  102026 |          1 |           0 |       1
          20 |  102027 |          1 |           0 |       2
          21 |  102028 |          1 |           0 |       3
          22 |  102029 |          1 |           0 |       1
          23 |  102030 |          1 |           0 |       2
          24 |  102031 |          1 |           0 |       3
          25 |  102032 |          1 |           0 |       1
          26 |  102033 |          1 |           0 |       2
          27 |  102034 |          1 |           0 |       3
          28 |  102035 |          1 |           0 |       1
          29 |  102036 |          1 |           0 |       2
          30 |  102037 |          1 |           0 |       3
          31 |  102038 |          1 |           0 |       1
          32 |  102039 |          1 |           0 |       2
          33 |  102040 |          1 |           0 |       3
          34 |  102041 |          1 |           0 |       1
          35 |  102042 |          1 |           0 |       2
          36 |  102043 |          1 |           0 |       3
          37 |  102044 |          1 |           0 |       1
          38 |  102045 |          1 |           0 |       2
          39 |  102046 |          1 |           0 |       3
          40 |  102047 |          1 |           0 |       1
          41 |  102048 |          1 |           0 |       2
          42 |  102049 |          1 |           0 |       3
          43 |  102050 |          1 |           0 |       1
          44 |  102051 |          1 |           0 |       2
          45 |  102052 |          1 |           0 |       3
          46 |  102053 |          1 |           0 |       1
          47 |  102054 |          1 |           0 |       2
          48 |  102055 |          1 |           0 |       3
          49 |  102056 |          1 |           0 |       1
          50 |  102057 |          1 |           0 |       2
          51 |  102058 |          1 |           0 |       3
          52 |  102059 |          1 |           0 |       1
          53 |  102060 |          1 |           0 |       2
          54 |  102061 |          1 |           0 |       3
          55 |  102062 |          1 |           0 |       1
          56 |  102063 |          1 |           0 |       2
          57 |  102064 |          1 |           0 |       3
          58 |  102065 |          1 |           0 |       1
          59 |  102066 |          1 |           0 |       2
          60 |  102067 |          1 |           0 |       3
          61 |  102068 |          1 |           0 |       1
          62 |  102069 |          1 |           0 |       2
          63 |  102070 |          1 |           0 |       3
          64 |  102071 |          1 |           0 |       1
(64 rows)
```



## 参考


- https://mp.weixin.qq.com/s/3ow_FGTFRvC01X8ZUDid2Q
- https://chenhuajun.github.io/2020/09/07/%E5%9F%BA%E4%BA%8EPatroni%E7%9A%84Citus%E9%AB%98%E5%8F%AF%E7%94%A8%E7%8E%AF%E5%A2%83%E9%83%A8%E7%BD%B2.html
- https://chenhuajun.github.io/2020/09/07/%E5%9F%BA%E4%BA%8EPatroni%E7%9A%84PostgreSQL%E9%AB%98%E5%8F%AF%E7%94%A8%E7%8E%AF%E5%A2%83%E9%83%A8%E7%BD%B2.html

