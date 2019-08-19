---
layout: post
title: "使用kubectl插件检查deployment的所有pod是否就绪"
date: "2019-08-19 16:00:00"
category: kubernetes
tags: kubernetes
author: lework
---
* content
{:toc}

描述

我们在进行`ci/cd`流程时，应用了`deployment`资源时，无法知道修改的`pod`是否处在就绪状态，并正常工作。但是我们的应用操作时成功执行了，如果说这时`pod`没有正常就绪工作，也就是说我们要改变的`pod`没有改变，目标集群运行的还是之前的`pod`，如果说我们有对`deployment`的不可用pod进行监控，那我们得到的通知时间也是很滞后了，因此，我们需要一个在`ci/cd`流程中检查`deployment`的所有`pod`是否`就绪并正常工作`。`kubectl-check`就是为了解决这个场景而诞生的。




## 插件使用

**安装插件**

```bash
wget -O /usr/local/bin/kubectl-check https://raw.githubusercontent.com/lework/kubectl-check/master/kubectl-check
chmod +x /usr/local/bin/kubectl-check
```

**查看帮助**

```bash
kubectl check -h
Check Kubernetes deployment status.

Usage: /usr/local/bin/kubectl-check [-d deploy] [-v verbose] [ -i interval] [-t total] | [-h]
  -n,--namespace     Specify namespace, default is default
  -d,--deploy        Depoyment name
  -i,--interval      Check the deployment status interval
  -t,--total         Total number of inspections
  -v,--verbose       Verbose info
  -h,--help          View help
```

**检查deployment状态**

> 呈轮询式检查deployment的状态，如果检查到deployment的所有pod`启动成功`并`就绪后`后检查脚本则退出,状态码返回0; 如果检查超过默认次数(60)后还未成功则超时退出，返回状态码1.

```bash
kubectl check -d deploy-name # 指定deploy名称
kubectl check -n default -d deploy-name -v # 指定命名空间
kubectl check -d deploy-name -i 5 -t 10 # 指定检查等待时间时间(单位秒)和检查次数
kubectl check -d deploy-name -v # 打印详细信息
```

检查deployment的变更成功

```bash
# kubectl apply -f ci-k8s.yaml; kubectl check -d test-ci
deployment.extensions/test-ci configured
service/test-ci unchanged
[Number of inspections] 1
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 2
[Pods]: test-ci-7d6c97fdbb-rb9mc
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Pods]: test-ci-7d6c97fdbb-vdlpr
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 2
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 2
[Pods]: test-ci-7d6c97fdbb-rb9mc
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Pods]: test-ci-7d6c97fdbb-vdlpr
  phase: Running
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

......

[Number of inspections] 6
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 1
[Pods]: test-ci-597f5488fd-qfb6c
  phase: Running
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 7
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: null
[Deploy] pod success.
[Reuslt] This success.
```

检查deployment的变更失败

> 默认检查60次，间隔2秒钟一次检查,超过60次，则超时失败

```bash
# kubectl apply -f ci-k8s.yaml; kubectl check -d test-ci
deployment.extensions/test-ci configured
service/test-ci unchanged
[Number of inspections] 1
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 2
[Pods]: test-ci-8694cbd94-h9wgc
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Pods]: test-ci-8694cbd94-z8bwp
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 2
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 2
[Pods]: test-ci-8694cbd94-h9wgc
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Pods]: test-ci-8694cbd94-z8bwp
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

......

[Number of inspections] 59
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 2
[Pods]: test-ci-8694cbd94-h9wgc
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Pods]: test-ci-8694cbd94-z8bwp
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 60
[Deployment] test-ci
  status: True
  message: Deployment has minimum availability.
  unavailableReplicas: 2
[Pods]: test-ci-8694cbd94-h9wgc
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Pods]: test-ci-8694cbd94-z8bwp
  phase: Pending
  conditions:
    status=False, type=Ready, message=containers with unready status: [test-ci]
    status=False, type=ContainersReady, message=containers with unready status: [test-ci]
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Reuslt] Check timeout, this time failed.
```

## Docker使用

**使用kubeconfig**

```bash
KUBERNETES_KUBECONFIG=$(base64 -w 0 ~/.kube/config)
docker run --rm -e KUBERNETES_KUBECONFIG=$KUBERNETES_KUBECONFIG lework/kubectl-check:latest kubectl check -d deploy-name
```

**使用kube token**

```bash
kubectl create serviceaccount def-ns-admin -n default
kubectl create rolebinding def-ns-admin --clusterrole=admin --serviceaccount=default:def-ns-admin

KUBERNETES_SERVER="https://192.168.77.130:6443"
KUBERNETES_TOKEN=$(kubectl get secret $(kubectl get sa def-ns-admin -o jsonpath={.secrets[].name}) -o jsonpath={.data.token})
KUBERNETES_CERT=$(kubectl get secret $(kubectl get sa def-ns-admin -o jsonpath={.secrets[].name}) -o "jsonpath={.data.ca\.crt}")

docker run --rm -e KUBERNETES_SERVER=$KUBERNETES_SERVER -e KUBERNETES_TOKEN=$KUBERNETES_TOKEN -e KUBERNETES_CERT=$KUBERNETES_CERT lework/kubectl-check:latest kubectl check -d deploy-name
```
