---
layout: post
title: '运维小需求：动态更新 nginx upstream 的后端地址'
date: '2021-06-26 18:02'
category: 运维小需求
tags: 运维小需求 nginx docker
author: lework
---
* content
{:toc}

## 需求

定时轮训指定名称的 `docker container` `IP` 地址， 将可用的容器 `IP` 动态更新到 `nginx` 中的 `upstream`




## 实现

技术栈： bash 脚本（轻量小巧）

大概流程：

1. 循环执行主流程检测。
2. 获取指定的 `docker container`  `IP` 地址。
3. 筛选获取的地址，去除不正常访问的地址。
4. 获取指定的 `upstream` 的 `server` 地址。
5. 对比两方的地址情况，少了就加，多了就删。
6. 有更改时，备份配置文件，更新地址并重载nginx。
7. 发送变动通知。

## demo 

脚本信息 [check-container-ip.sh](https://github.com/lework/script/blob/master/shell/check-container-ip.sh)

```bash
# bash check-container-ip.sh 
Detect container ip addresses and dynamically update nginx upstream configuration.

Usage:
  check-container-ip.sh [options]

Options:
  --container-name  Docker container name
  --config-file     Nginx conf path
  --upstream-name   Nginx upstream name
  --service-port    Service port
  --retries         Retries number, default:9223372036854775807
  --wait            Retries wait time, default:5
  -h,--help         View help

Example:
  check-container-ip.sh --container-name api-test \
     --config-file /etc/nginx/conf.d/api.conf \
     --upstream-name api \
     --service-port 8000 \
     --retries 20 \
     --wait 5
```

执行示例

```bash
# bash check-container-ip.sh \
  --container-name test \
  --config-file /etc/nginx/conf.d/app.conf \
  --upstream-name jdc \
  --service-port 80 \
  --wait 10

[2021-06-26T18:11:44.689876022+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:11:44.691175649+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:11:44.848207545+0800]: INFO:    [container] IP: 172.17.0.4 172.17.0.2 
[2021-06-26T18:11:44.849298269+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:11:44.866825303+0800]: INFO:    [container] Check success:  172.17.0.4 172.17.0.2
[2021-06-26T18:11:44.868169682+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:11:44.873904890+0800]: INFO:    [nginx] Upstream host: 172.17.0.4 172.17.0.2 
[2021-06-26T18:11:44.875084081+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:11:44.885852957+0800]: INFO:    [nginx] IP: 172.17.0.4 ip already exists on the upstream host.
[2021-06-26T18:11:44.889362447+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.

[2021-06-26T18:11:54.895074423+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:11:54.896400190+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:11:55.022315191+0800]: INFO:    [container] IP: 172.17.0.4 172.17.0.2 
[2021-06-26T18:11:55.024691081+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:11:55.046834370+0800]: INFO:    [container] Check success:  172.17.0.4 172.17.0.2
[2021-06-26T18:11:55.048455315+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:11:55.058293420+0800]: INFO:    [nginx] Upstream host: 172.17.0.4 172.17.0.2 
[2021-06-26T18:11:55.060708823+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:11:55.070984879+0800]: INFO:    [nginx] IP: 172.17.0.4 ip already exists on the upstream host.
[2021-06-26T18:11:55.075567487+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.

[2021-06-26T18:12:05.079230194+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:12:05.081411012+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:12:05.210756842+0800]: INFO:    [container] IP: 172.17.0.4 172.17.0.2 
[2021-06-26T18:12:05.212034230+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:12:05.227071927+0800]: INFO:    [container] Check success:  172.17.0.4 172.17.0.2
[2021-06-26T18:12:05.228430952+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:12:05.232234299+0800]: INFO:    [nginx] Upstream host: 172.17.0.4 172.17.0.2 
[2021-06-26T18:12:05.233529434+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:12:05.240125763+0800]: INFO:    [nginx] IP: 172.17.0.4 ip already exists on the upstream host.
[2021-06-26T18:12:05.241538682+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.

[2021-06-26T18:12:15.244381936+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:12:15.249022097+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:12:15.369714380+0800]: INFO:    [container] IP: 172.17.0.2 
[2021-06-26T18:12:15.371114977+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:12:15.378531792+0800]: INFO:    [container] Check success:  172.17.0.2
[2021-06-26T18:12:15.380109775+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:12:15.384894510+0800]: INFO:    [nginx] Upstream host: 172.17.0.4 172.17.0.2 
[2021-06-26T18:12:15.386182081+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:12:15.391151216+0800]: INFO:    [nginx] IP: 172.17.0.4 delete from upstream host.
[2021-06-26T18:12:15.394097602+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.
[2021-06-26T18:12:15.399146322+0800]: INFO:    [nginx] Backup nginx config to /tmp/app.conf-1624961535
[2021-06-26T18:12:15.403289497+0800]: INFO:    [nginx] Apply nginx config file.
[2021-06-26T18:12:15.406450126+0800]: INFO:    [nginx] Reload nginx.
[2021-06-26T18:12:15.423804139+0800]: INFO:    [nginx] Reload nginx success.
[2021-06-26T18:12:15.425168496+0800]: INFO:    [notice] select feishu
[2021-06-26T18:12:15.730561435+0800]: INFO:    [notice] send feishu success.

[2021-06-26T18:12:25.734062812+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:12:25.735668081+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:12:25.855734726+0800]: INFO:    [container] IP: 172.17.0.2 
[2021-06-26T18:12:25.856897778+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:12:25.864057139+0800]: INFO:    [container] Check success:  172.17.0.2
[2021-06-26T18:12:25.865536108+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:12:25.871325777+0800]: INFO:    [nginx] Upstream host: 172.17.0.2 
[2021-06-26T18:12:25.872879719+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:12:25.877713259+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.

[2021-06-26T18:12:35.880746441+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:12:35.882388085+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:12:36.013987039+0800]: INFO:    [container] IP: 172.17.0.3 172.17.0.4 172.17.0.2 
[2021-06-26T18:12:36.015466094+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:12:36.037020420+0800]: INFO:    [container] Check success:  172.17.0.3 172.17.0.4 172.17.0.2
[2021-06-26T18:12:36.038286881+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:12:36.043353802+0800]: INFO:    [nginx] Upstream host: 172.17.0.2 
[2021-06-26T18:12:36.044992484+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:12:36.052393380+0800]: INFO:    [nginx] IP: 172.17.0.3 add to upstream hosts.
[2021-06-26T18:12:36.056616206+0800]: INFO:    [nginx] IP: 172.17.0.4 add to upstream hosts.
[2021-06-26T18:12:36.059997855+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.
[2021-06-26T18:12:36.063132066+0800]: INFO:    [nginx] Backup nginx config to /tmp/app.conf-1624961556
[2021-06-26T18:12:36.068179039+0800]: INFO:    [nginx] Apply nginx config file.
[2021-06-26T18:12:36.073193553+0800]: INFO:    [nginx] Reload nginx.
[2021-06-26T18:12:36.094124818+0800]: INFO:    [nginx] Reload nginx success.
[2021-06-26T18:12:36.095663246+0800]: INFO:    [notice] select feishu
[2021-06-26T18:12:36.506075275+0800]: INFO:    [notice] send feishu success.

[2021-06-26T18:12:46.514596534+0800]: INFO:    [check] Start inspection.
[2021-06-26T18:12:46.516122793+0800]: INFO:    [container] Get container ip address.
[2021-06-26T18:12:46.634794500+0800]: INFO:    [container] IP: 172.17.0.3 172.17.0.4 172.17.0.2 
[2021-06-26T18:12:46.635920918+0800]: INFO:    [container] Check container service status.
[2021-06-26T18:12:46.655990108+0800]: INFO:    [container] Check success:  172.17.0.3 172.17.0.4 172.17.0.2
[2021-06-26T18:12:46.657474480+0800]: INFO:    [nginx] Get nginx upstream jdc host.
[2021-06-26T18:12:46.662043763+0800]: INFO:    [nginx] Upstream host: 172.17.0.4 172.17.0.3 172.17.0.2 
[2021-06-26T18:12:46.663385986+0800]: INFO:    [nginx] Config nginx upstream jdc host.
[2021-06-26T18:12:46.672982088+0800]: INFO:    [nginx] IP: 172.17.0.4 ip already exists on the upstream host.
[2021-06-26T18:12:46.674180166+0800]: INFO:    [nginx] IP: 172.17.0.3 ip already exists on the upstream host.
[2021-06-26T18:12:46.675648645+0800]: INFO:    [nginx] IP: 172.17.0.2 ip already exists on the upstream host.
^C[2021-06-26T18:12:48.251628392+0800]: INFO:    [stop] check total: 6
```

消息通知


![home](/assets/images/2021/demand/check-container-ip.png)
