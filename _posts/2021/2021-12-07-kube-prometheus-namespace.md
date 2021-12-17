---
layout: post
title: 'kube-prometheus 添加新的 namespace 到promethues监控中'
date: '2021-12-07 21:00'
category: prometheus
tags: kubernetes prometheus
author: lework 
---
* content
{:toc}


使用 [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) 安装的 prometheus 只会监控 `default` `kube-system` `monitoring`（kube-prometheus 自己创建的ns）三个命名空间，而如果想要添加其他的命名空间，就需要另外的操作。




## 1. 监控其他 namespace 中的 endpoint 资源


需要做的操作

1. 创建新增命名空间的 role， 用于获取监控的信息。
2. 将创建的role绑定到 monitoring 命名空间中的 prometheus-k8s sa。

```bash
kubectl create ns test

namespace=test

cat <<EOF | kubectl apply -f - 
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: prometheus-k8s
  namespace: ${namespace}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prometheus-k8s
  namespace: ${namespace}
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
EOF
```



## 2. 监控其他 namespace 中的 serviceMonitor 资源

> serviceMonitorNamespaceSelector 命名空间的标签匹配，不指定时，只匹配自身命名空间的资源
>
> serviceMonitorSelector serviceMonitor的标签匹配， 不指定时，只匹配自身命名空间的资源

修改 Prometheus 资源配置

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
...
# 增加 ns 匹配的标签
  serviceMonitorNamespaceSelector:
    matchLabels:
      serviceMonitor: prometheus
      
# 或者 增加下面的匹配，用来匹配 serviceMonitor
  serviceMonitorSelector:
    matchLabels:
      serviceMonitor: prometheus
```

添加命名空间的标签

```bash
for ns in default kube-system monitoring test; do 
  kubectl patch ns $ns --patch '{"metadata":{"labels":{"serviceMonitor": "prometheus" } } }'
done
```

添加 serviceMonitor 的标签

```bash
kubectl patch -n test servicemonitor demo-app --patch '{"metadata":{"labels":{"serviceMonitor":"prometheus"}}}' --type=merge
```

测试 

```
cat <<EOF | kubectl-test apply -f - 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-demo-app
  namespace: test
  labels:
    app: ingress-demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ingress-demo-app
  template:
    metadata:
      labels:
        app: ingress-demo-app
        namespace: test
    spec:
      containers:
      - name: whoami
        image: traefik/whoami:v1.6.1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-demo-app
  namespace: test
  labels:
    app: ingress-demo-app
spec:
  type: ClusterIP
  selector:
    app: ingress-demo-app
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-demo-app
  namespace: test
  labels:
    app: ingress-demo-app
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: app.demo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ingress-demo-app
            port:
              number: 80
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    name: ingress-demo-app
  name: ingress-demo-app
  namespace: test
spec:
  endpoints:
  - port: http
    path: /health
    interval: 5s
  selector:
    matchLabels:
      app: ingress-demo-app
EOF
```

