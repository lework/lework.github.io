---
layout: post
title: "利用ansible来做kubernetes 1.10.3集群高可用的一键部署"
date: "2018-05-29 10:51:08"
categories: kubernetes
tags: Ansible kubernetes
excerpt: "请读者务必保持环境一致 安装过程中需要下载所需系统包，请务必使所有节点连上互联网。 本次安装的集群节点信息 实验环境：VMware的虚拟机 另外..."
auth: lework
---
* content
{:toc}

> 请读者务必保持环境一致

> 安装过程中需要下载所需系统包，请务必使所有节点**连上互联网**。

# 本次安装的集群节点信息
---
实验环境：VMware的虚拟机

IP地址 | 主机名 | CPU|内存 
 :-: | :-: |  :-: |  :-:
192.168.77.133 | k8s-m1| 6核|6G
192.168.77.134 | k8s-m2| 6核|6G
192.168.77.135 | k8s-m3| 6核|6G
192.168.77.136 | k8s-n1| 6核|6G
192.168.77.137 | k8s-n2| 6核|6G
192.168.77.138 | k8s-n3| 6核|6G

**另外由所有 master节点提供一组VIP `192.168.77.140`。**


# 本次安装的集群拓扑图
---

![image.png](/assets/images/Ansible/3629406-ccd39e6cc3daaae5.png)

# 本次使用到的ROLE
- [Ansible Role 系统环境 之【epel源设置】](http://www.jianshu.com/p/6d749a8012b9)
- [Ansible Role 系统环境 之【hostnames】](https://www.jianshu.com/p/59173fcf0085)
- [Ansible Role 容器 之【docker】](http://www.jianshu.com/p/9fc767887b12)
- [Ansible Role 容器 之【kubernetes】](https://www.jianshu.com/p/731b945c4603)

> ansible role怎么用请看下面文章
- [ Ansible Role【怎么用？】](http://www.jianshu.com/p/585303ab4b02)

# 集群安装方式

以static pod方式安装kubernetes ha高可用集群。

# Ansible管理节点操作
---

OS： `CentOS Linux release 7.4.1708 (Core)`
ansible:  `2.5.3`

######  安装Ansible
```bash
# yum -y install ansible
# ansible --version
ansible 2.5.3
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Aug  4 2017, 00:39:18) [GCC 4.8.5 20150623 (Red Hat 4.8.5-16)]
```
######  配置ansible
```bash
# sed -i 's|#host_key_checking|host_key_checking|g' /etc/ansible/ansible.cfg
```

######  下载role
```bash
# yum -y install git
# git clone https://github.com/lework/Ansible-roles.git /etc/ansible/roles
正克隆到 '/etc/ansible/roles'...
remote: Counting objects: 1767, done.
remote: Compressing objects: 100% (20/20), done.
remote: Total 1767 (delta 5), reused 24 (delta 4), pack-reused 1738
接收对象中: 100% (1767/1767), 427.96 KiB | 277.00 KiB/s, done.
处理 delta 中: 100% (639/639), done.
```

######  下载kubernetes-files.zip文件
> 这是为了适应国情，导出所需的谷歌docker image，方便大家使用。

文件下载链接：`https://pan.baidu.com/s/1BNMJLEVzCE8pvegtT7xjyQ` 密码：`qm4k`

```bash
# yum -y install unzip
# unzip kubernetes-files.zip -d /etc/ansible/roles/kubernetes/files/
```

######  配置主机信息
```bash
# cat /etc/ansible/hosts
[k8s-master]
192.168.77.133
192.168.77.134
192.168.77.135
[k8s-node]
192.168.77.136
192.168.77.137
192.168.77.138
[k8s-cluster:children]
k8s-master
k8s-node
[k8s-cluster:vars]
ansible_ssh_pass=123456
```
> `k8s-master`组为所有的master节点主机。`k8s-node`组为所有的node节点主机。`k8s-cluster`包含`k8s-master`和`k8s-node`组的所有主机。

> 请注意, 主机名称请用小写字母, 大写字母会出现找不到主机的问题。

######  配置playbook
```bash
# cat /etc/ansible/k8s.yml
---
# 初始化集群
- hosts: k8s-cluster
  serial: "100%"
  any_errors_fatal: true
  vars:
    - ipnames:
        '192.168.77.133': 'k8s-m1'
        '192.168.77.134': 'k8s-m2'
        '192.168.77.135': 'k8s-m3'
        '192.168.77.136': 'k8s-n1'
        '192.168.77.137': 'k8s-n2'
        '192.168.77.138': 'k8s-n3'
  roles:
    - hostnames
    - repo-epel
    - docker

# 安装master节点
- hosts: k8s-master
  any_errors_fatal: true
  vars:
    - kubernetes_master: true
    - kubernetes_apiserver_vip: 192.168.77.140
  roles:
    - kubernetes

# 安装node节点
- hosts: k8s-node
  any_errors_fatal: true
  vars:
    - kubernetes_node: true
    - kubernetes_apiserver_vip: 192.168.77.140
  roles:
    - kubernetes
    
# 安装addons应用
- hosts: k8s-master
  any_errors_fatal: true
  vars:
    - kubernetes_addons: true
    - kubernetes_ingress_controller: nginx
    - kubernetes_apiserver_vip: 192.168.77.140
  roles:
    - kubernetes
```
> kubernetes_ingress_controller  还可以选择`traefik`

######  执行playbook
```bash
# ansible-playbook /etc/ansible/k8s.yml
......
real	26m44.153s
user	1m53.698s
sys	0m55.509s
```
[![asciicast](/assets/images/Ansible/3629406-4bdab1fccecd4185.png)](https://asciinema.org/a/1saMof2HuDhY0ujkoS2UyuSl7)

######  验证集群版本
```bash
# kubectl version
Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:17:39Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.3", GitCommit:"2bba0127d85d5a46ab4b778548be28623b32d0b0", GitTreeState:"clean", BuildDate:"2018-05-21T09:05:37Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
```

######  验证集群状态
```bash
kubectl -n kube-system get po -o wide -l k8s-app=kube-proxy
kubectl -n kube-system get po -l k8s-app=kube-dns
kubectl -n kube-system get po -l k8s-app=calico-node -o wide
calicoctl node status
kubectl -n kube-system get po,svc -l k8s-app=kubernetes-dashboard
kubectl -n kube-system get po,svc | grep -E 'monitoring|heapster|influxdb'
kubectl -n ingress-nginx get pods
kubectl -n kube-system get po -l app=helm
kubectl -n kube-system logs -f kube-scheduler-k8s-m2
helm version
```
> 这里就不写结果了。

######  查看addons访问信息

> 在第一台master服务器上
```
kubectl cluster-info
Kubernetes master is running at https://192.168.77.140:6443
Elasticsearch is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy
heapster is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/heapster/proxy
Kibana is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
kube-dns is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
monitoring-grafana is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
monitoring-influxdb is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb:http/proxy
```

```bash
# cat ~/k8s_addons_access
```

> 集群部署完成后，建议重启集群所有节点。

> 如遇到问题，请加QQ群咨询： `425931784`
