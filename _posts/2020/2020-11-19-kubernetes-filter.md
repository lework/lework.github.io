---
layout: post
title: "过滤 kubernetes 的导出信息"
date: "2020-11-19 18:36"
category: kubernetes
tags: kubernetes
author: lework
---
* content
{:toc}

在 k8s 中导出(`kubectl get xx -o yaml`)资源描述信息时，会带出一些k8s系统添加的信息，但这些都不是我们需要的信息，官方没有提供过滤的选项，在下面我给出了几种方式来处理这种情况。




## 输出 yaml

使用 [yq](https://github.com/mikefarah/yq) 项目过滤 yaml 信息

```bash
wget https://gh.con.sh/https://github.com/mikefarah/yq/releases/download/3.4.1/yq_linux_amd64 -O /usr/local/sbin/yq
chmod +x /usr/local/sbin/yq

kubectl_yaml() {
  kubectl -o yaml "$@" \
    | yq d - 'items[*].metadata.managedFields' \
    | yq d - 'metadata.managedFields' \
    | yq d - 'items[*].metadata.ownerReferences' \
    | yq d - 'metadata.ownerReferences' \
    | yq d - 'items[*].metadata.annotations."kubectl.kubernetes.io/last-applied-configuration"' \
    | yq d - 'metadata.annotations."kubectl.kubernetes.io/last-applied-configuration"' \
    | yq d - 'items[*].metadata.creationTimestamp' \
    | yq d - 'metadata.creationTimestamp' \
    | yq d - 'items[*].metadata.resourceVersion' \
    | yq d - 'metadata.resourceVersion' \
    | yq d - 'items[*].metadata.generateName' \
    | yq d - 'metadata.generateName' \
    | yq d - 'items[*].metadata.selfLink' \
    | yq d - 'metadata.selfLink' \
    | yq d - 'items[*].metadata.uid' \
    | yq d - 'metadata.uid' \
    | yq d - 'items[*].status' \
    | yq d - 'status'
}
```
测试下获取

```bash
# kubectl_yaml get pods
apiVersion: v1
items:
  - apiVersion: v1
    kind: Pod
    metadata:
      labels:
        app: ingress-demo-app
        pod-template-hash: 78ccc7c466
      name: ingress-demo-app-78ccc7c466-8kz5h
      namespace: default
    spec:
      containers:
        - image: traefik/whoami:v1.6.0
          imagePullPolicy: IfNotPresent
          name: whoami
          ports:
            - containerPort: 80
              protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
              name: default-token-qksdb
              readOnly: true
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
      nodeName: k8s-worker-node1
      preemptionPolicy: PreemptLowerPriority
      priority: 0
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: default
      serviceAccountName: default
      terminationGracePeriodSeconds: 30
      tolerations:
        - effect: NoExecute
          key: node.kubernetes.io/not-ready
          operator: Exists
          tolerationSeconds: 300
        - effect: NoExecute
          key: node.kubernetes.io/unreachable
          operator: Exists
          tolerationSeconds: 300
      volumes:
        - name: default-token-qksdb
          secret:
            defaultMode: 420
            secretName: default-token-qksdb
  - apiVersion: v1
    kind: Pod
    metadata:
      labels:
        app: ingress-demo-app
        pod-template-hash: 78ccc7c466
      name: ingress-demo-app-78ccc7c466-m5slh
      namespace: default
    spec:
      containers:
        - image: traefik/whoami:v1.6.0
          imagePullPolicy: IfNotPresent
          name: whoami
          ports:
            - containerPort: 80
              protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
              name: default-token-qksdb
              readOnly: true
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
      nodeName: k8s-worker-node2
      preemptionPolicy: PreemptLowerPriority
      priority: 0
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: default
      serviceAccountName: default
      terminationGracePeriodSeconds: 30
      tolerations:
        - effect: NoExecute
          key: node.kubernetes.io/not-ready
          operator: Exists
          tolerationSeconds: 300
        - effect: NoExecute
          key: node.kubernetes.io/unreachable
          operator: Exists
          tolerationSeconds: 300
      volumes:
        - name: default-token-qksdb
          secret:
            defaultMode: 420
            secretName: default-token-qksdb
kind: List
metadata: {}

# kubectl_yaml get pod ingress-demo-app-78ccc7c466-m5slh
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: ingress-demo-app
    pod-template-hash: 78ccc7c466
  name: ingress-demo-app-78ccc7c466-m5slh
  namespace: default
spec:
  containers:
    - image: traefik/whoami:v1.6.0
      imagePullPolicy: IfNotPresent
      name: whoami
      ports:
        - containerPort: 80
          protocol: TCP
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: default-token-qksdb
          readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: k8s-worker-node2
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
      tolerationSeconds: 300
  volumes:
    - name: default-token-qksdb
      secret:
        defaultMode: 420
        secretName: default-token-qksdb
```

```bash
# kubectl_yaml get service
kubectl_yaml get service
apiVersion: v1
items:
  - apiVersion: v1
    kind: Service
    metadata:
      annotations: {}
      name: ingress-demo-app
      namespace: default
    spec:
      clusterIP: 10.96.44.53
      ports:
        - name: http
          port: 80
          protocol: TCP
          targetPort: 80
      selector:
        app: ingress-demo-app
      sessionAffinity: None
      type: ClusterIP
  - apiVersion: v1
    kind: Service
    metadata:
      labels:
        component: apiserver
        provider: kubernetes
      name: kubernetes
      namespace: default
    spec:
      clusterIP: 10.96.0.1
      ports:
        - name: https
          port: 443
          protocol: TCP
          targetPort: 6443
      sessionAffinity: None
      type: ClusterIP
kind: List
metadata: {}

# kubectl_yaml get service ingress-demo-app
apiVersion: v1
kind: Service
metadata:
  annotations: {}
  name: ingress-demo-app
  namespace: default
spec:
  clusterIP: 10.96.44.53
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: ingress-demo-app
  sessionAffinity: None
  type: ClusterIP
```

## 输出 json

使用 [jq](https://github.com/stedolan/jq) 项目过滤 json 信息

```bash
wget https://gh.con.sh/https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64 -O /usr/local/sbin/jq
chmod +x /usr/local/sbin/jq

kubectl_json() {
  kubectl -o json "$@" \
    | jq 'del(.items[]?.metadata.managedFields)' \
    | jq 'del(.metadata.managedFields)' \
    | jq 'del(.items[]?.metadata.ownerReferences)' \
    | jq 'del(.metadata.ownerReferences)' \
    | jq 'del(.items[]?.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration")' \
    | jq 'del(.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration")' \
    | jq 'del(.items[]?.metadata.creationTimestamp)' \
    | jq 'del(.metadata.creationTimestamp)' \
    | jq 'del(.items[]?.metadata.resourceVersion)' \
    | jq 'del(.metadata.resourceVersion)' \
    | jq 'del(.items[]?.metadata.generateName)' \
    | jq 'del(.metadata.generateName)' \
    | jq 'del(.items[]?.metadata.selfLink)' \
    | jq 'del(.metadata.selfLink)' \
    | jq 'del(.items[]?.metadata.uid)' \
    | jq 'del(.metadata.uid)' \
    | jq 'del(.items[]?.status)' \
    | jq 'del(.status)'
}

```

测试下获取

```bash
# kubectl_json get pods
{
  "apiVersion": "v1",
  "items": [
    {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {
        "labels": {
          "app": "ingress-demo-app",
          "pod-template-hash": "78ccc7c466"
        },
        "name": "ingress-demo-app-78ccc7c466-8kz5h",
        "namespace": "default"
      },
      "spec": {
        "containers": [
          {
            "image": "traefik/whoami:v1.6.0",
            "imagePullPolicy": "IfNotPresent",
            "name": "whoami",
            "ports": [
              {
                "containerPort": 80,
                "protocol": "TCP"
              }
            ],
            "resources": {},
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "volumeMounts": [
              {
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount",
                "name": "default-token-qksdb",
                "readOnly": true
              }
            ]
          }
        ],
        "dnsPolicy": "ClusterFirst",
        "enableServiceLinks": true,
        "nodeName": "k8s-worker-node1",
        "preemptionPolicy": "PreemptLowerPriority",
        "priority": 0,
        "restartPolicy": "Always",
        "schedulerName": "default-scheduler",
        "securityContext": {},
        "serviceAccount": "default",
        "serviceAccountName": "default",
        "terminationGracePeriodSeconds": 30,
        "tolerations": [
          {
            "effect": "NoExecute",
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "tolerationSeconds": 300
          },
          {
            "effect": "NoExecute",
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "tolerationSeconds": 300
          }
        ],
        "volumes": [
          {
            "name": "default-token-qksdb",
            "secret": {
              "defaultMode": 420,
              "secretName": "default-token-qksdb"
            }
          }
        ]
      }
    },
    {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {
        "labels": {
          "app": "ingress-demo-app",
          "pod-template-hash": "78ccc7c466"
        },
        "name": "ingress-demo-app-78ccc7c466-m5slh",
        "namespace": "default"
      },
      "spec": {
        "containers": [
          {
            "image": "traefik/whoami:v1.6.0",
            "imagePullPolicy": "IfNotPresent",
            "name": "whoami",
            "ports": [
              {
                "containerPort": 80,
                "protocol": "TCP"
              }
            ],
            "resources": {},
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "volumeMounts": [
              {
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount",
                "name": "default-token-qksdb",
                "readOnly": true
              }
            ]
          }
        ],
        "dnsPolicy": "ClusterFirst",
        "enableServiceLinks": true,
        "nodeName": "k8s-worker-node2",
        "preemptionPolicy": "PreemptLowerPriority",
        "priority": 0,
        "restartPolicy": "Always",
        "schedulerName": "default-scheduler",
        "securityContext": {},
        "serviceAccount": "default",
        "serviceAccountName": "default",
        "terminationGracePeriodSeconds": 30,
        "tolerations": [
          {
            "effect": "NoExecute",
            "key": "node.kubernetes.io/not-ready",
            "operator": "Exists",
            "tolerationSeconds": 300
          },
          {
            "effect": "NoExecute",
            "key": "node.kubernetes.io/unreachable",
            "operator": "Exists",
            "tolerationSeconds": 300
          }
        ],
        "volumes": [
          {
            "name": "default-token-qksdb",
            "secret": {
              "defaultMode": 420,
              "secretName": "default-token-qksdb"
            }
          }
        ]
      }
    }
  ],
  "kind": "List",
  "metadata": {}
}

# kubectl_json get service
{
  "apiVersion": "v1",
  "items": [
    {
      "apiVersion": "v1",
      "kind": "Service",
      "metadata": {
        "annotations": {},
        "name": "ingress-demo-app",
        "namespace": "default"
      },
      "spec": {
        "clusterIP": "10.96.44.53",
        "ports": [
          {
            "name": "http",
            "port": 80,
            "protocol": "TCP",
            "targetPort": 80
          }
        ],
        "selector": {
          "app": "ingress-demo-app"
        },
        "sessionAffinity": "None",
        "type": "ClusterIP"
      }
    },
    {
      "apiVersion": "v1",
      "kind": "Service",
      "metadata": {
        "labels": {
          "component": "apiserver",
          "provider": "kubernetes"
        },
        "name": "kubernetes",
        "namespace": "default"
      },
      "spec": {
        "clusterIP": "10.96.0.1",
        "ports": [
          {
            "name": "https",
            "port": 443,
            "protocol": "TCP",
            "targetPort": 6443
          }
        ],
        "sessionAffinity": "None",
        "type": "ClusterIP"
      }
    }
  ],
  "kind": "List",
  "metadata": {}
}
```
## 使用插件

[kubectl-neat](https://github.com/itaysk/kubectl-neat) 清理Kuberntes yaml和json输出，使其具有可读性.

```bash
wget https://gh.con.sh/https://github.com/itaysk/kubectl-neat/releases/download/v2.0.1/kubectl-neat_linux.tar.gz
tar zxf kubectl-neat_linux.tar.gz  -C /usr/local/sbin/
chmod +x /usr/local/sbin/kubectl-neat
```

测试使用
```bash
# kubectl neat get -- pods -oyaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: ingress-demo-app
      pod-template-hash: 78ccc7c466
    name: ingress-demo-app-78ccc7c466-8kz5h
    namespace: default
  spec:
    containers:
    - image: traefik/whoami:v1.6.0
      name: whoami
      ports:
      - containerPort: 80
    preemptionPolicy: PreemptLowerPriority
    priority: 0
    serviceAccountName: default
    tolerations:
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
      tolerationSeconds: 300
- apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: ingress-demo-app
      pod-template-hash: 78ccc7c466
    name: ingress-demo-app-78ccc7c466-m5slh
    namespace: default
  spec:
    containers:
    - image: traefik/whoami:v1.6.0
      name: whoami
      ports:
      - containerPort: 80
    preemptionPolicy: PreemptLowerPriority
    priority: 0
    serviceAccountName: default
    tolerations:
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
      tolerationSeconds: 300
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
  
# kubectl get svc -o json | kubectl neat                 
{
    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "v1",
            "kind": "Service",
            "metadata": {"name":"ingress-demo-app","namespace":"default"},
            "spec": {
                "clusterIP": "10.96.44.53",
                "ports": [
                    {
                        "name": "http",
                        "port": 80
                    }
                ],
                "selector": {
                    "app": "ingress-demo-app"
                }
            }
        },
        {
            "apiVersion": "v1",
            "kind": "Service",
            "metadata": {"labels":{"component":"apiserver","provider":"kubernetes"},"name":"kubernetes","namespace":"default"},
            "spec": {
                "clusterIP": "10.96.0.1",
                "ports": [
                    {
                        "name": "https",
                        "port": 443,
                        "targetPort": 6443
                    }
                ]
            }
        }
    ],
    "kind": "List",
    "metadata": {
        "resourceVersion": "",
        "selfLink": ""
    }
}
```

更多例子
```bash
kubectl get pod mypod -o yaml | kubectl neat
kubectl get pod mypod -oyaml | kubectl neat -o json
kubectl neat -f - <./my-pod.json
kubectl neat -f ./my-pod.json
kubectl neat -f ./my-pod.json --output yaml
kubectl neat get -- pod mypod -oyaml
kubectl neat get -- svc -n default myservice --output json
```

## 使用 sed

> 使用sed时，需时刻注意匹配关键字

```bash
# kubectl get svc ingress-demo-app -o yaml \
  | sed -n '/ managedFields:/{p; :a; N; / name: ingress-demo-app/!ba; s/.*\n//}; p' \
  | sed -e 's/ uid:.*//g' \
        -e 's/ resourceVersion:.*//g' \
        -e 's/ selfLink:.*//g' \
        -e 's/ creationTimestamp:.*//g' \
        -e 's/ managedFields:.*//g' \
        -e 's/ generateName:.*//g' \
        -e '/^\s*$/d'
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"ingress-demo-app","namespace":"default"},"spec":{"ports":[{"name":"http","port":80,"targetPort":80}],"selector":{"app":"ingress-demo-app"},"type":"ClusterIP"}}
  name: ingress-demo-app
  namespace: default
spec:
  clusterIP: 10.96.44.53
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: ingress-demo-app
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```


