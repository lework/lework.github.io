---
layout: post
title: 'rabbitmq使用docker方式安装'
date: '2019-12-23 20:00:00'
category: rabbitmq
tags: rabbitmq
author: lework
---
* content
{:toc}


## 安装docker

```bash
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
sed -i 's#download.docker.com#mirrors.ustc.edu.cn/docker-ce#g' /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce bash-completion
cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
mkdir  /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "3"
    },
    "default-ulimits": {
        "nofile": {
             "Name": "nofile",
             "Hard": 64000,
             "Soft": 64000
         }
    },
    "live-restore": true,
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true"
    ],
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn/"
    ]
}
EOF

systemctl enable --now docker
```



## 安装docker-compose

```bash
curl -L https://raw.githubusercontent.com/docker/compose/1.25.0/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
curl -L https://github.com/docker/compose/releases/download/1.25.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## 启动单实例

```bash
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3.8.2-management

docker exec rabbitmq  rabbitmqctl status
```

web访问  http://192.168.77.130:15672 使用guest用户登录，密码guest


## 启动集群

下载 docker-compose file

```bash
git clone https://github.com/lework/Docker-compose-file.git
cd Docker-compose-file/rabbitmq/

cat docker-compose-cluster.yml
version: "3"

networks:
  rabbitmq:
  
services:
  rmq0: &rabbitmq 
    image: rabbitmq:3.8.2-management
    hostname: rmq0
    container_name: rmq0
    environment:
      RABBITMQ_ERLANG_COOKIE: rabbitmq-cluster-cookie
    volumes:
      - ./cluster/conf/enabled_plugins:/etc/rabbitmq/enabled_plugins:ro
      - ./cluster/conf/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./cluster/conf/rabbitmq-definitions.json:/etc/rabbitmq/rabbitmq-definitions.json:ro
    ports:
      - "5673:5672"
      - "15673:15672"
      - "15693:15692"
    networks:
      - rabbitmq
    cap_add:
      - ALL
    ulimits:
      nofile:
        soft: "2000"
        hard: "2000"
    restart: always
  rmq1:
    << : *rabbitmq
    hostname: rmq1
    container_name: rmq1
    ports:
      - "5674:5672"
      - "15674:15672"
      - "15694:15692"
  rmq2:
    << : *rabbitmq
    hostname: rmq2
    container_name: rmq2
    ports:
      - "5675:5672"
      - "15675:15672"
      - "15695:15692"


cat 1.init-cluster.sh

#!/bin/bash


exec="docker-compose -f docker-compose-cluster.yml exec"


# rmq1,内存节点

$exec rmq1 rabbitmqctl stop_app # 停止rabbitmq服务
$exec rmq1 rabbitmqctl reset # 清空节点状态
$exec rmq1 rabbitmqctl join_cluster --ram rabbit@rmq0 # rmq1和rmq0构成集群,rmq1必须能通过rmq0的主机名ping通
$exec rmq1 rabbitmqctl start_app  # 开启rabbitmq服务

# rmq2,内存节点

$exec rmq2 rabbitmqctl stop_app # 停止rabbitmq服务
$exec rmq2 rabbitmqctl reset # 清空节点状态
$exec rmq2 rabbitmqctl join_cluster --ram rabbit@rmq0 # rmq2和rmq0构成集群,rmq2必须能通过rmq0的主机名ping通
$exec rmq2 rabbitmqctl start_app  # 开启rabbitmq服务


$exec rmq0 rabbitmqctl cluster_status


cat 2.create-queue.sh 
#!/bin/bash


exec="docker-compose -f docker-compose-cluster.yml exec rmq0"


# queue
echo "__create queue________"
$exec rabbitmqadmin declare queue --vhost=/ name=ha1.queue durable=true
$exec rabbitmqadmin declare queue --vhost=/ name=ha2.queue durable=true
$exec rabbitmqadmin declare queue --vhost=/ name=ha3.queue durable=true
$exec rabbitmqadmin declare queue --vhost=/ name=all.queue
$exec rabbitmqadmin declare queue --vhost=/ name=nodes.queue


# exchange
echo "__create exchange________"
$exec rabbitmqadmin declare exchange --vhost=/ name=test.exchange type=direct durable=true

# binding 

$exec rabbitmqadmin declare binding --vhost=/ source=test.exchange destination=ha1.queue routing_key=ha1.queue
$exec rabbitmqadmin declare binding --vhost=/ source=test.exchange destination=ha2.queue routing_key=ha2.queue
$exec rabbitmqadmin declare binding --vhost=/ source=test.exchange destination=ha3.queue routing_key=ha3.queue

# list

echo "__queues___________________"
$exec rabbitmqctl list_queues -p /
echo "__exchanges___________________"
$exec rabbitmqctl list_exchanges -p /
echo "__bindings___________________"
$exec rabbitmqctl list_bindings -p /

# send message
echo "__send message________"
$exec rabbitmqadmin publish routing_key=ha1.queue payload="just for queue"
$exec rabbitmqadmin publish exchange=test.exchange routing_key=ha1.queue payload="hello, world"

echo "__get queue________"
$exec rabbitmqadmin get queue=ha1.queue

# consumer
echo "__consumer message_______"
$exec rabbitmqadmin get queue=ha1.queue ackmode=ack_requeue_false
$exec rabbitmqadmin get queue=ha1.queue ackmode=ack_requeue_false
echo "__queue message______________"
$exec rabbitmqadmin get queue=ha1.queue
```

**启动集群**
```bash
docker-compose -f docker-compose-cluster.yml up -d
```

**初始化集群**
```bash
bash 1.init-cluster.sh
```

**创建队列，测试集群**

```bash
bash 2.create-queue.sh 
```

**节点信息**
```
主机	节点名称	amqp	http
10.1.223.62	rabbit@rmq0	5673	15673
10.1.223.62	rabbit@rmq1	5674	15674
10.1.223.62	rabbit@rmq2	5675	15675
```

web( http://10.1.223.62:15673/ ) 界面使用admin用户登陆，密码admin
