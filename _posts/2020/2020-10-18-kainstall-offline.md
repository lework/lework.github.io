---
layout: post
title: "使用 kainstall 离线部署 kubernetes v1.19.3 ha 集群"
date: "2020-10-18 19:40"
category: kubernetes
tags: kubernetes k8s-install
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
  - 配置 `chrony` 时间同步
  - 添加 `ssh-login-info` 信息
  - 配置 `audit` 审计
  - 安装 `ipvs` 模块
  - 更新内核
- 安装`docker`, `kube`组件。
- 初始化`kubernetes`集群,以及增加或删除节点。
- 安装`ingress`组件，可选`nginx`，`traefik`。
- 安装`network`组件，可选`flannel`，`calico`， 需在初始化时指定。
- 安装`monitor`组件，可选`prometheus`。
- 安装`log`组件，可选`elasticsearch`。
- 安装`storage`组件，可选`rook`，`longhorn`。
- 安装`web ui`组件，可选`dashboard`, `kubesphere`。
- 升级到`kubernetes`指定版本。
- 更新集群证书。
- 添加运维操作，如备份etcd快照。
- 支持**离线部署**。
- 支持**sudo特权**。
- 支持**10年证书期限**。

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
  -p,--password        ssh password
     --private-key     ssh private key
  -P,--port            ssh port, default: 22
  -v,--version         kube version, default: latest
  -n,--network         cluster network, choose: [flannel,calico], default: flannel
  -i,--ingress         ingress controller, choose: [nginx,traefik], default: nginx
  -ui,--ui             cluster web ui, choose: [dashboard,kubesphere], default: dashboard
  -M,--monitor         cluster monitor, choose: [prometheus]
  -l,--log             cluster log, choose: [elasticsearch]
  -s,--storage         cluster storage, choose: [rook,longhorn]
  -U,--upgrade-kernel  upgrade kernel
  -of,--offline-file   specify the offline package file to load
      --10years        the certificate period is 10 years.
      --sudo           sudo mode
      --sudo-user      sudo user
      --sudo-password  sudo user password

Example:
  [init cluster]
  kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134,192.168.77.135 \
  --user root \
  --password 123456 \
  --version 1.19.3

  [reset cluster]
  kainstall.sh reset \
  --user root \
  --password 123456

  [add node]
  kainstall.sh add \
  --master 192.168.77.140,192.168.77.141 \
  --worker 192.168.77.143,192.168.77.144 \
  --user root \
  --password 123456 \
  --version 1.19.3

  [del node]
  kainstall.sh del \
  --master 192.168.77.140,192.168.77.141 \
  --worker 192.168.77.143,192.168.77.144 \
  --user root \
  --password 123456
 
  [other]
  kainstall.sh renew-cert --user root --password 123456
  kainstall.sh upgrade --version 1.19.3 --user root --password 123456
  kainstall.sh add --ingress traefik
  kainstall.sh add --monitor prometheus
  kainstall.sh add --log elasticsearch
  kainstall.sh add --storage rook
  kainstall.sh add --ui dashboard
```

### 默认设置

在脚本的开头存放着默认配置，可根据自己需求修改这些。

```bash
# 版本
DOCKER_VERSION="${DOCKER_VERSION:-latest}"
KUBE_VERSION="${KUBE_VERSION:-latest}"
FLANNEL_VERSION="${FLANNEL_VERSION:-0.13.0}"
METRICS_SERVER_VERSION="${METRICS_SERVER_VERSION:-0.3.7}"
INGRESS_NGINX="${INGRESS_NGINX:-0.40.2}"
TRAEFIK_VERSION="${TRAEFIK_VERSION:-2.3.2}"
CALICO_VERSION="${CALICO_VERSION:-3.16.3}"
KUBE_PROMETHEUS_VERSION="${KUBE_PROMETHEUS_VERSION:-0.6.0}"
ELASTICSEARCH_VERSION="${ELASTICSEARCH_VERSION:-7.9.2}"
ROOK_VERSION="${ROOK_VERSION:-1.4.6}"
LONGHORN_VERSION="${LONGHORN_VERSION:-1.0.2}"
KUBERNETES_DASHBOARD_VERSION="${KUBERNETES_DASHBOARD_VERSION:-2.0.4}"
KUBESPHERE_VERSION="${KUBESPHERE_VERSION:-3.0.0}"

# 集群配置
KUBE_APISERVER="${KUBE_APISERVER:-apiserver.cluster.local}"
KUBE_POD_SUBNET="${KUBE_POD_SUBNET:-10.244.0.0/16}"
KUBE_SERVICE_SUBNET="${KUBE_SERVICE_SUBNET:-10.96.0.0/16}"
KUBE_IMAGE_REPO="${KUBE_IMAGE_REPO:-registry.aliyuncs.com/k8sxio}"
KUBE_NETWORK="${KUBE_NETWORK:-flannel}"
KUBE_INGRESS="${KUBE_INGRESS:-nginx}"
KUBE_MONITOR="${KUBE_MONITOR:-prometheus}"
KUBE_STORAGE="${KUBE_STORAGE:-rook}"
KUBE_LOG="${KUBE_LOG:-elasticsearch}"
KUBE_UI="${KUBE_UI:-dashboard}"

# 定义的master和worker节点地址，以逗号分隔
MASTER_NODES="${MASTER_NODES:-}"
WORKER_NODES="${WORKER_NODES:-}"

# 定义在哪个节点上进行设置
MGMT_NODE="${MGMT_NODE:-127.0.0.1}"

# 节点的连接信息
SSH_USER="${SSH_USER:-root}"
SSH_PASSWORD="${SSH_PASSWORD:-}"
SSH_PRIVATE_KEY="${SSH_PRIVATE_KEY:-}"
SSH_PORT="${SSH_PORT:-22}"
SUDO_USER="${SUDO_USER:-root}"

# 节点设置
HOSTNAME_PREFIX="${HOSTNAME_PREFIX:-k8s}"

# 脚本设置
TMP_DIR="$(rm -rf /tmp/kainstall* && mktemp -d -t kainstall.XXXXXXXXXX)"
LOG_FILE="${TMP_DIR}/kainstall.log"
SSH_OPTIONS="-o ConnectTimeout=600 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
ERROR_INFO="\n\033[31mERROR Summary: \033[0m\n  "
ACCESS_INFO="\n\033[32mACCESS Summary: \033[0m\n  "
COMMAND_OUTPUT=""
SCRIPT_PARAMETER="$*"
OFFLINE_DIR="/tmp/kainstall-offline-file/"
OFFLINE_FILE=""
OS_SUPPORT="centos7 centos8"
GITHUB_PROXY="${GITHUB_PROXY:-https://gh.lework.workers.dev/}"
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
[2020-10-18T21:42:09.991605289+0800]: INFO:    [start] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --port 22 --version 1.19.3 --upgrade-kernel --offline-file centos7.tgz
[2020-10-18T21:42:10.045617473+0800]: INFO:    [check] ssh command exists.
[2020-10-18T21:42:10.048010906+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T21:42:10.050417991+0800]: INFO:    [check] wget command exists.
[2020-10-18T21:42:10.052660876+0800]: INFO:    [check] tar command exists.
[2020-10-18T21:42:10.242451328+0800]: INFO:    [check] ssh 192.168.77.130 connection succeeded.
[2020-10-18T21:42:10.723910819+0800]: INFO:    [check] ssh 192.168.77.131 connection succeeded.
[2020-10-18T21:42:11.180985283+0800]: INFO:    [check] ssh 192.168.77.132 connection succeeded.
[2020-10-18T21:42:11.731500574+0800]: INFO:    [check] ssh 192.168.77.133 connection succeeded.
[2020-10-18T21:42:12.226653547+0800]: INFO:    [check] ssh 192.168.77.134 connection succeeded.
[2020-10-18T21:42:12.228322169+0800]: INFO:    [check] os support: centos7 centos8
[2020-10-18T21:42:12.398095318+0800]: INFO:    [check] 192.168.77.130 os support succeeded.
[2020-10-18T21:42:12.597884440+0800]: INFO:    [check] 192.168.77.131 os support succeeded.
[2020-10-18T21:42:12.770756258+0800]: INFO:    [check] 192.168.77.132 os support succeeded.
[2020-10-18T21:42:12.941320567+0800]: INFO:    [check] 192.168.77.133 os support succeeded.
[2020-10-18T21:42:13.120887677+0800]: INFO:    [check] 192.168.77.134 os support succeeded.
[2020-10-18T21:42:13.128865777+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T21:42:20.249161081+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T21:42:20.251740755+0800]: INFO:    [offline] master 192.168.77.130: load offline file
[2020-10-18T21:42:20.451982301+0800]: INFO:    [offline] 192.168.77.130: mkdir offline dir succeeded.
[2020-10-18T21:42:23.554143021+0800]: INFO:    [offline] scp kernel file to 192.168.77.130 succeeded.
[2020-10-18T21:42:23.556736859+0800]: INFO:    [offline] master 192.168.77.130: install package
[2020-10-18T21:43:35.778095231+0800]: INFO:    [offline] master 192.168.77.130: install package succeeded.
[2020-10-18T21:43:35.970087400+0800]: INFO:    [offline] 192.168.77.130: clean offline file succeeded.
[2020-10-18T21:43:35.972149149+0800]: INFO:    [offline] master 192.168.77.131: load offline file
[2020-10-18T21:43:36.169007922+0800]: INFO:    [offline] 192.168.77.131: mkdir offline dir succeeded.
[2020-10-18T21:43:39.146796678+0800]: INFO:    [offline] scp kernel file to 192.168.77.131 succeeded.
[2020-10-18T21:43:39.149157285+0800]: INFO:    [offline] master 192.168.77.131: install package
[2020-10-18T21:44:58.392150628+0800]: INFO:    [offline] master 192.168.77.131: install package succeeded.
[2020-10-18T21:44:58.617949996+0800]: INFO:    [offline] 192.168.77.131: clean offline file succeeded.
[2020-10-18T21:44:58.622343691+0800]: INFO:    [offline] master 192.168.77.132: load offline file
[2020-10-18T21:44:58.799858235+0800]: INFO:    [offline] 192.168.77.132: mkdir offline dir succeeded.
[2020-10-18T21:45:01.432936541+0800]: INFO:    [offline] scp kernel file to 192.168.77.132 succeeded.
[2020-10-18T21:45:01.434979649+0800]: INFO:    [offline] master 192.168.77.132: install package
[2020-10-18T21:46:23.350449761+0800]: INFO:    [offline] master 192.168.77.132: install package succeeded.
[2020-10-18T21:46:23.567690675+0800]: INFO:    [offline] 192.168.77.132: clean offline file succeeded.
[2020-10-18T21:46:23.741204363+0800]: INFO:    [offline] scp manifests file to 192.168.77.130 succeeded.
[2020-10-18T21:46:24.441241749+0800]: INFO:    [offline] scp bins file to 192.168.77.130 succeeded.
[2020-10-18T21:46:24.443510172+0800]: INFO:    [offline] worker 192.168.77.133: load offline file
[2020-10-18T21:46:24.612133596+0800]: INFO:    [offline] 192.168.77.133: mkdir offline dir succeeded.
[2020-10-18T21:46:27.481599149+0800]: INFO:    [offline] scp kernel file to 192.168.77.133 succeeded.
[2020-10-18T21:46:27.483851513+0800]: INFO:    [offline] worker 192.168.77.133: install package
[2020-10-18T21:47:47.142959536+0800]: INFO:    [offline] worker 192.168.77.133: install package succeeded.
[2020-10-18T21:47:47.323802728+0800]: INFO:    [offline] 192.168.77.133: clean offline file succeeded.
[2020-10-18T21:47:47.326079215+0800]: INFO:    [offline] worker 192.168.77.134: load offline file
[2020-10-18T21:47:47.492807185+0800]: INFO:    [offline] 192.168.77.134: mkdir offline dir succeeded.
[2020-10-18T21:47:50.811504365+0800]: INFO:    [offline] scp kernel file to 192.168.77.134 succeeded.
[2020-10-18T21:47:50.813642792+0800]: INFO:    [offline] worker 192.168.77.134: install package
[2020-10-18T21:49:14.433373584+0800]: INFO:    [offline] worker 192.168.77.134: install package succeeded.
[2020-10-18T21:49:14.639421606+0800]: INFO:    [offline] 192.168.77.134: clean offline file succeeded.
[2020-10-18T21:49:14.841252245+0800]: INFO:    [offline] scp manifests file to 192.168.77.130 succeeded.
[2020-10-18T21:49:15.315825065+0800]: INFO:    [offline] scp bins file to 192.168.77.130 succeeded.
[2020-10-18T21:49:15.318612033+0800]: INFO:    [init] upgrade kernel: 192.168.77.130
[2020-10-18T21:49:17.227522701+0800]: INFO:    [init] upgrade kernel 192.168.77.130 succeeded.
[2020-10-18T21:49:17.230234671+0800]: INFO:    [init] upgrade kernel: 192.168.77.131
[2020-10-18T21:49:19.277022344+0800]: INFO:    [init] upgrade kernel 192.168.77.131 succeeded.
[2020-10-18T21:49:19.279160142+0800]: INFO:    [init] upgrade kernel: 192.168.77.132
[2020-10-18T21:49:21.075936241+0800]: INFO:    [init] upgrade kernel 192.168.77.132 succeeded.
[2020-10-18T21:49:21.078309182+0800]: INFO:    [init] upgrade kernel: 192.168.77.133
[2020-10-18T21:49:22.929862866+0800]: INFO:    [init] upgrade kernel 192.168.77.133 succeeded.
[2020-10-18T21:49:22.931795072+0800]: INFO:    [init] upgrade kernel: 192.168.77.134
[2020-10-18T21:49:24.774297569+0800]: INFO:    [init] upgrade kernel 192.168.77.134 succeeded.
[2020-10-18T21:49:24.942734507+0800]: INFO:    [init] 192.168.77.130: Wait for 15s to restart succeeded.
[2020-10-18T21:49:25.132654008+0800]: INFO:    [init] 192.168.77.131: Wait for 15s to restart succeeded.
[2020-10-18T21:49:25.356312411+0800]: INFO:    [init] 192.168.77.132: Wait for 15s to restart succeeded.
[2020-10-18T21:49:25.566742158+0800]: INFO:    [init] 192.168.77.133: Wait for 15s to restart succeeded.
[2020-10-18T21:49:25.732063202+0800]: INFO:    [init] 192.168.77.134: Wait for 15s to restart succeeded.
[2020-10-18T21:49:25.734562241+0800]: INFO:    [notice] Please execute the command again!

ACCESS Summary: 
  [command] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --port 22 --version 1.19.3 --offline-file centos7.tgz
  

  See detailed log >>> /tmp/kainstall.YDB4c8l5Zx/kainstall.log 
```

- `ERROR Summary` 记录错误的信息，在执行有错误时显示
- `ACCESS Summary` 记录用于访问的信息
- `See detailed log`  指出脚本执行的详细日志

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
[2020-10-18T21:51:08.539179453+0800]: INFO:    [start] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --version 1.19.3 --offline-file centos7.tgz
[2020-10-18T21:51:08.544349578+0800]: INFO:    [check] ssh command exists.
[2020-10-18T21:51:08.546149857+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T21:51:08.548333730+0800]: INFO:    [check] wget command exists.
[2020-10-18T21:51:08.550216323+0800]: INFO:    [check] tar command exists.
[2020-10-18T21:51:08.680033422+0800]: INFO:    [check] ssh 192.168.77.130 connection succeeded.
[2020-10-18T21:51:08.833056103+0800]: INFO:    [check] ssh 192.168.77.131 connection succeeded.
[2020-10-18T21:51:08.987923522+0800]: INFO:    [check] ssh 192.168.77.132 connection succeeded.
[2020-10-18T21:51:09.126323254+0800]: INFO:    [check] ssh 192.168.77.133 connection succeeded.
[2020-10-18T21:51:09.286383414+0800]: INFO:    [check] ssh 192.168.77.134 connection succeeded.
[2020-10-18T21:51:09.293131511+0800]: INFO:    [check] os support: centos7 centos8
[2020-10-18T21:51:09.495731287+0800]: INFO:    [check] 192.168.77.130 os support succeeded.
[2020-10-18T21:51:09.651436712+0800]: INFO:    [check] 192.168.77.131 os support succeeded.
[2020-10-18T21:51:09.787117163+0800]: INFO:    [check] 192.168.77.132 os support succeeded.
[2020-10-18T21:51:09.929785125+0800]: INFO:    [check] 192.168.77.133 os support succeeded.
[2020-10-18T21:51:10.050411430+0800]: INFO:    [check] 192.168.77.134 os support succeeded.
[2020-10-18T21:51:10.053980056+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T21:51:16.877370152+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T21:51:16.879105422+0800]: INFO:    [offline] master 192.168.77.130: load offline file
[2020-10-18T21:51:17.029000287+0800]: INFO:    [offline] 192.168.77.130: mkdir offline dir succeeded.
[2020-10-18T21:51:17.030919904+0800]: INFO:    [offline] master 192.168.77.130: copy offline file
[2020-10-18T21:51:17.710969139+0800]: INFO:    [offline] scp kube file to 192.168.77.130 succeeded.
[2020-10-18T21:51:18.806917672+0800]: INFO:    [offline] scp all file to 192.168.77.130 succeeded.
[2020-10-18T21:51:20.292026979+0800]: INFO:    [offline] scp master images to 192.168.77.130 succeeded.
[2020-10-18T21:51:20.985802759+0800]: INFO:    [offline] scp all images to 192.168.77.130 succeeded.
[2020-10-18T21:51:20.988842336+0800]: INFO:    [offline] master 192.168.77.130: install package
[2020-10-18T21:51:50.441953361+0800]: INFO:    [offline] master 192.168.77.130: install package succeeded.
[2020-10-18T21:52:03.078755937+0800]: INFO:    [offline] 192.168.77.130: load images succeeded.
[2020-10-18T21:52:03.235625112+0800]: INFO:    [offline] 192.168.77.130: clean offline file succeeded.
[2020-10-18T21:52:03.238647924+0800]: INFO:    [offline] master 192.168.77.131: load offline file
[2020-10-18T21:52:03.367849950+0800]: INFO:    [offline] 192.168.77.131: mkdir offline dir succeeded.
[2020-10-18T21:52:03.369952603+0800]: INFO:    [offline] master 192.168.77.131: copy offline file
[2020-10-18T21:52:04.424441984+0800]: INFO:    [offline] scp kube file to 192.168.77.131 succeeded.
[2020-10-18T21:52:06.093321285+0800]: INFO:    [offline] scp all file to 192.168.77.131 succeeded.
[2020-10-18T21:52:07.993862796+0800]: INFO:    [offline] scp master images to 192.168.77.131 succeeded.
[2020-10-18T21:52:08.824155283+0800]: INFO:    [offline] scp all images to 192.168.77.131 succeeded.
[2020-10-18T21:52:08.825712188+0800]: INFO:    [offline] master 192.168.77.131: install package
[2020-10-18T21:52:41.073132955+0800]: INFO:    [offline] master 192.168.77.131: install package succeeded.
[2020-10-18T21:52:54.276713409+0800]: INFO:    [offline] 192.168.77.131: load images succeeded.
[2020-10-18T21:52:54.425439749+0800]: INFO:    [offline] 192.168.77.131: clean offline file succeeded.
[2020-10-18T21:52:54.427578253+0800]: INFO:    [offline] master 192.168.77.132: load offline file
[2020-10-18T21:52:54.594371757+0800]: INFO:    [offline] 192.168.77.132: mkdir offline dir succeeded.
[2020-10-18T21:52:54.596104052+0800]: INFO:    [offline] master 192.168.77.132: copy offline file
[2020-10-18T21:52:55.486203819+0800]: INFO:    [offline] scp kube file to 192.168.77.132 succeeded.
[2020-10-18T21:52:57.081853815+0800]: INFO:    [offline] scp all file to 192.168.77.132 succeeded.
[2020-10-18T21:52:58.862609849+0800]: INFO:    [offline] scp master images to 192.168.77.132 succeeded.
[2020-10-18T21:52:59.683067206+0800]: INFO:    [offline] scp all images to 192.168.77.132 succeeded.
[2020-10-18T21:52:59.684659626+0800]: INFO:    [offline] master 192.168.77.132: install package
[2020-10-18T21:53:31.334159317+0800]: INFO:    [offline] master 192.168.77.132: install package succeeded.
[2020-10-18T21:53:44.058064730+0800]: INFO:    [offline] 192.168.77.132: load images succeeded.
[2020-10-18T21:53:44.198155241+0800]: INFO:    [offline] 192.168.77.132: clean offline file succeeded.
[2020-10-18T21:53:44.350241174+0800]: INFO:    [offline] scp manifests file to 192.168.77.130 succeeded.
[2020-10-18T21:53:45.065091203+0800]: INFO:    [offline] scp bins file to 192.168.77.130 succeeded.
[2020-10-18T21:53:45.066808810+0800]: INFO:    [offline] worker 192.168.77.133: load offline file
[2020-10-18T21:53:45.213395983+0800]: INFO:    [offline] 192.168.77.133: mkdir offline dir succeeded.
[2020-10-18T21:53:45.218761688+0800]: INFO:    [offline] worker 192.168.77.133: copy offline file
[2020-10-18T21:53:46.259586173+0800]: INFO:    [offline] scp kube file to 192.168.77.133 succeeded.
[2020-10-18T21:53:47.975509014+0800]: INFO:    [offline] scp all file to 192.168.77.133 succeeded.
[2020-10-18T21:53:48.179467973+0800]: INFO:    [offline] scp worker file to 192.168.77.133 succeeded.
[2020-10-18T21:53:51.147284088+0800]: INFO:    [offline] scp worker images to 192.168.77.133 succeeded.
[2020-10-18T21:53:51.974652519+0800]: INFO:    [offline] scp all images to 192.168.77.133 succeeded.
[2020-10-18T21:53:51.976243508+0800]: INFO:    [offline] worker 192.168.77.133: install package
[2020-10-18T21:54:25.220823168+0800]: INFO:    [offline] worker 192.168.77.133: install package succeeded.
[2020-10-18T21:54:44.277731296+0800]: INFO:    [offline] 192.168.77.133: load images succeeded.
[2020-10-18T21:54:44.543267754+0800]: INFO:    [offline] 192.168.77.133: clean offline file succeeded.
[2020-10-18T21:54:44.545300540+0800]: INFO:    [offline] worker 192.168.77.134: load offline file
[2020-10-18T21:54:44.742257322+0800]: INFO:    [offline] 192.168.77.134: mkdir offline dir succeeded.
[2020-10-18T21:54:44.743957302+0800]: INFO:    [offline] worker 192.168.77.134: copy offline file
[2020-10-18T21:54:45.815976910+0800]: INFO:    [offline] scp kube file to 192.168.77.134 succeeded.
[2020-10-18T21:54:47.430197725+0800]: INFO:    [offline] scp all file to 192.168.77.134 succeeded.
[2020-10-18T21:54:47.648762032+0800]: INFO:    [offline] scp worker file to 192.168.77.134 succeeded.
[2020-10-18T21:54:50.407928664+0800]: INFO:    [offline] scp worker images to 192.168.77.134 succeeded.
[2020-10-18T21:54:51.310691200+0800]: INFO:    [offline] scp all images to 192.168.77.134 succeeded.
[2020-10-18T21:54:51.312541343+0800]: INFO:    [offline] worker 192.168.77.134: install package
[2020-10-18T21:55:26.324625676+0800]: INFO:    [offline] worker 192.168.77.134: install package succeeded.
[2020-10-18T21:56:01.330140626+0800]: INFO:    [offline] 192.168.77.134: load images succeeded.
[2020-10-18T21:56:04.315308937+0800]: INFO:    [offline] 192.168.77.134: clean offline file succeeded.
[2020-10-18T21:56:04.481405871+0800]: INFO:    [offline] scp manifests file to 192.168.77.130 succeeded.
[2020-10-18T21:56:04.891091039+0800]: INFO:    [offline] scp bins file to 192.168.77.130 succeeded.
[2020-10-18T21:56:04.893883241+0800]: INFO:    [init] master: 192.168.77.130
[2020-10-18T21:56:11.366837744+0800]: INFO:    [init] init master 192.168.77.130 succeeded.
[2020-10-18T21:56:14.246269729+0800]: INFO:    [init] 192.168.77.130 set hostname and hostname resolution succeeded.
[2020-10-18T21:56:14.252221846+0800]: INFO:    [init] 192.168.77.130: set audit-policy file.
[2020-10-18T13:56:15.634555723+0800]: INFO:    [init] 192.168.77.130: set audit-policy file succeeded.
[2020-10-18T13:56:15.637998053+0800]: INFO:    [init] master: 192.168.77.131
[2020-10-18T13:56:19.974503246+0800]: INFO:    [init] init master 192.168.77.131 succeeded.
[2020-10-18T13:56:21.877699513+0800]: INFO:    [init] 192.168.77.131 set hostname and hostname resolution succeeded.
[2020-10-18T13:56:21.879676004+0800]: INFO:    [init] 192.168.77.131: set audit-policy file.
[2020-10-18T13:56:22.169729544+0800]: INFO:    [init] 192.168.77.131: set audit-policy file succeeded.
[2020-10-18T13:56:22.171647833+0800]: INFO:    [init] master: 192.168.77.132
[2020-10-18T13:56:26.090562866+0800]: INFO:    [init] init master 192.168.77.132 succeeded.
[2020-10-18T13:56:27.199232457+0800]: INFO:    [init] 192.168.77.132 set hostname and hostname resolution succeeded.
[2020-10-18T13:56:27.201289087+0800]: INFO:    [init] 192.168.77.132: set audit-policy file.
[2020-10-18T13:56:27.545977931+0800]: INFO:    [init] 192.168.77.132: set audit-policy file succeeded.
[2020-10-18T13:56:27.547586872+0800]: INFO:    [init] worker: 192.168.77.133
[2020-10-18T13:56:33.532461365+0800]: INFO:    [init] init worker 192.168.77.133 succeeded.
[2020-10-18T13:56:34.657860701+0800]: INFO:    [init] worker: 192.168.77.134
[2020-10-18T13:56:43.623407687+0800]: INFO:    [init] init worker 192.168.77.134 succeeded.
[2020-10-18T13:56:44.680636763+0800]: INFO:    [install] install docker on 192.168.77.130.
[2020-10-18T13:56:48.512812380+0800]: INFO:    [install] install docker on 192.168.77.130 succeeded.
[2020-10-18T13:56:48.515522061+0800]: INFO:    [install] install kube on 192.168.77.130
[2020-10-18T13:56:49.238638170+0800]: INFO:    [install] install kube on 192.168.77.130 succeeded.
[2020-10-18T13:56:49.240970424+0800]: INFO:    [install] install docker on 192.168.77.131.
[2020-10-18T13:56:52.418778002+0800]: INFO:    [install] install docker on 192.168.77.131 succeeded.
[2020-10-18T13:56:52.420416590+0800]: INFO:    [install] install kube on 192.168.77.131
[2020-10-18T13:56:52.986138495+0800]: INFO:    [install] install kube on 192.168.77.131 succeeded.
[2020-10-18T13:56:52.987881051+0800]: INFO:    [install] install docker on 192.168.77.132.
[2020-10-18T13:56:56.449328201+0800]: INFO:    [install] install docker on 192.168.77.132 succeeded.
[2020-10-18T13:56:56.451446936+0800]: INFO:    [install] install kube on 192.168.77.132
[2020-10-18T13:56:57.051724786+0800]: INFO:    [install] install kube on 192.168.77.132 succeeded.
[2020-10-18T13:56:57.054021108+0800]: INFO:    [install] install docker on 192.168.77.133.
[2020-10-18T13:57:00.767132752+0800]: INFO:    [install] install docker on 192.168.77.133 succeeded.
[2020-10-18T13:57:00.768729975+0800]: INFO:    [install] install kube on 192.168.77.133
[2020-10-18T13:57:01.329373990+0800]: INFO:    [install] install kube on 192.168.77.133 succeeded.
[2020-10-18T13:57:01.331097243+0800]: INFO:    [install] install docker on 192.168.77.134.
[2020-10-18T13:57:05.597148573+0800]: INFO:    [install] install docker on 192.168.77.134 succeeded.
[2020-10-18T13:57:05.598718700+0800]: INFO:    [install] install kube on 192.168.77.134
[2020-10-18T13:57:06.226108156+0800]: INFO:    [install] install kube on 192.168.77.134 succeeded.
[2020-10-18T13:57:06.228258415+0800]: INFO:    [install] install haproxy on 192.168.77.133
[2020-10-18T13:57:06.579791485+0800]: INFO:    [install] install haproxy on 192.168.77.133 succeeded.
[2020-10-18T13:57:06.581912132+0800]: INFO:    [install] install haproxy on 192.168.77.134
[2020-10-18T13:57:06.925802358+0800]: INFO:    [install] install haproxy on 192.168.77.134 succeeded.
[2020-10-18T13:57:06.928318827+0800]: INFO:    [kubeadm init] kubeadm init on 192.168.77.130
[2020-10-18T13:57:06.930726448+0800]: INFO:    [kubeadm init] 192.168.77.130: set kubeadmcfg.yaml
[2020-10-18T13:57:07.425830585+0800]: INFO:    [kubeadm init] 192.168.77.130: set kubeadmcfg.yaml succeeded.
[2020-10-18T13:57:07.431101382+0800]: INFO:    [kubeadm init] 192.168.77.130: kubeadm init start.
[2020-10-18T13:57:30.606310299+0800]: INFO:    [kubeadm init] 192.168.77.130: kubeadm init succeeded.
[2020-10-18T13:57:33.610003004+0800]: INFO:    [kubeadm init] 192.168.77.130: set kube config.
[2020-10-18T13:57:34.146568449+0800]: INFO:    [kubeadm init] 192.168.77.130: set kube config succeeded.
[2020-10-18T13:57:34.703751547+0800]: INFO:    [kubeadm init] Auto-Approve kubelet cert csr succeeded.
[2020-10-18T13:57:34.707748023+0800]: INFO:    [kubeadm join] master: get join token and cert info
[2020-10-18T13:57:35.062412159+0800]: INFO:    [command] get CACRT_HASH value succeeded.
[2020-10-18T13:57:36.658265368+0800]: INFO:    [command] get INTI_CERTKEY value succeeded.
[2020-10-18T13:57:37.251803647+0800]: INFO:    [command] get INIT_TOKEN value succeeded.
[2020-10-18T13:57:37.254365069+0800]: INFO:    [kubeadm join] master 192.168.77.131 join cluster.
[2020-10-18T13:58:13.159858723+0800]: INFO:    [kubeadm join] master 192.168.77.131 join cluster succeeded.
[2020-10-18T13:58:13.165775671+0800]: INFO:    [kubeadm join] 192.168.77.131: set kube config.
[2020-10-18T13:58:13.820488067+0800]: INFO:    [kubeadm join] 192.168.77.131: set kube config succeeded.
[2020-10-18T13:58:14.203650706+0800]: INFO:    [kubeadm join] master 192.168.77.132 join cluster.
[2020-10-18T13:58:53.940537678+0800]: INFO:    [kubeadm join] master 192.168.77.132 join cluster succeeded.
[2020-10-18T13:58:53.943256036+0800]: INFO:    [kubeadm join] 192.168.77.132: set kube config.
[2020-10-18T13:58:54.643852500+0800]: INFO:    [kubeadm join] 192.168.77.132: set kube config succeeded.
[2020-10-18T13:58:55.004027619+0800]: INFO:    [kubeadm join] worker 192.168.77.133 join cluster.
[2020-10-18T13:59:03.013022818+0800]: INFO:    [kubeadm join] worker 192.168.77.133 join cluster succeeded.
[2020-10-18T13:59:03.016824096+0800]: INFO:    [kubeadm join] set 192.168.77.133 worker node role.
[2020-10-18T13:59:03.663889419+0800]: INFO:    [kubeadm join] set 192.168.77.133 worker node role succeeded.
[2020-10-18T13:59:03.667271210+0800]: INFO:    [kubeadm join] worker 192.168.77.134 join cluster.
[2020-10-18T13:59:12.194455638+0800]: INFO:    [kubeadm join] worker 192.168.77.134 join cluster succeeded.
[2020-10-18T13:59:12.197607689+0800]: INFO:    [kubeadm join] set 192.168.77.134 worker node role.
[2020-10-18T13:59:12.815763213+0800]: INFO:    [kubeadm join] set 192.168.77.134 worker node role succeeded.
[2020-10-18T13:59:12.819391043+0800]: INFO:    [network] add flannel
[2020-10-18T13:59:12.838444350+0800]: INFO:    [download] kube-flannel.yml
[2020-10-18T13:59:13.152383331+0800]: INFO:    [download] kube-flannel.yml succeeded.
[2020-10-18T13:59:13.451558977+0800]: INFO:    [flannel] change flannel pod subnet succeeded.
[2020-10-18T13:59:13.453465923+0800]: INFO:    [apply] /tmp/kainstall-offline-file//manifests/kube-flannel.yml
[2020-10-18T13:59:14.060178621+0800]: INFO:    [apply] add /tmp/kainstall-offline-file//manifests/kube-flannel.yml succeeded.
[2020-10-18T13:59:14.062512993+0800]: INFO:    [addon] download metrics-server manifests
[2020-10-18T13:59:14.067971377+0800]: INFO:    [download] metrics-server.yml
[2020-10-18T13:59:14.421245726+0800]: INFO:    [download] metrics-server.yml succeeded.
[2020-10-18T13:59:14.858800505+0800]: INFO:    [addon] change metrics-server parameter succeeded.
[2020-10-18T13:59:14.863092844+0800]: INFO:    [apply] /tmp/kainstall-offline-file//manifests/metrics-server.yml
[2020-10-18T13:59:15.950478749+0800]: INFO:    [apply] add /tmp/kainstall-offline-file//manifests/metrics-server.yml succeeded.
[2020-10-18T13:59:15.966622676+0800]: INFO:    [ingress] add ingress-nginx
[2020-10-18T13:59:15.994750091+0800]: INFO:    [download] ingress-nginx.yml
[2020-10-18T13:59:16.641426756+0800]: INFO:    [download] ingress-nginx.yml succeeded.
[2020-10-18T13:59:17.200347107+0800]: INFO:    [ingress] change ingress-nginx manifests succeeded.
[2020-10-18T13:59:17.203416240+0800]: INFO:    [apply] /tmp/kainstall-offline-file//manifests/ingress-nginx.yml
[2020-10-18T13:59:18.301341007+0800]: INFO:    [apply] add /tmp/kainstall-offline-file//manifests/ingress-nginx.yml succeeded.
[2020-10-18T13:59:21.308149626+0800]: INFO:    [waiting] waiting ingress-nginx
[2020-10-18T13:59:39.485401752+0800]: INFO:    [waiting] ingress-nginx pod ready succeeded.
[2020-10-18T13:59:42.494994636+0800]: INFO:    [ingress] add ingress default-http-backend
[2020-10-18T13:59:42.497669955+0800]: INFO:    [apply] default-http-backend
[2020-10-18T13:59:43.081525736+0800]: INFO:    [apply] add default-http-backend succeeded.
[2020-10-18T13:59:43.084439240+0800]: INFO:    [ingress] add ingress app demo
[2020-10-18T13:59:43.093182748+0800]: INFO:    [apply] ingress-demo-app
[2020-10-18T13:59:43.805771280+0800]: INFO:    [apply] add ingress-demo-app succeeded.
[2020-10-18T13:59:44.251696117+0800]: INFO:    [command] get node_ip value succeeded.
[2020-10-18T13:59:44.776322329+0800]: INFO:    [command] get node_port value succeeded.
[2020-10-18T13:59:44.781769928+0800]: INFO:    [ui] add kubernetes dashboard
[2020-10-18T13:59:44.787873395+0800]: INFO:    [download] kubernetes-dashboard.yml
[2020-10-18T13:59:45.099422339+0800]: INFO:    [download] kubernetes-dashboard.yml succeeded.
[2020-10-18T13:59:45.102741741+0800]: INFO:    [apply] /tmp/kainstall-offline-file//manifests/kubernetes-dashboard.yml
[2020-10-18T13:59:45.825561002+0800]: INFO:    [apply] add /tmp/kainstall-offline-file//manifests/kubernetes-dashboard.yml succeeded.
[2020-10-18T13:59:45.836095789+0800]: INFO:    [apply] kubernetes dashboard ingress
[2020-10-18T13:59:46.547949116+0800]: INFO:    [apply] add kubernetes dashboard ingress succeeded.
[2020-10-18T13:59:46.975136425+0800]: INFO:    [command] get node_ip value succeeded.
[2020-10-18T13:59:47.486750357+0800]: INFO:    [command] get node_port value succeeded.
[2020-10-18T13:59:48.146034030+0800]: INFO:    [ui] create kubernetes dashboard admin service account succeeded.
[2020-10-18T13:59:48.667999747+0800]: INFO:    [command] get dashboard_token value succeeded.
[2020-10-18T13:59:48.674452079+0800]: INFO:    [ops] add etcd snapshot cronjob
[2020-10-18T13:59:49.124515516+0800]: INFO:    [command] get etcd_image value succeeded.
[2020-10-18T13:59:49.128495317+0800]: INFO:    [apply] etcd-snapshot
[2020-10-18T13:59:49.624364122+0800]: INFO:    [apply] add etcd-snapshot succeeded.
[2020-10-18T13:59:54.631774235+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-master-node1   Ready    master   2m29s   v1.19.3   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   115s    v1.19.3   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   64s     v1.19.3   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   53s     v1.19.3   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   44s     v1.19.3   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE              NAME                                         READY   STATUS              RESTARTS   AGE
default                ingress-demo-app-78ccc7c466-gqc7g            1/1     Running             0          12s
default                ingress-demo-app-78ccc7c466-r4s7w            1/1     Running             0          12s
ingress-nginx          ingress-nginx-admission-create-g8jkh         0/1     Completed           0          37s
ingress-nginx          ingress-nginx-admission-patch-6jtwd          0/1     Completed           0          37s
ingress-nginx          ingress-nginx-controller-69b8c77547-gl5wg    1/1     Running             0          37s
kube-system            coredns-59c898cd69-gfc9w                     1/1     Running             0          2m10s
kube-system            coredns-59c898cd69-sddmr                     1/1     Running             0          2m10s
kube-system            default-http-backend-78894999d9-nn8bt        1/1     Running             0          12s
kube-system            etcd-k8s-master-node1                        1/1     Running             0          2m19s
kube-system            etcd-k8s-master-node2                        1/1     Running             0          111s
kube-system            kube-apiserver-k8s-master-node1              1/1     Running             0          2m19s
kube-system            kube-apiserver-k8s-master-node2              1/1     Running             0          114s
kube-system            kube-controller-manager-k8s-master-node1     1/1     Running             1          2m19s
kube-system            kube-controller-manager-k8s-master-node2     1/1     Running             0          114s
kube-system            kube-controller-manager-k8s-master-node3     1/1     Running             0          15s
kube-system            kube-flannel-ds-cvggq                        1/1     Running             0          41s
kube-system            kube-flannel-ds-gf5c2                        1/1     Running             0          41s
kube-system            kube-flannel-ds-plzsm                        1/1     Running             0          41s
kube-system            kube-flannel-ds-t5hbv                        1/1     Running             0          41s
kube-system            kube-flannel-ds-zbrsk                        1/1     Running             0          41s
kube-system            kube-proxy-f4665                             1/1     Running             0          44s
kube-system            kube-proxy-flpfk                             1/1     Running             0          2m10s
kube-system            kube-proxy-g2bsp                             1/1     Running             0          53s
kube-system            kube-proxy-tbtbg                             1/1     Running             0          115s
kube-system            kube-proxy-zz5dv                             1/1     Running             0          64s
kube-system            kube-scheduler-k8s-master-node1              1/1     Running             1          2m19s
kube-system            kube-scheduler-k8s-master-node2              1/1     Running             0          114s
kube-system            kube-scheduler-k8s-master-node3              1/1     Running             0          14s
kube-system            metrics-server-57b9d596cc-rkmfd              1/1     Running             0          39s
kubernetes-dashboard   dashboard-metrics-scraper-7b59f7d4df-7wnd7   1/1     Running             0          10s
kubernetes-dashboard   kubernetes-dashboard-665f4c5ff-tv2s5         0/1     ContainerCreating   0          10s
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

**注意：** 这里需要绑定下主机名解析,将下面的内容放入到`hosts`文件中

```bash
192.168.77.133 kubernetes-dashboard.cluster.local
```

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
[2020-10-18T14:10:20.653045713+0800]: INFO:    [start] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.3 --upgrade-kernel --offline-file centos7.tgz
[2020-10-18T14:10:20.658968844+0800]: INFO:    [check] ssh command exists.
[2020-10-18T14:10:20.660855327+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T14:10:20.674528824+0800]: INFO:    [check] wget command exists.
[2020-10-18T14:10:20.694267744+0800]: INFO:    [check] tar command exists.
[2020-10-18T14:10:21.150752973+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-10-18T14:10:21.657667614+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-10-18T14:10:21.667550504+0800]: INFO:    [check] os support: centos7 centos8
[2020-10-18T14:10:21.840945814+0800]: INFO:    [check] 192.168.77.135 os support succeeded.
[2020-10-18T14:10:22.014314743+0800]: INFO:    [check] 192.168.77.136 os support succeeded.
[2020-10-18T14:10:22.296882718+0800]: INFO:    [check] conn apiserver succeeded.
[2020-10-18T14:10:22.298928416+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T14:10:35.338142207+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T14:10:35.975947744+0800]: INFO:    [offline] master 192.168.77.135: load offline file
[2020-10-18T14:10:36.468118059+0800]: INFO:    [offline] 192.168.77.135: mkdir offline dir succeeded.
[2020-10-18T14:10:41.414214558+0800]: INFO:    [offline] scp kernel file to 192.168.77.135 succeeded.
[2020-10-18T14:10:41.417093064+0800]: INFO:    [offline] master 192.168.77.135: install package
[2020-10-18T14:12:12.572258150+0800]: INFO:    [offline] master 192.168.77.135: install package succeeded.
[2020-10-18T14:12:12.766348336+0800]: INFO:    [offline] 192.168.77.135: clean offline file succeeded.
[2020-10-18T14:12:12.777464566+0800]: INFO:    [offline] scp manifests file to 127.0.0.1 succeeded.
[2020-10-18T14:12:12.845309404+0800]: INFO:    [offline] scp bins file to 127.0.0.1 succeeded.
[2020-10-18T14:12:12.847304634+0800]: INFO:    [offline] worker 192.168.77.136: load offline file
[2020-10-18T14:12:13.020949799+0800]: INFO:    [offline] 192.168.77.136: mkdir offline dir succeeded.
[2020-10-18T14:12:16.132723034+0800]: INFO:    [offline] scp kernel file to 192.168.77.136 succeeded.
[2020-10-18T14:12:16.135669266+0800]: INFO:    [offline] worker 192.168.77.136: install package
[2020-10-18T14:13:55.490464733+0800]: INFO:    [offline] worker 192.168.77.136: install package succeeded.
[2020-10-18T14:13:55.681027773+0800]: INFO:    [offline] 192.168.77.136: clean offline file succeeded.
[2020-10-18T14:13:55.689767598+0800]: INFO:    [offline] scp manifests file to 127.0.0.1 succeeded.
[2020-10-18T14:13:55.759109974+0800]: INFO:    [offline] scp bins file to 127.0.0.1 succeeded.
[2020-10-18T14:13:55.761067975+0800]: INFO:    [init] upgrade kernel: 192.168.77.135
[2020-10-18T14:13:57.505316156+0800]: INFO:    [init] upgrade kernel 192.168.77.135 succeeded.
[2020-10-18T14:13:57.507752513+0800]: INFO:    [init] upgrade kernel: 192.168.77.136
[2020-10-18T14:13:59.525797083+0800]: INFO:    [init] upgrade kernel 192.168.77.136 succeeded.
[2020-10-18T14:13:59.697026513+0800]: INFO:    [init] 192.168.77.135: Wait for 15s to restart succeeded.
[2020-10-18T14:13:59.866398263+0800]: INFO:    [init] 192.168.77.136: Wait for 15s to restart succeeded.
[2020-10-18T14:13:59.868817958+0800]: INFO:    [notice] Please execute the command again!

ACCESS Summary: 
  [command] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.3 --offline-file centos7.tgz
  

  See detailed log >>> /tmp/kainstall.bXsMiOCiqb/kainstall.log 
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
[2020-10-18T14:16:06.417507582+0800]: INFO:    [start] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.3 --offline-file centos7.tgz
[2020-10-18T14:16:06.422920094+0800]: INFO:    [check] ssh command exists.
[2020-10-18T14:16:06.424908186+0800]: INFO:    [check] sshpass command exists.
[2020-10-18T14:16:06.427047771+0800]: INFO:    [check] wget command exists.
[2020-10-18T14:16:06.429630788+0800]: INFO:    [check] tar command exists.
[2020-10-18T14:16:06.569872324+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-10-18T14:16:06.719132312+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-10-18T14:16:06.720982620+0800]: INFO:    [check] os support: centos7 centos8
[2020-10-18T14:16:06.851460338+0800]: INFO:    [check] 192.168.77.135 os support succeeded.
[2020-10-18T14:16:06.980814222+0800]: INFO:    [check] 192.168.77.136 os support succeeded.
[2020-10-18T14:16:07.138515101+0800]: INFO:    [check] conn apiserver succeeded.
[2020-10-18T14:16:07.140841336+0800]: INFO:    [offline] Unzip offline package on local.
[2020-10-18T14:16:14.923737134+0800]: INFO:    [offline] Unzip offline package succeeded.
[2020-10-18T14:16:14.926541590+0800]: INFO:    [offline] master 192.168.77.135: load offline file
[2020-10-18T14:16:15.078073207+0800]: INFO:    [offline] 192.168.77.135: mkdir offline dir succeeded.
[2020-10-18T14:16:15.079967998+0800]: INFO:    [offline] master 192.168.77.135: copy offline file
[2020-10-18T14:16:16.160275215+0800]: INFO:    [offline] scp kube file to 192.168.77.135 succeeded.
[2020-10-18T14:16:18.028812808+0800]: INFO:    [offline] scp all file to 192.168.77.135 succeeded.
[2020-10-18T14:16:20.135428808+0800]: INFO:    [offline] scp master images to 192.168.77.135 succeeded.
[2020-10-18T14:16:21.285284367+0800]: INFO:    [offline] scp all images to 192.168.77.135 succeeded.
[2020-10-18T14:16:21.288061030+0800]: INFO:    [offline] master 192.168.77.135: install package
[2020-10-18T14:16:58.833438603+0800]: INFO:    [offline] master 192.168.77.135: install package succeeded.
[2020-10-18T14:17:13.175976318+0800]: INFO:    [offline] 192.168.77.135: load images succeeded.
[2020-10-18T14:17:13.337820519+0800]: INFO:    [offline] 192.168.77.135: clean offline file succeeded.
[2020-10-18T14:17:13.350726640+0800]: INFO:    [offline] scp manifests file to 127.0.0.1 succeeded.
[2020-10-18T14:17:13.424018191+0800]: INFO:    [offline] scp bins file to 127.0.0.1 succeeded.
[2020-10-18T14:17:13.426559619+0800]: INFO:    [offline] worker 192.168.77.136: load offline file
[2020-10-18T14:17:13.604385417+0800]: INFO:    [offline] 192.168.77.136: mkdir offline dir succeeded.
[2020-10-18T14:17:13.606559541+0800]: INFO:    [offline] worker 192.168.77.136: copy offline file
[2020-10-18T14:17:14.589604511+0800]: INFO:    [offline] scp kube file to 192.168.77.136 succeeded.
[2020-10-18T14:17:16.199852534+0800]: INFO:    [offline] scp all file to 192.168.77.136 succeeded.
[2020-10-18T14:17:16.347005064+0800]: INFO:    [offline] scp worker file to 192.168.77.136 succeeded.
[2020-10-18T14:17:19.682769578+0800]: INFO:    [offline] scp worker images to 192.168.77.136 succeeded.
[2020-10-18T14:17:20.789042994+0800]: INFO:    [offline] scp all images to 192.168.77.136 succeeded.
[2020-10-18T14:17:20.792106919+0800]: INFO:    [offline] worker 192.168.77.136: install package
[2020-10-18T14:17:56.386076474+0800]: INFO:    [offline] worker 192.168.77.136: install package succeeded.
[2020-10-18T14:18:17.541650635+0800]: INFO:    [offline] 192.168.77.136: load images succeeded.
[2020-10-18T14:18:17.714584367+0800]: INFO:    [offline] 192.168.77.136: clean offline file succeeded.
[2020-10-18T14:18:17.723100877+0800]: INFO:    [offline] scp manifests file to 127.0.0.1 succeeded.
[2020-10-18T14:18:17.812906102+0800]: INFO:    [offline] scp bins file to 127.0.0.1 succeeded.
[2020-10-18T14:18:18.300699059+0800]: INFO:    [command] get MGMT_NODE value succeeded.
[2020-10-18T14:18:25.758224621+0800]: INFO:    [command] get node_hosts value succeeded.
[2020-10-18T14:18:26.572992334+0800]: INFO:    [command] get master_index value succeeded.
[2020-10-18T14:18:26.997009500+0800]: INFO:    [command] get worker_index value succeeded.
[2020-10-18T14:18:27.333385254+0800]: INFO:    [init] 192.168.77.130 add new node hostname resolution succeeded.
[2020-10-18T14:18:27.773465680+0800]: INFO:    [init] 192.168.77.131 add new node hostname resolution succeeded.
[2020-10-18T14:18:28.369733389+0800]: INFO:    [init] 192.168.77.132 add new node hostname resolution succeeded.
[2020-10-18T14:18:29.107446713+0800]: INFO:    [init] 192.168.77.133 add new node hostname resolution succeeded.
[2020-10-18T14:18:29.617971389+0800]: INFO:    [init] 192.168.77.134 add new node hostname resolution succeeded.
[2020-10-18T14:18:29.620266271+0800]: INFO:    [init] master: 192.168.77.135
[2020-10-18T14:18:48.601528633+0800]: INFO:    [init] init master 192.168.77.135 succeeded.
[2020-10-18T14:18:49.141843830+0800]: INFO:    [init] 192.168.77.135 set hostname and hostname resolution succeeded.
[2020-10-18T14:18:49.143791055+0800]: INFO:    [init] 192.168.77.135: set audit-policy file.
[2020-10-18T14:18:49.419293235+0800]: INFO:    [init] 192.168.77.135: set audit-policy file succeeded.
[2020-10-18T14:18:49.421821454+0800]: INFO:    [init] worker: 192.168.77.136
[2020-10-18T14:19:14.028259409+0800]: INFO:    [init] init worker 192.168.77.136 succeeded.
[2020-10-18T14:19:14.491764382+0800]: INFO:    [install] install docker on 192.168.77.135.
[2020-10-18T14:19:17.887969455+0800]: INFO:    [install] install docker on 192.168.77.135 succeeded.
[2020-10-18T14:19:17.892198861+0800]: INFO:    [install] install kube on 192.168.77.135
[2020-10-18T14:19:18.444299897+0800]: INFO:    [install] install kube on 192.168.77.135 succeeded.
[2020-10-18T14:19:18.446644340+0800]: INFO:    [install] install docker on 192.168.77.136.
[2020-10-18T14:19:21.955198579+0800]: INFO:    [install] install docker on 192.168.77.136 succeeded.
[2020-10-18T14:19:21.957091068+0800]: INFO:    [install] install kube on 192.168.77.136
[2020-10-18T14:19:22.578405133+0800]: INFO:    [install] install kube on 192.168.77.136 succeeded.
[2020-10-18T14:19:23.033902451+0800]: INFO:    [command] get apiservers value succeeded.
[2020-10-18T14:19:23.036083295+0800]: INFO:    [install] install haproxy on 192.168.77.136
[2020-10-18T14:19:23.398123957+0800]: INFO:    [install] install haproxy on 192.168.77.136 succeeded.
[2020-10-18T14:19:23.400599345+0800]: INFO:    [kubeadm join] master: get join token and cert info
[2020-10-18T14:19:23.730072438+0800]: INFO:    [command] get CACRT_HASH value succeeded.
[2020-10-18T14:19:25.757056775+0800]: INFO:    [command] get INTI_CERTKEY value succeeded.
[2020-10-18T14:19:26.268372756+0800]: INFO:    [command] get INIT_TOKEN value succeeded.
[2020-10-18T14:19:26.271329665+0800]: INFO:    [kubeadm join] master 192.168.77.135 join cluster.
[2020-10-18T14:19:44.161476875+0800]: INFO:    [kubeadm join] master 192.168.77.135 join cluster succeeded.
[2020-10-18T14:19:44.163729239+0800]: INFO:    [kubeadm join] 192.168.77.135: set kube config.
[2020-10-18T14:19:44.631128982+0800]: INFO:    [kubeadm join] 192.168.77.135: set kube config succeeded.
[2020-10-18T14:19:45.078285873+0800]: INFO:    [kubeadm join] worker 192.168.77.136 join cluster.
[2020-10-18T14:19:53.700319800+0800]: INFO:    [kubeadm join] worker 192.168.77.136 join cluster succeeded.
[2020-10-18T14:19:53.703167849+0800]: INFO:    [kubeadm join] set 192.168.77.136 worker node role.
[2020-10-18T14:19:54.393071916+0800]: INFO:    [kubeadm join] set 192.168.77.136 worker node role succeeded.
[2020-10-18T14:19:59.411002169+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
k8s-master-node1   Ready    master   22m   v1.19.3   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   21m   v1.19.3   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   21m   v1.19.3   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node4   Ready    master   22s   v1.19.3   192.168.77.135   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   20m   v1.19.3   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   20m   v1.19.3   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node3   Ready    worker   6s    v1.19.3   192.168.77.136   <none>        CentOS Linux 7 (Core)   5.9.1-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE              NAME                                         READY   STATUS      RESTARTS   AGE
default                ingress-demo-app-78ccc7c466-gqc7g            1/1     Running     0          20m
default                ingress-demo-app-78ccc7c466-r4s7w            1/1     Running     0          20m
ingress-nginx          ingress-nginx-admission-create-g8jkh         0/1     Completed   0          20m
ingress-nginx          ingress-nginx-admission-patch-6jtwd          0/1     Completed   0          20m
ingress-nginx          ingress-nginx-controller-69b8c77547-gl5wg    1/1     Running     0          20m
kube-system            coredns-59c898cd69-gfc9w                     1/1     Running     0          22m
kube-system            coredns-59c898cd69-sddmr                     1/1     Running     0          22m
kube-system            default-http-backend-78894999d9-nn8bt        1/1     Running     0          20m
kube-system            etcd-k8s-master-node1                        1/1     Running     0          22m
kube-system            etcd-k8s-master-node2                        1/1     Running     0          21m
kube-system            etcd-k8s-master-node3                        1/1     Running     0          19m
kube-system            etcd-k8s-master-node4                        0/1     Running     0          20s
kube-system            kube-apiserver-k8s-master-node1              1/1     Running     0          22m
kube-system            kube-apiserver-k8s-master-node2              1/1     Running     0          21m
kube-system            kube-apiserver-k8s-master-node3              1/1     Running     1          19m
kube-system            kube-apiserver-k8s-master-node4              1/1     Running     0          21s
kube-system            kube-controller-manager-k8s-master-node1     1/1     Running     1          22m
kube-system            kube-controller-manager-k8s-master-node2     1/1     Running     0          21m
kube-system            kube-controller-manager-k8s-master-node3     1/1     Running     0          20m
kube-system            kube-controller-manager-k8s-master-node4     0/1     Running     0          21s
kube-system            kube-flannel-ds-cml5t                        1/1     Running     0          7s
kube-system            kube-flannel-ds-cvggq                        1/1     Running     0          20m
kube-system            kube-flannel-ds-gf5c2                        1/1     Running     0          20m
kube-system            kube-flannel-ds-plzsm                        1/1     Running     0          20m
kube-system            kube-flannel-ds-t5hbv                        1/1     Running     0          20m
kube-system            kube-flannel-ds-zbd9f                        1/1     Running     0          23s
kube-system            kube-flannel-ds-zbrsk                        1/1     Running     0          20m
kube-system            kube-proxy-f4665                             1/1     Running     0          20m
kube-system            kube-proxy-flpfk                             1/1     Running     0          22m
kube-system            kube-proxy-g2bsp                             1/1     Running     0          20m
kube-system            kube-proxy-jtchb                             1/1     Running     0          23s
kube-system            kube-proxy-qj8qg                             1/1     Running     0          7s
kube-system            kube-proxy-tbtbg                             1/1     Running     0          22m
kube-system            kube-proxy-zz5dv                             1/1     Running     0          21m
kube-system            kube-scheduler-k8s-master-node1              1/1     Running     1          22m
kube-system            kube-scheduler-k8s-master-node2              1/1     Running     0          21m
kube-system            kube-scheduler-k8s-master-node3              1/1     Running     0          20m
kube-system            kube-scheduler-k8s-master-node4              0/1     Running     0          21s
kube-system            metrics-server-57b9d596cc-rkmfd              1/1     Running     0          20m
kubernetes-dashboard   dashboard-metrics-scraper-7b59f7d4df-7wnd7   1/1     Running     0          20m
kubernetes-dashboard   kubernetes-dashboard-665f4c5ff-tv2s5         1/1     Running     0          20m

  See detailed log >>> /tmp/kainstall.apVkPCLmf8/kainstall.log 
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

