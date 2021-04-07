---
layout: post
title: "使用ansible来做kubernetes 1.14.3集群高可用的一键部署"
date: "2019-07-13 09:22:33"
category: kubernetes
tags: ansible kubernetes k8s-install
author: lework
---
* content
{:toc}

> 请读者务必保持环境一致

> 安装过程中需要下载所需系统包，请务必使所有节点**连上互联网**。

## 本次安装的集群节点信息

实验环境：VMware的虚拟机

| System OS          |   IP Address   | Docker  |    Kernel    | Hostname | Cpu  | Memory | Application      |
| ------------------ | :------------: | :-----: | :----------: | :------: | ---- | :----: | ---------------- |
| CentOS    7.4.1708 | 192.168.77.130 | 18.09.6 | 5.1.11-1.el7 |  k8s-m1  | 2C   |   4G   | k8s-master、etcd |
| CentOS    7.4.1708 | 192.168.77.131 | 18.09.6 | 5.1.11-1.el7 |  k8s-m2  | 2C   |   4G   | k8s-master、etcd |
| CentOS    7.4.1708 | 192.168.77.132 | 18.09.6 | 5.1.11-1.el7 |  k8s-m3  | 2C   |   4G   | k8s-master、etcd |
| CentOS    7.4.1708 | 192.168.77.133 | 18.09.6 | 5.1.11-1.el7 |  k8s-n1  | 2C   |   4G   | k8s-node         |
| CentOS    7.4.1708 | 192.168.77.134 | 18.09.6 | 5.1.11-1.el7 |  k8s-n2  | 2C   |   4G   | k8s-node         |

## 本次安装的集群拓扑图

![k8s-node-ha](/assets/images/kubernetes/k8s-node-ha.png)




## 网络信息

- Cluster IP CIDR: `10.244.0.0/16`
- Service Cluster IP CIDR: `10.96.0.0/12`
- Service DNS IP: `10.96.0.10`
- DNS DN: `cluster.local`
- INGRESS IP: `192.168.77.140`
- External DNS IP: `192.168.77.141`

## 本次使用到的ROLE
- [Ansible Role 系统环境 之【epel源设置】](https://github.com/lework/Ansible-roles/tree/master/repo-epel)
- [Ansible Role 系统环境 之【hostnames】](https://github.com/lework/Ansible-roles/tree/master/hostnames)
- [Ansible Role 系统环境 之【ssh-keys】](https://github.com/lework/Ansible-roles/tree/master/ssh-keys)
- [Ansible Role 系统环境 之【ntp】](https://github.com/lework/Ansible-roles/tree/master/ntp)
- [Ansible Role 系统环境 之【update-kernel 】](https://github.com/lework/Ansible-roles/tree/master/update-kernel) 
- [Ansible Role 容器 之【docker】](https://github.com/lework/Ansible-roles/tree/master/docker)
- [Ansible Role 容器 之【kubernetes-bin】](https://github.com/lework/Ansible-roles/tree/master/kubernetes-bin)

> ansible role怎么用请看下面文章

- [ Ansible Role【怎么用？】](http://www.jianshu.com/p/585303ab4b02)

## 集群安装方式

以hyperkube二进制方式安装kubernetes 1.14.3 ha 集群。

## Ansible管理节点操作
---

OS： `CentOS Linux release 7.4.1708 (Core)`
ansible:  `2.8.1`

###  安装Ansible
```
# yum -y install ansible
# ansible --version
ansible 2.8.1
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Aug  4 2017, 00:39:18) [GCC 4.8.5 20150623 (Red Hat 4.8.5-16)]
```
###  配置ansible
```
# sed -i 's|#host_key_checking|host_key_checking|g' /etc/ansible/ansible.cfg
```

###  下载role
```
# yum -y install git
# git clone https://github.com/lework/Ansible-roles.git /etc/ansible/roles
```

###  下载k8s-v1.14.3.7za文件
> 为了加速部署过程，这里将需要下载的镜像准备好了.

文件下载链接：`https://pan.baidu.com/s/1eBPPI6kDxvbynH43--ly5g` 密码：`y39z`

```
# yum -y install p7zip
# 7za x k8s-v1.14.3.7za -r -o/opt/
# cp -rf  v1.14.3/* /etc/ansible/roles/kubernetes-bin/files/
```

###  配置主机信息
```
# cat /etc/ansible/hosts
[k8s_master]
192.168.77.130
192.168.77.131
192.168.77.132
[k8s_node]
192.168.77.133
192.168.77.134
[k8s_cluster:children]
k8s_master
k8s_node
[k8s_cluster:vars]
ansible_ssh_pass=123456
```
> `k8s_master`组为所有的master节点主机。`k8s_node`组为所有的node节点主机。`k8s_cluster`包含`k8s_master`和`k8s_node`组的所有主机。

> 请注意, 主机名称请用小写字母, 大写字母会出现找不到主机的问题。

###  配置playbook
```
# cat /etc/ansible/k8s.yml
---
# 初始化节点
- hosts: k8s_cluster
    serial: "100%"
    any_errors_fatal: true
    vars:
    - ipnames:
        '192.168.77.130': 'k8s-m1'
        '192.168.77.131': 'k8s-m2'
        '192.168.77.132': 'k8s-m3'
        '192.168.77.133': 'k8s-n1'
        '192.168.77.134': 'k8s-n2'
    roles:
    - hostnames
    - { role: ssh-keys, ssh_keys_host: '192.168.77.130' }
    - repo-epel
    - ntp
    - docker
    - update-kernel 

# 安装master节点
- hosts: k8s_master
    any_errors_fatal: true
    vars:
    - kubernetes_master: true
    roles:
    - kubernetes-bin

# 安装node节点
- hosts: k8s_node
    any_errors_fatal: true
    vars:
    - kubernetes_node: true
    roles:
    - kubernetes-bin

# 安装addons组件
- hosts: k8s_master
    any_errors_fatal: true
    vars:
    - kubernetes_addons: true
    - kubernetes_ingress_ip: 192.168.77.140
    - kubernetes_external_dns_ip: 192.168.77.141
    roles:
    - kubernetes-bin
```

- `ipnames` 变量设置的是`/etc/hosts`里的内容
- `ssh_keys_host` 指定哪台主机免密登录其他节点
- `kubernetes_master` 部署k8s master节点
- `kubernetes_node` 部署k8s node节点
- `kubernetes_addons` 部署k8s addons组件
- `kubernetes_ingress_ip` 指定ingress svc的externalIP
- `kubernetes_external_dns_ip` 指定core svc的 externalIP

更多变量设置请看默认变量文件`/etc/ansible/roles/kubernetes-bin/defaults/main.yml`

###  执行playbook
```
# time ansible-playbook /etc/ansible/k8s.yml
......
real    25m17.295s
user    3m13.059s
sys     1m46.286s 
```
[![asciicast](https://asciinema.org/a/257020.svg)](https://asciinema.org/a/257020)

###  验证集群版本
```
# kubectl version
Client Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.3", GitCommit:"5e53fd6bc17c0dec8434817e69b04a25d8ae0ff0", GitTreeState:"clean", BuildDate:"2019-06-06T01:36:19Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.3", GitCommit:"5e53fd6bc17c0dec8434817e69b04a25d8ae0ff0", GitTreeState:"clean", BuildDate:"2019-06-06T01:36:19Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

###  验证集群状态
```
kubectl get cs
kubectl get csr
kubectl get nodes
kubectl get ns
kubectl get all --all-namespaces=true
helm version
etcdctl member list
kubectl -n kube-system exec calicoctl -- calicoctl get node -o wide
ipvsadm -Ln
```
> 这里就不写结果了。

### 查看addons访问信息

> 在第一台master服务器上

```
cat ~/k8s-addons-access.md

## secret
dashboard_secret: 
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1kZ3p4ayIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImY1NDgwMzQ0LWE0OGYtMTFlOS05MzU3LTAwMGMyOTg4MDFjMCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.wKXs_K633Y-EtfGCPRQjMVsklzpuO7NshhCwgTasmrRpB1MP48kx5pa2D6Mqg6BAv40AGFGdq6m6EK6Beuj-E7brGvvdg3h6P2nDUKBUxEq6dXwBDyNTRiGaI-QmSJHjn8yt59gl7rdgCWrrL_B5bo-umsjV3jKk4tIOMX7RgdqSB6sDkgPoILiC9cNOKl3JIfqH3dXwYHJwJylS2dwvxCbMFpNZtXhCKGP_lciaU0ESr3OGK03-kHCUmWyisX-WnwBbCYvmNIArwhYD8QwHOXU9PI4i2DF48Fg6TAxMFsOpnJd3AvzWCIdaRVmfvQdNTYMEZmkA04oFyPGI_Q9c_g


## host bing
192.168.77.140 kubernetes-dashboard.k8s.local
192.168.77.140 alertmanager.monitoring.k8s.local
192.168.77.140 grafana.monitoring.k8s.local
192.168.77.140 prometheus.monitoring.k8s.local
192.168.77.140 scope.weave.k8s.local


## http access
https://kubernetes-dashboard.k8s.local
http://alertmanager.monitoring.k8s.local
http://grafana.monitoring.k8s.local
http://prometheus.monitoring.k8s.local
http://scope.weave.k8s.local

## DNS
TCP 192.168.77.141
UDP 192.168.77.141

dig @192.168.77.141 A scope.weave.k8s.local +noall +answer
```

> 集群部署完成后，建议重启集群所有节点。

> 如遇到问题，请加QQ群咨询： `425931784`  (已满)   `756527917`

如果你想手动的一步一步的安装集群，请看 [全手工使用hyperkube二进制安装Kubernetes v14.3 ha集群
](https://lework.github.io/2019/06/30/k8s-ha-bin-install/)