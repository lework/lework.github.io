---
layout: post
title: 'rabbitmq监控'
date: '2019-12-24 20:00:00'
category: rabbitmq
tags: rabbitmq
author: lework
---
* content
{:toc}


从3.8.0版开始，RabbitMQ附带了内置的Prometheus＆Grafana支持。3.8之前的RabbitMQ版本可以使用单独的插件 `prometheus_rabbitmq_exporter` 来向Prometheus公开指标。

只需开启`rabbitmq_prometheus`插件，就可以启用prometheus的metrics接口，默认端口为15692




## 启用命令

```bash
rabbitmq-plugins enable rabbitmq_prometheus
```

访问 http://localhost:15692/metrics 就可以获取rabbitmq的指标数据

集群或者单机部署，这里就不在描述了，具体可以查看docker方式的安装情况


进入我们的docker-compose 配置目录中
```bash
git clone https://github.com/lework/Docker-compose-file.git
cd Docker-compose-file/rabbitmq/
```

**启动prometheus监控套件**

```bash
docker-compose -f docker-compose-metrics.yml up -d
```

**访问prometheus**

http://localhost:9090

![rabbitmq](/assets/images/rabbitmq/rabbitmq7.png)
![rabbitmq](/assets/images/rabbitmq/rabbitmq8.png)


**访问grafana**

http://localhost:3000  用户admin, 默认密码admin

![rabbitmq](/assets/images/rabbitmq/rabbitmq9.png)
