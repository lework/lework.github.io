---
layout: post
title: Kubernetes-v1-10-3-HA-集群手工安装教程
categories: kubernetes
tags: Ansible kubernetes k8s-install
date: 2018-06-01 12:04
excerpt: "之所以这么累的手工安装Kubernetes v1.10.3 HA 集群，是为了深入的了解Kubernetes各个组件的关系和功能，本次使用static pod方式安装服务。"
auth: lework
---
* content
{:toc}


# 目的

之所以这么累的手工安装Kubernetes v1.10.3 HA 集群，是为了深入的了解Kubernetes各个组件的关系和功能，本次使用static pod方式安装服务。

> 如果不想手动，或者想学习自动化部署的，可以看看[Ansible 应用 之 【利用ansible来做kubernetes 1.10.3集群高可用的一键部署】](https://www.jianshu.com/p/265cfb0811b2) 文章。

# 本次安装的集群节点信息
---
> 请读者务必保持环境一致

实验环境：VMware的虚拟机
系统： centos 7.4 x64

IP地址 | 主机名 | CPU|内存 
 :-: | :-: |  :-: |  :-:
192.168.77.133 | k8s-m1| 6核|6G
192.168.77.134 | k8s-m2| 6核|6G
192.168.77.135 | k8s-m3| 6核|6G
192.168.77.136 | k8s-n1| 6核|6G
192.168.77.137 | k8s-n2| 6核|6G
192.168.77.138 | k8s-n3| 6核|6G

**另外由所有 master节点提供一组VIP `192.168.77.140`。**
>  这里`m`为主要控制节点，`n`为应用工作节点。所有操作全部用`root`用户通过`ssh`协议`22`端口。

# 本次安装的软件版本
- Kubernetes v1.10.3
- CNI v2.0.5
- Etcd v3.1.13
- Calico v3.0.4
- DNS 1.14.10
- Haproxy 1.8.9
- Keepalived 1.3.9
- Dashboard v1.8.3
- Elasticsearch v6.2.4
- Traefik v1.6.2
- Nginx 0.15.0
- Docker CE latest version

![image.png](/assets/images/kubernetes/3629406-ccd39e6cc3daaae5.png)

# 安装前准备

> 安装过程中需要下载所需系统包，请务必使所有节点**连上互联网**。

**在ks8-m1节点上操作**

> 请注意, 主机名称请用小写字母, 大写字母会出现找不到主机的问题。

设置主机名解析
```bash
cat <<EOF > /etc/hosts
192.168.77.133 k8s-m1
192.168.77.134 k8s-m2
192.168.77.135 k8s-m3
192.168.77.136 k8s-n1
192.168.77.137 k8s-n2
192.168.77.138 k8s-n3
EOF
```
建立免密登录其他节点
```bash
ssh-keygen -t rsa -P '' -f /root/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir ~/.ssh/"
    scp ~/.ssh/authorized_keys ${NODE}:~/.ssh/authorized_keys
done
```

同步主机名到其他节点
```bash
for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    scp /etc/hosts ${NODE}:/etc/hosts
done
```
关闭防火墙
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "systemctl stop firewalld && systemctl disable firewalld"
done
```
关闭网络服务
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "systemctl stop NetworkManager && systemctl disable NetworkManager"
done
```
关闭selinux
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "setenforce 0 && sed -i 's#=enforcing#=disabled#g'/etc/selinux/config"
done
```
修改系统参数
```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.nr_hugepages=757
vm.swappiness=0
EOF
sysctl -p /etc/sysctl.d/k8s.conf
for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    scp /etc/sysctl.d/k8s.conf ${NODE}:/etc/sysctl.d/k8s.conf
    ssh ${NODE} "sysctl -p /etc/sysctl.d/k8s.conf"
done
```
关闭swap
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "swapoff -a && sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab"
done
```
安装docker-ce
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo && \
      sed -i 's#download.docker.com#mirrors.ustc.edu.cn/docker-ce#g' /etc/yum.repos.d/docker-ce.repo && \
      yum -y install docker-ce && \
      systemctl enable docker && systemctl start docker"
done
```

加载所需要的镜像
> 这是为了适应国情，导出所需的谷歌docker image，方便大家使用。

文件下载链接：`https://pan.baidu.com/s/1BNMJLEVzCE8pvegtT7xjyQ` 密码：`qm4k`
```bash
yum -y install unzip
unzip  kubernetes-files.zip -d /opt/files
find /opt/files/images/all/ -type f -name '*.tar' -exec docker load -i {} \;
find  /opt/files/images/master/ -type f -name '*.tar' -exec docker load -i {} \;

for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /opt/software/kubernetes/images"
    scp /opt/files/images/all/*  ${NODE}:/opt/software/kubernetes/images/
done

for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    scp /opt/files/images/master/*  ${NODE}:/opt/software/kubernetes/images/
done

for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "find /opt/software/kubernetes/images/ -type f -name '*.tar' -exec docker load -i {} \;"
done
```
部署kubelet
```bash
mdkir -p /usr/local/kubernetes/bin/
cp /opt/files/bin/kubelet /usr/local/kubernetes/bin/kubelet
chmod +x /usr/local/kubernetes/bin/kubelet
for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mdkir -p /usr/local/kubernetes/bin/"
    scp /usr/local/kubernetes/bin/kubelet ${NODE}:/usr/local/kubernetes/bin/kubelet
    ssh ${NODE} "chmod +x /usr/local/kubernetes/bin/kubelet"
done
```

部署kubectl
```bash
cp /opt/files/bin/kubectl /usr/local/kubernetes/bin/
chmod +x /usr/local/kubernetes/bin/kubectl
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    scp /usr/local/kubernetes/bin/kubectl ${NODE}:/usr/local/kubernetes/bin/kubectl
    ssh ${NODE} "chmod +x /usr/local/kubernetes/bin/kubectl"
done
```

部署Kubernetes CNI
```bash
mkdir -p /usr/local/cni/bin /etc/cni/net.d
cp /opt/files/bin/cni/* /usr/local/cni/bin
echo 'PATH=/usr/local/cni/bin:$PATH' >> /etc/profile
source /etc/profile
for NODE in k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /usr/local/cni/bin /etc/cni/net.d && \
       echo 'PATH=/usr/local/cni/bin:$PATH' >> /etc/profile"
    scp /usr/local/cni/bin*  ${NODE}:/usr/local/cni/bin
done
```

部署CFSSL工具
```bash
mkdir  -p /usr/local/cfssl/bin
cp /opt/files/bin/cfssl/* /usr/local/cfssl/bin/
chmod +x /usr/local/cfssl/bin/*
echo 'PATH=/usr/local/cfssl/bin:$PATH' >> /etc/profile
source /etc/profile
```

下载配置文件
> 下载本次使用的配置文件
```bash
yum -y install git
git clone https://github.com/lework/kubernetes-manual.git /opt/kubernetes-manual
cp -rf /opt/kubernetes-manual/v1.10/conf/kubernetes /etc/kubernetes
```


# 建立CA和证书

**在k8s-m1节点上操作**
创建Etcd所需的证书
```bash
mkdir -p /etc/etcd/ssl && cd /etc/etcd/ssl
echo '{"signing":{"default":{"expiry":"87600h"},"profiles":{"kubernetes":{"usages":["signing","key encipherment","server auth","client auth"],"expiry":"87600h"}}}}' > ca-config.json
echo '{"CN":"etcd","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"etcd","OU":"Etcd Security"}]}' > etcd-ca-csr.json
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare etcd-ca
echo '{"CN":"etcd","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"etcd","OU":"Etcd Security"}]}' > etcd-csr.json
cfssl gencert \
  -ca=etcd-ca.pem \
  -ca-key=etcd-ca-key.pem \
  -config=ca-config.json \
  -hostname=127.0.0.1,192.168.77.133,192.168.77.134,192.168.77.135 \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd
```
> -hostname需修改成所有 masters 节点。

将etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem 文件拷贝到其它master节点上
```bash
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/etcd/ssl"
    for FILE in etcd-ca-key.pem  etcd-ca.pem etcd-key.pem etcd.pem; do
      scp /etc/etcd/ssl/${FILE} ${NODE}:/etc/etcd/ssl/${FILE}
    done
  done
```

Kubernetes
```bash
mkdir -p /etc/kubernetes/pki && cd /etc/kubernetes/pki
echo '{"signing":{"default":{"expiry":"87600h"},"profiles":{"kubernetes":{"usages":["signing","key encipherment","server auth","client auth"],"expiry":"87600h"}}}}' > ca-config.json
echo '{"CN":"kubernetes","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"Kubernetes","OU":"Kubernetes-manual"}]}' > ca-csr.json
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

API Server Certificate
```bash
echo '{"CN":"kube-apiserver","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"Kubernetes","OU":"Kubernetes-manual"}]}' > apiserver-csr.json 
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.96.0.1,192.168.77.140,127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  apiserver-csr.json | cfssljson -bare apiserver
```
> 这边-hostname的10.96.0.1是Cluster IP的Kubernetes端点;
192.168.77.140为虚拟IP 位址(VIP);
kubernetes.default为Kubernets DN。

Front Proxy Certificate
```bash
echo '{"CN":"kubernetes","key":{"algo":"rsa","size":2048}}' > front-proxy-ca-csr.json
cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare front-proxy-ca
echo '{"CN":"front-proxy-client","key":{"algo":"rsa","size":2048}}' > front-proxy-client-csr.json
cfssl gencert \
  -ca=front-proxy-ca.pem \
  -ca-key=front-proxy-ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  front-proxy-client-csr.json | cfssljson -bare front-proxy-client
```

Admin Certificate
```bash
echo '{"CN":"admin","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"system:masters","OU":"Kubernetes-manual"}]}' > admin-csr.json 
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

生成admin.conf的kubeconfig 
```bash
# admin set cluster
kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://192.168.77.140:6443 \
    --kubeconfig=../admin.conf

# admin set credentials
kubectl config set-credentials kubernetes-admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=../admin.conf

# admin set context
kubectl config set-context kubernetes-admin@kubernetes \
    --cluster=kubernetes \
    --user=kubernetes-admin \
    --kubeconfig=../admin.conf

# admin set default context
kubectl config use-context kubernetes-admin@kubernetes \
    --kubeconfig=../admin.conf
```
> --server要更换成VIP地址

Controller Manager Certificate
```bash
echo '{"CN":"system:kube-controller-manager","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"syroller-manager","OU":"Kubernetes-manual"}]}' > manager-csr.json
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  manager-csr.json | cfssljson -bare controller-manager
```

生成controller-manager的 kubeconfig
```bash
# controller-manager set cluster
kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://192.168.77.140:6443 \
    --kubeconfig=../controller-manager.conf

# controller-manager set credentials
kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=controller-manager.pem \
    --client-key=controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=../controller-manager.conf

# controller-manager set context
kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=../controller-manager.conf

# controller-manager set default context
kubectl config use-context system:kube-controller-manager@kubernetes \
    --kubeconfig=../controller-manager.conf
```
> --server要更换成VIP地址

Scheduler Certificate
```bash
echo '{"CN":"system:kube-scheduler","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","ST":"Shanghai","L":"Shanghai","O":"system:kube-scheduler","OU":"Kubernetes-manual"}]}' > scheduler-csr.json
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  scheduler-csr.json | cfssljson -bare scheduler
```

生成scheduler的 kubeconfig
```bash
# scheduler set cluster
kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://192.168.77.140:6443 \
    --kubeconfig=../scheduler.conf

# scheduler set credentials
kubectl config set-credentials system:kube-scheduler \
    --client-certificate=scheduler.pem \
    --client-key=scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=../scheduler.conf

# scheduler set context
kubectl config set-context system:kube-scheduler@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-scheduler \
    --kubeconfig=../scheduler.conf

# scheduler use default context
kubectl config use-context system:kube-scheduler@kubernetes \
    --kubeconfig=../scheduler.conf
```
> --server要更换成VIP地址

Master Kubelet Certificate
```bash
echo '{"CN":"system:node:$NODE","key":{"algo":"rsa","size":2048},"names":[{"C":"CN","L":"Shanghai","ST":"Shanghai","O":"system:nodes","OU":"Kubernetes-manual"}]}' > kubelet-csr.json

for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    cp kubelet-csr.json kubelet-$NODE-csr.json;
    sed -i "s/\$NODE/$NODE/g" kubelet-$NODE-csr.json;
    cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=ca-config.json \
      -hostname=$NODE \
      -profile=kubernetes \
      kubelet-$NODE-csr.json | cfssljson -bare kubelet-$NODE
  done
```
完成后复制kubelet证书到其它节点
```bash
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/kubernetes/pki"
    for FILE in kubelet-$NODE-key.pem kubelet-$NODE.pem ca.pem; do
      scp /etc/kubernetes/pki/${FILE} ${NODE}:/etc/kubernetes/pki/${FILE}
    done
  done
```

生成kubelet的 kubeconfig
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "source /etc/profile; cd /etc/kubernetes/pki && \
      kubectl config set-cluster kubernetes \
        --certificate-authority=ca.pem \
        --embed-certs=true \
        --server=https://192.168.77.140:6443 \
        --kubeconfig=../kubelet.conf && \
      kubectl config set-credentials system:node:${NODE} \
        --client-certificate=kubelet-${NODE}.pem \
        --client-key=kubelet-${NODE}-key.pem \
        --embed-certs=true \
        --kubeconfig=../kubelet.conf && \
      kubectl config set-context system:node:${NODE}@kubernetes \
        --cluster=kubernetes \
        --user=system:node:${NODE} \
        --kubeconfig=../kubelet.conf && \
      kubectl config use-context system:node:${NODE}@kubernetes \
        --kubeconfig=../kubelet.conf"
  done
```
> --server要更换成VIP地址

Service Account Key

Service account 不是透过 CA 进行认证，因此不要透过 CA 来做 Service account key 的检查，这边建立一组 Private 与 Public 金钥提供给 Service account key 使用：
```bash
openssl genrsa -out sa.key 2048
openssl rsa -in sa.key -pubout -out sa.pub
```

复制凭证档案至其他master节点
```bash
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    for FILE in $(ls ca*.pem sa.* apiserver*.pem front*.pem scheduler*.pem); do
      scp /etc/kubernetes/pki/${FILE} ${NODE}:/etc/kubernetes/pki/${FILE}
    done
  done
```

复制Kubernetes config 到其它master节点
```bash
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    for FILE in admin.conf controller-manager.conf scheduler.conf; do
      scp /etc/kubernetes/${FILE} ${NODE}:/etc/kubernetes/${FILE}
    done
  done
```

> 如网卡名称不是`ens33`, 请修改keepalived文件

Kubernetes Masters 角色配置manifests
```bash
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/kubernetes/manifests"
    scp /etc/kubernetes/manifests/* ${NODE}:/etc/kubernetes/manifests/
  done
```

> 如改变ip的话，请修改配置文件

生成encryption.yml
```bash
cat <<EOF > /etc/kubernetes/encryption.yml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: $(head -c 32 /dev/urandom | base64)
      - identity: {}
EOF
```

建立audit-policy.yml
```bash
cat <<EOF > /etc/kubernetes/audit-policy.yml
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF
```

拷贝到其他master节点
```bash
for NODE in k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    for FILE in encryption.yml audit-policy.yml; do
      scp /etc/kubernetes/${FILE} ${NODE}:/etc/kubernetes/${FILE}
    done
  done
```

配置etcd
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    scp /opt/kubernetes-manual/v1.10/conf/etcd/etcd.config.yml ${NODE}:/etc/etcd/etcd.config.yml
    ssh ${NODE} "sed -i "s/\${HOSTNAME}/${HOSTNAME}/g" /etc/etcd/etcd.config.yml && \
      sed -i "s/\${PUBLIC_IP}/$(hostname -i)/g" /etc/etcd/etcd.config.yml"
done
```
> 如改变ip的话，etcd.config.yml 配置文件中的集群地址记得更改

配置haproxy.cfg
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/haproxy/"
    scp /opt/kubernetes-manual/v1.10/conf/haproxy/haproxy.cfg ${NODE}:/etc/haproxy/haproxy.cfg
done
```
> 如改变ip的话，haproxy.cfg 配置文件中的节点地址记得更改

配置kubelet.service
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/systemd/system/kubelet.service.d"
    scp /opt/kubernetes-manual/v1.10/conf/master/kubelet.service ${NODE}:/lib/systemd/system/kubelet.service
    scp /opt/kubernetes-manual/v1.10/conf/master/10-kubelet.conf ${NODE}:/etc/systemd/system/kubelet.service.d/10-kubelet.conf
done
```

建立存储目录
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /var/lib/kubelet /var/log/kubernetes /var/lib/etcd"
done
```

启动服务
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "systemctl enable kubelet.service && systemctl start kubelet.service"
done
```

验证集群
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3; do
    echo "--- $NODE ---"
    ssh ${NODE} "cp /etc/kubernetes/admin.conf ~/.kube/config"
done

kubectl get node
NAME       STATUS     ROLES     AGE       VERSION
k8s-m1     NotReady   master    31m       v1.10.3
k8s-m2     NotReady   master    31m       v1.10.3
k8s-m3     NotReady   master    31m       v1.10.3
```

定义权限
```bash
cd /etc/kubernetes/
kubectl apply -f apiserver-to-kubelet-rbac.yml
clusterrole.rbac.authorization.k8s.io "system:kube-apiserver-to-kubelet" created
clusterrolebinding.rbac.authorization.k8s.io "system:kube-apiserver" created

kubectl -n kube-system logs -f kube-scheduler-k8s-m2
…
I0518 06:38:59.423822       1 leaderelection.go:175] attempting to acquire leader lease  kube-system/kube-scheduler...
I0518 06:38:59.484043       1 leaderelection.go:184] successfully acquired lease kube-system/kube-scheduler
```

设定master节点允许Taint
```bash
kubectl taint nodes node-role.kubernetes.io/master="":NoSchedule --all
```

建立 TLS Bootstrapping RBAC
```bash
cd /etc/kubernetes/pki
export TOKEN_ID=$(openssl rand 3 -hex)
export TOKEN_SECRET=$(openssl rand 8 -hex)
export BOOTSTRAP_TOKEN=${TOKEN_ID}.${TOKEN_SECRET}
export KUBE_APISERVER="https://192.168.77.140:6443"

# bootstrap set cluster
kubectl config set-cluster kubernetes \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=../bootstrap-kubelet.conf

# bootstrap set credentials
kubectl config set-credentials tls-bootstrap-token-user \
    --token=${BOOTSTRAP_TOKEN} \
    --kubeconfig=../bootstrap-kubelet.conf

# bootstrap set context
kubectl config set-context tls-bootstrap-token-user@kubernetes \
    --cluster=kubernetes \
    --user=tls-bootstrap-token-user \
    --kubeconfig=../bootstrap-kubelet.conf

# bootstrap use default context
kubectl config use-context tls-bootstrap-token-user@kubernetes \
    --kubeconfig=../bootstrap-kubelet.conf
```

接著在k8s-m1建立 TLS bootstrap secret 來提供自動簽證使用：
```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-${TOKEN_ID}
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: ${TOKEN_ID}
  token-secret: ${TOKEN_SECRET}
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups: system:bootstrappers:default-node-token
EOF
```

在k8s-m1建立 TLS Bootstrap Autoapprove RBAC：
```bash
cd /etc/kubernetes/
kubectl apply -f kubelet-bootstrap-rbac.yml
```

# 部署Kubernetes Nodes

在k8s-m1节点上拷贝证书文件
```bash
for NODE in k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/kubernetes/{pki,manifests}"
    ssh ${NODE} "mkdir -p /etc/etcd/ssl"
    # Etcd
    for FILE in etcd-ca.pem etcd.pem etcd-key.pem; do
      scp /etc/etcd/ssl/${FILE} ${NODE}:/etc/etcd/ssl/${FILE}
    done
    # Kubernetes
    for FILE in pki/ca.pem pki/ca-key.pem bootstrap-kubelet.conf; do
      scp /etc/kubernetes/${FILE} ${NODE}:/etc/kubernetes/${FILE}
    done
  done
```
配置kubelet.service
```bash
for NODE in k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /etc/systemd/system/kubelet.service.d"
    scp /opt/kubernetes-manual/v1.10/conf/node/kubelet.service ${NODE}:/lib/systemd/system/kubelet.service
    scp /opt/kubernetes-manual/v1.10/conf/node/10-kubelet.conf ${NODE}:/etc/systemd/system/kubelet.service.d/10-kubelet.conf
done
```

建立存储目录
```bash
for NODE in k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "mkdir -p /var/lib/kubelet /var/log/kubernetes /var/lib/etcd"
done
```

启动服务
```bash
for NODE in k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "systemctl enable kubelet.service && systemctl start kubelet.service"
done
```


查看集群状态
```bash
kubectl get csr
NAME                                                   AGE       REQUESTOR                 CONDITION
csr-dgt8g                                              20m       system:node:k8s-m2        Approved,Issued
csr-tdlxx                                              33m       system:node:k8s-m2        Approved,Issued
csr-v8j9p                                              20m       system:node:k8s-m1        Approved,Issued
csr-vvxfv                                              33m       system:node:k8s-m1        Approved,Issued
csr-w8mpq                                              33m       system:node:k8s-m3        Approved,Issued
csr-zhvn6                                              20m       system:node:k8s-m3        Approved,Issued
node-csr-LpmPGk2_tkxJlas75v8RuNcf0WMh_uSlWzSC2R4_NGI   2m        system:bootstrap:40ed9e   Approved,Issued
node-csr-Y_f2xcvkLEJmyi3ogSPkx-eYPjuS2osGlH2n1jIxPXY   2m        system:bootstrap:40ed9e   Approved,Issued
node-csr-kSXMp4fYysfK5ttM-k7ZdGGmuh7hMWvVXiZVft1MWx4   2m        system:bootstrap:40ed9e   Approved,Issued

kubectl get node
NAME       STATUS     ROLES     AGE       VERSION
k8s-m1     NotReady   master    33m       v1.10.3
k8s-m2     NotReady   master    33m       v1.10.3
k8s-m3     NotReady   master    33m       v1.10.3
k8s-n1   NotReady   node      2m        v1.10.3
k8s-n2   NotReady   node      2m        v1.10.3
K8s-n3   NotReady   node      2m        v1.10.3
```

# 部署Kubernetes Core Addons 

### Kubernetes Proxy

使用kube-proxy.yml.conf 建立Kubernetes Proxy Addon
```bash
cd /etc/kubernetes/
kubectl apply -f addons/kube-proxy.yml
serviceaccount "kube-proxy" created
clusterrolebinding.rbac.authorization.k8s.io "system:kube-proxy" created
configmap "kube-proxy" created
daemonset.apps "kube-proxy" created

kubectl -n kube-system get po -o wide -l k8s-app=kube-proxy
NAME               READY     STATUS    RESTARTS   AGE       IP               NODE
kube-proxy-2qwqx   1/1       Running   0          2m        192.168.77.135   k8s-m3
kube-proxy-csf4q   1/1       Running   0          2m        192.168.77.134   k8s-m2
kube-proxy-f4phm   1/1       Running   0          2m        192.168.77.133   k8s-m1
kube-proxy-ffqcm   1/1       Running   0          2m        192.168.77.138   k8s-n3
kube-proxy-vngnj   1/1       Running   0          2m        192.168.77.137   k8s-n2
kube-proxy-xhrzg   1/1       Running   0          2m        192.168.77.136   k8s-n1
```

### Kubernetes DNS
```bash
[root@k8s-m1 ~]# kubectl apply -f addons/kube-dns.yml
serviceaccount "kube-dns" created
service "kube-dns" created
deployment.extensions "kube-dns" created
[root@k8s-m1 ~]# kubectl -n kube-system get po -l k8s-app=kube-dns
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-74ffb4cdb8-xhk28   0/3       Pending   0          14s
```
这边会发现处于Pending状态，是由于Kubernetes Pod Network 还未建立完成，因此所有节点会处于NotReady状态，而造成Pod 无法被排程分配到指定节点上启动，由于为了解决该问题，下节将说明如何建立Pod Network。

###  Calico Network 安裝

> 如网卡名称不是`ens33`, 请修改文件

```bash
[root@k8s-m1 ~]# kubectl apply -f  calico.yml
configmap "calico-config" created
daemonset.extensions "calico-node" created
deployment.extensions "calico-kube-controllers" created
clusterrolebinding.rbac.authorization.k8s.io "calico-cni-plugin" created
clusterrole.rbac.authorization.k8s.io "calico-cni-plugin" created
serviceaccount "calico-cni-plugin" created
clusterrolebinding.rbac.authorization.k8s.io "calico-kube-controllers" created
clusterrole.rbac.authorization.k8s.io "calico-kube-controllers" created
serviceaccount "calico-kube-controllers" created

[root@k8s-m1 ~]# kubectl -n kube-system get po -l k8s-app=calico-node -o wide
NAME                READY     STATUS    RESTARTS   AGE       IP               NODE
calico-node-4gjvm   2/2       Running   0          1m        192.168.77.134   k8s-m2
calico-node-4pxn7   2/2       Running   0          1m        192.168.77.135   k8s-m3
calico-node-6kdhz   2/2       Running   0          1m        192.168.77.136   k8s-n1
calico-node-f69tx   2/2       Running   0          1m        192.168.77.137   k8s-n2
calico-node-msgvs   2/2       Running   0          1m        192.168.77.138   k8s-n3
calico-node-qct95   2/2       Running   0          1m        192.168.77.133   k8s-m1
```
在k8s-m1下载 Calico CLI 来查看 Calico nodes
```bash
cp bin/calic/calicoctl /usr/local/bin/
chmod u+x /usr/local/bin/calicoctl
cat <<EOF > ~/calico-rc
export ETCD_ENDPOINTS="https://192.168.77.133:2379,https://192.168.77.134:2379,https://192.168.77.135:2379"
export ETCD_CA_CERT_FILE="/etc/etcd/ssl/etcd-ca.pem"
export ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
export ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
EOF

. ~/calico-rc
calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+----------------+-------------------+-------+----------+-------------+
| 192.168.77.134 | node-to-node mesh | up    | 11:51:10 | Established |
| 192.168.77.135 | node-to-node mesh | up    | 11:51:08 | Established |
| 192.168.77.136 | node-to-node mesh | up    | 11:51:16 | Established |
| 192.168.77.138 | node-to-node mesh | up    | 11:51:11 | Established |
| 192.168.77.137 | node-to-node mesh | up    | 11:51:26 | Established |
+----------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

查看 pending 的 pod 是否已执行：
```bash
kubectl -n kube-system get po -l k8s-app=kube-dns
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-74ffb4cdb8-xhk28   3/3       Running   0          1h
```

# Kubernetes Extra Addons 部署

### Dashboard
```bash
kubectl apply -f addons/kubernetes-dashboard.yaml 
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created

kubectl -n kube-system get po,svc -l k8s-app=kubernetes-dashboard
NAME                                        READY     STATUS    RESTARTS   AGE
pod/kubernetes-dashboard-7d5dcdb6d9-kdnbd   1/1       Running   0          5m

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes-dashboard   ClusterIP   10.96.107.225   <none>        443/TCP   5m
```
这边会额外建立一个名称为open-api Cluster Role Binding，这仅作为方便测试时使用，在一般情况下不要开启，不然就会直接被存取所有 API:
```bash
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: open-api
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: system:anonymous
EOF
```

通过`https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login`访问

在 1.7 版本以后的 Dashboard 将不再提供所有权限，因此需要建立一个 service account 来绑定 cluster-admin role：
```bash
kubectl -n kube-system create sa dashboard
kubectl create clusterrolebinding dashboard --clusterrole cluster-admin --serviceaccount=kube-system:dashboard
SECRET=$(kubectl -n kube-system get sa dashboard -o yaml | awk '/dashboard-token/ {print $3}')
kubectl -n kube-system describe secrets ${SECRET} | awk '/token:/{print $2}'
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtdG9rZW4tajg1dmQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZmM1NjFjN2EtNWI1Zi0xMWU4LTgzMTItMDAwYzI5ZGJkMTM5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZCJ9.C24d9VjAjquEUxRaCbVFbCb-jT47bfWxP9je3hS594VqNqhwnLWqD2Hhqrc0DTbBna-WMWFXq1yiSVNFHe-c9QSgfE81p6qDQZrrGPadQLw0uhFtS71OLVD1hPz91IIhIYQsE2vCKlc0PQqOqK6AIDXFE49o0YlU_ecQ75oIbi4FeYQWMH0HrCNHEXvS2PknNINnPRuRAC2m15yhlmj2lqDO1XYYyYvYWYWv3WUd_mLH1wD_h5N5MYy1RYp3rNkDUzYE5dAwSH0A42L6M4Wf7zaY-zJkkWSwGzzL1z5uQdFteTFHmgKOl043iQ-X1I5phO-eqfS5hBu03r4lXjy9UA
```
在页面上输入令牌，就可以进入管理页面了。
![image.png]/assets/images/kubernetes/3629406-fee9f96036fc53ff.png
![image.png]/assets/images/kubernetes/3629406-80f0aceb17edc7e6.png
如果令牌过期，可以用下面命令重新获取
```bash
kubectl -n kube-system describe secrets | sed -rn '/\sdashboard-token-/,/^token/{/^token/s#\S+\s+##p}' 
```

### Heapster
Heapster 是 Kubernetes 社区维护的容器丛集监控与效能分析工具。 Heapster 会从Kubernetes apiserver 取得所有Node 资讯，然后再透过这些Node 来取得kubelet 上的资料，最后再将所有收集到资料送到Heapster 的后台储存InfluxDB，最后利用Grafana 来抓取InfluxDB 的资料源来进行视觉化。

```bash
kubectl apply -f addons/kube-monitor.yml
kubectl -n kube-system get po,svc | grep -E 'monitoring|heapster|influxdb'
pod/heapster-8655976fbd-tnzfv                  4/4       Running   0          2m
pod/influxdb-grafana-848cd4dd9c-vhvqf          2/2       Running   0          3m

service/heapster               ClusterIP   10.105.56.202    <none>        80/TCP              3m
service/monitoring-grafana     ClusterIP   10.105.75.81     <none>        80/TCP              3m
service/monitoring-influxdb    ClusterIP   10.110.213.212   <none>        8083/TCP,8086/TCP   3m
```
访问grafana
`https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy/?orgId=1`
![image.png]/assets/images/kubernetes/3629406-d1f856e188b05f12.png

### Ingress Controller
Ingress是利用 Nginx 或 HAProxy 等负载平衡器来曝露丛集内服务的元件，Ingress 主要透过设定 Ingress 规格来定义 Domain Name 映射 Kubernetes 内部 Service，这种方式可以避免掉使用过多的 NodePort 问题。

在k8s-m1通过 kubectl 来建立 Ingress Controller 即可：
```bash
kubectl create ns ingress-nginx
kubectl apply -f addons/ingress-controller.yml

serviceaccount "nginx-ingress-serviceaccount" created
clusterrole.rbac.authorization.k8s.io "nginx-ingress-clusterrole" created
role.rbac.authorization.k8s.io "nginx-ingress-role" created
rolebinding.rbac.authorization.k8s.io "nginx-ingress-role-nisa-binding" created
clusterrolebinding.rbac.authorization.k8s.io "nginx-ingress-clusterrole-nisa-binding" created
configmap "nginx-configuration" created
configmap "tcp-services" created
service "default-http-backend" created
service "ingress-nginx" created
deployment.extensions "default-http-backend" created
deployment.extensions "nginx-ingress-controller" created

[root@k8s-m1 ~]# kubectl -n ingress-nginx get pods
NAME                                        READY     STATUS    RESTARTS   AGE
default-http-backend-5c6d95c48-62cx6        1/1       Running   0          15m
nginx-ingress-controller-5868fdbf7c-6lnlt   1/1       Running   0          42s
```

测试 Ingress 功能
这边先建立一个 Nginx HTTP server Deployment 与 Service：
```bash
kubectl run nginx-dp --image nginx --port 80
kubectl expose deploy nginx-dp --port 80
kubectl get po,svc
cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-nginx-ingress
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: test.nginx.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-dp
          servicePort: 80
EOF
```

测试
```bash
curl 192.168.77.140 -H 'Host: test.nginx.com'
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

curl 192.168.77.140 -H 'Host: test.nginx.com1'
default backend - 404
```
使用traefik作为Ingress  Controller
```bash
kubectl create ns ingress-traefik
kubectl apply -f addons/ingress-traefik.yml
kubectl -n ingress-traefik get all

NAME                        READY     STATUS    RESTARTS   AGE
pod/traefik-ingress-69mbq   1/1       Running   0          1h
pod/traefik-ingress-mqhv4   1/1       Running   0          1h
pod/traefik-ingress-nwmgm   1/1       Running   0          1h
pod/traefik-ingress-snblq   1/1       Running   0          1h
pod/traefik-ingress-wbx6h   1/1       Running   0          1h
pod/traefik-ingress-xpq4k   1/1       Running   0          1h

NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                       AGE
service/traefik-ingress-service   LoadBalancer   10.110.41.24   192.168.77.140   80:30582/TCP,8580:31593/TCP   1h

NAME                             DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/traefik-ingress   6         6         6         6            6           <none>          1h

kubectl run nginx-dp --image nginx --port 80 -n ingress-traefik
kubectl expose deploy nginx-dp --port 80 -n ingress-traefik

cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress-traefik
  namespace: ingress-traefik
  annotations:
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: test.traefik.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-dp
          servicePort: 80
EOF
kubectl -n ingress-traefik get all

curl 192.168.77.140 -H 'Host: test.traefik.com'
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

访问网址 `http://192.168.77.140:31593/dashboard/`

![image.png]/assets/images/kubernetes/3629406-556512e8de122e67.png


### Helm Tiller Server
Helm 是 Kubernetes Chart 的管理工具，Kubernetes Chart 是一套预先组态的 Kubernetes 资源套件。其中Tiller Server主要负责接收来至 Client 的指令，并透过 kube-apiserver 与 Kubernetes 丛集做沟通，根据 Chart 定义的内容，来产生与管理各​​种对应 API 物件的 Kubernetes 部署档案(又称为 Release)。

首先在k8s-m1安裝 Helm tool：
```bash
mkdir -p /usr/local/helm/bin/
cp bin/helm/helm /usr/local/helm/bin/
echo 'PATH=/usr/local/helm/bin:$PATH' >> /etc/profile
source /etc/profile
```
另外在所有node节点安装socat
```bash
for NODE in k8s-m1 k8s-m2 k8s-m3 k8s-n1 k8s-n2 k8s-n3; do
    echo "--- $NODE ---"
    ssh ${NODE} "yum -y install socat"
done
```

接着初始化 Helm(这边会安装 Tiller Server)：
```bash
kubectl -n kube-system create sa tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller

Creating /root/.helm 
Creating /root/.helm/repository 
Creating /root/.helm/repository/cache 
Creating /root/.helm/repository/local 
Creating /root/.helm/plugins 
Creating /root/.helm/starters 
Creating /root/.helm/cache/archive 
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!

kubectl -n kube-system get po -l app=helm
NAME                             READY     STATUS    RESTARTS   AGE
tiller-deploy-5c688d5f9b-5ltmk   1/1       Running   0          1m

helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

测试helm功能
```bash
测试helm功能
helm install --name demo --set Persistence.Enabled=false stable/jenkins
kubectl get po,svc  -l app=demo-jenkins

NAME                                READY     STATUS    RESTARTS   AGE
pod/demo-jenkins-5954bfc7b5-nx9vr   1/1       Running   0          8m

NAME                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/demo-jenkins         LoadBalancer   10.98.50.125    <pending>     8080:30659/TCP   2h
service/demo-jenkins-agent   ClusterIP      10.98.193.230   <none>        50000/TCP        2h
```

获取admin帐号
```bash
printf $(kubectl get secret --namespace default demo-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
TTSJLhPWkM
```
访问Jenkins `http://192.168.77.140:30659/`
![image.png]/assets/images/kubernetes/3629406-39938d65d8bb63e9.png

# 日志
```bash
kubectl apply -f addons/elasticsearch.yml
kubectl apply -f addons/fluentd.yml
kubectl apply -f addons/kibana.yml

kubectl get all -n kube-system | grep -E 'elastic|fluentd|kibana'
pod/elasticsearch-logging-0                    1/1       Running   0          15h
pod/fluentd-es-m5lt7                           1/1       Running   0          15h
pod/fluentd-es-qvl9w                           1/1       Running   0          15h
pod/fluentd-es-s9rvq                           1/1       Running   0          15h
pod/fluentd-es-t7k2j                           1/1       Running   0          15h
pod/fluentd-es-tt5cc                           1/1       Running   0          15h
pod/fluentd-es-zmxkx                           1/1       Running   0          15h
pod/kibana-logging-86c85d6956-4vx24            1/1       Running   0          15h


service/elasticsearch-logging   ClusterIP   10.99.209.193    <none>        9200/TCP            15h
service/kibana-logging          ClusterIP   10.105.151.197   <none>        5601/TCP            15h
daemonset.apps/fluentd-es    6         6         6         6            6           <none>          15h

deployment.apps/kibana-logging            1         1         1            1           15h

replicaset.apps/kibana-logging-86c85d6956            1         1         1         15h

statefulset.apps/elasticsearch-logging   1         1         15h
```
访问elasticsearch `https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy`
![image.png]/assets/images/kubernetes/3629406-a51590f39b3c68e4.png
访问kibana `https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy`
![image.png]/assets/images/kubernetes/3629406-9bdbce7f37e9ebda.png
![image.png]/assets/images/kubernetes/3629406-51784a1683db675a.png

# 集群信息
```bash
kubectl cluster-info
Kubernetes master is running at https://192.168.77.140:6443
Elasticsearch is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy
heapster is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/heapster/proxy
Kibana is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
kube-dns is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
monitoring-grafana is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
monitoring-influxdb is running at https://192.168.77.140:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb:http/proxy
```
# 测试集群
关闭k8s-m2，即vip所在的节点。
```bash
poweroff
```
进入k8s-m1查看集群状态
```bash
kubectl get cs
NAME                 STATUS      MESSAGE                                                                                              ERROR
controller-manager   Healthy     ok                                                                                                   
scheduler            Healthy     ok                                                                                                   
etcd-1               Healthy     {"health": "true"}                                                                                   
etcd-2               Healthy     {"health": "true"}                                                                                   
etcd-0               Unhealthy   Get https://192.168.77.134:2379/health: dial tcp 192.168.77.134:2379: getsockopt: no route to host 
```
测试是否可以建立pod
```bash
kubectl run nginx --image nginx --restart=Never --port 80
kubectl get po
NAME             READY     STATUS              RESTARTS   AGE
nginx  1/1       Running             0                1m
```


> 本次实验还需改进，缺少集群的存储方案，如es，influxdb，如果pod分配到其他节点，数据就没了。后续会有ceph和GlusterFS存储方案

> 累了吧，试试一键部署 [Ansible 应用 之 【利用ansible来做kubernetes 1.10.3集群高可用的一键部署】](https://www.jianshu.com/p/265cfb0811b2) 。
