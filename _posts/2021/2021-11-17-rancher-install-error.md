---
layout: post
title: 'rancher 重置未果，集群节点反而全被删'
date: '2021-11-17 21:00'
category: rancher
tags: kubernetes rancher
author: lework 
---
* content
{:toc}

记录下安装 rancher 过程的一个悲剧

**痛心: rancher 重置未果，集群节点反而全被删！**




## 实现步骤


1. 安装 rancher

   > 版本： 2.6.2

   ```bash
   kubectl apply -f rancher.yaml
   ```

2. 清理 rancher

    > 为啥要清理，因为我想把https去掉，想重来。。。。

    ```bash
    # 删除rancher
    kubectl delete -f rancher.yaml
    # 删除ns
    kubectl delete ns p-qxl8j p-vdwl2 local fleet-default  fleet-local cattle-fleet-local-system cattle-fleet-system cattle-global-data  cattle-global-nt  cattle-impersonation-system cattle-system cattle-fleet-clusters-system  cluster-fleet-local-local-1a3d67d0a899
    
    # 这里要等很久，且删不掉，ns处于tem, 用下列命令强制删除
    for ns in p-qxl8j p-vdwl2 local fleet-default  fleet-local cattle-fleet-local-system cattle-fleet-system cattle-global-data  cattle-global-nt  cattle-impersonation-system cattle-system cattle-fleet-clusters-system  cluster-fleet-local-local-1a3d67d0a899
    do
    kubectl get namespace "${ns}" -o json \
                | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/" \
                | kubectl replace --raw /api/v1/namespaces/${ns}/finalize -f -
    done
    
    # 这里因为有webhook验证，所以需要删除。PS: 不知道是不是官方故意放置，防止后续的删除操作。
    kubectl delete mutatingwebhookconfigurations mutating-webhook-configuration rancher.cattle.io
    kubectl delete validatingwebhookconfigurations validating-webhook-configuration rancher.cattle.io
    ```

3. 重新安装 rancher

   > 罪恶的操作来了。

   ```bash
   kubectl apply -f rancher.yaml
   ```

   到这里，rancher 会输出下列日志，然后节点就被删除了。

   ```
   2021/11/17 14:11:07 [INFO] Waiting for initial data to be populated
   2021/11/17 14:11:09 [INFO] Waiting for initial data to be populated
   2021/11/17 14:11:11 [INFO] Waiting for initial data to be populated
   ```

   集群事件

   > 集群所有节点全被删除。

   ```
   0s          Normal   RemovingNode                node/k8s-worker-node2   Node k8s-worker-node2 event: Removing Node k8s-worker-node2 from Controller
   0s          Normal   RemovingNode                node/k8s-master-node1   Node k8s-master-node1 event: Removing Node k8s-master-node1 from Controller
   0s          Normal   RemovingNode                node/k8s-worker-node1   Node k8s-worker-node1 event: Removing Node k8s-worker-node1 from Controller
   ```

4. 恢复集群

   > 背锅吧，少年。

   重启所有节点服务器就行。

5. 生成的 `rancher.yaml`

    ```bash
    ./helm repo add rancher-stable https://releases.rancher.com/server-charts/stable  
    ./helm fetch --untar --untardir . 'rancher-stable/rancher'
    ./helm template --output-dir './yamls2' './rancher/'  --set tls=external --set postDelete.enabled=false
    ```

    ```yaml
    # Source: rancher/templates/serviceAccount.yaml
    kind: ServiceAccount
    apiVersion: v1
    metadata:
      name: rancher
      namespace: ops
      labels:
        app: rancher
        chart: rancher-2.6.2
        heritage: Helm
    ---
    # Source: rancher/templates/clusterRoleBinding.yaml
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: rancher
      namespace: ops
      labels:
        app: rancher
        chart: rancher-2.6.2
        heritage: Helm
    subjects:
    - kind: ServiceAccount
      name: rancher
      namespace: ops
    roleRef:
      kind: ClusterRole
      name: cluster-admin
      apiGroup: rbac.authorization.k8s.io
    ---
    # Source: rancher/templates/secret.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: "bootstrap-secret"
      namespace: ops
    type: Opaque
    data:
      bootstrapPassword: ""
    
    ---
    # Source: rancher/templates/deployment.yaml
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: rancher
      namespace: ops
      annotations:
      labels:
        app: rancher
        chart: rancher-2.6.2
        heritage: Helm
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: rancher
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 1
        type: RollingUpdate
      template:
        metadata:
          labels:
            app: rancher
        spec:
          serviceAccountName: rancher
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - rancher
                  topologyKey: kubernetes.io/hostname
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms: 
                  - matchExpressions:
                    - key: kubernetes.io/os
                      operator: NotIn
                      values:
                      - windows
          tolerations: 
            - key: "cattle.io/os"
              value: "linux"
              effect: "NoSchedule"
              operator: "Equal"
          containers:
          - image: rancher/rancher:v2.6.2
            imagePullPolicy: IfNotPresent
            name: rancher
            ports:
            - containerPort: 80
              protocol: TCP
            args:
            # Public trusted CA - clear ca certs
            - "--no-cacerts"
            - "--http-listen-port=80"
            - "--https-listen-port=443"
            - "--add-local=true"
            env:
            - name: CATTLE_NAMESPACE
              value: ops
            - name: CATTLE_PEER_SERVICE
              value: rancher
            - name: CATTLE_BOOTSTRAP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: "bootstrap-secret"
                  key: "bootstrapPassword"
            livenessProbe:
              httpGet:
                path: /healthz
                port: 80
              initialDelaySeconds: 60
              periodSeconds: 30
            readinessProbe:
              httpGet:
                path: /healthz
                port: 80
              initialDelaySeconds: 5
              periodSeconds: 30
            resources:
              {}
            volumeMounts:
          volumes:
    
    ---
    # Source: rancher/templates/service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: rancher
      namespace: ops
      labels:
        app: rancher
        chart: rancher-2.6.2
        heritage: Helm
    spec:
      ports:
      - port: 80
        targetPort: 80
        protocol: TCP
        name: http
      - port: 443
        targetPort: 444
        protocol: TCP
        name: https-internal
      selector:
        app: rancher
    
    ---
    # Source: rancher/templates/ingress.yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: rancher
      namespace: ops
      labels:
        app: rancher
        chart: rancher-2.6.2
        heritage: Helm
      annotations:
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/ssl-redirect: "false" # turn off ssl redirect for external.
        nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
        nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
        nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
    spec:
      rules:
      - host: rancher.ops.putaoevent.com
        http:
          paths:
          - backend:
              serviceName: rancher
              servicePort: 80
    ```

