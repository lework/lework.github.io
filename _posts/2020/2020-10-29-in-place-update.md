---
layout: post
title: "使用 OpenKruise 原地升级 POD 资源"
date: "2020-10-29 18:36"
category: kubernetes
tags: kubernetes deploy
author: lework
---
* content
{:toc}

## OpenKruise

[OpenKruise](https://github.com/openkruise/kruise) 是阿里云开源的大规模应用自动化管理引擎，在功能上对标了 Kubernetes 原生的 Deployment / StatefulSet 等控制器，但 OpenKruise 提供了更多的增强功能如：优雅原地升级、发布优先级/打散策略、多可用区workload抽象管理、统一 sidecar 容器注入管理等，都是经历了阿里巴巴超大规模应用场景打磨出的核心能力。这些 feature 帮助我们应对更加多样化的部署环境和需求、为集群维护者和应用开发者带来更加灵活的部署发布组合策略。

目前，Kruise 提供了以下 workload 控制器：
- `CloneSet`: 提供了更加高效、确定可控的应用管理和部署能力，支持优雅原地升级、指定删除、发布顺序可配置、并行/灰度发布等丰富的策略，可以满足更多样化的应用场景。
- `Advanced StatefulSet`: 基于原生 StatefulSet 之上的增强版本，默认行为与原生完全一致，在此之外提供了原地升级、并行发布（最大不可用）、发布暂停等功能。
- `SidecarSet`: 对 sidecar 容器做统一管理，在满足 selector 条件的 Pod 中注入指定的 sidecar 容器。
- `UnitedDeployment`: 通过多个 subset workload 将应用部署到多个可用区。
- `BroadcastJob`: 配置一个 job，在集群中所有满足条件的 Node 上都跑一个 Pod 任务。
- `Advanced DaemonSet`: 基于原生 DaemonSet 之上的增强版本，默认行为与原生一致，在此之外提供了灰度分批、按 Node label 选择、暂停、热升级等发布策略。

> 更多介绍见： https://openkruise.io

本文将使用 OpenKruise 的 `CloneSet`  **原地升级** 功能来更新 pod 资源。




## 升级方式

### Deployment
使用 Deployment 部署时，那么升级过程中 Deployment 会触发新版本 ReplicaSet 创建 Pod，并删除旧版本 Pod。
在本次升级过程中，原 Pod 对象被删除，一个新 Pod 对象被创建。新 Pod 被调度到另一个 Node 上，分配到一个新的 IP，并把 foo、bar 两个容器在这个 Node 上重新拉取镜像、启动容器。

### StatefulSet
使用 StatefulSet 部署，那么升级过程中 StatefulSet 会先删除旧 Pod 对象，等删除完成后用同样的名字在创建一个新的 Pod 对象。
值得注意的是，尽管新旧两个 Pod 名字都叫 pod-0，但其实是两个完全不同的 Pod 对象（uid也变了）。StatefulSet 等到原先的 pod-0 对象完全从 Kubernetes 集群中被删除后，才会提交创建一个新的 pod-0 对象。而这个新的 Pod 也会被重新调度、分配IP、拉镜像、启动容器。

### 原地升级

所谓原地升级模式，就是在应用升级过程中避免将整个 Pod 对象删除、新建，而是基于原有的 Pod 对象升级其中某一个或多个容器的镜像版本。
在原地升级的过程中，我们仅仅更新了原 Pod 对象中 foo 容器的 image 字段来触发 foo 容器升级到新版本。而不管是 Pod 对象，还是 Node、IP 都没有发生变化，甚至 foo 容器升级的过程中 bar 容器还一直处于运行状态。
这种只更新 Pod 中某一个或多个容器版本、而不影响整个 Pod 对象、其余容器的升级方式，被我们称为 Kubernetes 中的原地升级。


## 原地升级的收益

首先，这种原地升级的模式极大地提升了应用发布的效率，根据非完全统计数据，在阿里环境下原地升级至少比完全重建升级提升了 80% 以上的发布速度。这其实很容易理解，原地升级为发布效率带来了以下优化点：

- 节省了调度的耗时，Pod 的位置、资源都不发生变化；
- 节省了分配网络的耗时，Pod 还使用原有的 IP；
- 节省了分配、挂载远程盘的耗时，Pod 还使用原有的 PV（且都是已经在 Node 上挂载好的）；
- 节省了大部分拉取镜像的耗时，因为 Node 上已经存在了应用的旧镜像，当拉取新版本镜像时只需要下载很少的几层 layer。

其次，当我们升级 Pod 中一些 sidecar 容器（如采集日志、监控等）时，其实并不希望干扰到业务容器的运行。但面对这种场景，Deployment 或 StatefulSet 的升级都会将整个 Pod 重建，势必会对业务造成一定的影响。而容器级别的原地升级变动的范围非常可控，只会将需要升级的容器做重建，其余容器包括网络、挂载盘都不会受到影响。

最后，原地升级也为我们带来了集群的稳定性和确定性。当一个 Kubernetes 集群中大量应用触发重建 Pod 升级时，可能造成大规模的 Pod 飘移，以及对 Node 上一些低优先级的任务 Pod 造成反复的抢占迁移。这些大规模的 Pod 重建，本身会对 apiserver、scheduler、网络/磁盘分配等中心组件造成较大的压力，而这些组件的延迟也会给 Pod 重建带来恶性循环。而采用原地升级后，整个升级过程只会涉及到 controller 对 Pod 对象的更新操作和 kubelet 重建对应的容器。


## 使用 OpenKruise
### 安装 OpenKruise

```bash
# kubectl  get node
NAME               STATUS   ROLES    AGE     VERSION
k8s-master-node1   Ready    master   4h24m   v1.19.3
k8s-master-node2   Ready    master   4h23m   v1.19.3
k8s-master-node3   Ready    master   4h22m   v1.19.3
k8s-worker-node1   Ready    worker   4h22m   v1.19.3
k8s-worker-node2   Ready    worker   4h22m   v1.19.3

# curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
# helm install kruise https://github.com/openkruise/kruise/releases/download/v0.6.1/kruise-chart.tgz
```

OpenKruise 安装的资源

```bash
# kubectl get crds | grep kruise
broadcastjobs.apps.kruise.io       2020-10-29T07:58:23Z
clonesets.apps.kruise.io           2020-10-29T07:58:23Z
daemonsets.apps.kruise.io          2020-10-29T07:58:23Z
sidecarsets.apps.kruise.io         2020-10-29T07:58:23Z
statefulsets.apps.kruise.io        2020-10-29T07:58:23Z
uniteddeployments.apps.kruise.io   2020-10-29T07:58:23Z
```

```bash
# kubectl get all -n kruise-system     
NAME                                            READY   STATUS    RESTARTS   AGE
pod/kruise-controller-manager-86d459f65-qb7j4   1/1     Running   1          6m19s
pod/kruise-controller-manager-86d459f65-xcdvs   1/1     Running   0          6m19s

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kruise-webhook-service   ClusterIP   10.96.229.106   <none>        443/TCP   6m21s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kruise-controller-manager   2/2     2            2           6m21s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/kruise-controller-manager-86d459f65   2         2         2       6m20s
```

### 使用 CloneSet

这里我们部署一个 nginx 资源

```bash
cat <<EOF | kubectl apply -f -
---
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  labels:
    app: sample
  name: sample
spec:
  replicas: 2
  updateStrategy:
    type: InPlaceIfPossible # 优先尝试原地升级
    inPlaceUpdateStrategy:
      gracePeriodSeconds: 5 # not ready后等待一段时间
    maxUnavailable: 20% # 默认 20%
    maxSurge: 0  # 默认 0
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      labels:
        app: sample
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: sample
  labels:
    app: sample
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: sample
EOF
```

创建的 cloneset 资源信息

```bash
# kubectl get clonesets sample
NAME     DESIRED   UPDATED   UPDATED_READY   READY   TOTAL   AGE
sample   2         2         2               2       2       15s

# kubectl describe clonesets sample   
Name:         sample
Namespace:    default
Labels:       app=sample
Annotations:  <none>
API Version:  apps.kruise.io/v1alpha1
Kind:         CloneSet
Metadata:
  Creation Timestamp:  2020-10-29T09:50:56Z
  Generation:          1
  Managed Fields:
    API Version:  apps.kruise.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
        f:labels:
          .:
          f:app:
      f:spec:
        .:
        f:replicas:
        f:selector:
          .:
          f:matchLabels:
            .:
            f:app:
        f:template:
          .:
          f:metadata:
            .:
            f:labels:
              .:
              f:app:
          f:spec:
            .:
            f:containers:
        f:updateStrategy:
          .:
          f:inPlaceUpdateStrategy:
            .:
            f:gracePeriodSeconds:
          f:maxSurge:
          f:maxUnavailable:
          f:type:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2020-10-29T09:50:56Z
    API Version:  apps.kruise.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:availableReplicas:
        f:collisionCount:
        f:labelSelector:
        f:observedGeneration:
        f:readyReplicas:
        f:replicas:
        f:updateRevision:
        f:updatedReadyReplicas:
        f:updatedReplicas:
    Manager:         manager
    Operation:       Update
    Time:            2020-10-29T09:51:00Z
  Resource Version:  59660
  Self Link:         /apis/apps.kruise.io/v1alpha1/namespaces/default/clonesets/sample
  UID:               b2307fae-6bbd-4e2a-92f1-fa753300c2c8
Spec:
  Replicas:                2
  Revision History Limit:  10
  Scale Strategy:
  Selector:
    Match Labels:
      App:  sample
  Template:
    Metadata:
      Creation Timestamp:  <nil>
      Labels:
        App:  sample
    Spec:
      Containers:
        Image:              nginx:alpine
        Image Pull Policy:  Always
        Name:               nginx
        Resources:
        Termination Message Path:    /dev/termination-log
        Termination Message Policy:  File
      Dns Policy:                    ClusterFirst
      Restart Policy:                Always
      Scheduler Name:                default-scheduler
      Security Context:
      Termination Grace Period Seconds:  30
  Update Strategy:
    In Place Update Strategy:
      Grace Period Seconds:  5
    Max Surge:               0
    Max Unavailable:         20%
    Partition:               0
    Type:                    InPlaceIfPossible
Status:
  Available Replicas:      2
  Collision Count:         0
  Label Selector:          app=sample
  Observed Generation:     1
  Ready Replicas:          2
  Replicas:                2
  Update Revision:         sample-d4d4fb5bd
  Updated Ready Replicas:  2
  Updated Replicas:        2
Events:
  Type    Reason            Age   From                 Message
  ----    ------            ----  ----                 -------
  Normal  SuccessfulCreate  21s   cloneset-controller  succeed to create pod sample-j5ht6
  Normal  SuccessfulCreate  21s   cloneset-controller  succeed to create pod sample-m2dvb
```

CloneSet status 中的字段说明：
- `status.replicas`: Pod 总数
- `status.readyReplicas`: ready Pod 数量
- `status.availableReplicas`: ready and available Pod 数量 (满足 minReadySeconds)
- `status.updateRevision`: 最新版本的 revision hash 值
- `status.updatedReplicas`: 最新版本的 Pod 数量
- `status.updatedReadyReplicas`: 最新版本的 ready Pod 数量


查看其中一个pod的资源信息

```bash
# kubectl describe pods sample-j5ht6
Name:         sample-j5ht6
Namespace:    default
Priority:     0
Node:         k8s-worker-node1/192.168.77.133
Start Time:   Thu, 29 Oct 2020 17:50:56 +0800
Labels:       app=sample
              apps.kruise.io/cloneset-instance-id=j5ht6
              controller-revision-hash=sample-d4d4fb5bd
              lifecycle.apps.kruise.io/state=Normal
Annotations:  lifecycle.apps.kruise.io/timestamp: 2020-10-29T09:50:56Z
Status:       Running
IP:           10.244.3.9
IPs:
  IP:           10.244.3.9
Controlled By:  CloneSet/sample
Containers:
  nginx:
    Container ID:   docker://e306b3dec3dd73c09086272428bbf9d9129188cfe749b06fd621025a71ee0076
    Image:          nginx:alpine
    Image ID:       docker-pullable://nginx@sha256:5aa44b407756b274a600c7399418bdfb1d02c33317ae27fd5e8a333afb115db1
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 29 Oct 2020 17:51:00 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-tcprk (ro)
Readiness Gates:
  Type                 Status
  InPlaceUpdateReady   True 
Conditions:
  Type                 Status
  InPlaceUpdateReady   True 
  Initialized          True 
  Ready                True 
  ContainersReady      True 
  PodScheduled         True 
Volumes:
  default-token-tcprk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-tcprk
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m46s  default-scheduler  Successfully assigned default/sample-j5ht6 to k8s-worker-node1
  Normal  Pulling    3m46s  kubelet            Pulling image "nginx:alpine"
  Normal  Pulled     3m43s  kubelet            Successfully pulled image "nginx:alpine" in 2.599105357s
  Normal  Created    3m43s  kubelet            Created container nginx
  Normal  Started    3m43s  kubelet            Started container nginx
```
`Readiness Gates` 有 kruise 定义的 `InPlaceUpdateReady` 状态标识，从而控制 pod 的 ready 状态。

**访问 pod** 

```bash
# kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
ingress-demo-app   ClusterIP   10.96.181.11    <none>        80/TCP    3h57m
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP   3h59m
sample             ClusterIP   10.96.112.166   <none>        80/TCP    6m25s

# curl 10.96.112.166
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
访问正常，没什么问题的。

查看pod的一些信息: 名称, pod主机ip, pod ip, image name, 资源id
```bash
# printf "$(kubectl get pods -l app=sample -o json  -o jsonpath='{range.items[*]}{.metadata.name} {.status.hostIP} {.status.podIP } {.spec.containers[*].image} {.metadata.resourceVersion}\n{end}')"
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:alpine 59657
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 59649
```

接下来，我们进行更新容器镜像。

```bash
kubectl patch CloneSet sample --type=merge -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.19.1"}]}}}}'
```
> 注意, 此合并操作是覆盖containers里的信息, 多个容器镜像时不可用这个方式更新。

观察pod的资源变化
```bash
for i in `seq 20`; do printf "$(kubectl get pods -l app=sample -o json  -o jsonpath='{range.items[*]}{.metadata.name} {.status.hostIP} {.status.podIP } {.spec.containers[*].image} {.metadata.resourceVersion}\n{end}')"; sleep 3;done 
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:alpine 59657
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 59649
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62259
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 59649
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62259
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 59649
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62259
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 59649
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62338
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 62348
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62362
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:alpine 62348
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62362
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:1.19.1 62375
sample-j5ht6 192.168.77.133 10.244.3.9 nginx:1.19.1 62362
sample-m2dvb 192.168.77.134 10.244.4.10 nginx:1.19.1 62375
``

可以看到 pod 只改变了容器镜像和资源id, pod名称, pod主机, pod ip 都没有改变。

再去 pod 节点上查看容器信息
{% raw %}
​```bash
docker ps --filter label=io.kubernetes.pod.name=sample-j5ht6 --format "table {{.Image}}\t{{.CreatedAt}}\t{{.RunningFor}}\t{{.Names}}"      
IMAGE                                    CREATED AT                      CREATED             NAMES
nginx                                    2020-10-29 18:00:57 +0800 CST   4 minutes ago       k8s_nginx_sample-j5ht6_default_0c0ad06e-792f-487c-b218-321fb33b1695_1
registry.aliyuncs.com/k8sxio/pause:3.2   2020-10-29 17:50:56 +0800 CST   14 minutes ago      k8s_POD_sample-j5ht6_default_0c0ad06e-792f-487c-b218-321fb33b1695_0
```
{% endraw %}
可以看到 pod 里 只有 nginx 这个容器重新创建了, pause 容器还是之前的。

cloneset 的执行事件

```bash
kubectl get event | grep cloneset/sample
16m         Normal   SuccessfulCreate             cloneset/sample                          succeed to create pod sample-j5ht6
16m         Normal   SuccessfulCreate             cloneset/sample                          succeed to create pod sample-m2dvb
6m49s       Normal   SuccessfulUpdatePodInPlace   cloneset/sample                          successfully update pod sample-j5ht6 in-place(revision sample-7b66b966b7)
6m28s       Normal   SuccessfulUpdatePodInPlace   cloneset/sample                          successfully update pod sample-m2dvb in-place(revision sample-7b66b966b7)
```

最后, 我们通过使用 `cloneset` 的方式实现了 pod 原地升级的功能, 减少了pod升级时间。

## 其他示例

> https://github.com/openkruise/kruise/tree/master/docs/tutorial

### DaemonSet
```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: DaemonSet
metadata:
  name: guestbook-ds
spec:
  selector:
    matchLabels:
      app: guestbook-daemon
  template:
    metadata:
      labels:
        app: guestbook-daemon
    spec:
      containers:
        - name: daemon
          image: openkruise/guestbook:v1
          resources:
            limits:
              cpu: "100m"
              memory: "200Mi"
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
      partition: 0
      rollingUpdateType: Standard
    type: RollingUpdate
```
### SidecarSet

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
metadata:
  name: guestbook-sidecar
spec:
  selector: # select the pods to be injected with sidecar containers
    matchLabels:
      app.kubernetes.io/name: guestbook-with-sidecar
  containers:
    - name: guestbook-sidecar
      image: openkruise/guestbook:sidecar
      imagePullPolicy: Always
      ports:
        - name: sidecar-server
          containerPort: 4000 # different from main guestbook containerPort which is 3000
      volumeMounts:
        - name: log-volume
          mountPath: /var/log
  volumes:
    - name: log-volume
      emptyDir: {}
```

### StatefulSet
```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: StatefulSet
metadata:
  name: demo-v1-guestbook-kruise
  labels:
    app.kubernetes.io/name: guestbook-kruise
    app.kubernetes.io/instance: demo-v1
spec:
  replicas: 20
  serviceName: demo-v1-guestbook-kruise
  selector:
    matchLabels:
      app.kubernetes.io/name: guestbook-kruise
      app.kubernetes.io/instance: demo-v1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: guestbook-kruise
        app.kubernetes.io/instance: demo-v1
    spec:
      readinessGates:
        # A new condition that ensures the pod remains at NotReady state while the in-place update is happening
      - conditionType: InPlaceUpdateReady
      containers:
      - name: guestbook-kruise
        image: openkruise/guestbook:v1
        imagePullPolicy: Always
        ports:
        - name: http-server
          containerPort: 3000
  podManagementPolicy: Parallel  # allow parallel updates, works together with maxUnavailable
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # Do in-place update if possible, currently only image update is supported for in-place update
      podUpdatePolicy: InPlaceIfPossible
      # Allow parallel updates with max number of unavailable instances equals to 2
      maxUnavailable: 3
```

### UnitedDeployment
```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: UnitedDeployment
metadata:
  labels:
    app.kubernetes.io/name: guestbook-kruise
    app.kubernetes.io/instance: demo
  name: demo-guestbook-kruise
spec:
  replicas: 10
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/name: guestbook-kruise
      app.kubernetes.io/instance: demo
  template:
    statefulSetTemplate:
      metadata:
        labels:
          app.kubernetes.io/name: guestbook-kruise
          app.kubernetes.io/instance: demo
      spec:
        template:
          metadata:
            labels:
              app.kubernetes.io/name: guestbook-kruise
              app.kubernetes.io/instance: demo
          spec:
            containers:
              - name: guestbook-kruise
                image: openkruise/guestbook:v1
                imagePullPolicy: Always
                ports:
                  - name: http-server
                    containerPort: 3000
  topology:
    subsets:
      - name: subset-a
        replicas: 2
        nodeSelectorTerm:
          matchExpressions:
            - key: az
              operator: In
              values:
                - zone-a
      - name: subset-b
        replicas: 50%
        nodeSelectorTerm:
          matchExpressions:
            - key: az
              operator: In
              values:
                - zone-b
      - name: subset-c
        nodeSelectorTerm:
          matchExpressions:
            - key: az
              operator: In
              values:
                - zone-c
```

### BroadcastJob
```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: BroadcastJob
metadata:
  name: download-image
spec:
  template:
    spec:
      containers:
        - name: guestbook
          image: openkruise/guestbook:v3
          command: ["echo",  "started"] # a dummy command to do nothing
      restartPolicy: Never
  completionPolicy:
    type: Always
    ttlSecondsAfterFinished: 60 # the job will be deleted after 60 seconds
```

## 参考

- https://openkruise.io/zh-cn/docs/
- https://developer.aliyun.com/article/765421
- https://yqh.aliyun.com/detail/14018
- http://qiankunli.github.io/2020/08/12/openkruise_note.html