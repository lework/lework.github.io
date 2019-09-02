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

我们在进行`ci/cd`流程时，应用了`deployment`资源时，无法知道修改的`pod`是否处在就绪状态，并正常工作。但是我们的应用操作时成功执行了，如果说这时`pod`没有正常就绪工作，也就是说我们要改变的`pod`没有改变，目标集群运行的还是之前的`pod`，如果说我们有对`deployment`的不可用pod进行监控，那我们得到的通知时间也是很滞后了，因此，我们需要一个在`ci/cd`流程中检查`deployment`的所有`pod`是否`就绪并正常工作`。[kubectl-check](https://github.com/lework/kubectl-check)就是为了解决这个场景而诞生的。




## 插件使用

**所需权限**

```bash
# 创建rabc权限
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deploy
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: ci-deploy
  namespace: default
rules:
  - apiGroups: ["apps", "extensions", ""]
    resources: ["pods", "deployments", "deployments/scale", "services", "replicasets"]
    verbs: ["create","get","list","patch","update"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: ci-deploy
  namespace: default
subjects:
  - kind: ServiceAccount
    name: ci-deploy
roleRef:
  kind: Role
  name: ci-deploy
  apiGroup: rbac.authorization.k8s.io
EOF
```

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
deployment.apps/ci-test configured
service/ci-test unchanged
[Number of inspections] 1
[Time] 2019-08-19 13:16:58
[Deployment] ci-test
  conditions:
    status=True, type=Available, reason=MinimumReplicasAvailable, message=Deployment has minimum availability.
  unavailableReplicas: 2
[Pods] ci-test-6686546958-cw49b
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Pods] ci-test-6686546958-mr97p
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 2
[Time] 2019-08-19 13:17:03
[Deployment] ci-test
  conditions:
    status=True, type=Available, reason=MinimumReplicasAvailable, message=Deployment has minimum availability.
  unavailableReplicas: 2
[Pods] ci-test-6686546958-cw49b
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Pods] ci-test-6686546958-mr97p
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Pods] ci-test-6b98ffbbc4-2rj6j
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Reuslt] Deployment is not ready.
[Sleep] 2s

......

[Number of inspections] 6
[Time] 2019-08-19 13:17:19
[Deployment] ci-test
  conditions:
    status=True, type=Available, reason=MinimumReplicasAvailable, message=Deployment has minimum availability.
  unavailableReplicas: 1
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 7
[Time] 2019-08-19 13:17:22
[Deployment] ci-test
  conditions:
    status=True, type=Available, reason=MinimumReplicasAvailable, message=Deployment has minimum availability.
  unavailableReplicas: null
[Deploy] Pod success.
[Reuslt] Deploy success.
```

检查deployment的变更失败

> 默认检查60次，间隔2秒钟一次检查,超过60次，则超时失败.最后一次则输出详细信息，用作查找问题.

```bash
# kubectl apply -f ci-k8s.yaml; kubectl check -d test-ci
deployment.extensions/test-ci configured
service/test-ci unchanged

.....

[Number of inspections] 59
[Time] 2019-08-19 13:22:06
[Deployment] ci-test
  conditions:
    status=True, type=Available, reason=MinimumReplicasAvailable, message=Deployment has minimum availability.
  unavailableReplicas: 2
[Pods] ci-test-66db988df4-fnkq9
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Pods] ci-test-66db988df4-s8s6g
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
[Reuslt] Deployment is not ready.
[Sleep] 2s

[Number of inspections] 60
[Time] 2019-08-19 13:22:09
[Deployment] ci-test
  namespace: default
  selector: app=ci-test,environment=test
  conditions:
    status=True, type=Available, reason=MinimumReplicasAvailable, message=Deployment has minimum availability.
  replicas: 3
  availableReplicas: 1
  unavailableReplicas: 2
  updatedReplicas: 2
  observedGeneration: 6
[Pods] ci-test-66db988df4-fnkq9
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
  podIP: 10.244.0.65
  qosClass: Burstable
  startTime: 2019-08-19T05:19:00Z
  containerStatuses:
    [
      {
        "image": "192.168.77.133:5000/root/ci-test:dev2",
        "imageID": "",
        "lastState": {},
        "name": "ci-test",
        "ready": false,
        "restartCount": 0,
        "state": {
          "waiting": {
            "message": "Back-off pulling image \"192.168.77.133:5000/root/ci-test:dev2\"",
            "reason": "ImagePullBackOff"
          }
        }
      }
    ]
[Pods] ci-test-66db988df4-s8s6g
  phase: Pending
  conditions:
    status=False, type=Ready, reason=ContainersNotReady, message=containers with unready status: [ci-test]
    status=False, type=ContainersReady, reason=ContainersNotReady, message=containers with unready status: [ci-test]
  hostIP: 172.17.0.2
  podIP: 10.244.0.64
  qosClass: Burstable
  startTime: 2019-08-19T05:19:00Z
  containerStatuses:
    [
      {
        "image": "192.168.77.133:5000/root/ci-test:dev2",
        "imageID": "",
        "lastState": {},
        "name": "ci-test",
        "ready": false,
        "restartCount": 0,
        "state": {
          "waiting": {
            "message": "rpc error: code = Unknown desc = failed to resolve image \"192.168.77.133:5000/root/ci-test:dev2\": no available registry endpoint: failed to do request: Head https://192.168.77.133:5000/v2/root/ci-test/manifests/dev2: http: server gave HTTP response to HTTPS client",
            "reason": "ErrImagePull"
          }
        }
      }
    ]
[Reuslt] Deployment is not ready.

[Version comparison]
--- /dev/fd/63
+++ /dev/fd/62
@@ -1,11 +1,11 @@
-deployment.extensions/ci-test with revision #5
+deployment.extensions/ci-test with revision #6
 Pod Template:
   Labels:      app=ci-test
        environment=test
-       pod-template-hash=6686546958
+       pod-template-hash=66db988df4
   Containers:
    ci-test:
-    Image:     192.168.77.133:5000/root/ci-test:latest
+    Image:     192.168.77.133:5000/root/ci-test:dev2
     Port:      80/TCP
     Host Port: 0/TCP
     Requests:
[Reuslt] Check timeout, this time failed.
```

## Docker使用

**使用kubeconfig**

```bash
# 设定kubeconfig
KUBERNETES_KUBECONFIG=$(base64 -w 0 ~/.kube/config)

# 执行cmd命令
docker run --rm -e KUBERNETES_KUBECONFIG=$KUBERNETES_KUBECONFIG lework/kubectl-check:latest kubectl check -d deploy-name

# 以变量的形式指定deploy name
docker run --rm -e KUBERNETES_KUBECONFIG=$KUBERNETES_KUBECONFIG -e KUBERNETES_DEPLOY=deploy-name lework/kubectl-check:latest
```

**使用kube token**

```bash
kubectl create serviceaccount def-ns-admin -n default
kubectl create rolebinding def-ns-admin --clusterrole=admin --serviceaccount=default:def-ns-admin

KUBERNETES_SERVER="https://192.168.77.130:6443"
KUBERNETES_TOKEN=$(kubectl get secret $(kubectl get sa def-ns-admin -o jsonpath={.secrets[].name}) -o jsonpath={.data.token})
KUBERNETES_CERT=$(kubectl get secret $(kubectl get sa def-ns-admin -o jsonpath={.secrets[].name}) -o "jsonpath={.data.ca\.crt}")

# 执行cmd命令
docker run --rm -e KUBERNETES_SERVER=$KUBERNETES_SERVER -e KUBERNETES_TOKEN=$KUBERNETES_TOKEN -e KUBERNETES_CERT=$KUBERNETES_CERT lework/kubectl-check:latest kubectl check -d deploy-name

# 以变量的形式指定deploy name
docker run --rm -e KUBERNETES_SERVER=$KUBERNETES_SERVER -e KUBERNETES_TOKEN=$KUBERNETES_TOKEN -e KUBERNETES_CERT=$KUBERNETES_CERT -e KUBERNETES_DEPLOY=deploy-name lework/kubectl-check:latest
```
