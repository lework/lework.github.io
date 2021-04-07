---
layout: post
title: "使用kubeadm更新k8s集群"
date: "2019-10-01 10:00:00"
category: kubernetes
tags: kubernetes k8s-install
author: lework
---
* content
{:toc}

本次我们将实际利用Kubeadm工具来更新集群，这边沿用之前分享的[使用kubeadm安装Kubernetes v15.4 ha集群](/2019/10/01/kubeadm-install/)，部署的HA丛集来进行`v1.15.4`更新至 `v1.16.0`。





## kubeadm更新步骤

本部分将依据以下步骤进行，并通过Kubeadm 升级既有Kubernetes集群至新版本:

- 主节点(Masters)
  - 更新kube-apiserver, controller manager, scheduler 与etcd
  - 更新Addons。如: kube-proxy, CoreDNS。
  - 更新kubelet binary file与maniftest。
  - (optional) 更新Node bootstrap tokens 的RBAC 规则。
- 工作节点(Nodes)
  - 在主节点使用kubectl drain来驱赶Pods到其他节点，并进入运维模式。另外建议使用PodDisruptionBudge
  - 确保应用程式在Kubernetes集群的可用与不可用数。
  - 更新kubelet 二进制档与maniftest
  - 在主节点使用kubectl uncordon 让节点能够被调度。

##　事前准备
在开始更新丛集前，请确保以下条件已达成:

1. 用kubeadm建立一座Kubernetes集群。可以参考[使用kubeadm安装Kubernetes v15.4 ha集群](/2019/09/30/kubeadm-install/)文章进行。
2. 确保丛集的所有节点处于Ready 状态。
3. 确保应用程式利用进阶的Kubernetes API 建立，如Deployment。并利用多副本机制来避免服务中断。

>  以下步骤请一台一台来更新，这是为了确保Kubernetes 功能不会因为更新而中断。



## 更新主节点(Masters)

在`k8s-m1节点`上安装新版的kubeadm

``` bash
yum install -y kubeadm-1.16.0 --disableexcludes=kubernetes
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:34:01Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```
在`k8s-m1节点`上更新控制平面

``` bash
kubeadm upgrade plan v1.16.0
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.15.4
[upgrade/versions] kubeadm version: v1.16.0

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     5 x v1.15.4   v1.16.0

Upgrade to the latest version in the v1.15 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.15.4   v1.16.0
Controller Manager   v1.15.4   v1.16.0
Scheduler            v1.15.4   v1.16.0
Kube Proxy           v1.15.4   v1.16.0
CoreDNS              1.3.1     1.6.2
Etcd                 3.3.10    3.3.15-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.16.0
        
# 看到上面信息后，就可以更新节点了
kubeadm upgrade apply v1.16.0
......

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.16.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

在`其他主节点`上更新控制平面组件

``` bash
yum install -y kubeadm-1.16.0 --disableexcludes=kubernetes
kubeadm upgrade node
```

主节点更新完成后，开始在主节点上更新kubelet和kubectl

```bash
yum install -y kubelet-1.16.0 kubectl-1.16.0 --disableexcludes=kubernetes
```

接着重启kubelet
``` bash
systemctl daemon-reload
systemctl restart kubelet
```

在master节点上查看集群信息

``` bash
kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:27:17Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}

kubectl get no
NAME     STATUS   ROLES    AGE   VERSION
k8s-m1   Ready    master   16h   v1.16.0
k8s-m2   Ready    master   16h   v1.16.0
k8s-m3   Ready    master   16h   v1.16.0
k8s-n1   Ready    <none>   15h   v1.15.4
k8s-n2   Ready    <none>   15h   v1.15.4
```


## 更新node节点

可以在所有node节点上先更新`kubeadm`

``` bash
yum install -y kubeadm-1.16.0 --disableexcludes=kubernetes
kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:34:01Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```
### 更新k8s-n1节点

在任一台主节点上执行`kubectl drain`命令

``` bash
kubectl drain k8s-n1 --ignore-daemonsets
node/k8s-n1 already cordoned
WARNING: deleting Pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: default/nginx; ignoring DaemonSet-managed Pods: kube-system/calico-node-9fltz, kube-system/kube-proxy-65jrh
evicting pod "coredns-5c468949c8-qjwxg"
pod/coredns-5c468949c8-qjwxg evicted
node/k8s-n1 evicted
```
> 确保节点上的pods都被迁移到其他node中

更新组件

``` bash
kubeadm upgrade node
```
更新kubelet
``` bash
yum install -y kubelet-1.16.0-0 --disableexcludes=kubernetes
```
接着重启kubelet
``` bash
systemctl daemon-reload
systemctl restart kubelet
```
在主机点中恢复node节点
``` bash
kubectl uncordon k8s-n1
```

验证
``` bash
kubectl get no
NAME     STATUS   ROLES    AGE   VERSION
k8s-m1   Ready    master   16h   v1.16.0
k8s-m2   Ready    master   16h   v1.16.0
k8s-m3   Ready    master   16h   v1.16.0
k8s-n1   Ready    <none>   15h   v1.16.0
k8s-n2   Ready    <none>   15h   v1.15.4
```

### 更新k8s-n2节点

在任一台主节点上执行`kubectl drain`命令

``` bash
kubectl drain k8s-n2 --ignore-daemonsets
node/k8s-n2 cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-drqzh, kube-system/kube-proxy-rp96f
evicting pod "coredns-5c468949c8-xvkz7"
pod/coredns-5c468949c8-xvkz7 evicted
node/k8s-n2 evicted
```

> 确保节点上的pods都被迁移到其他node中



更新组件

``` bash
kubeadm upgrade node
```

更新kubelet

``` bash
yum install -y kubelet-1.16.0-0 --disableexcludes=kubernetes
```

接着重启kubelet
``` bash
systemctl daemon-reload
systemctl restart kubelet
```

在恢复驱逐状态

``` bash
kubectl uncordon k8s-n2
```

验证
``` bash
kubectl get no
NAME     STATUS   ROLES    AGE   VERSION
k8s-m1   Ready    master   16h   v1.16.0
k8s-m2   Ready    master   16h   v1.16.0
k8s-m3   Ready    master   16h   v1.16.0
k8s-n1   Ready    <none>   16h   v1.16.0
k8s-n2   Ready    <none>   16h   v1.16.0
```

## Kubeadm upgrade 流程

从上面的更新流程可以看到，kubeadm的更新命令流程：

![k8s-node-ha](/assets/images/kubernetes/kubeadm-update.png)

可以看到使用少数的命令就能更新集群，实际上每个命令背后都做了不少事情，这里我们将描述每个命令做了哪些事情

### kubeadm upgrade plan

该指令主要检查是否可以升级目前集群到指定的Kubernetes 版本，这过程会进行验证以下几件事情:

1. 检查当前集群中的Kubeadm 组件。
2. 检查当前集群是否处于健康状态。
3. 取得指定版本是否可以使用。若未指定版本的话，则以当前官方的最新稳定版本。
4. 确认是否符合[Version Skew Policies](https://kubernetes.io/docs/setup/release/version-skew-policy/)。
5. 若上面执行都没问题的话，会显示当前集群组件版本，以及目标集群组件版本。

### kubeadm upgrade apply

一但检查都没问题后，我们就可以执行`kubeadm upgrade apply`来进行主节点的更新，这时kubeadm会开始进行以下流程:

1. 检查组件参数是否有效，并载入预设参数与使用者输入的参数。另外也会检查当前是否为root。
2. kubeadm会透过admin user来与API server沟通，并检查当前集群是否处于健康状态，以及所有节点是否处于Ready状态。当检查都完成后，会读取集群中的kubeadm InitConfiguration组件信息，并将内容调整成新版的信息后，更新至当前集群中，以在后续执行更新时使用。另外该阶段也会再次检查是否符合[Version Skew Policies](https://kubernetes.io/docs/setup/release/version-skew-policy/)。
3. 当上述确认后，kubeadm 会与API server 沟通，以建立一个DaemonSet 来取得指定版本的控制平面组件容器镜像。这边原理是利用Kubernetes 机制来下载镜像，当Pod 被建立时，容器Runtime 就会先下载镜像才启动，因此可以确保指定容器镜像档被载入到节点上。这也是为何前面需要确保节点处于Ready 状态原因。另外Kubeadm 不直接从容器Runtime 拉取镜像，是因为现在有太多容器Runtime 被使用，因此这么做的话，会增加程序与维护的困难与复杂性。
4. 一但镜像下载完成后，kubeadm 就会执行组件升级。首先会备份etcd 的mainifest 数据，并更新成新版本内容，接着等待etcd 启动。接着会开始对控制组件件进行更新，过程中跟etcd 类似，会先写入新版本YAML 到/tmp 底下，接着将新版本信息放到mainifest 目录，然后将旧的mainifest 信息进行备份，最后等待监听kubelet重新启动Static Pod 的变动。另外过程中，如果kubeadm 检查发现相关凭证将在180 天过期的话，kubeadm 会自动更新相关TLS 凭证，以确保集群不会因为过期而出问题。
5. 若控制平面组件都升级完成的话，会开始进行以下步骤:
   - 将新版本的kubeadm ClusterStatus 与kubelet 组件更新至集群中，并建立相对应的RBAC 权限，以以便于其他节点使用。
   - 从集群取得最新版本的kubelet内容，并覆写到`/var/lib/kubelet/config.yaml`中。
   - 更新当前节点的`/var/lib/kubelet/kubeadm-flags.env`信息，以确保使用新版本的参数。
   - 设定主节点的Annotations(CRISocket)。
   - 更新CoreDNS 与kube-proxy 的DaemonSet 内容，以通过Rolling upgrade 方式更新至新版本。

### kubeadm upgrade node

在更新集群中，kubeadm会分为`第一主节点`、`其他主节点`与`工作节点`来进行，而想要更新到新版本，必须先在任一台主节点上执行`kubeadm upgrade apply`指令，来确保新版本的组件信息被新增到Kubernetes集群中。一但有了新版本组态信息后，就能在其他节点执行`kubeadm upgrade node`进行更新，而执行该指令时，又会依据节点的角色分成`主节点`与`工作节点`进行，这两者流程如下所示:

1. 取得集群中的kubeadm ClusterConfiguration 信息，并识别该节点为什么角色。若是主节点的话，则会进行控制平面组件的更新(流程同kubeadm upgrade apply)。
2. 从集群取得最新版本的kubelet内容，并覆写到`/var/lib/kubelet/config.yaml`中。
3. 更新当前节点的`/var/lib/kubelet/kubeadm-flags.env`内容，以确保使用新版本的参数。