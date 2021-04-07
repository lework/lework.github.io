---
layout: post
title: "使用 kainstall 工具一键部署 kubernetes 高可用集群"
date: "2020-09-26 14:40"
category: kubernetes
tags: kubernetes k8s-install
author: lework
---
* content
{:toc}

今天给大家介绍一款工具:  [kainstall](https://github.com/lework/kainstall) 一个由纯bash脚本编写的工具。可一键部署 kubernetes 高可用集群，增删节点，管理k8s集群变得省时省力。

话不多说，请看下面介绍和案例使用



## 工具介绍

### Github

[github.com/lework/kainstall](github.com/lework/kainstall)

### 功能

- 服务器初始化。
  - 关闭 `selinux`
  - 关闭 `swap`
  - 关闭 `firewalld`
  - 配置 `epel` 源
  - 修改 `limits`
  - 配置内核参数
  - 配置 `history` 记录
  - 配置 `journal` 日志
  - 配置 `chrony` 时间同步
  - 安装 `ipvs` 模块
  - 更新内核
- 安装 `docker`, `kube`组件。
- 初始化 `kubernetes` 集群,以及增加或删除节点。
- 安装 `ingress` 组件，可选 `nginx` ，`traefik`。
- 安装 `network` 组件，可选 `flannel`，`calico`， 需在初始化时指定。
- 安装 `monitor` 组件，可选 `prometheus`。
- 安装 `log`组件，可选 `elasticsearch`。
- 安装 `storage` 组件，可选 `rook`。
- 升级到 `kubernetes` 指定版本。
- 添加运维操作，如备份etcd快照。

### 帮助信息

```bash
bash kainstall.sh

Install kubernetes cluster using kubeadm.

Usage:
  kainstall.sh [command]

Available Commands:
  init            init Kubernetes cluster.
  reset           reset Kubernetes cluster.
  add             add nodes to the cluster.
  del             remove node from the cluster.
  upgrade         Upgrading kubeadm clusters.

Flag:
  -m,--master          master node, default: ''
  -w,--worker          work node, default: ''
  -u,--user            ssh user, default: root
  -p,--password        ssh password,default: 123456
  -P,--port            ssh port, default: 22
  -v,--version         kube version, default: latest
  -n,--network         cluster network, choose: [flannel,calico], default: flannel
  -i,--ingress         ingress controller, choose: [nginx,traefik], default: nginx
  -M,--monitor         cluster monitor, choose: [prometheus]
  -l,--log             cluster log, choose: [elasticsearch]
  -s,--storage         cluster storage, choose: [rook]
  -U,--upgrade-kernel  upgrade kernel

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
  kainstall.sh upgrade --version 1.19.2
  kainstall.sh add --ingress traefik
  kainstall.sh add --monitor prometheus
  kainstall.sh add --log elasticsearch
  kainstall.sh add --storage rook


ERROR Summary: 
  
ACCESS Summary: 
  

  See detailed log >>> /tmp/kainstall.BUpFzDT7FZ/kainstall.log 

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
TRAEFIK_VERSION="2.3.0"
CALICO_VERSION="3.16.1"
KUBE_PROMETHEUS_VERSION="0.6.0"
ELASTICSEARCH_VERSION="7.9.2"
ROOK_VERSION="1.3.11"
 
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
```

## 案例

### 节点列表

| name    | ip             | os         | cpu  | mem  | role   |
| ------- | -------------- | ---------- | ---- | ---- | ------ |
| node130 | 192.168.77.130 | CentOS 7.7 | 2C   | 2G   | master |
| node131 | 192.168.77.131 | CentOS 7.7 | 2C   | 2G   | master |
| node132 | 192.168.77.132 | CentOS 7.7 | 2C   | 2G   | master |
| node133 | 192.168.77.133 | CentOS 7.7 | 2C   | 4G   | worker |
| node134 | 192.168.77.134 | CentOS 7.7 | 2C   | 4G   | worker |
| node135 | 192.168.77.135 | CentOS 7.7 | 2C   | 4G   | worker |
| node136 | 192.168.77.136 | CentOS 7.7 | 2C   | 2G   | master |



### 拓扑架构

![k8s-node-ha](/assets/images/kubernetes/k8s-node-ha.png)

> 如需按照步骤安装集群，可参考 https://lework.github.io/2019/10/01/kubeadm-install/

### 初始化集群

在集群master节点中的一台机器上执行

```bash
wget https://cdn.jsdelivr.net/gh/lework/kainstall/kainstall.sh

bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.1 \
  --upgrade-kernel
```

执行日志

```bash
[2020-09-26T19:34:26.459489744+0800]: INFO:    [check] ssh command exists.
[2020-09-26T19:34:26.461036250+0800]: WARNING: [check] I require sshpass but it's not installed.
[2020-09-26T19:34:26.462759018+0800]: WARNING: [check] install sshpass package.
[2020-09-26T19:34:41.397206233+0800]: INFO:    [check] wget command exists.
[2020-09-26T19:34:41.670216283+0800]: INFO:    [check] ssh 192.168.77.130 connection succeeded.
[2020-09-26T19:34:42.306829816+0800]: INFO:    [check] ssh 192.168.77.131 connection succeeded.
[2020-09-26T19:34:42.701455725+0800]: INFO:    [check] ssh 192.168.77.132 connection succeeded.
[2020-09-26T19:34:43.178118541+0800]: INFO:    [check] ssh 192.168.77.133 connection succeeded.
[2020-09-26T19:34:43.703409982+0800]: INFO:    [check] ssh 192.168.77.134 connection succeeded.
[2020-09-26T19:34:43.709025501+0800]: INFO:    [init] upgrade kernel: 192.168.77.130
[2020-09-26T19:36:26.701116185+0800]: INFO:    [init] upgrade kernel 192.168.77.130 succeeded.
[2020-09-26T19:36:26.704679444+0800]: INFO:    [init] upgrade kernel: 192.168.77.131
[2020-09-26T19:38:17.347449947+0800]: INFO:    [init] upgrade kernel 192.168.77.131 succeeded.
[2020-09-26T19:38:17.349094402+0800]: INFO:    [init] upgrade kernel: 192.168.77.132
[2020-09-26T19:40:27.150739054+0800]: INFO:    [init] upgrade kernel 192.168.77.132 succeeded.
[2020-09-26T19:40:27.152675454+0800]: INFO:    [init] upgrade kernel: 192.168.77.133
[2020-09-26T19:42:26.281733266+0800]: INFO:    [init] upgrade kernel 192.168.77.133 succeeded.
[2020-09-26T19:42:26.283591784+0800]: INFO:    [init] upgrade kernel: 192.168.77.134
[2020-09-26T19:44:09.543560100+0800]: INFO:    [init] upgrade kernel 192.168.77.134 succeeded.
[2020-09-26T19:44:09.702391495+0800]: INFO:    [init] 192.168.77.130: Wait for 10s to restart succeeded.
[2020-09-26T19:44:09.856233007+0800]: INFO:    [init] 192.168.77.131: Wait for 10s to restart succeeded.
[2020-09-26T19:44:10.031792288+0800]: INFO:    [init] 192.168.77.132: Wait for 10s to restart succeeded.
[2020-09-26T19:44:10.193391432+0800]: INFO:    [init] 192.168.77.133: Wait for 10s to restart succeeded.
[2020-09-26T19:44:10.365485041+0800]: INFO:    [init] 192.168.77.134: Wait for 10s to restart succeeded.
[2020-09-26T19:44:10.367602186+0800]: INFO:    [notice] Please execute the command again!
[2020-09-26T19:44:10.369296613+0800]: INFO:    [cmd] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --version 1.19.1 

ERROR Summary: 
  
ACCESS Summary: 
  [cmd] bash kainstall.sh init --master 192.168.77.130,192.168.77.131,192.168.77.132 --worker 192.168.77.133,192.168.77.134 --user root --password 123456 --version 1.19.1 
  

  See detailed log >>> /tmp/kainstall.tLZPFD0eXr/kainstall.log 
```

> 如果执行有错误时，会在`ERROR Summary ` 汇总栏中显示，然后去查看详细日志找出错误原因，以便修正。

因为添加了`--upgrade-kernel` 更新系统内核，所以在内核更新后，需要重启系统应用新内核。

等待集群重启完成后，进入系统查看内核

```bash
# uname -a
Linux node130 5.8.12-1.el7.elrepo.x86_64 #1 SMP Fri Sep 25 17:49:38 EDT 2020 x86_64 x86_64 x86_64 GNU/Linux
```

根据日志提示命令，重新执行脚本进行初始化。

```bash
bash kainstall.sh init \
  --master 192.168.77.130,192.168.77.131,192.168.77.132 \
  --worker 192.168.77.133,192.168.77.134 \
  --user root \
  --password 123456 \
  --version 1.19.1
```

执行日志

```bash
[2020-09-26T19:46:25.946622259+0800]: INFO:    [check] ssh command exists.
[2020-09-26T19:46:25.948277694+0800]: INFO:    [check] sshpass command exists.
[2020-09-26T19:46:25.950110724+0800]: INFO:    [check] wget command exists.
[2020-09-26T19:46:26.076943829+0800]: INFO:    [check] ssh 192.168.77.130 connection succeeded.
[2020-09-26T19:46:26.232390735+0800]: INFO:    [check] ssh 192.168.77.131 connection succeeded.
[2020-09-26T19:46:26.392081683+0800]: INFO:    [check] ssh 192.168.77.132 connection succeeded.
[2020-09-26T19:46:26.560956819+0800]: INFO:    [check] ssh 192.168.77.133 connection succeeded.
[2020-09-26T19:46:26.725249860+0800]: INFO:    [check] ssh 192.168.77.134 connection succeeded.
[2020-09-26T19:46:26.736265421+0800]: INFO:    [init] master: 192.168.77.130
[2020-09-26T19:46:37.918863784+0800]: INFO:    [init] init master 192.168.77.130 succeeded.
[2020-09-26T19:46:38.274476436+0800]: INFO:    [init] 192.168.77.130 set hostname succeeded.
[2020-09-26T19:46:38.402683420+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.130 hostname resolve succeeded.
[2020-09-26T19:46:38.525377766+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.131 hostname resolve succeeded.
[2020-09-26T19:46:38.637292863+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.132 hostname resolve succeeded.
[2020-09-26T19:46:38.760145422+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.133 hostname resolve succeeded.
[2020-09-26T19:46:38.889994751+0800]: INFO:    [init] 192.168.77.130: add 192.168.77.134 hostname resolve succeeded.
[2020-09-26T19:46:38.894336889+0800]: INFO:    [init] master: 192.168.77.131
[2020-09-26T11:46:53.376421587+0800]: INFO:    [init] init master 192.168.77.131 succeeded.
[2020-09-26T11:46:53.547822393+0800]: INFO:    [init] 192.168.77.131 set hostname succeeded.
[2020-09-26T11:46:53.665028713+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.130 hostname resolve succeeded.
[2020-09-26T11:46:53.778767613+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.131 hostname resolve succeeded.
[2020-09-26T11:46:53.923914169+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.132 hostname resolve succeeded.
[2020-09-26T11:46:54.047473290+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.133 hostname resolve succeeded.
[2020-09-26T11:46:54.194096170+0800]: INFO:    [init] 192.168.77.131: add 192.168.77.134 hostname resolve succeeded.
[2020-09-26T11:46:54.197494179+0800]: INFO:    [init] master: 192.168.77.132
[2020-09-26T11:47:09.159827418+0800]: INFO:    [init] init master 192.168.77.132 succeeded.
[2020-09-26T11:47:09.315204650+0800]: INFO:    [init] 192.168.77.132 set hostname succeeded.
[2020-09-26T11:47:09.446887217+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.130 hostname resolve succeeded.
[2020-09-26T11:47:09.678959207+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.131 hostname resolve succeeded.
[2020-09-26T11:47:09.799634337+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.132 hostname resolve succeeded.
[2020-09-26T11:47:09.930902704+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.133 hostname resolve succeeded.
[2020-09-26T11:47:10.046943191+0800]: INFO:    [init] 192.168.77.132: add 192.168.77.134 hostname resolve succeeded.
[2020-09-26T11:47:10.048554165+0800]: INFO:    [init] worker: 192.168.77.133
[2020-09-26T11:47:32.159253479+0800]: INFO:    [init] init worker 192.168.77.133 succeeded.
[2020-09-26T11:47:32.652773670+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.130 hostname resolve succeeded.
[2020-09-26T11:47:32.876000925+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.131 hostname resolve succeeded.
[2020-09-26T11:47:33.046043614+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.132 hostname resolve succeeded.
[2020-09-26T11:47:33.222017099+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.133 hostname resolve succeeded.
[2020-09-26T11:47:33.354227725+0800]: INFO:    [init] 192.168.77.133: add 192.168.77.134 hostname resolve succeeded.
[2020-09-26T11:47:33.357053654+0800]: INFO:    [init] worker: 192.168.77.134
[2020-09-26T11:47:57.322105894+0800]: INFO:    [init] init worker 192.168.77.134 succeeded.
[2020-09-26T11:47:57.623716596+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.130 hostname resolve succeeded.
[2020-09-26T11:47:57.765056044+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.131 hostname resolve succeeded.
[2020-09-26T11:47:57.899200522+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.132 hostname resolve succeeded.
[2020-09-26T11:47:58.021554473+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.133 hostname resolve succeeded.
[2020-09-26T11:47:58.183393828+0800]: INFO:    [init] 192.168.77.134: add 192.168.77.134 hostname resolve succeeded.
[2020-09-26T11:47:58.185286660+0800]: INFO:    [install] install docker on 192.168.77.130.
[2020-09-26T11:48:35.916945860+0800]: INFO:    [install] install docker on 192.168.77.130 succeeded.
[2020-09-26T11:48:35.921387143+0800]: INFO:    [install] install kube on 192.168.77.130
[2020-09-26T11:49:02.706121316+0800]: INFO:    [install] install kube on 192.168.77.130 succeeded.
[2020-09-26T11:49:02.707542107+0800]: INFO:    [install] install docker on 192.168.77.131.
[2020-09-26T11:49:41.025354511+0800]: INFO:    [install] install docker on 192.168.77.131 succeeded.
[2020-09-26T11:49:41.027644188+0800]: INFO:    [install] install kube on 192.168.77.131
[2020-09-26T11:50:06.424983791+0800]: INFO:    [install] install kube on 192.168.77.131 succeeded.
[2020-09-26T11:50:06.426483383+0800]: INFO:    [install] install docker on 192.168.77.132.
[2020-09-26T11:50:43.758318647+0800]: INFO:    [install] install docker on 192.168.77.132 succeeded.
[2020-09-26T11:50:43.760462914+0800]: INFO:    [install] install kube on 192.168.77.132
[2020-09-26T11:51:11.098803542+0800]: INFO:    [install] install kube on 192.168.77.132 succeeded.
[2020-09-26T11:51:11.100417166+0800]: INFO:    [install] install docker on 192.168.77.133.
[2020-09-26T11:51:51.310424725+0800]: INFO:    [install] install docker on 192.168.77.133 succeeded.
[2020-09-26T11:51:51.312599081+0800]: INFO:    [install] install kube on 192.168.77.133
[2020-09-26T11:52:22.777088620+0800]: INFO:    [install] install kube on 192.168.77.133 succeeded.
[2020-09-26T11:52:22.778935700+0800]: INFO:    [install] install docker on 192.168.77.134.
[2020-09-26T11:52:58.535362604+0800]: INFO:    [install] install docker on 192.168.77.134 succeeded.
[2020-09-26T11:52:58.537367098+0800]: INFO:    [install] install kube on 192.168.77.134
[2020-09-26T11:53:27.279733828+0800]: INFO:    [install] install kube on 192.168.77.134 succeeded.
[2020-09-26T11:53:27.281412913+0800]: INFO:    [install] install haproxy on 192.168.77.133
[2020-09-26T11:53:30.047661397+0800]: INFO:    [install] install haproxy on 192.168.77.133 succeeded.
[2020-09-26T11:53:30.049763650+0800]: INFO:    [install] install haproxy on 192.168.77.134
[2020-09-26T11:53:32.374731522+0800]: INFO:    [install] install haproxy on 192.168.77.134 succeeded.
[2020-09-26T11:53:32.377325440+0800]: INFO:    [kubeadm init] kubeadm init on 192.168.77.130
[2020-09-26T11:53:32.379493352+0800]: INFO:    [kubeadm init] 192.168.77.130: set kubeadmcfg.yaml
[2020-09-26T11:53:32.491013753+0800]: INFO:    [kubeadm init] 192.168.77.130: set kubeadmcfg.yaml succeeded.
[2020-09-26T11:53:32.495493161+0800]: INFO:    [kubeadm init] 192.168.77.130: kubeadm init start.
[2020-09-26T11:55:00.626003854+0800]: INFO:    [kubeadm init] 192.168.77.130: kubeadm init succeeded.
[2020-09-26T11:55:03.632757780+0800]: INFO:    [kubeadm init] 192.168.77.130: set kube config.
[2020-09-26T11:55:03.809875845+0800]: INFO:    [kubeadm init] 192.168.77.130: set kube config succeeded.
[2020-09-26T11:55:03.812377275+0800]: INFO:    [kubeadm join] master: get CACRT_HASH
[2020-09-26T11:55:03.935890567+0800]: INFO:    [kubeadm join] master: get INTI_CERTKEY
[2020-09-26T11:55:05.276641010+0800]: INFO:    [kubeadm join] master: get INIT_TOKEN
[2020-09-26T11:55:05.553460941+0800]: INFO:    [kubeadm join] master 192.168.77.131 join cluster.
[2020-09-26T11:56:55.625479574+0800]: INFO:    [kubeadm join] master 192.168.77.131 join cluster succeeded.
[2020-09-26T11:56:55.629529983+0800]: INFO:    [kubeadm join] 192.168.77.131: set kube config.
[2020-09-26T11:56:55.933395661+0800]: INFO:    [kubeadm join] 192.168.77.131: set kube config succeeded.
[2020-09-26T11:56:56.094497175+0800]: INFO:    [kubeadm join] master 192.168.77.132 join cluster.
[2020-09-26T11:58:19.070833078+0800]: INFO:    [kubeadm join] master 192.168.77.132 join cluster succeeded.
[2020-09-26T11:58:19.072853190+0800]: INFO:    [kubeadm join] 192.168.77.132: set kube config.
[2020-09-26T11:58:19.600490688+0800]: INFO:    [kubeadm join] 192.168.77.132: set kube config succeeded.
[2020-09-26T11:58:19.772673379+0800]: INFO:    [kubeadm join] worker 192.168.77.133 join cluster.
[2020-09-26T11:58:28.101625880+0800]: INFO:    [kubeadm join] worker 192.168.77.133 join cluster succeeded.
[2020-09-26T11:58:28.103397011+0800]: INFO:    [kubeadm join] set 192.168.77.133 worker node role.
[2020-09-26T11:58:28.560512519+0800]: INFO:    [kubeadm join] set 192.168.77.133 worker node role succeeded.
[2020-09-26T11:58:28.563285607+0800]: INFO:    [kubeadm join] worker 192.168.77.134 join cluster.
[2020-09-26T11:58:37.563822532+0800]: INFO:    [kubeadm join] worker 192.168.77.134 join cluster succeeded.
[2020-09-26T11:58:37.565662188+0800]: INFO:    [kubeadm join] set 192.168.77.134 worker node role.
[2020-09-26T11:58:37.898761094+0800]: INFO:    [kubeadm join] set 192.168.77.134 worker node role succeeded.
[2020-09-26T11:58:37.901845530+0800]: INFO:    [network] download flannel manifests
[2020-09-26T11:58:37.989451181+0800]: INFO:    [apply] /tmp/kainstall.ytdInAV7HY/kube-flannel.yml
[2020-09-26T11:58:38.505798491+0800]: INFO:    [apply] add /tmp/kainstall.ytdInAV7HY/kube-flannel.yml succeeded.
[2020-09-26T11:58:38.515945316+0800]: INFO:    [addon] download metrics-server manifests
[2020-09-26T11:58:40.176114199+0800]: INFO:    [apply] /tmp/kainstall.ytdInAV7HY/metrics-server.yml
[2020-09-26T11:58:40.841404313+0800]: INFO:    [apply] add /tmp/kainstall.ytdInAV7HY/metrics-server.yml succeeded.
[2020-09-26T11:58:40.847441032+0800]: INFO:    [ingress] download ingress-nginx manifests
[2020-09-26T11:58:40.918537008+0800]: INFO:    [apply] /tmp/kainstall.ytdInAV7HY/ingress-nginx.yml
[2020-09-26T11:58:42.119990528+0800]: INFO:    [apply] add /tmp/kainstall.ytdInAV7HY/ingress-nginx.yml succeeded.
[2020-09-26T11:58:42.123026214+0800]: INFO:    [waiting] waiting ingress-nginx
[2020-09-26T12:00:41.212906179+0800]: INFO:    [waiting] ingress-nginx pod ready succeeded.
[2020-09-26T12:00:44.231255486+0800]: INFO:    [ingress] add ingress app demo
[2020-09-26T12:00:44.236614945+0800]: INFO:    [apply] ingress-demo-app
[2020-09-26T12:00:46.130477868+0800]: INFO:    [apply] add ingress-demo-app succeeded.
[2020-09-26T12:00:46.597676812+0800]: INFO:    [ops] add etcd snapshot cronjob
[2020-09-26T12:00:46.599542801+0800]: INFO:    [apply] etcd-snapshot
[2020-09-26T12:00:46.919659982+0800]: INFO:    [apply] add etcd-snapshot succeeded.
[2020-09-26T12:00:51.924281184+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master-node1   Ready    master   5m56s   v1.19.1   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   4m15s   v1.19.1   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   2m39s   v1.19.1   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   2m25s   v1.19.1   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   2m15s   v1.19.1   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
default         ingress-demo-app-589b64597f-bdm8d           1/1     Running     0          8s
default         ingress-demo-app-589b64597f-rm5q7           1/1     Running     0          8s
ingress-nginx   ingress-nginx-admission-create-lm7qj        0/1     Completed   0          2m10s
ingress-nginx   ingress-nginx-admission-patch-zcbtn         0/1     Completed   0          2m10s
ingress-nginx   ingress-nginx-controller-859658cbfd-xrqff   1/1     Running     0          2m11s
kube-system     coredns-59c898cd69-97bjz                    1/1     Running     0          5m36s
kube-system     coredns-59c898cd69-qw5s7                    1/1     Running     0          5m36s
kube-system     etcd-k8s-master-node1                       1/1     Running     0          5m46s
kube-system     etcd-k8s-master-node2                       1/1     Running     0          4m14s
kube-system     etcd-k8s-master-node3                       1/1     Running     0          2m37s
kube-system     kube-apiserver-k8s-master-node1             1/1     Running     0          5m46s
kube-system     kube-apiserver-k8s-master-node2             1/1     Running     0          4m15s
kube-system     kube-apiserver-k8s-master-node3             1/1     Running     0          2m37s
kube-system     kube-controller-manager-k8s-master-node1    1/1     Running     1          5m46s
kube-system     kube-controller-manager-k8s-master-node2    1/1     Running     0          4m14s
kube-system     kube-controller-manager-k8s-master-node3    1/1     Running     0          2m38s
kube-system     kube-flannel-ds-amd64-5mv5r                 1/1     Running     0          2m14s
kube-system     kube-flannel-ds-amd64-7sf8s                 1/1     Running     0          2m14s
kube-system     kube-flannel-ds-amd64-kfhbf                 1/1     Running     0          2m14s
kube-system     kube-flannel-ds-amd64-lj8rb                 1/1     Running     0          2m14s
kube-system     kube-flannel-ds-amd64-qmcdm                 1/1     Running     0          2m14s
kube-system     kube-proxy-bqzxq                            1/1     Running     0          2m25s
kube-system     kube-proxy-bxdc5                            1/1     Running     0          2m15s
kube-system     kube-proxy-dz9gz                            1/1     Running     0          5m36s
kube-system     kube-proxy-fzwvb                            1/1     Running     0          4m15s
kube-system     kube-proxy-zrmr8                            1/1     Running     0          2m39s
kube-system     kube-scheduler-k8s-master-node1             1/1     Running     1          5m46s
kube-system     kube-scheduler-k8s-master-node2             1/1     Running     0          4m14s
kube-system     kube-scheduler-k8s-master-node3             1/1     Running     0          2m38s
kube-system     metrics-server-5488bc7945-d84wv             1/1     Running     0          2m11s
ERROR Summary: 
  
ACCESS Summary: 
  [ingress] curl -H 'Host:app.demo.com' 192.168.77.133:32366
  

  See detailed log >>> /tmp/kainstall.ytdInAV7HY/kainstall.log 
```

日志中显示，集群节点已经处于`Ready`, 如果你的不是，请等待一会。

 根据日志，可以看到访问ingress-demo-app的方法

```bash
curl -H 'Host:app.demo.com' 192.168.77.133:32366

Hostname: ingress-demo-app-589b64597f-bdm8d
IP: 127.0.0.1
IP: 10.244.4.4
RemoteAddr: 10.244.4.3:47584
GET / HTTP/1.1
Host: app.demo.com
User-Agent: curl/7.29.0
Accept: */*
X-Forwarded-For: 10.244.3.0
X-Forwarded-Host: app.demo.com
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Real-Ip: 10.244.3.0
X-Request-Id: cf9899fac67011b07f082df64d6f74cf
X-Scheme: http
```

### 添加节点

在任意 master 节点上操作

首先更新内核

```bash
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.1 \
  --upgrade-kernel
```

> master 和 worker 节点可同时添加, 也可以分开

执行日志

```bash
[2020-09-26T12:10:40.778771610+0800]: INFO:    [check] ssh command exists.
[2020-09-26T12:10:40.780779404+0800]: INFO:    [check] sshpass command exists.
[2020-09-26T12:10:40.782390416+0800]: INFO:    [check] wget command exists.
[2020-09-26T12:10:41.085664234+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-09-26T12:10:41.272253392+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-09-26T12:10:41.430699498+0800]: INFO:    [check] conn apiserver succeeded.
[2020-09-26T12:10:41.432763541+0800]: INFO:    [init] upgrade kernel: 192.168.77.135
[2020-09-26T12:12:32.636836184+0800]: INFO:    [init] upgrade kernel 192.168.77.135 succeeded.
[2020-09-26T12:12:32.638639161+0800]: INFO:    [init] upgrade kernel: 192.168.77.136
[2020-09-26T12:13:53.146157237+0800]: INFO:    [init] upgrade kernel 192.168.77.136 succeeded.
[2020-09-26T12:13:53.298473735+0800]: INFO:    [init] 192.168.77.135: Wait for 10s to restart succeeded.
[2020-09-26T12:13:53.497501679+0800]: INFO:    [init] 192.168.77.136: Wait for 10s to restart succeeded.
[2020-09-26T12:13:53.500811026+0800]: INFO:    [notice] Please execute the command again!
[2020-09-26T12:13:53.504536417+0800]: INFO:    [cmd] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --version 1.19.1 

ERROR Summary: 
  
ACCESS Summary: 
  [cmd] bash kainstall.sh add --master 192.168.77.135 --worker 192.168.77.136 --user root --password 123456 --port 22 --version 1.19.1 
  

  See detailed log >>> /tmp/kainstall.qJJkFP71Dx/kainstall.log 
```

等待节点重启后，接着，进行添加节点操作

```bash
bash kainstall.sh add \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user root \
  --password 123456 \
  --port 22 \
  --version 1.19.1
```

执行日志

```bash
[2020-09-26T12:22:46.647653101+0800]: INFO:    [check] ssh command exists.
[2020-09-26T12:22:46.649340322+0800]: INFO:    [check] sshpass command exists.
[2020-09-26T12:22:46.651057425+0800]: INFO:    [check] wget command exists.
[2020-09-26T12:22:46.815046260+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-09-26T12:22:46.929671775+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-09-26T12:22:47.081347319+0800]: INFO:    [check] conn apiserver succeeded.
[2020-09-26T12:22:47.357101360+0800]: INFO:    [init] master: 192.168.77.135
[2020-09-26T12:23:09.879064458+0800]: INFO:    [init] init master 192.168.77.135 succeeded.
[2020-09-26T12:23:10.094492921+0800]: INFO:    [init] 192.168.77.135 set hostname and resolve succeeded.
[2020-09-26T12:23:10.311056515+0800]: INFO:    [init] worker: 192.168.77.136
[2020-09-26T12:23:24.483441954+0800]: INFO:    [init] init worker 192.168.77.136 succeeded.
[2020-09-26T12:23:24.620888424+0800]: INFO:    [init] 192.168.77.136 set hostname and resolve succeeded.
[2020-09-26T12:23:24.622966200+0800]: INFO:    [install] install docker on 192.168.77.135.
[2020-09-26T12:24:01.931334066+0800]: INFO:    [install] install docker on 192.168.77.135 succeeded.
[2020-09-26T12:24:01.932992920+0800]: INFO:    [install] install kube on 192.168.77.135
[2020-09-26T12:24:28.422064249+0800]: INFO:    [install] install kube on 192.168.77.135 succeeded.
[2020-09-26T12:24:28.424247318+0800]: INFO:    [install] install docker on 192.168.77.136.
[2020-09-26T12:25:04.168995847+0800]: INFO:    [install] install docker on 192.168.77.136 succeeded.
[2020-09-26T12:25:04.170921864+0800]: INFO:    [install] install kube on 192.168.77.136
[2020-09-26T12:25:31.296654405+0800]: INFO:    [install] install kube on 192.168.77.136 succeeded.
[2020-09-26T12:25:31.499777726+0800]: INFO:    [install] install haproxy on 192.168.77.136
[2020-09-26T12:25:33.917217803+0800]: INFO:    [install] install haproxy on 192.168.77.136 succeeded.
[2020-09-26T12:25:33.923687567+0800]: INFO:    [kubeadm join] master: get CACRT_HASH
[2020-09-26T12:25:34.052053277+0800]: INFO:    [kubeadm join] master: get INTI_CERTKEY
[2020-09-26T12:25:37.912162412+0800]: INFO:    [kubeadm join] master: get INIT_TOKEN
[2020-09-26T12:25:38.256108904+0800]: INFO:    [kubeadm join] master 192.168.77.135 join cluster.
[2020-09-26T12:27:10.984050414+0800]: INFO:    [kubeadm join] master 192.168.77.135 join cluster succeeded.
[2020-09-26T12:27:10.986733582+0800]: INFO:    [kubeadm join] 192.168.77.135: set kube config.
[2020-09-26T12:27:12.218104279+0800]: INFO:    [kubeadm join] 192.168.77.135: set kube config succeeded.
[2020-09-26T12:27:13.185195086+0800]: INFO:    [kubeadm join] worker 192.168.77.136 join cluster.
[2020-09-26T12:27:21.010415890+0800]: INFO:    [kubeadm join] worker 192.168.77.136 join cluster succeeded.
[2020-09-26T12:27:21.014874111+0800]: INFO:    [kubeadm join] set 192.168.77.136 worker node role.
[2020-09-26T12:27:21.512029132+0800]: INFO:    [kubeadm join] set 192.168.77.136 worker node role succeeded.
[2020-09-26T12:27:21.746590260+0800]: INFO:    [del] 192.168.77.133: add apiserver from haproxy
[2020-09-26T12:27:21.919701213+0800]: INFO:    [del] 192.168.77.133: add apiserver(192.168.77.135) from haproxy succeeded.
[2020-09-26T12:27:21.921559102+0800]: INFO:    [del] 192.168.77.134: add apiserver from haproxy
[2020-09-26T12:27:22.387721177+0800]: INFO:    [del] 192.168.77.134: add apiserver(192.168.77.135) from haproxy succeeded.
[2020-09-26T12:27:22.389910068+0800]: INFO:    [del] 192.168.77.136: add apiserver from haproxy
[2020-09-26T12:27:22.584832514+0800]: INFO:    [del] 192.168.77.136: add apiserver(192.168.77.135) from haproxy succeeded.
[2020-09-26T12:27:27.588148051+0800]: INFO:    [cluster] cluster status

NAME               STATUS     ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master-node1   Ready      master   32m   v1.19.1   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready      master   30m   v1.19.1   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready      master   29m   v1.19.1   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node4   NotReady   master   40s   v1.19.1   192.168.77.135   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready      worker   29m   v1.19.1   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready      worker   28m   v1.19.1   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node3   NotReady   worker   7s    v1.19.1   192.168.77.136   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE       NAME                                        READY   STATUS              RESTARTS   AGE
default         ingress-demo-app-589b64597f-bdm8d           1/1     Running             0          26m
default         ingress-demo-app-589b64597f-rm5q7           1/1     Running             0          26m
ingress-nginx   ingress-nginx-admission-create-lm7qj        0/1     Completed           0          28m
ingress-nginx   ingress-nginx-admission-patch-zcbtn         0/1     Completed           0          28m
ingress-nginx   ingress-nginx-controller-859658cbfd-xrqff   1/1     Running             0          28m
kube-system     coredns-59c898cd69-97bjz                    1/1     Running             0          32m
kube-system     coredns-59c898cd69-qw5s7                    1/1     Running             0          32m
kube-system     etcd-k8s-master-node1                       1/1     Running             0          32m
kube-system     etcd-k8s-master-node2                       1/1     Running             0          30m
kube-system     etcd-k8s-master-node3                       1/1     Running             0          29m
kube-system     etcd-k8s-master-node4                       0/1     Running             0          38s
kube-system     kube-apiserver-k8s-master-node1             1/1     Running             0          32m
kube-system     kube-apiserver-k8s-master-node2             1/1     Running             0          30m
kube-system     kube-apiserver-k8s-master-node3             1/1     Running             0          29m
kube-system     kube-apiserver-k8s-master-node4             0/1     Running             0          39s
kube-system     kube-controller-manager-k8s-master-node1    1/1     Running             1          32m
kube-system     kube-controller-manager-k8s-master-node2    1/1     Running             0          30m
kube-system     kube-controller-manager-k8s-master-node3    1/1     Running             0          29m
kube-system     kube-controller-manager-k8s-master-node4    0/1     Running             0          39s
kube-system     kube-flannel-ds-amd64-5mv5r                 1/1     Running             0          28m
kube-system     kube-flannel-ds-amd64-7sf8s                 1/1     Running             0          28m
kube-system     kube-flannel-ds-amd64-bwdm4                 1/1     Running             0          40s
kube-system     kube-flannel-ds-amd64-kfhbf                 1/1     Running             0          28m
kube-system     kube-flannel-ds-amd64-lj8rb                 1/1     Running             0          28m
kube-system     kube-flannel-ds-amd64-qmcdm                 1/1     Running             0          28m
kube-system     kube-flannel-ds-amd64-rmjtq                 0/1     Init:0/1            0          6s
kube-system     kube-proxy-6bbgx                            1/1     Running             0          40s
kube-system     kube-proxy-bqzxq                            1/1     Running             0          29m
kube-system     kube-proxy-bxdc5                            1/1     Running             0          28m
kube-system     kube-proxy-dz9gz                            1/1     Running             0          32m
kube-system     kube-proxy-fzwvb                            1/1     Running             0          30m
kube-system     kube-proxy-jtxnc                            0/1     ContainerCreating   0          7s
kube-system     kube-proxy-zrmr8                            1/1     Running             0          29m
kube-system     kube-scheduler-k8s-master-node1             1/1     Running             1          32m
kube-system     kube-scheduler-k8s-master-node2             1/1     Running             0          30m
kube-system     kube-scheduler-k8s-master-node3             1/1     Running             0          29m
kube-system     kube-scheduler-k8s-master-node4             0/1     Running             0          38s
kube-system     metrics-server-5488bc7945-d84wv             1/1     Running             0          28m
ERROR Summary: 
  
ACCESS Summary: 
  

  See detailed log >>> /tmp/kainstall.o6O1DyKKzV/kainstall.log 

```

等待一会，查看节点状态就`Ready` 了

```bash
# kubectl get node -o wide       
NAME               STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master-node1   Ready    master   47m   v1.19.1   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   45m   v1.19.1   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   44m   v1.19.1   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node4   Ready    master   15m   v1.19.1   192.168.77.135   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   43m   v1.19.1   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   43m   v1.19.1   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node3   Ready    worker   15m   v1.19.1   192.168.77.136   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
```



### 删除节点

在任意 master 节点上操作

```bash
bash kainstall.sh del \
  --master 192.168.77.135 \
  --worker 192.168.77.136 \
  --user root \
  --password 123456 \
  --port 22
```

> master 和 worker 节点可同时添加, 也可以分开

执行日志

```bash
[2020-09-26T12:48:25.242258713+0800]: INFO:    [check] ssh command exists.
[2020-09-26T12:48:25.244405320+0800]: INFO:    [check] sshpass command exists.
[2020-09-26T12:48:25.246265095+0800]: INFO:    [check] wget command exists.
[2020-09-26T12:48:25.369521762+0800]: INFO:    [check] ssh 192.168.77.135 connection succeeded.
[2020-09-26T12:48:25.494157980+0800]: INFO:    [check] ssh 192.168.77.136 connection succeeded.
[2020-09-26T12:48:25.593280942+0800]: INFO:    [check] conn apiserver succeeded.
[2020-09-26T12:48:25.685816991+0800]: INFO:    [del] 192.168.77.133: remove apiserver from haproxy
[2020-09-26T12:48:25.862584056+0800]: INFO:    [del] 192.168.77.133: remove apiserver(192.168.77.135) from haproxy succeeded.
[2020-09-26T12:48:25.864431069+0800]: INFO:    [del] 192.168.77.134: remove apiserver from haproxy
[2020-09-26T12:48:26.109456387+0800]: INFO:    [del] 192.168.77.134: remove apiserver(192.168.77.135) from haproxy succeeded.
[2020-09-26T12:48:26.111961334+0800]: INFO:    [del] 192.168.77.136: remove apiserver from haproxy
[2020-09-26T12:48:26.255811074+0800]: INFO:    [del] 192.168.77.136: remove apiserver(192.168.77.135) from haproxy succeeded.
[2020-09-26T12:48:26.260324829+0800]: INFO:    [del] node 192.168.77.135
[2020-09-26T12:48:26.356543093+0800]: INFO:    [del] drain 192.168.77.135
[2020-09-26T12:48:26.493261213+0800]: INFO:    [del] 192.168.77.135: drain succeeded.
[2020-09-26T12:48:26.494910986+0800]: INFO:    [del] delete node 192.168.77.135
[2020-09-26T12:48:26.630122612+0800]: INFO:    [del] 192.168.77.135: delete succeeded.
[2020-09-26T12:48:29.633223344+0800]: INFO:    [reset] node 192.168.77.135
[2020-09-26T12:48:31.891804528+0800]: INFO:    [reset] 192.168.77.135: reset succeeded.
[2020-09-26T12:48:31.893591911+0800]: INFO:    [del] node 192.168.77.136
[2020-09-26T12:48:31.997434876+0800]: INFO:    [del] drain 192.168.77.136
[2020-09-26T12:48:32.138634028+0800]: INFO:    [del] 192.168.77.136: drain succeeded.
[2020-09-26T12:48:32.140027184+0800]: INFO:    [del] delete node 192.168.77.136
[2020-09-26T12:48:32.242612594+0800]: INFO:    [del] 192.168.77.136: delete succeeded.
[2020-09-26T12:48:35.246905712+0800]: INFO:    [reset] node 192.168.77.136
[2020-09-26T12:48:36.432741765+0800]: INFO:    [reset] 192.168.77.136: reset succeeded.
[2020-09-26T12:48:41.436578053+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master-node1   Ready    master   53m   v1.19.1   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   52m   v1.19.1   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   50m   v1.19.1   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   50m   v1.19.1   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   50m   v1.19.1   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
default         ingress-demo-app-589b64597f-bdm8d           1/1     Running     0          47m
default         ingress-demo-app-589b64597f-rm5q7           1/1     Running     0          47m
ingress-nginx   ingress-nginx-admission-create-lm7qj        0/1     Completed   0          49m
ingress-nginx   ingress-nginx-admission-patch-zcbtn         0/1     Completed   0          49m
ingress-nginx   ingress-nginx-controller-859658cbfd-xrqff   1/1     Running     0          50m
kube-system     coredns-59c898cd69-97bjz                    1/1     Running     0          53m
kube-system     coredns-59c898cd69-qw5s7                    1/1     Running     0          53m
kube-system     etcd-k8s-master-node1                       1/1     Running     0          53m
kube-system     etcd-k8s-master-node2                       1/1     Running     0          52m
kube-system     etcd-k8s-master-node3                       1/1     Running     0          50m
kube-system     kube-apiserver-k8s-master-node1             1/1     Running     0          53m
kube-system     kube-apiserver-k8s-master-node2             1/1     Running     0          52m
kube-system     kube-apiserver-k8s-master-node3             1/1     Running     0          50m
kube-system     kube-controller-manager-k8s-master-node1    1/1     Running     1          53m
kube-system     kube-controller-manager-k8s-master-node2    1/1     Running     0          52m
kube-system     kube-controller-manager-k8s-master-node3    1/1     Running     0          50m
kube-system     kube-flannel-ds-amd64-5mv5r                 1/1     Running     0          50m
kube-system     kube-flannel-ds-amd64-7sf8s                 1/1     Running     0          50m
kube-system     kube-flannel-ds-amd64-bwdm4                 1/1     Running     0          21m
kube-system     kube-flannel-ds-amd64-kfhbf                 1/1     Running     0          50m
kube-system     kube-flannel-ds-amd64-lj8rb                 1/1     Running     0          50m
kube-system     kube-flannel-ds-amd64-qmcdm                 1/1     Running     0          50m
kube-system     kube-flannel-ds-amd64-rmjtq                 1/1     Running     0          21m
kube-system     kube-proxy-6bbgx                            1/1     Running     0          21m
kube-system     kube-proxy-bqzxq                            1/1     Running     0          50m
kube-system     kube-proxy-bxdc5                            1/1     Running     0          50m
kube-system     kube-proxy-dz9gz                            1/1     Running     0          53m
kube-system     kube-proxy-fzwvb                            1/1     Running     0          52m
kube-system     kube-proxy-jtxnc                            1/1     Running     0          21m
kube-system     kube-proxy-zrmr8                            1/1     Running     0          50m
kube-system     kube-scheduler-k8s-master-node1             1/1     Running     1          53m
kube-system     kube-scheduler-k8s-master-node2             1/1     Running     0          52m
kube-system     kube-scheduler-k8s-master-node3             1/1     Running     0          50m
kube-system     metrics-server-5488bc7945-d84wv             1/1     Running     0          50m
ERROR Summary: 
  
ACCESS Summary: 
  

  See detailed log >>> /tmp/kainstall.WDzfoqveo9/kainstall.log 
```



### 添加监控

在任意 master 节点上操作

```bash
bash kainstall.sh add --monitor prometheus
```

执行日志

```bash
[2020-09-26T12:49:19.134408423+0800]: INFO:    [check] ssh command exists.
[2020-09-26T12:49:19.136139609+0800]: INFO:    [check] sshpass command exists.
[2020-09-26T12:49:19.138114744+0800]: INFO:    [check] wget command exists.
[2020-09-26T12:49:19.225561467+0800]: INFO:    [check] conn apiserver succeeded.
[2020-09-26T12:49:19.227222444+0800]: INFO:    [monitor] download prometheus manifests
[2020-09-26T12:49:21.708567888+0800]: INFO:    [monitor] download prometheus succeeded.
[2020-09-26T12:49:21.710197755+0800]: INFO:    [monitor] apply prometheus manifests
[2020-09-26T12:49:33.952204798+0800]: INFO:    [apply] add prometheus succeeded.
[2020-09-26T12:49:33.984385715+0800]: INFO:    [monitor] set controller-manager and scheduler prometheus discovery service
[2020-09-26T12:49:34.837517322+0800]: INFO:    [apply] set controller-manager and scheduler prometheus discovery succeeded.
[2020-09-26T12:49:34.839992554+0800]: INFO:    [monitor] add prometheus ingress
[2020-09-26T12:49:43.308108521+0800]: INFO:    [apply] add prometheus ingress succeeded.

ERROR Summary: 
  
ACCESS Summary: 
  [ingress] curl -H 'Host:grafana.monitoring.cluster.local' 192.168.77.133:32366
  [ingress] curl -H 'Host:prometheus.monitoring.cluster.local' 192.168.77.133:32366
  [ingress] curl -H 'Host:alertmanager.monitoring.cluster.local' 192.168.77.133:32366
  

  See detailed log >>> /tmp/kainstall.8x27tSeHnr/kainstall.log 

```

等待一段时间后，po正常运行后，就可以访问了。

```bash
# kubectl -n monitoring get pods
NAME                                   READY   STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2     Running   0          2m14s
alertmanager-main-1                    2/2     Running   0          2m14s
alertmanager-main-2                    2/2     Running   0          2m14s
grafana-7c9bc466d8-vccmr               1/1     Running   0          2m38s
kube-state-metrics-66b65b78bc-x5r57    3/3     Running   0          2m38s
node-exporter-2mwlp                    2/2     Running   0          2m37s
node-exporter-jdpxl                    2/2     Running   0          2m37s
node-exporter-lqsjz                    2/2     Running   0          2m37s
node-exporter-qr7vf                    2/2     Running   0          2m37s
node-exporter-r992d                    2/2     Running   0          2m37s
prometheus-adapter-557648f58c-22r6z    1/1     Running   0          2m37s
prometheus-k8s-0                       3/3     Running   1          116s
prometheus-k8s-1                       3/3     Running   1          116s
prometheus-operator-5b7946f4d6-sxlsq   2/2     Running   0          2m42s
```

通过主机host绑定，来访问对应的服务

```bash
192.168.77.133 grafana.monitoring.cluster.local
192.168.77.133 prometheus.monitoring.cluster.local
192.168.77.133 alertmanager.monitoring.cluster.local
```
访问 grafana
url： http://grafana.monitoring.cluster.local:32366/

![image-20200928125811772](/assets/images/kubernetes/kainstall-grafana.png)

访问 prometheus 
url： http://prometheus.monitoring.cluster.local:32366/
![image-20200928125910590](/assets/images/kubernetes/kainstall-prometheus.png)

访问 alertmanager
url： http://alertmanager.monitoring.cluster.local:32366/
![image-20200928125945019](/assets/images/kubernetes/kainstall-alertmanager.png)

### 升级版本

在任意 master 节点上操作

```bash
bash kainstall.sh upgrade \
  --version 1.19.2 \
  --user root \
  --password 123456 \
  --port 22
```

执行日志

```bash
[2020-09-26T13:02:49.517325174+0800]: INFO:    [check] ssh command exists.
[2020-09-26T13:02:49.521616700+0800]: INFO:    [check] sshpass command exists.
[2020-09-26T13:02:49.524099456+0800]: INFO:    [check] wget command exists.
[2020-09-26T13:02:49.623708070+0800]: INFO:    [check] conn apiserver succeeded.
[2020-09-26T13:02:49.634978307+0800]: INFO:    [upgrade] upgrade to 1.19.2
[2020-09-26T13:02:50.681561173+0800]: INFO:    [upgrade] node: k8s-master-node1
[2020-09-26T13:03:07.023576290+0800]: INFO:    [upgrade] drain k8s-master-node1 node succeeded.
[2020-09-26T13:08:20.765654416+0800]: INFO:    [upgrade] plan and upgrade cluster on k8s-master-node1 succeeded.
[2020-09-26T13:08:20.915498559+0800]: INFO:    [upgrade] k8s-master-node1 ready succeeded.
[2020-09-26T13:08:26.038396891+0800]: INFO:    [upgrade] uncordon k8s-master-node1 node succeeded.
[2020-09-26T13:08:26.043680543+0800]: INFO:    [upgrade] node: k8s-master-node2
[2020-09-26T13:08:36.483122019+0800]: INFO:    [upgrade] drain k8s-master-node2 node succeeded.
[2020-09-26T13:13:23.274510315+0800]: INFO:    [upgrade] upgrade k8s-master-node2 node succeeded.
[2020-09-26T13:13:24.184048070+0800]: INFO:    [upgrade] k8s-master-node2 ready succeeded.
[2020-09-26T13:13:29.302988338+0800]: INFO:    [upgrade] uncordon k8s-master-node2 node succeeded.
[2020-09-26T13:13:29.306162582+0800]: INFO:    [upgrade] node: k8s-master-node3
[2020-09-26T13:13:29.471089742+0800]: INFO:    [upgrade] drain k8s-master-node3 node succeeded.
[2020-09-26T13:19:15.432440076+0800]: INFO:    [upgrade] upgrade k8s-master-node3 node succeeded.
[2020-09-26T13:19:15.539758058+0800]: INFO:    [upgrade] k8s-master-node3 ready succeeded.
[2020-09-26T13:19:20.657324323+0800]: INFO:    [upgrade] uncordon k8s-master-node3 node succeeded.
[2020-09-26T13:19:20.659345723+0800]: INFO:    [upgrade] node: k8s-worker-node1
[2020-09-26T13:19:58.682543928+0800]: INFO:    [upgrade] drain k8s-worker-node1 node succeeded.
[2020-09-26T13:20:26.319268990+0800]: INFO:    [upgrade] upgrade k8s-worker-node1 node succeeded.
[2020-09-26T13:20:26.464842633+0800]: INFO:    [upgrade] k8s-worker-node1 ready succeeded.
[2020-09-26T13:20:31.583020236+0800]: INFO:    [upgrade] uncordon k8s-worker-node1 node succeeded.
[2020-09-26T13:20:31.585730169+0800]: INFO:    [upgrade] node: k8s-worker-node2
[2020-09-26T13:20:50.060684932+0800]: INFO:    [upgrade] drain k8s-worker-node2 node succeeded.
[2020-09-26T13:21:25.641352602+0800]: INFO:    [upgrade] upgrade k8s-worker-node2 node succeeded.
[2020-09-26T13:21:25.738130750+0800]: INFO:    [upgrade] k8s-worker-node2 ready succeeded.
[2020-09-26T13:21:30.847023987+0800]: INFO:    [upgrade] uncordon k8s-worker-node2 node succeeded.
[2020-09-26T13:21:35.849913889+0800]: INFO:    [cluster] cluster status

NAME               STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master-node1   Ready    master   86m   v1.19.2   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   84m   v1.19.2   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   83m   v1.19.2   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   83m   v1.19.2   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   82m   v1.19.2   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13

NAMESPACE       NAME                                        READY   STATUS              RESTARTS   AGE
default         ingress-demo-app-589b64597f-6d4tv           0/1     ContainerCreating   0          64s
default         ingress-demo-app-589b64597f-tgks5           0/1     ContainerCreating   0          64s
ingress-nginx   ingress-nginx-controller-859658cbfd-lfvrc   0/1     Running             0          65s
kube-system     coredns-59c898cd69-njtns                    1/1     Running             0          64s
kube-system     coredns-59c898cd69-rx8xn                    1/1     Running             0          2m15s
kube-system     etcd-k8s-master-node1                       1/1     Running             0          86m
kube-system     etcd-k8s-master-node2                       1/1     Running             0          84m
kube-system     etcd-k8s-master-node3                       1/1     Running             0          83m
kube-system     kube-apiserver-k8s-master-node1             1/1     Running             0          86m
kube-system     kube-apiserver-k8s-master-node2             1/1     Running             0          84m
kube-system     kube-apiserver-k8s-master-node3             1/1     Running             0          83m
kube-system     kube-controller-manager-k8s-master-node1    1/1     Running             1          86m
kube-system     kube-controller-manager-k8s-master-node2    1/1     Running             0          84m
kube-system     kube-controller-manager-k8s-master-node3    1/1     Running             0          83m
kube-system     kube-flannel-ds-amd64-5mv5r                 1/1     Running             0          82m
kube-system     kube-flannel-ds-amd64-7sf8s                 1/1     Running             0          82m
kube-system     kube-flannel-ds-amd64-kfhbf                 1/1     Running             0          82m
kube-system     kube-flannel-ds-amd64-lj8rb                 1/1     Running             0          82m
kube-system     kube-flannel-ds-amd64-qmcdm                 1/1     Running             0          82m
kube-system     kube-proxy-bqzxq                            1/1     Running             0          83m
kube-system     kube-proxy-bxdc5                            1/1     Running             0          82m
kube-system     kube-proxy-dz9gz                            1/1     Running             0          86m
kube-system     kube-proxy-fzwvb                            1/1     Running             0          84m
kube-system     kube-proxy-zrmr8                            1/1     Running             0          83m
kube-system     kube-scheduler-k8s-master-node1             1/1     Running             1          86m
kube-system     kube-scheduler-k8s-master-node2             1/1     Running             0          84m
kube-system     kube-scheduler-k8s-master-node3             1/1     Running             0          83m
kube-system     metrics-server-5488bc7945-c4f5q             1/1     Running             0          64s
monitoring      alertmanager-main-0                         2/2     Running             0          54s
monitoring      alertmanager-main-1                         2/2     Running             0          51s
monitoring      alertmanager-main-2                         2/2     Running             0          56s
monitoring      grafana-7c9bc466d8-t2fg5                    0/1     ContainerCreating   0          63s
monitoring      kube-state-metrics-66b65b78bc-rg47g         0/3     ContainerCreating   0          64s
monitoring      node-exporter-2mwlp                         2/2     Running             0          32m
monitoring      node-exporter-jdpxl                         2/2     Running             0          32m
monitoring      node-exporter-lqsjz                         2/2     Running             0          32m
monitoring      node-exporter-qr7vf                         2/2     Running             0          32m
monitoring      node-exporter-r992d                         2/2     Running             0          32m
monitoring      prometheus-adapter-557648f58c-8gcnv         1/1     Running             0          63s
monitoring      prometheus-k8s-0                            3/3     Running             1          49s
monitoring      prometheus-k8s-1                            3/3     Running             1          57s
monitoring      prometheus-operator-5b7946f4d6-tfwg9        2/2     Running             0          63s
ERROR Summary: 
  
ACCESS Summary: 
  

  See detailed log >>> /tmp/kainstall.Zvel7elRqV/kainstall.log 
```

等待一会，查看集群状态

```bash
# kubectl version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.2", GitCommit:"f5743093fd1c663cb0cbc89748f730662345d44d", GitTreeState:"clean", BuildDate:"2020-09-16T13:41:02Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.1", GitCommit:"206bcadf021e76c27513500ca24182692aabd17e", GitTreeState:"clean", BuildDate:"2020-09-09T11:18:22Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}

# kubectl get nodes -o wide
NAME               STATUS   ROLES    AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
k8s-master-node1   Ready    master   87m   v1.19.2   192.168.77.130   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node2   Ready    master   85m   v1.19.2   192.168.77.131   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-master-node3   Ready    master   84m   v1.19.2   192.168.77.132   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node1   Ready    worker   83m   v1.19.2   192.168.77.133   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13
k8s-worker-node2   Ready    worker   83m   v1.19.2   192.168.77.134   <none>        CentOS Linux 7 (Core)   5.8.12-1.el7.elrepo.x86_64   docker://19.3.13

# kubectl get pods -A
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE
default         ingress-demo-app-589b64597f-6d4tv           1/1     Running   0          2m31s
default         ingress-demo-app-589b64597f-tgks5           1/1     Running   0          2m31s
ingress-nginx   ingress-nginx-controller-859658cbfd-lfvrc   1/1     Running   0          2m32s
kube-system     coredns-59c898cd69-njtns                    1/1     Running   0          2m31s
kube-system     coredns-59c898cd69-rx8xn                    1/1     Running   0          3m42s
kube-system     etcd-k8s-master-node1                       1/1     Running   0          87m
kube-system     etcd-k8s-master-node2                       1/1     Running   0          86m
kube-system     etcd-k8s-master-node3                       1/1     Running   0          84m
kube-system     kube-apiserver-k8s-master-node1             1/1     Running   0          87m
kube-system     kube-apiserver-k8s-master-node2             1/1     Running   0          86m
kube-system     kube-apiserver-k8s-master-node3             1/1     Running   0          84m
kube-system     kube-controller-manager-k8s-master-node1    1/1     Running   1          87m
kube-system     kube-controller-manager-k8s-master-node2    1/1     Running   0          86m
kube-system     kube-controller-manager-k8s-master-node3    1/1     Running   0          84m
kube-system     kube-flannel-ds-amd64-5mv5r                 1/1     Running   0          84m
kube-system     kube-flannel-ds-amd64-7sf8s                 1/1     Running   0          84m
kube-system     kube-flannel-ds-amd64-kfhbf                 1/1     Running   0          84m
kube-system     kube-flannel-ds-amd64-lj8rb                 1/1     Running   0          84m
kube-system     kube-flannel-ds-amd64-qmcdm                 1/1     Running   0          84m
kube-system     kube-proxy-bqzxq                            1/1     Running   0          84m
kube-system     kube-proxy-bxdc5                            1/1     Running   0          84m
kube-system     kube-proxy-dz9gz                            1/1     Running   0          87m
kube-system     kube-proxy-fzwvb                            1/1     Running   0          86m
kube-system     kube-proxy-zrmr8                            1/1     Running   0          84m
kube-system     kube-scheduler-k8s-master-node1             1/1     Running   1          87m
kube-system     kube-scheduler-k8s-master-node2             1/1     Running   0          86m
kube-system     kube-scheduler-k8s-master-node3             1/1     Running   0          84m
kube-system     metrics-server-5488bc7945-c4f5q             1/1     Running   0          2m31s
monitoring      alertmanager-main-0                         2/2     Running   0          2m21s
monitoring      alertmanager-main-1                         2/2     Running   0          2m18s
monitoring      alertmanager-main-2                         2/2     Running   0          2m23s
monitoring      grafana-7c9bc466d8-t2fg5                    1/1     Running   0          2m30s
monitoring      kube-state-metrics-66b65b78bc-rg47g         3/3     Running   0          2m31s
monitoring      node-exporter-2mwlp                         2/2     Running   0          33m
monitoring      node-exporter-jdpxl                         2/2     Running   0          33m
monitoring      node-exporter-lqsjz                         2/2     Running   0          33m
monitoring      node-exporter-qr7vf                         2/2     Running   0          33m
monitoring      node-exporter-r992d                         2/2     Running   0          33m
monitoring      prometheus-adapter-557648f58c-8gcnv         1/1     Running   0          2m30s
monitoring      prometheus-k8s-0                            3/3     Running   1          2m16s
monitoring      prometheus-k8s-1                            3/3     Running   1          2m24s
monitoring      prometheus-operator-5b7946f4d6-tfwg9        2/2     Running   0          2m30s
```

集群升级完成。



### 其他操作

**注意：** 添加组件时请保持节点的内存和cpu至少为`2C4G`的空闲。否则会导致节点下线且服务器卡死。

```bash
# 添加 traefik ingress
bash kainstall.sh add --ingress traefik

# 添加 elasticsearch
bash kainstall.sh add --log elasticsearch

# 添加 rook
bash kainstall.sh add --storage rook
```



## 最后

是不是很简单，你只需要准备好服务器，操作下脚本就行了，敲一条命令，喝杯咖啡摸摸鱼，集群就好了。这生活岂不美哉，自动化的魅力！。

