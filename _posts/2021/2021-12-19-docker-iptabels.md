---
layout: post
title: 'Iptables 和 Docker 共存'
date: '2021-12-19 21:00'
category: docker
tags: linux docker
author: lework 
---
* content
{:toc}


## 需求

> Docker Server Version: 20.10.12

  在一台装有Docker的服务器上，需要利用iptables作为防火墙限制外部访问。




## Iptables && Docker

iptables 分为表(tables)、链(chain)和规则(rules)三个层面，我们一般只用到 filter 表，其中包含：

- INPUT，输入链。发往本机的数据包通过此链。
- OUTPUT，输出链。从本机发出的数据包通过此链。
- FORWARD，转发链。本机转发的数据包通过此链。

由于 Docker 默认通过 NAT（网络地址转换）连接默认虚拟网桥(docker0)和容器的默认网关(ens33)，所以设置INPUT 是没有用的，应该设置 **FORWARD 链**。而由于FORWARD链默认为 `DROP`, 所有转发都被 `DOCKER-USER ` 拦截。

```bash
Chain FORWARD (policy DROP)
target     prot opt source               destination         
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0           
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0     
```

因此，我们要利用 `DOCKER-USER` 链设置容器访问规则，具体实现操作如下。



**注意: 防火墙管理程序要关闭。**

```bash
# 关闭防火墙管理程序
for target in firewalld python-firewall firewalld-filesystem iptables; do
  systemctl stop $target &>/dev/null || true
  systemctl disable $target &>/dev/null || true
done
```



## 实现1

> 这里需要安装下 iptables-services
>  yum install -y iptables-services

1. 系统的规则

   ```bash
   # 先允许所有,这里很重要，不加会被拒绝
   iptables -P INPUT ACCEPT 
   
   iptables -F  # 清空所有默认规则
   iptables -X  # 清空所有自定义规则
   iptables -Z  # 所有计数器归0
   
   # 对外开放的端口
   iptables -A INPUT -p tcp -m multiport --dport 22  -j ACCEPT
   
   # 允许ping
   iptables -A INPUT -p icmp -j ACCEPT
   
   # 允许主动连接其他主机
   iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
   
   # 其他入站一律丢弃
   iptables -P INPUT DROP
   
   # 所有出站一律绿灯
   iptables -P OUTPUT ACCEPT
   
   # 保存规则
   service iptables save
   ```
   
2. docker 的规则

   ```bash
   # 重启docker, 生成docker的规则
   systemctl restart docker
   ```

3. `DOCKER-USER` 链 的规则

   ```bash
   # 其他入站一律丢弃
   iptables -I DOCKER-USER -i ens33 ! -s 127.0.0.1 -j DROP
   
   # 允许主动连接其他主机
   iptables -I DOCKER-USER -m state --state established,related -j ACCEPT
   
   iptables -A DOCKER-USER -m conntrack --ctstate established,related -j ACCEPT
   
   # 对外开放的端口
   iptables -I DOCKER-USER -p tcp -m conntrack --ctorigdstport 80  -j ACCEPT
   iptables -I DOCKER-USER -p tcp -m conntrack --ctorigdstport 443  -j ACCEPT
   
   # 保存规则
   service iptables save
   ```
   > ens33 是我的节点外网接口
   
4. 此时的 iptables 规则

   ```bash
   # iptables -L -n
   Chain INPUT (policy DROP)
   target     prot opt source               destination         
   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 22
   ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
   
   Chain FORWARD (policy DROP)
   target     prot opt source               destination         
   DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0           
   DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0           
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
   DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain OUTPUT (policy ACCEPT)
   target     prot opt source               destination         
   
   Chain DOCKER (1 references)
   target     prot opt source               destination         
   ACCEPT     tcp  --  0.0.0.0/0            172.17.0.2           tcp dpt:8080
   ACCEPT     tcp  --  0.0.0.0/0            172.17.0.2           tcp dpt:80
   ACCEPT     tcp  --  0.0.0.0/0            172.17.0.3           tcp dpt:80
   
   Chain DOCKER-ISOLATION-STAGE-1 (1 references)
   target     prot opt source               destination         
   DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain DOCKER-ISOLATION-STAGE-2 (1 references)
   target     prot opt source               destination         
   DROP       all  --  0.0.0.0/0            0.0.0.0/0           
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain DOCKER-USER (1 references)
   target     prot opt source               destination         
   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            ctorigdstport 80
   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            ctorigdstport 443
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
   DROP       all  -- !127.0.0.1            0.0.0.0/0                     
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   ```
   
   这个实现方式，有个缺陷，就是要每次 docker 容器发生改变的时候，都需要保存下当前iptables的配置，不然重启 iptables 服务的时候，就会加载 老的规则，从而导致iptables 规则混乱。


## 实现2 

为了解决上述问题，我们可以

- 创建一个名为 FILTERS 的新链，来自 INPUT 和 DOCKER-USER 的网络流量被放入其中，并将此配置存放到一个文件中。
- 创建一个 iptables systemd 服务，用来重载 iptables 规则。

> 这里我们不需要系统安装的 iptables-services , 使用命令给他卸载掉。
> yum remove -y iptables-services  

1. 创建 iptables.conf 规则文件

    ```bash
    cat <<EOF > /etc/iptables.conf
    *filter

    # Reset counters
    :INPUT ACCEPT [0:0]
    :FORWARD DROP [0:0]
    :OUTPUT ACCEPT [0:0]
    :DOCKER-USER - [0:0]
    :FILTERS - [0:0]

    # Flush
    -F INPUT
    -F DOCKER-USER
    -F FILTERS

    # Select
    -A INPUT -i lo -j ACCEPT
    -A INPUT -i ens33 -j FILTERS
    -A DOCKER-USER -i ens33 -j FILTERS
    -A DOCKER-USER -j RETURN

    # Filters
    ## icmp
    -A FILTERS -p icmp --icmp-type any -j ACCEPT

    ## Activate established connexions
    -A FILTERS -m state --state ESTABLISHED,RELATED -j ACCEPT
    -A FILTERS -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

    ## Allow ssh
    -A FILTERS -p tcp -m multiport --dport 22  -j ACCEPT

    ## Allow all on http/https
    -A FILTERS -p tcp -m tcp -m conntrack --ctorigdstport 80 -j ACCEPT
    -A FILTERS -p tcp -m tcp -m conntrack --ctorigdstport 443 -j ACCEPT
    
    # Block all external
    -A FILTERS -i ens33 -j DROP
    -A FILTERS -j RETURN

    ## Commit
    COMMIT
    EOF
    ```
    > ens33 是我的节点外网接口

2. 创建 system 服务

    ```bash
    cat << EOF >  /usr/lib/systemd/system/iptables.service
    [Unit]
    Description=Restore iptables firewall rules
    Before=network-pre.target

    [Service]
    Type=oneshot
    ExecStart=/sbin/iptables-restore -n /etc/iptables.conf

    [Install]
    WantedBy=multi-user.target
    EOF

    systemctl enable iptables
    systemctl start iptables
    ```

    > iptables-restore -n  关闭隐式全局刷新，只执行我们手动的显式刷新。

3. 此时的 iptables 规则

   ```bash
   Chain INPUT (policy ACCEPT)
   target     prot opt source               destination         
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
   FILTERS    all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain FORWARD (policy DROP)
   target     prot opt source               destination         
   DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0           
   DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0           
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
   DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain OUTPUT (policy ACCEPT)
   target     prot opt source               destination         
   
   Chain DOCKER (1 references)
   target     prot opt source               destination         
   
   Chain DOCKER-ISOLATION-STAGE-1 (1 references)
   target     prot opt source               destination         
   DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain DOCKER-ISOLATION-STAGE-2 (1 references)
   target     prot opt source               destination         
   DROP       all  --  0.0.0.0/0            0.0.0.0/0           
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain DOCKER-USER (1 references)
   target     prot opt source               destination         
   FILTERS    all  --  0.0.0.0/0            0.0.0.0/0           
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   
   Chain FILTERS (2 references)
   target     prot opt source               destination         
   ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 255
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
   ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 22
   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp ctorigdstport 80
   ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp ctorigdstport 443
   DROP       all  --  0.0.0.0/0            0.0.0.0/0           
   RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
   ```

   

如果说我们需要更新规则，直接编辑 `/etc/iptables.conf` 文件，然后执行 `systemctl restart iptables`  重新加载下规则即可。



## 参考

- https://docs.docker.com/network/iptables/
- https://github.com/boTux-fr/systemd-service-iptables
- https://blog.bestucloud.com/tech/iptables-and-docker-conflicts/
- https://unrouted.io/2017/08/15/docker-firewall/
- https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html
- https://www.path8.net/docs/iptables-tutorial_cn/iptables-tutorial-1.2.2-cn.pdf