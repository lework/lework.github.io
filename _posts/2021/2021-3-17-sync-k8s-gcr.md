---
layout: post
title: '运维小需求: 同步 k8s.gcr.io 镜像'
date: '2021-03-17 22:00'
category: 运维小需求
tags: 运维小需求 kubernetes skopeo 
author: lework
---
* content
{:toc}

## 痛点

1. `k8s.gcr.io` 仓库中的镜像难以下载, 国内使用 `kubernetes` 困难重重。
2. 国内网友实现的同步方案比较复杂，不容易新增和删除镜像。

## 目的

实现一个自动同步所需镜像从到 `k8s.gcr.io` 国内( `aliyun` )的镜像仓库，且新增和配置镜像操作需简单化。





## 技术架构

- `python` 脚本：对比**源**和**目的仓库镜像**同步信息，生成需要同步的镜像配置。

- `skopeo`：使用生成的同步配置，进行同步操作。

- `Github Action`：每天自动运行脚本同步镜像到阿里云。 


## 功能

1. 支持动态同步镜像列表(默认获取最新的 5 个 tag 用于同步)。
2. 支持静态同步镜像列表(使用指定的 tag 用于同步)。
3. 方便调整同步位置。


## 使用

动态镜像同步配置

> 采用 yaml 结构配置，方便清晰。fork 仓库后修改配置文件提交 pr 即可。

```yaml
last: 5
images:
  k8s.gcr.io:
    - etcd
    - coredns
    - kube-proxy
    - kube-apiserver
    - kube-scheduler
    - kube-controller-manager
    - metrics-server/metrics-server
    - ingress-nginx/controller
    - dns/k8s-dns-node-cache
```

静态镜像同步配置

```yaml
k8s.gcr.io:
  images:
    defaultbackend-amd64: 
     - '1.5'
    pause:
     - '3.5'
     - '3.4.1'
     - '3.3'
```

同步规则

```
k8s.gcr.io/{image_name}  ==>  registry.cn-hangzhou.aliyuncs.com/kainstall/{image_name}
```

拉取镜像
```bash
$ docker pull registry.cn-hangzhou.aliyuncs.com/kainstall/kube-scheduler:[镜像版本号]
```


## 具体实现

请看 Github 仓库 [sync_image](https://github.com/lework/sync_image)

