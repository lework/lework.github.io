---
layout: post
title: "使用 kainstall 离线部署 kubernetes v1.19.3 ha 集群"
date: "2020-10-18 19:40"
category: kubernetes
tags: kubernetes install
author: lework
---
* content
{:toc}

今天给大家介绍一款工具:  [kainstall](https://github.com/lework/kainstall) 一个由纯bash脚本编写的工具。可一键离线部署 kubernetes 高可用集群，增删节点，管理k8s集群变得省时省力。

话不多说，请看下面介绍和案例使用



## 工具介绍

### Github

[github.com/lework/kainstall](github.com/lework/kainstall)

### 功能

- 服务器初始化。
  - 关闭 `selinux`
  - 关闭 `swap`
  - 关闭 `firewalld`
  - 关闭大内存页
  - 配置 `epel` 源
  - 修改 `limits`
  - 配置内核参数
  - 配置 `history` 记录
  - 配置 `journal` 日志
  - 配置 `chrony`时间同步
  - 安装 `ipvs` 模块
  - 更新内核
- 安装`docker`, `kube`组件。
- 初始化`kubernetes`集群,以及增加或删除节点。
- 安装`ingress`组件，可选`nginx`，`traefik`。
- 安装`network`组件，可选`flannel`，`calico`， 需在初始化时指定。
- 安装`monitor`组件，可选`prometheus`。
- 安装`log`组件，可选`elasticsearch`。
- 安装`storage`组件，可选`rook`。
- 安装`web ui`组件，可选`dashboard`。
- 升级到`kubernetes`指定版本。
- 更新证书。
- 添加运维操作，如备份etcd快照。
- 支持**离线部署**。

### 帮助信息

```bash
# bash kainstall.sh 

Install kubernetes cluster using kubeadm.

Usage:
  kainstall.sh [command]

Available Commands:
  init            Init Kubernetes cluster.
  reset           Reset Kubernetes cluster.
  add             Add nodes to the cluster.
  del             Remove node from the cluster.
  upgrade         Upgrading kubeadm clusters.
  renew-cert      Renew all available certificates.

Flag:
  -m,--master          master node, default: ''
  -w,--worker          work node, default: ''
  -u,--user            ssh user, default: root
  -p,--password        ssh password,default: 123456
  -P,--port            ssh port, default: 22
  -v,--version         kube version, default: latest
  -n,--network         cluster network, choose: [flannel,calico], default: flannel
  -i,--ingress         ingress controller, choose: [nginx,traefik], default: nginx
  -ui,--ui             cluster web ui, choose: [dashboard], default: dashboard
  -M,--monitor         cluster monitor, choose: [prometheus]
  -l,--log             cluster log, choose: [elasticsearch]
  -s,--storage         cluster storage, choose: [rook]
  -U,--upgrade-kernel  upgrade kernel
  -of,--offline-file   specify the offline package file to load

Example:
  [cluster node]
  kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134,192.168.77.135 \
  --user root \
  --password 123456 \
  --version 1.19.2

  [cluster node]
  kainstall.sh reset \
  --user root \
  --password 123456

  [add node]
  kainstall.sh add \
  --master 192.168.77.140,192.168.77.141 \
  --worker 192.168.77.143,192.168.77.144 \
  --user root \
  --password 123456 \
  --version 1.19.2

  [del node]
  kainstall.sh del \
  --master 192.168.77.140,192.168.77.141 \
  --worker 192.168.77.143,192.168.77.144 \
  --user root \
  --password 123456
 
  [other]
  kainstall.sh renew-cert
  kainstall.sh upgrade --version 1.19.2
  kainstall.sh add --ingress traefik
  kainstall.sh add --monitor prometheus
  kainstall.sh add --log elasticsearch
  kainstall.sh add --storage rook
  kainstall.sh add --ui dashboard


ERROR Summary: 
  
ACCESS Summary: 
  

  See detailed log >>> /tmp/kainstall.GVl5r0ASAV/kainstall.log 


```

- `ERROR Summary` 记录错误的信息
- `ACCESS Summary` 记录用于访问的信息
- `  See detailed log`  指出脚本执行的详细日志

### 默认设置

在脚本的开头存放着默认配置，可根据自己需求修改这些。

```bash
# 版本
DOCKER_VERSION="latest"
KUBE_VERSION="latest"
FLANNEL_VERSION="0.12.0"
METRICS_SERVER_VERSION="0.3.7"
INGRESS_NGINX="0.35.0"
TRAEFIK_VERSION="2.3.1"
CALICO_VERSION="3.16.1"
KUBE_PROMETHEUS_VERSION="0.6.0"
ELASTICSEARCH_VERSION="7.9.2"
ROOK_VERSION="1.4.5"
KUBERNETES_DASHBOARD_VERSION="2.0.4"

# 集群配置
KUBE_APISERVER="apiserver.cluster.local"
KUBE_POD_SUBNET="10.244.0.0/16"
KUBE_SERVICE_SUBNET="10.96.0.0/16"
KUBE_IMAGE_REPO="registry.aliyuncs.com/k8sxio"
KUBE_NETWORK="flannel"
KUBE_INGRESS="nginx"
KUBE_MONITOR="prometheus"
KUBE_STORAGE="rook"
KUBE_LOG="elasticsearch"
KUBE_UI="dashboard"

# 定义的master和worker节点地址，以逗号分隔
MASTER_NODES=""
WORKER_NODES=""

# 定义在哪个节点上进行设置
INIT_NODE="127.0.0.1"

# 节点的连接信息
SSH_USER="root"
SSH_PASSWORD="123456"
SSH_PORT="22"

# 节点设置
HOSTNAME_PREFIX="k8s"

# 脚本设置
TMP_DIR="$(mktemp -d -t kainstall.XXXXXXXXXX)"
LOG_FILE="${TMP_DIR}/kainstall.log"
SSH_OPTIONS="-o ConnectTimeout=600 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
ERROR_INFO="\n\033[31mERROR Summary: \033[0m\n  "
ACCESS_INFO="\n\033[32mACCESS Summary: \033[0m\n  "
COMMAND_OUTPUT=""
SCRIPT_PARAMETER="$*"
# http://kainstall.oss-cn-shanghai.aliyuncs.com/1.19.3/centos7.tgz
OFFLINE_FILE=""
```

## 案例

### 节点列表

| name    | ip             | os         | cpu  | mem  | role   |
| ------- | -------------- | ---------- | ---- | ---- | ------ |
| node130 | 192.168.77.130 | CentOS 7.7 | 2C   | 2G   | master |
| node131 | 192.168.77.131 | CentOS 7.7 | 2C   | 2G   | master |
| node132 | 192.168.77.132 | CentOS 7.7 | 2C   | 2G   | master |
| node133 | 192.168.77.133 | CentOS 7.7 | 2C   | 2G   | worker |
| node134 | 192.168.77.134 | CentOS 7.7 | 2C   | 2G   | worker |
| node135 | 192.168.77.135 | CentOS 7.7 | 2C   | 2G   | master |
| node136 | 192.168.77.136 | CentOS 7.7 | 2C   | 2G   | worker |

### 拓扑架构

![k8s-node-ha](/assets/images/kubernetes/k8s-node-ha.png)

### 下载离线包

下载 kubernetes 在 centos7 系统上的 `1.19.3` 离线包

```bash
wget http://kainstall.oss-cn-shanghai.aliyuncs.com/1.19.3/centos7.tgz
```

> 更多版本的离线包下载地址见： https://github.com/lework/kainstall-offline

### 下载脚本

```bash
wget https://cdn.jsdelivr.net/gh/lework/kainstall/kainstall.sh
```



### 初始化集群

将离线包和脚本一起拷贝到集群master节点中的一台机器上。



在集群master节点中的一台机器上执行

**首先，需要安装一些基础包**

> 需要注意的是，因为需要在本地解压离线包和处理数据，需要在线安装一些基础包

```bash
yum install -y wget tar sshpass openssh
```

**接着，更新集群节点内核**

```bash
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3 \
  --upgrade-kernel \
  --offline-file centos7.tgz
```

> 这里，我们指定了--upgrade-kernel 来先更新内核，不需要的可以去掉。--offline-file 来指定离线包

执行日志

```bash
[2020-10-18T17:50:43.479706158+0800]: INFO:    [start] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --port 22 --version 1.19.3 --upgrade-kernel --offline-file centos7.tgz
[2020-10-18T17:50:43.485071042+0800]: INFO:    [check] ssh command exists.
[2020-10-18T17:50:43.486902907+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T17:50:43.488731203+0800]: INFO:    [check] wget command exists.
[2020-10-18T17:50:43.490673490+0800]: INFO:    [check] tar command exists.
[2020-10-18T17:50:43.760114367+0800]: INFO:    [check] ssh 192.168.77.130 connection succeeded.
[2020-10-18T17:50:44.259013296+0800]: INFO:    [check] ssh 192.168.77.131 connection succeeded.
[2020-10-18T17:50:44.764440460+0800]: INFO:    [check] ssh 192.168.77.132 connection succeeded.
[2020-10-18T17:50:44.995870090+0800]: INFO:    [check] ssh 192.168.77.133 connection succeeded.
[2020-10-18T17:50:45.474574513+0800]: INFO:    [check] ssh 192.168.77.134 connection succeeded.
[2020-10-18T17:50:45.495325447+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T17:50:52.992739503+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T17:50:52.994872979+0800]: INFO:    [offline] master 192.168.77.130: load offline file
[2020-10-18T17:50:53.214190160+0800]: INFO:    [offline] 192.168.77.130: mkdir offline dir succeeded.
[2020-10-18T17:50:55.772872391+0800]: INFO:    [offline] scp kernel file to 192.168.77.130 succeeded.
[2020-10-18T17:50:55.775491454+0800]: INFO:    [offline] master 192.168.77.130: install package
[2020-10-18T17:52:06.359160752+0800]: INFO:    [offline] master 192.168.77.130: install package succeeded.
[2020-10-18T17:52:06.560955231+0800]: INFO:    [offline] 192.168.77.130: clean offline file succeeded.
[2020-10-18T17:52:06.563867537+0800]: INFO:    [offline] master 192.168.77.131: load offline file
[2020-10-18T17:52:06.730818115+0800]: INFO:    [offline] 192.168.77.131: mkdir offline dir succeeded.
[2020-10-18T17:52:11.718231332+0800]: INFO:    [offline] scp kernel file to 192.168.77.131 succeeded.
[2020-10-18T17:52:11.720206185+0800]: INFO:    [offline] master 192.168.77.131: install package
[2020-10-18T17:53:53.304112678+0800]: INFO:    [offline] master 192.168.77.131: install package succeeded.
[2020-10-18T17:53:53.513266123+0800]: INFO:    [offline] 192.168.77.131: clean offline file succeeded.
[2020-10-18T17:53:53.515889603+0800]: INFO:    [offline] master 192.168.77.132: load offline file
[2020-10-18T17:53:53.684757325+0800]: INFO:    [offline] 192.168.77.132: mkdir offline dir succeeded.
[2020-10-18T17:53:56.499055863+0800]: INFO:    [offline] scp kernel file to 192.168.77.132 succeeded.
[2020-10-18T17:53:56.501573806+0800]: INFO:    [offline] master 192.168.77.132: install package
[2020-10-18T17:55:51.360719303+0800]: INFO:    [offline] master 192.168.77.132: install package succeeded.
[2020-10-18T17:55:51.547706910+0800]: INFO:    [offline] 192.168.77.132: clean offline file succeeded.
[2020-10-18T17:55:51.550060760+0800]: INFO:    [offline] worker 192.168.77.133: load offline file
[2020-10-18T17:55:51.733192673+0800]: INFO:    [offline] 192.168.77.133: mkdir offline dir succeeded.
[2020-10-18T17:55:55.344433611+0800]: INFO:    [offline] scp kernel file to 192.168.77.133 succeeded.
[2020-10-18T17:55:55.346646263+0800]: INFO:    [offline] worker 192.168.77.133: install package
[2020-10-18T17:57:15.130350421+0800]: INFO:    [offline] worker 192.168.77.133: install package succeeded.
[2020-10-18T17:57:15.316636719+0800]: INFO:    [offline] 192.168.77.133: clean offline file succeeded.
[2020-10-18T17:57:15.319319015+0800]: INFO:    [offline] worker 192.168.77.134: load offline file
[2020-10-18T17:57:15.472594633+0800]: INFO:    [offline] 192.168.77.134: mkdir offline dir succeeded.
[2020-10-18T17:57:18.366452869+0800]: INFO:    [offline] scp kernel file to 192.168.77.134 succeeded.
[2020-10-18T17:57:18.369041547+0800]: INFO:    [offline] worker 192.168.77.134: install package
[2020-10-18T17:58:40.398587876+0800]: INFO:    [offline] worker 192.168.77.134: install package succeeded.
[2020-10-18T17:58:40.616468510+0800]: INFO:    [offline] 192.168.77.134: clean offline file succeeded.
[2020-10-18T17:58:40.618252525+0800]: INFO:    [init] upgrade kernel: 192.168.77.130
[2020-10-18T17:58:42.516137803+0800]: INFO:    [init] upgrade kernel 192.168.77.130 succeeded.
[2020-10-18T17:58:42.518768445+0800]: INFO:    [init] upgrade kernel: 192.168.77.131
[2020-10-18T17:58:44.524662269+0800]: INFO:    [init] upgrade kernel 192.168.77.131 succeeded.
[2020-10-18T17:58:44.526621690+0800]: INFO:    [init] upgrade kernel: 192.168.77.132
[2020-10-18T17:58:46.136360769+0800]: INFO:    [init] upgrade kernel 192.168.77.132 succeeded.
[2020-10-18T17:58:46.138388263+0800]: INFO:    [init] upgrade kernel: 192.168.77.133
[2020-10-18T17:58:47.961893118+0800]: INFO:    [init] upgrade kernel 192.168.77.133 succeeded.
[2020-10-18T17:58:47.964063820+0800]: INFO:    [init] upgrade kernel: 192.168.77.134
[2020-10-18T17:58:49.720038765+0800]: INFO:    [init] upgrade kernel 192.168.77.134 succeeded.
[2020-10-18T17:58:49.880400284+0800]: INFO:    [init] 192.168.77.130: Wait for 15s to restart succeeded.
[2020-10-18T17:58:50.041030690+0800]: INFO:    [init] 192.168.77.131: Wait for 15s to restart succeeded.
[2020-10-18T17:58:50.207276379+0800]: INFO:    [init] 192.168.77.132: Wait for 15s to restart succeeded.
[2020-10-18T17:58:50.388503905+0800]: INFO:    [init] 192.168.77.133: Wait for 15s to restart succeeded.
[2020-10-18T17:58:50.549808718+0800]: INFO:    [init] 192.168.77.134: Wait for 15s to restart succeeded.
[2020-10-18T17:58:50.555445201+0800]: INFO:    [notice] Please execute the command again!

ERROR Summary: 
  
ACCESS Summary: 
  [command] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --port 22 --version 1.19.3 --offline-file centos7.tgz
  

  See detailed log >>> /tmp/kainstall.8dA57pKTKx/kainstall.log 
```

> 如果执行有错误时，会在`ERROR Summary ` 汇总栏中显示，然后去查看详细日志找出错误原因，以便修正。

因为添加了`--upgrade-kernel` 更新系统内核，所以在内核更新后，需要重启系统应用新内核。

等待集群重启完成后，进入系统查看内核

```bash
# uname -a
Linux base 5.9.1-1.el7.elrepo.x86_64 #1 SMP Fri Oct 16 10:55:30 EDT 2020 x86_64 x86_64 x86_64 GNU/Linux
```

**然后，根据日志提示命令，重新执行脚本进行集群初始化。**

```bash
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --version 1.19.3 \
  --offline-file centos7.tgz
```

执行日志

```bash
[2020-10-18T18:00:51.641665968+0800]: INFO:    [start] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --version 1.19.3 --offline-file centos7.tgz
[2020-10-18T18:00:51.646516966+0800]: INFO:    [check] ssh command exists.
[2020-10-18T18:00:51.648068830+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T18:00:51.650135586+0800]: INFO:    [check] wget command exists.
[2020-10-18T18:00:51.651810955+0800]: INFO:    [check] tar command exists.
[2020-10-18T18:00:51.812440436+0800]: INFO:    [check] ssh 192.168.77.130 connection succeeded.
[2020-10-18T18:00:51.938382652+0800]: INFO:    [check] ssh 192.168.77.131 connection succeeded.
[2020-10-18T18:00:52.071576601+0800]: INFO:    [check] ssh 192.168.77.132 connection succeeded.
[2020-10-18T18:00:52.213264078+0800]: INFO:    [check] ssh 192.168.77.133 connection succeeded.
[2020-10-18T18:00:52.346026578+0800]: INFO:    [check] ssh 192.168.77.134 connection succeeded.
[2020-10-18T18:00:52.349868918+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T18:01:01.688704874+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T18:01:01.690265067+0800]: INFO:    [offline] master 192.168.77.130: load offline file
[2020-10-18T18:01:01.816416626+0800]: INFO:    [offline] 192.168.77.130: mkdir offline dir succeeded.
[2020-10-18T18:01:01.819485922+0800]: INFO:    [offline] master 192.168.77.130: copy offline file
[2020-10-18T18:01:02.525748036+0800]: INFO:    [offline] scp kube file to 192.168.77.130 succeeded.
[2020-10-18T18:01:03.612461494+0800]: INFO:    [offline] scp all file to 192.168.77.130 succeeded.
[2020-10-18T18:01:04.978550197+0800]: INFO:    [offline] scp master images to 192.168.77.130 succeeded.
[2020-10-18T18:01:05.594666731+0800]: INFO:    [offline] scp all images to 192.168.77.130 succeeded.
[2020-10-18T18:01:05.597493436+0800]: INFO:    [offline] master 192.168.77.130: install package
[2020-10-18T18:01:29.551571165+0800]: INFO:    [offline] master 192.168.77.130: install package succeeded.
[2020-10-18T18:01:42.219257013+0800]: INFO:    [offline] 192.168.77.130: load images succeeded.
[2020-10-18T18:01:42.361405866+0800]: INFO:    [offline] 192.168.77.130: clean offline file succeeded.
[2020-10-18T18:01:42.363070534+0800]: INFO:    [offline] master 192.168.77.131: load offline file
[2020-10-18T18:01:42.476112160+0800]: INFO:    [offline] 192.168.77.131: mkdir offline dir succeeded.
[2020-10-18T18:01:42.478169911+0800]: INFO:    [offline] master 192.168.77.131: copy offline file
[2020-10-18T18:01:43.463409622+0800]: INFO:    [offline] scp kube file to 192.168.77.131 succeeded.
[2020-10-18T18:01:45.079246187+0800]: INFO:    [offline] scp all file to 192.168.77.131 succeeded.
[2020-10-18T18:01:46.887937128+0800]: INFO:    [offline] scp master images to 192.168.77.131 succeeded.
[2020-10-18T18:01:47.641855519+0800]: INFO:    [offline] scp all images to 192.168.77.131 succeeded.
[2020-10-18T18:01:47.643321094+0800]: INFO:    [offline] master 192.168.77.131: install package
[2020-10-18T18:02:21.040855099+0800]: INFO:    [offline] master 192.168.77.131: install package succeeded.
[2020-10-18T18:02:39.219135942+0800]: INFO:    [offline] 192.168.77.131: load images succeeded.
[2020-10-18T18:02:39.344169063+0800]: INFO:    [offline] 192.168.77.131: clean offline file succeeded.
[2020-10-18T18:02:39.345716924+0800]: INFO:    [offline] master 192.168.77.132: load offline file
[2020-10-18T18:02:39.454673547+0800]: INFO:    [offline] 192.168.77.132: mkdir offline dir succeeded.
[2020-10-18T18:02:39.456252252+0800]: INFO:    [offline] master 192.168.77.132: copy offline file
[2020-10-18T18:02:40.430450945+0800]: INFO:    [offline] scp kube file to 192.168.77.132 succeeded.
[2020-10-18T18:02:42.049006464+0800]: INFO:    [offline] scp all file to 192.168.77.132 succeeded.
[2020-10-18T18:02:43.493442354+0800]: INFO:    [offline] scp master images to 192.168.77.132 succeeded.
[2020-10-18T18:02:44.264894585+0800]: INFO:    [offline] scp all images to 192.168.77.132 succeeded.
[2020-10-18T18:02:44.266822762+0800]: INFO:    [offline] master 192.168.77.132: install package
[2020-10-18T18:03:12.927560145+0800]: INFO:    [offline] master 192.168.77.132: install package succeeded.
[2020-10-18T18:03:25.557692814+0800]: INFO:    [offline] 192.168.77.132: load images succeeded.
[2020-10-18T18:03:25.736460540+0800]: INFO:    [offline] 192.168.77.132: clean offline file succeeded.
[2020-10-18T18:03:25.738196893+0800]: INFO:    [offline] worker 192.168.77.133: load offline file
[2020-10-18T18:03:25.845499568+0800]: INFO:    [offline] 192.168.77.133: mkdir offline dir succeeded.
[2020-10-18T18:03:25.847396426+0800]: INFO:    [offline] worker 192.168.77.133: copy offline file
[2020-10-18T18:03:26.850058727+0800]: INFO:    [offline] scp kube file to 192.168.77.133 succeeded.
[2020-10-18T18:03:28.640564572+0800]: INFO:    [offline] scp all file to 192.168.77.133 succeeded.
[2020-10-18T18:03:28.768797400+0800]: INFO:    [offline] scp worker file to 192.168.77.133 succeeded.
[2020-10-18T18:03:31.644611323+0800]: INFO:    [offline] scp worker images to 192.168.77.133 succeeded.
[2020-10-18T18:03:32.492342310+0800]: INFO:    [offline] scp all images to 192.168.77.133 succeeded.
[2020-10-18T18:03:32.494053478+0800]: INFO:    [offline] worker 192.168.77.133: install package
[2020-10-18T18:04:01.012115117+0800]: INFO:    [offline] worker 192.168.77.133: install package succeeded.
[2020-10-18T18:04:18.863719345+0800]: INFO:    [offline] 192.168.77.133: load images succeeded.
[2020-10-18T18:04:19.250587940+0800]: INFO:    [offline] 192.168.77.133: clean offline file succeeded.
[2020-10-18T18:04:19.252936712+0800]: INFO:    [offline] worker 192.168.77.134: load offline file
[2020-10-18T18:04:19.382678623+0800]: INFO:    [offline] 192.168.77.134: mkdir offline dir succeeded.
[2020-10-18T18:04:19.387020349+0800]: INFO:    [offline] worker 192.168.77.134: copy offline file
[2020-10-18T18:04:20.360780849+0800]: INFO:    [offline] scp kube file to 192.168.77.134 succeeded.
[2020-10-18T18:04:21.999403626+0800]: INFO:    [offline] scp all file to 192.168.77.134 succeeded.
[2020-10-18T18:04:22.123890841+0800]: INFO:    [offline] scp worker file to 192.168.77.134 succeeded.
[2020-10-18T18:04:24.833011418+0800]: INFO:    [offline] scp worker images to 192.168.77.134 succeeded.
[2020-10-18T18:04:25.575660040+0800]: INFO:    [offline] scp all images to 192.168.77.134 succeeded.
[2020-10-18T18:04:25.577322069+0800]: INFO:    [offline] worker 192.168.77.134: install package
[2020-10-18T18:04:54.909816799+0800]: INFO:    [offline] worker 192.168.77.134: install package succeeded.
[2020-10-18T18:05:20.385195764+0800]: INFO:    [offline] 192.168.77.134: load images succeeded.
[2020-10-18T18:05:21.977597051+0800]: INFO:    [offline] 192.168.77.134: clean offline file succeeded.
[2020-10-18T18:05:21.982316659+0800]: INFO:    [init] master: 192.168.77.130
[2020-10-18T18:05:25.757680992+0800]: INFO:    [init] init master 192.168.77.130 succeeded.
[2020-10-18T18:05:26.474281930+0800]: INFO:    [init] 192.168.77.130 set hostname succeeded.
[2020-10-18T18:05:26.619002757+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.130 hostname resolve succeeded.
[2020-10-18T18:05:26.832646262+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.131 hostname resolve succeeded.
[2020-10-18T18:05:26.951831389+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.132 hostname resolve succeeded.
[2020-10-18T18:05:27.119422614+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.133 hostname resolve succeeded.
[2020-10-18T18:05:27.247035367+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.134 hostname resolve succeeded.
[2020-10-18T18:05:27.250319636+0800]: INFO:    [init] master: 192.168.77.131
[2020-10-18T18:05:31.605352100+0800]: INFO:    [init] init master 192.168.77.131 succeeded.
[2020-10-18T18:05:34.460507941+0800]: INFO:    [init] 192.168.77.131 set hostname succeeded.
[2020-10-18T18:05:34.585966001+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.130 hostname resolve succeeded.
[2020-10-18T18:05:34.709178940+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.131 hostname resolve succeeded.
[2020-10-18T18:05:34.826757277+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.132 hostname resolve succeeded.
[2020-10-18T18:05:34.958073876+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.133 hostname resolve succeeded.
[2020-10-18T18:05:35.131631055+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.134 hostname resolve succeeded.
[2020-10-18T18:05:35.133788135+0800]: INFO:    [init] master: 192.168.77.132
[2020-10-18T18:05:37.347162700+0800]: INFO:    [init] init master 192.168.77.132 succeeded.
[2020-10-18T18:05:38.027498432+0800]: INFO:    [init] 192.168.77.132 set hostname succeeded.
[2020-10-18T18:05:38.145505572+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.130 hostname resolve succeeded.
[2020-10-18T18:05:38.260811273+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.131 hostname resolve succeeded.
[2020-10-18T18:05:38.378261868+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.132 hostname resolve succeeded.
[2020-10-18T18:05:38.488026509+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.133 hostname resolve succeeded.
[2020-10-18T18:05:38.599722344+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.134 hostname resolve succeeded.
[2020-10-18T18:05:38.601467616+0800]: INFO:    [init] worker: 192.168.77.133
[2020-10-18T18:05:41.149146441+0800]: INFO:    [init] init worker 192.168.77.133 succeeded.
[2020-10-18T18:05:41.687413480+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.130 hostname resolve succeeded.
[2020-10-18T18:05:41.891135221+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.131 hostname resolve succeeded.
[2020-10-18T18:05:42.012691572+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.132 hostname resolve succeeded.
[2020-10-18T18:05:42.131046887+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.133 hostname resolve succeeded.
[2020-10-18T18:05:42.241093057+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.134 hostname resolve succeeded.
[2020-10-18T18:05:42.243155517+0800]: INFO:    [init] worker: 192.168.77.134
[2020-10-18T18:05:44.875762726+0800]: INFO:    [init] init worker 192.168.77.134 succeeded.
[2020-10-18T18:05:45.362965567+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.130 hostname resolve succeeded.
[2020-10-18T18:05:45.487128062+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.131 hostname resolve succeeded.
[2020-10-18T18:05:45.603889091+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.132 hostname resolve succeeded.
[2020-10-18T18:05:45.729523800+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.133 hostname resolve succeeded.
[2020-10-18T18:05:45.854246655+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.134 hostname resolve succeeded.
[2020-10-18T18:05:45.856807395+0800]: INFO:    [install] install docker on 192.168.77.130.
[2020-10-18T18:05:49.452927897+0800]: INFO:    [install] install docker on 192.168.77.130 succeeded.
[2020-10-18T18:05:49.454349026+0800]: INFO:    [install] install kube on 192.168.77.130
[2020-10-18T18:05:49.746063731+0800]: INFO:    [install] install kube on 192.168.77.130 succeeded.
[2020-10-18T18:05:49.750284566+0800]: INFO:    [install] install docker on 192.168.77.131.
[2020-10-18T18:05:52.455838267+0800]: INFO:    [install] install docker on 192.168.77.131 succeeded.
[2020-10-18T18:05:52.457465360+0800]: INFO:    [install] install kube on 192.168.77.131
[2020-10-18T18:05:52.752861298+0800]: INFO:    [install] install kube on 192.168.77.131 succeeded.
[2020-10-18T18:05:52.754566275+0800]: INFO:    [install] install docker on 192.168.77.132.
[2020-10-18T18:05:55.743422256+0800]: INFO:    [install] install docker on 192.168.77.132 succeeded.
[2020-10-18T18:05:55.745093588+0800]: INFO:    [install] install kube on 192.168.77.132
[2020-10-18T18:05:56.029917356+0800]: INFO:    [install] install kube on 192.168.77.132 succeeded.
[2020-10-18T18:05:56.031805208+0800]: INFO:    [install] install docker on 192.168.77.133.
[2020-10-18T18:05:59.872730399+0800]: INFO:    [install] install docker on 192.168.77.133 succeeded.
[2020-10-18T18:05:59.874743749+0800]: INFO:    [install] install kube on 192.168.77.133
[2020-10-18T18:06:00.207880853+0800]: INFO:    [install] install kube on 192.168.77.133 succeeded.
[2020-10-18T18:06:00.210035473+0800]: INFO:    [install] install docker on 192.168.77.134.
[2020-10-18T18:06:03.169253150+0800]: INFO:    [install] install docker on 192.168.77.134 succeeded.
[2020-10-18T18:06:03.170952143+0800]: INFO:    [install] install kube on 192.168.77.134
[2020-10-18T18:06:03.447316237+0800]: INFO:    [install] install kube on 192.168.77.134 succeeded.
[2020-10-18T18:06:03.449016747+0800]: INFO:    [install] install haproxy on 192.168.77.133
[2020-10-18T18:06:03.630295386+0800]: INFO:    [install] install haproxy on 192.168.77.133 succeeded.
[2020-10-18T18:06:03.632200979+0800]: INFO:    [install] install haproxy on 192.168.77.134
[2020-10-18T18:06:03.829133534+0800]: INFO:    [install] install haproxy on 192.168.77.134 succeeded.
[2020-10-18T18:06:03.830862951+0800]: INFO:    [kubeadm init] kubeadm init on 192.168.77.130
[2020-10-18T18:06:03.832675780+0800]: INFO:    [kubeadm init] 192.168.77.130: set kubeadmcfg.yaml
[2020-10-18T18:06:03.940682593+0800]: INFO:    [kubeadm init] 192.168.77.130: set kubeadmcfg.yaml succeeded.
[2020-10-18T18:06:03.943461170+0800]: INFO:    [kubeadm init] 192.168.77.130: kubeadm init start.
[2020-10-18T18:06:25.845389567+0800]: INFO:    [kubeadm init] 192.168.77.130: kubeadm init succeeded.
[2020-10-18T18:06:28.849357217+0800]: INFO:    [kubeadm init] 192.168.77.130: set kube config.
[2020-10-18T18:06:29.798465208+0800]: INFO:    [kubeadm init] 192.168.77.130: set kube config succeeded.
[2020-10-18T18:06:30.164401745+0800]: INFO:    [kubeadm init] Auto-Approve kubelet cert csr succeeded.
[2020-10-18T18:06:30.166423567+0800]: INFO:    [kubeadm join] master: get CACRT_HASH
[2020-10-18T18:06:30.480977712+0800]: INFO:    [kubeadm join] master: get INTI_CERTKEY
[2020-10-18T18:06:32.100713596+0800]: INFO:    [kubeadm join] master: get INIT_TOKEN
[2020-10-18T18:06:32.459615146+0800]: INFO:    [kubeadm join] master 192.168.77.131 join cluster.
[2020-10-18T18:07:09.162308049+0800]: INFO:    [kubeadm join] master 192.168.77.131 join cluster succeeded.
[2020-10-18T18:07:09.191372624+0800]: INFO:    [kubeadm join] 192.168.77.131: set kube config.
[2020-10-18T18:07:10.055564197+0800]: INFO:    [kubeadm join] 192.168.77.131: set kube config succeeded.
[2020-10-18T18:07:10.184134471+0800]: INFO:    [kubeadm join] master 192.168.77.132 join cluster.
[2020-10-18T18:07:50.469443226+0800]: INFO:    [kubeadm join] master 192.168.77.132 join cluster succeeded.
[2020-10-18T18:07:50.471958596+0800]: INFO:    [kubeadm join] 192.168.77.132: set kube config.
[2020-10-18T18:07:51.345985019+0800]: INFO:    [kubeadm join] 192.168.77.132: set kube config succeeded.
[2020-10-18T18:07:51.511044583+0800]: INFO:    [kubeadm join] worker 192.168.77.133 join cluster.
[2020-10-18T18:07:59.935323665+0800]: INFO:    [kubeadm join] worker 192.168.77.133 join cluster succeeded.
[2020-10-18T18:07:59.937359685+0800]: INFO:    [kubeadm join] set 192.168.77.133 worker node role.
[2020-10-18T18:08:00.360750357+0800]: INFO:    [kubeadm join] set 192.168.77.133 worker node role succeeded.
[2020-10-18T18:08:00.375156232+0800]: INFO:    [kubeadm join] worker 192.168.77.134 join cluster.
[2020-10-18T18:08:08.570033447+0800]: INFO:    [kubeadm join] worker 192.168.77.134 join cluster succeeded.
[2020-10-18T18:08:08.574971042+0800]: INFO:    [kubeadm join] set 192.168.77.134 worker node role.
[2020-10-18T18:08:08.924406095+0800]: INFO:    [kubeadm join] set 192.168.77.134 worker node role succeeded.
[2020-10-18T18:08:08.929937286+0800]: INFO:    [network] download flannel manifests
[2020-10-18T18:08:08.937822435+0800]: INFO:    [apply] /tmp/kainstall.NNT5bkFwiO/kube-flannel.yml
[2020-10-18T18:08:09.425851082+0800]: INFO:    [apply] add /tmp/kainstall.NNT5bkFwiO/kube-flannel.yml succeeded.
[2020-10-18T18:08:09.429300481+0800]: INFO:    [addon] download metrics-server manifests
[2020-10-18T18:08:09.437455347+0800]: INFO:    [apply] /tmp/kainstall.NNT5bkFwiO/metrics-server.yml
[2020-10-18T18:08:10.526159524+0800]: INFO:    [apply] add /tmp/kainstall.NNT5bkFwiO/metrics-server.yml succeeded.
[2020-10-18T18:08:10.531540677+0800]: INFO:    [ingress] download ingress-nginx manifests
[2020-10-18T18:08:10.590763636+0800]: INFO:    [apply] /tmp/kainstall.NNT5bkFwiO/ingress-nginx.yml
[2020-10-18T18:08:11.867680397+0800]: INFO:    [apply] add /tmp/kainstall.NNT5bkFwiO/ingress-nginx.yml succeeded.
[2020-10-18T18:08:13.872129586+0800]: INFO:    [waiting] waiting ingress-nginx
[2020-10-18T18:08:36.094001306+0800]: INFO:    [waiting] ingress-nginx pod ready succeeded.
[2020-10-18T18:08:39.098984734+0800]: INFO:    [ingress] add ingress default-http-backend
[2020-10-18T18:08:39.103062996+0800]: INFO:    [apply] default-http-backend
[2020-10-18T18:08:39.870450439+0800]: INFO:    [apply] add default-http-backend succeeded.
[2020-10-18T18:08:39.879688704+0800]: INFO:    [ingress] add ingress app demo
[2020-10-18T18:08:39.887992290+0800]: INFO:    [apply] ingress-demo-app
[2020-10-18T18:08:40.391973897+0800]: INFO:    [apply] add ingress-demo-app succeeded.
[2020-10-18T18:08:40.845374133+0800]: INFO:    [ui] add kubernetes dashboard
[2020-10-18T18:08:40.848682763+0800]: INFO:    [ui] download dashboard manifests
[2020-10-18T18:08:40.853330773+0800]: INFO:    [apply] /tmp/kainstall.NNT5bkFwiO/kubernetes-dashboard.yml
[2020-10-18T18:08:41.488114910+0800]: INFO:    [apply] add /tmp/kainstall.NNT5bkFwiO/kubernetes-dashboard.yml succeeded.
[2020-10-18T18:08:42.022859793+0800]: INFO:    [apply] add kubernetes dashboard ingress succeeded.
[2020-10-18T18:08:42.870858170+0800]: INFO:    [ui] create kubernetes dashboard admin service account succeeded.
[2020-10-18T18:08:43.164213556+0800]: INFO:    [ui] get kubernetes dashboard admin token succeeded.
[2020-10-18T18:08:43.168717459+0800]: INFO:    [ops] add etcd snapshot cronjob
[2020-10-18T18:08:43.172897646+0800]: INFO:    [apply] etcd-snapshot
[2020-10-18T18:08:43.580689886+0800]: INFO:    [apply] add etcd-snapshot succeeded.
[2020-10-18T18:08:48.584777813+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-master-node1   Ready    master   2m27s   v1.19.3   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   113s    v1.19.3   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   60s     v1.19.3   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   49s     v1.19.3   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   40s     v1.19.3   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE              NAME                                         READY   STATUS              RESTARTS   AGE
default                ingress-demo-app-78ccc7c466-6g88r            1/1     Running             0          8s
default                ingress-demo-app-78ccc7c466-gxqxl            1/1     Running             0          8s
ingress-nginx          ingress-nginx-admission-create-wglxn         0/1     Completed           0          37s
ingress-nginx          ingress-nginx-admission-patch-jc2zc          0/1     Completed           0          37s
ingress-nginx          ingress-nginx-controller-5fdcfbdbd7-bgl2s    1/1     Running             0          37s
kube-system            coredns-59c898cd69-tzdqv                     1/1     Running             0          2m7s
kube-system            coredns-59c898cd69-vr55t                     1/1     Running             0          2m7s
kube-system            default-http-backend-78894999d9-x2c5b        1/1     Running             0          9s
kube-system            etcd-k8s-master-node1                        1/1     Running             0          2m16s
kube-system            etcd-k8s-master-node2                        1/1     Running             0          112s
kube-system            etcd-k8s-master-node3                        0/1     Running             0          58s
kube-system            kube-apiserver-k8s-master-node1              1/1     Running             0          2m16s
kube-system            kube-apiserver-k8s-master-node2              1/1     Running             0          112s
kube-system            kube-controller-manager-k8s-master-node1     1/1     Running             1          2m16s
kube-system            kube-controller-manager-k8s-master-node2     1/1     Running             0          112s
kube-system            kube-controller-manager-k8s-master-node3     0/1     Pending             0          0s
kube-system            kube-flannel-ds-amd64-65pqv                  1/1     Running             0          39s
kube-system            kube-flannel-ds-amd64-hzwls                  1/1     Running             0          39s
kube-system            kube-flannel-ds-amd64-ncdlq                  1/1     Running             0          39s
kube-system            kube-flannel-ds-amd64-r2dpr                  1/1     Running             0          39s
kube-system            kube-flannel-ds-amd64-vg82v                  1/1     Running             0          39s
kube-system            kube-proxy-fs6k7                             1/1     Running             0          113s
kube-system            kube-proxy-nswm8                             1/1     Running             0          2m7s
kube-system            kube-proxy-nzn9z                             1/1     Running             0          40s
kube-system            kube-proxy-vmgsz                             1/1     Running             0          49s
kube-system            kube-proxy-x2xv2                             1/1     Running             0          60s
kube-system            kube-scheduler-k8s-master-node1              1/1     Running             1          2m16s
kube-system            kube-scheduler-k8s-master-node2              1/1     Running             0          112s
kube-system            metrics-server-5488bc7945-rcmv7              1/1     Running             0          37s
kubernetes-dashboard   dashboard-metrics-scraper-7b59f7d4df-m4zrz   1/1     Running             0          7s
kubernetes-dashboard   kubernetes-dashboard-665f4c5ff-brf94         0/1     ContainerCreating   0          7s
ERROR Summary: 
  
ACCESS Summary: 
  [ingress] curl -H 'Host:app.demo.com' 192.168.77.133:48873
  [ingress] curl -H 'Host:kubernetes-dashboard.cluster.local' http://192.168.77.133:34886
  [Token] eyJhbGciOiJSUzI1NiIsImtpZCI6IjlOUHViSEtya0VNYWt3NDBiM19EVnpJZmY1bUgtR29SXzJ5V2pTSTVNNTQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi1zYS10b2tlbi0ydzVwaiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi1zYSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFjODVjMWQ4LWIzYTgtNDBiNi05YjJmLTQ0MWI4MzM2NzY5MyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi1zYSJ9.dLxZbkikxxqkcMFOt-ewiZi5YHPxlrwxZUqMipoaNvzrwYc7iRdoestS4Ph-Fi6HsjGafyTgVry8RnfayM79Jr-tAdUWh8YOMPTjEZZqOFo8BdJcopKTd7ji_3auDEt2LJeM-98iyRq07g00uI3hVm80-TbSjqS4l2L7S5gPAQLCrSuw53rkbSZtPGvIDHX60f2Xv3k35F19gdVypZIbtGpbhbwVxEEc_wHZ5Apcr-uL-tnUqH5q7R3WxnEppIXMeV2om16v3SZp6bFmBmFO1NK-eVY-y_HYh9M62K5eojsVK398vAEvgTe003NOhXWPS4qRdQ7kAmwD3W_WtUKAag
  

  See detailed log >>> /tmp/kainstall.NNT5bkFwiO/kainstall.log 
```

日志中显示，集群节点已经处于`Ready`, 如果你的不是，请等待一会。

 根据日志，可以看到访问 ingress-demo-app 和 dashboard 的方法

```bash
# curl -H 'Host:app.demo.com' 192.168.77.133:48873
Hostname: ingress-demo-app-78ccc7c466-6g88r
IP: 127.0.0.1
IP: 10.244.3.5
RemoteAddr: 10.244.4.3:34440
GET / HTTP/1.1
Host: app.demo.com
User-Agent: curl/7.29.0
Accept: */*
X-Forwarded-For: 10.244.3.0
X-Forwarded-Host: app.demo.com
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Real-Ip: 10.244.3.0
X-Request-Id: 6e64581fc824138e7621d4992f337410
X-Scheme: http

```

通过网页访问dashboard，并输入提供的token

![kainstall-dashboard](/assets/images/kubernetes/kainstall-dashboard.png)



### 添加节点

在任意 master 节点上操作

**首先，更新内核**

```bash
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3 \
  --upgrade-kernel \
  --offline-file centos7.tgz
```

> master 和 worker 节点可同时添加, 也可以分开

执行日志

```bash
[2020-10-18T18:15:40.400250201+0800]: INFO:    [start] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.3 --upgrade-kernel --offline-file centos7.tgz
[2020-10-18T18:15:40.408998882+0800]: INFO:    [check] ssh command exists.
[2020-10-18T18:15:40.412021302+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T18:15:40.440119635+0800]: INFO:    [check] wget command exists.
[2020-10-18T18:15:40.460242552+0800]: INFO:    [check] tar command exists.
[2020-10-18T18:15:41.522965498+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-10-18T18:15:43.406883854+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-10-18T18:15:43.498819032+0800]: INFO:    [check] conn apiserver succeeded.
[2020-10-18T18:15:43.501395496+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T18:15:58.792129124+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T18:15:58.794845194+0800]: INFO:    [offline] master 192.168.77.135: load offline file
[2020-10-18T18:16:00.446010038+0800]: INFO:    [offline] 192.168.77.135: mkdir offline dir succeeded.
[2020-10-18T18:16:04.433922760+0800]: INFO:    [offline] scp kernel file to 192.168.77.135 succeeded.
[2020-10-18T18:16:04.436073131+0800]: INFO:    [offline] master 192.168.77.135: install package
[2020-10-18T18:17:47.713240031+0800]: INFO:    [offline] master 192.168.77.135: install package succeeded.
[2020-10-18T18:17:47.902955709+0800]: INFO:    [offline] 192.168.77.135: clean offline file succeeded.
[2020-10-18T18:17:47.906697464+0800]: INFO:    [offline] worker 192.168.77.136: load offline file
[2020-10-18T18:17:48.065408296+0800]: INFO:    [offline] 192.168.77.136: mkdir offline dir succeeded.
[2020-10-18T18:17:51.445380172+0800]: INFO:    [offline] scp kernel file to 192.168.77.136 succeeded.
[2020-10-18T18:17:51.447944683+0800]: INFO:    [offline] worker 192.168.77.136: install package
[2020-10-18T18:19:30.406482951+0800]: INFO:    [offline] worker 192.168.77.136: install package succeeded.
[2020-10-18T18:19:30.584164487+0800]: INFO:    [offline] 192.168.77.136: clean offline file succeeded.
[2020-10-18T18:19:30.586313514+0800]: INFO:    [init] upgrade kernel: 192.168.77.135
[2020-10-18T18:19:32.512967664+0800]: INFO:    [init] upgrade kernel 192.168.77.135 succeeded.
[2020-10-18T18:19:32.514968009+0800]: INFO:    [init] upgrade kernel: 192.168.77.136
[2020-10-18T18:19:34.677703880+0800]: INFO:    [init] upgrade kernel 192.168.77.136 succeeded.
[2020-10-18T18:19:34.851881373+0800]: INFO:    [init] 192.168.77.135: Wait for 15s to restart succeeded.
[2020-10-18T18:19:35.013592852+0800]: INFO:    [init] 192.168.77.136: Wait for 15s to restart succeeded.
[2020-10-18T18:19:35.016088410+0800]: INFO:    [notice] Please execute the command again!

ERROR Summary: 
  
ACCESS Summary: 
  [command] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.3 --offline-file centos7.tgz
    

  See detailed log >>> /tmp/kainstall.f0qbybGokn/kainstall.log 

```

**接着，等待节点重启后，进行添加节点操作**

```bash
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.3 \
  --offline-file centos7.tgz
```

执行日志

```bash
[2020-10-18T18:52:45.597386838+0800]: INFO:    [start] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.3 --offline-file centos7.tgz
[2020-10-18T18:52:45.602816829+0800]: INFO:    [check] ssh command exists.
[2020-10-18T18:52:45.604694663+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T18:52:45.606616746+0800]: INFO:    [check] wget command exists.
[2020-10-18T18:52:45.608539007+0800]: INFO:    [check] tar command exists.
[2020-10-18T18:52:45.797108582+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-10-18T18:52:45.967563681+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-10-18T18:52:46.053187758+0800]: INFO:    [check] conn apiserver succeeded.
[2020-10-18T18:52:46.054975322+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T18:52:53.832419500+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T18:52:53.834586082+0800]: INFO:    [offline] master 192.168.77.135: load offline file
[2020-10-18T18:52:53.972850574+0800]: INFO:    [offline] 192.168.77.135: mkdir offline dir succeeded.
[2020-10-18T18:52:53.974760524+0800]: INFO:    [offline] master 192.168.77.135: copy offline file
[2020-10-18T18:52:54.879121696+0800]: INFO:    [offline] scp kube file to 192.168.77.135 succeeded.
[2020-10-18T18:52:56.721896983+0800]: INFO:    [offline] scp all file to 192.168.77.135 succeeded.
[2020-10-18T18:52:58.749584515+0800]: INFO:    [offline] scp master images to 192.168.77.135 succeeded.
[2020-10-18T18:52:59.715401348+0800]: INFO:    [offline] scp all images to 192.168.77.135 succeeded.
[2020-10-18T18:52:59.718796844+0800]: INFO:    [offline] master 192.168.77.135: install package
[2020-10-18T18:53:00.878588205+0800]: INFO:    [offline] master 192.168.77.135: install package succeeded.
[2020-10-18T18:53:07.586862309+0800]: INFO:    [offline] 192.168.77.135: load images succeeded.
[2020-10-18T18:53:07.771871821+0800]: INFO:    [offline] 192.168.77.135: clean offline file succeeded.
[2020-10-18T18:53:07.774597506+0800]: INFO:    [offline] worker 192.168.77.136: load offline file
[2020-10-18T18:53:07.899227690+0800]: INFO:    [offline] 192.168.77.136: mkdir offline dir succeeded.
[2020-10-18T18:53:07.901253758+0800]: INFO:    [offline] worker 192.168.77.136: copy offline file
[2020-10-18T18:53:08.829748280+0800]: INFO:    [offline] scp kube file to 192.168.77.136 succeeded.
[2020-10-18T18:53:10.398743297+0800]: INFO:    [offline] scp all file to 192.168.77.136 succeeded.
[2020-10-18T18:53:10.546975770+0800]: INFO:    [offline] scp worker file to 192.168.77.136 succeeded.
[2020-10-18T18:53:13.557398566+0800]: INFO:    [offline] scp worker images to 192.168.77.136 succeeded.
[2020-10-18T18:53:14.725548620+0800]: INFO:    [offline] scp all images to 192.168.77.136 succeeded.
[2020-10-18T18:53:14.727528400+0800]: INFO:    [offline] worker 192.168.77.136: install package
[2020-10-18T18:53:33.199862683+0800]: INFO:    [offline] worker 192.168.77.136: install package succeeded.
[2020-10-18T18:53:41.921894320+0800]: INFO:    [offline] 192.168.77.136: load images succeeded.
[2020-10-18T18:53:42.058701322+0800]: INFO:    [offline] 192.168.77.136: clean offline file succeeded.
[2020-10-18T18:53:42.333415267+0800]: INFO:    [init] master: 192.168.77.135
[2020-10-18T18:53:42.768963686+0800]: INFO:    [init] init master 192.168.77.135 succeeded.
[2020-10-18T18:53:42.906576969+0800]: INFO:    [init] 192.168.77.135 set hostname and resolve succeeded.
[2020-10-18T18:53:47.261398961+0800]: INFO:    [init] worker: 192.168.77.136
[2020-10-18T18:53:47.974938519+0800]: INFO:    [init] init worker 192.168.77.136 succeeded.
[2020-10-18T18:53:48.123461842+0800]: INFO:    [init] 192.168.77.136 set hostname and resolve succeeded.
[2020-10-18T18:53:48.125436382+0800]: INFO:    [install] install docker on 192.168.77.135.
[2020-10-18T18:53:51.234643362+0800]: INFO:    [install] install docker on 192.168.77.135 succeeded.
[2020-10-18T18:53:51.237040360+0800]: INFO:    [install] install kube on 192.168.77.135
[2020-10-18T18:53:51.536685731+0800]: INFO:    [install] install kube on 192.168.77.135 succeeded.
[2020-10-18T18:53:51.538362014+0800]: INFO:    [install] install docker on 192.168.77.136.
[2020-10-18T18:53:54.372379646+0800]: INFO:    [install] install docker on 192.168.77.136 succeeded.
[2020-10-18T18:53:54.374232226+0800]: INFO:    [install] install kube on 192.168.77.136
[2020-10-18T18:53:54.671053817+0800]: INFO:    [install] install kube on 192.168.77.136 succeeded.
[2020-10-18T18:53:55.084412660+0800]: INFO:    [install] install haproxy on 192.168.77.136
[2020-10-18T18:53:55.325471205+0800]: INFO:    [install] install haproxy on 192.168.77.136 succeeded.
[2020-10-18T18:53:55.327507514+0800]: INFO:    [kubeadm join] master: get CACRT_HASH
[2020-10-18T18:53:55.498850127+0800]: INFO:    [kubeadm join] master: get INTI_CERTKEY
[2020-10-18T18:53:57.079323697+0800]: INFO:    [kubeadm join] master: get INIT_TOKEN
[2020-10-18T18:53:57.416519009+0800]: INFO:    [kubeadm join] master 192.168.77.135 join cluster.
[2020-10-18T18:54:20.163773076+0800]: INFO:    [kubeadm join] master 192.168.77.135 join cluster succeeded.
[2020-10-18T18:54:20.166048717+0800]: INFO:    [kubeadm join] 192.168.77.135: set kube config.
[2020-10-18T18:54:20.337251185+0800]: INFO:    [kubeadm join] 192.168.77.135: set kube config succeeded.
[2020-10-18T18:54:20.513343208+0800]: INFO:    [kubeadm join] worker 192.168.77.136 join cluster.
[2020-10-18T18:54:29.007108805+0800]: INFO:    [kubeadm join] worker 192.168.77.136 join cluster succeeded.
[2020-10-18T18:54:29.010959477+0800]: INFO:    [kubeadm join] set 192.168.77.136 worker node role.
[2020-10-18T18:54:29.376213305+0800]: INFO:    [kubeadm join] set 192.168.77.136 worker node role succeeded.
[2020-10-18T18:54:29.664547838+0800]: INFO:    [del] 192.168.77.133: add apiserver from haproxy
[2020-10-18T18:54:29.811695371+0800]: INFO:    [del] 192.168.77.133: add apiserver(192.168.77.135) from haproxy succeeded.
[2020-10-18T18:54:29.814608074+0800]: INFO:    [del] 192.168.77.134: add apiserver from haproxy
[2020-10-18T18:54:29.964628322+0800]: INFO:    [del] 192.168.77.134: add apiserver(192.168.77.135) from haproxy succeeded.
[2020-10-18T18:54:29.967356743+0800]: INFO:    [del] 192.168.77.136: add apiserver from haproxy
[2020-10-18T18:54:30.205656063+0800]: INFO:    [del] 192.168.77.136: add apiserver(192.168.77.135) from haproxy succeeded.
[2020-10-18T18:54:35.209880703+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-master-node1   Ready    master   48m   v1.19.3   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   47m   v1.19.3   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   46m   v1.19.3   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node4   Ready    master   27s   v1.19.3   192.168.77.135   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   46m   v1.19.3   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   46m   v1.19.3   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node3   Ready    worker   7s    v1.19.3   192.168.77.136   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE              NAME                                         READY   STATUS      RESTARTS   AGE
default                ingress-demo-app-78ccc7c466-6g88r            1/1     Running     0          45m
default                ingress-demo-app-78ccc7c466-gxqxl            1/1     Running     0          45m
ingress-nginx          ingress-nginx-admission-create-wglxn         0/1     Completed   0          46m
ingress-nginx          ingress-nginx-admission-patch-jc2zc          0/1     Completed   0          46m
ingress-nginx          ingress-nginx-controller-5fdcfbdbd7-bgl2s    1/1     Running     0          46m
kube-system            coredns-59c898cd69-tzdqv                     1/1     Running     0          47m
kube-system            coredns-59c898cd69-vr55t                     1/1     Running     0          47m
kube-system            default-http-backend-78894999d9-x2c5b        1/1     Running     0          45m
kube-system            etcd-k8s-master-node1                        1/1     Running     0          48m
kube-system            etcd-k8s-master-node2                        1/1     Running     0          47m
kube-system            etcd-k8s-master-node3                        1/1     Running     0          46m
kube-system            etcd-k8s-master-node4                        0/1     Running     0          25s
kube-system            kube-apiserver-k8s-master-node1              1/1     Running     0          48m
kube-system            kube-apiserver-k8s-master-node2              1/1     Running     0          47m
kube-system            kube-apiserver-k8s-master-node3              1/1     Running     1          45m
kube-system            kube-apiserver-k8s-master-node4              1/1     Running     0          25s
kube-system            kube-controller-manager-k8s-master-node1     1/1     Running     1          48m
kube-system            kube-controller-manager-k8s-master-node2     1/1     Running     0          47m
kube-system            kube-controller-manager-k8s-master-node3     1/1     Running     0          45m
kube-system            kube-controller-manager-k8s-master-node4     0/1     Running     0          25s
kube-system            kube-flannel-ds-amd64-65pqv                  1/1     Running     0          46m
kube-system            kube-flannel-ds-amd64-hzwls                  1/1     Running     0          46m
kube-system            kube-flannel-ds-amd64-ncdlq                  1/1     Running     0          46m
kube-system            kube-flannel-ds-amd64-r2dpr                  1/1     Running     0          46m
kube-system            kube-flannel-ds-amd64-v2lbj                  1/1     Running     0          27s
kube-system            kube-flannel-ds-amd64-vg82v                  1/1     Running     0          46m
kube-system            kube-flannel-ds-amd64-w6g4s                  1/1     Running     0          7s
kube-system            kube-proxy-fs6k7                             1/1     Running     0          47m
kube-system            kube-proxy-nswm8                             1/1     Running     0          47m
kube-system            kube-proxy-nzn9z                             1/1     Running     0          46m
kube-system            kube-proxy-vmgsz                             1/1     Running     0          46m
kube-system            kube-proxy-x2xv2                             1/1     Running     0          46m
kube-system            kube-proxy-xhzhh                             1/1     Running     0          7s
kube-system            kube-proxy-znhr5                             1/1     Running     0          27s
kube-system            kube-scheduler-k8s-master-node1              1/1     Running     1          48m
kube-system            kube-scheduler-k8s-master-node2              1/1     Running     0          47m
kube-system            kube-scheduler-k8s-master-node3              1/1     Running     0          45m
kube-system            kube-scheduler-k8s-master-node4              0/1     Running     0          25s
kube-system            metrics-server-5488bc7945-rcmv7              1/1     Running     0          46m
kubernetes-dashboard   dashboard-metrics-scraper-7b59f7d4df-m4zrz   1/1     Running     0          45m
kubernetes-dashboard   kubernetes-dashboard-665f4c5ff-brf94         1/1     Running     0          45m
ERROR Summary: 
  
ACCESS Summary: 
  

  See detailed log >>> /tmp/kainstall.vkjw42E0u8/kainstall.log 
```

### 其他操作

**注意：** 添加组件时请保持节点的内存和cpu至少为`2C4G`的空闲。否则会导致节点下线且服务器卡死。

> 离线包中不包含以下软件，以下镜像还需自行下载。

```bash
# 添加 traefik ingress
bash kainstall.sh add --ingress traefik

# 添加 elasticsearch
bash kainstall.sh add --log elasticsearch

# 添加 rook
bash kainstall.sh add --storage rook

# 添加 prometheus
kainstall.sh add --monitor prometheus

# 更新证书
kainstall.sh renew-cert

# 更新集群版本
kainstall.sh upgrade --version 1.19.3
 
```



## 最后

是不是很简单，你只需要准备好服务器，操作下脚本就行了，敲一条命令，喝杯咖啡摸摸鱼，集群就好了。这生活岂不美哉，自动化的魅力！。

