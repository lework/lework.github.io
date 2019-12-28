---
layout: post
title: 'rabbitmq单机安装'
date: '2019-12-21 20:00:00'
category: rabbitmq
tags: rabbitmq
author: lework
---
* content
{:toc}


## 使用二进制版本安装




**centos7系统**

```bash
yum -y install socat
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v22.1.8/erlang-22.1.8-1.el7.x86_64.rpm
rpm -i erlang-22.1.8-1.el7.x86_64.rpm

wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.2/rabbitmq-server-generic-unix-3.8.2.tar.xz

tar xf rabbitmq-server-generic-unix-3.8.2.tar.xz  -C /usr/local/
ln -s /usr/local/rabbitmq_server-3.8.2 /usr/local/rabbitmq
echo 'export PATH=/usr/local/rabbitmq/sbin/:$PATH' >> /etc/profile
source /etc/profile

hostnamectl set-hostname rabbitmq
echo '127.0.0.1 rabbitmq' >> /etc/hosts

useradd rabbitmq
mkdir -p /var/lib/rabbitmq /var/log/rabbitmq
chown rabbitmq.rabbitmq -R /var/lib/rabbitmq /var/log/rabbitmq /usr/local/rabbitmq/

# 设置配置文件
cat >> /usr/local/rabbitmq/etc/rabbitmq/rabbitmq-env.conf <<EOF
RABBITMQ_NODENAME=rabbit@rabbitmq
RABBITMQ_NODE_IP_ADDRESS=0.0.0.0
RABBITMQ_NODE_PORT=5672
RABBITMQ_LOG_BASE=/var/log/rabbitmq
RABBITMQ_MNESIA_BASE=/var/lib/rabbitmq/mnesia
EOF

cat >> /usr/local/rabbitmq/etc/rabbitmq/rabbitmq.conf <<EOF
listeners.tcp.default = 5672
num_acceptors.tcp = 10

management.tcp.port = 15672
management.tcp.ip   = 0.0.0.0
management.http_log_dir = /var/log/rabbitmq/management_access
	
vm_memory_high_watermark.absolute = 512MiB
vm_memory_high_watermark_paging_ratio = 0.3

loopback_users.guest = true
EOF

cat >> /usr/local/rabbitmq/etc/rabbitmq/enabled_plugins <<EOF
[rabbitmq_management].
EOF

cat >> /etc/systemd/system/rabbitmq-server.service <<EOF
[Unit]
Description=RabbitMQ broker
After=syslog.target network.target

[Service]
Type=notify
User=rabbitmq
Group=rabbitmq
UMask=0027
NotifyAccess=all
TimeoutStartSec=3600
LimitNOFILE=32768
Restart=on-failure
RestartSec=10
WorkingDirectory=/var/lib/rabbitmq
ExecStart=/usr/local/rabbitmq/sbin/rabbitmq-server
ExecStop=/usr/local/rabbitmq/sbin/rabbitmqctl shutdown
SuccessExitStatus=69

[Install]
WantedBy=multi-user.target
EOF

# 设置cookie
echo "rabbitmq@sigle" >> ~/.erlang.cookie
echo "rabbitmq@sigle" >> /home/rabbitmq/.erlang.cookie
chown rabbitmq.rabbitmq /home/rabbitmq/.erlang.cookie
chmod 600 ~/.erlang.cookie  /home/rabbitmq/.erlang.cookie

systemctl enable --now rabbitmq-server

```


## 使用系统包安装

**centos7系统**

```bash
yum -y install socat
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v22.1.8/erlang-22.1.8-1.el7.x86_64.rpm
rpm -i erlang-22.1.8-1.el7.x86_64.rpm

wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.2/rabbitmq-server-3.8.2-1.el7.noarch.rpm
rpm -i rabbitmq-server-3.8.2-1.el7.noarch.rpm 

systemctl enable --now rabbitmq-server

lsof -i:5672
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
beam    2678 root   15u  IPv6  20971      0t0  TCP *:amqp (LISTEN)

```

**debian9系统**

```bash
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
dpkg -i erlang-solutions_1.0_all.deb
apt-get update
apt-get install erlang

apt-get -y install socat logrotate init-system-helpers adduser
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.2/rabbitmq-server_3.8.2-1_all.deb
dpkg -I rabbitmq-server_3.8.2-1_all.deb

systemctl enable --now rabbitmq-server
```

添加用户

```bash
rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin '.*' '.*' '.*'
rabbitmqctl list_users
```

启动插件

```bash
rabbitmq-plugins enable rabbitmq_management
```

查看状态
```bash
rabbitmqctl status
```

登录web页面

http://192.168.77.130:15672 

用户名admin，密码admin
