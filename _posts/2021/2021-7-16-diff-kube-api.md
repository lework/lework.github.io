---
layout: post
title: '运维小需求：查看及对比 kubernetes 各个版本的 API 资源对象信息'
date: '2021-07-16 18:02'
category: 运维小需求
tags: 运维小需求 kubernetes
author: lework
---
* content
{:toc}

## 需求

查看 kubernetes 各个版本的 API 资源对象信息，以及对比其变化。




## 实现

技术栈： vue，ant-design-vue, python

大概流程：

1. 由 [github action](https://github.com/lework/kube-api/blob/master/.github/workflows/update.yml) 提供 **收集数据** 功能。 在工作日 `10点` 检测是否有新版本; 当有新版本时，就会从 [Kubernetes API 文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/) 中拉取并整合 api 信息，存储到 `static/data/data.json`。
2. 由 [kube-api](https://github.com/lework/kube-api) 提供 **查询** 功能, 数据从 `static/data/data.json` 获取。

## demo 

- 网站:  [https://lework.github.io/kube-api/#/](https://lework.github.io/kube-api/#/)
- 源码:  [https://github.com/lework/kube-api](https://github.com/lework/kube-api)

![kube-api1](/assets/images/2021/demand/kube-api1.png)

![kube-api2](/assets/images/2021/demand/kube-api2.png)

![kube-api3](/assets/images/2021/demand/kube-api3.png)