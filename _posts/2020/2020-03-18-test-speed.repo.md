---
layout: post
title: "测试国内镜像站点的下载速度"
date: "2020-03-18 22:40"
category: linux
tags: linux
author: lework
---
* content
{:toc}

众所周知，国内有时很难访问国外的开源软件站点，能访问时，速度也会异常的慢。国内有很多公益组织纷纷镜像了国外站点，方便国内用户访问，这里非常感谢他们。中国地大物博，不同地区的访问镜像站点速度各有不同，为了大家能选择合适的镜像站点，我特意写了些脚本来测试站点的下载速度。





## linux系统软件源 测速

**测试网络** 

上海长城宽带

**执行测速**

```bash
curl -sSL https://cdn.jsdelivr.net/gh/lework/script/shell/os_repo_speed_test.sh | bash
```

**结果**

```bash

Os repo mirror site speed test

[Mirror Site]
edu_bit       :  http://mirror.bit.edu.cn/
tencent       :  https://mirrors.cloud.tencent.com/
edu_dlut      :  http://mirror.dlut.edu.cn/
edu_tsinghua  :  https://mirrors.tuna.tsinghua.edu.cn/
edu_lzu       :  http://mirror.lzu.edu.cn/
edu_zju       :  http://mirrors.zju.edu.cn/
aliyun        :  https://mirrors.aliyun.com/
netease       :  http://mirrors.cn99.com/
edu_bjtu      :  https://mirror.bjtu.edu.cn/
edu_sjtu      :  http://ftp.sjtu.edu.cn/
edu_ustc      :  https://mirrors.ustc.edu.cn/
huawei        :  https://mirrors.huaweicloud.com/
edu_cqu       :  http://mirrors.cqu.edu.cn/
sohu          :  http://mirrors.sohu.com/

[Test]
Test OS       : centos
Download File : centos/filelist.gz

Site Name     IPv4 address        File Size     Download Time       Download Speed
edu_bit       219.143.204.117     7.2M          41s                 182KB/s       
tencent       111.231.36.190      7.2M          1.6s                4.65MB/s      
edu_dlut      202.118.65.164      7.2M          36s                 206KB/s       
edu_tsinghua  101.6.8.193         7.2M          4.9s                1.46MB/s      
edu_lzu       202.201.0.160       7.2M          45s                 166KB/s       
edu_zju       210.32.158.231      7.2M          26s                 288KB/s       
aliyun        58.216.17.182       7.2M          28s                 269KB/s       
netease       59.111.0.251        7.2M          1.5s                4.83MB/s      
edu_bjtu      202.112.154.58      7.2M          5.0s                1.46MB/s      
edu_sjtu      202.38.97.230       7.2M          2.6s                2.82MB/s      
edu_ustc      202.141.176.110     7.2M          1.7s                4.15MB/s      
huawei        117.78.24.36        7.2M          1.5s                4.79MB/s      
edu_cqu       202.202.1.140       7.2M          1m58s               62.9KB/s
sohu          123.125.123.141     7.2M                     
```

同时，安利一个网站，在找开源软件的国内镜像点时，可以到[https://lework.github.io/lemonitor](https://lework.github.io/lemonitor) 站点去查找国内哪个服务商提供镜像


## Docker hub 测速

**网络地址**

阿里云网络

**执行测速**

```bash
curl -sSL https://cdn.jsdelivr.net/gh/lework/script/shell/docker_hub_speed_test.sh | bash
```

**结果**

```bash

Docker Hub mirror site speed test

[Mirror Site]
ustc          :  https://docker.mirrors.ustc.edu.cn
tencent       :  https://mirror.ccs.tencentyun.com
netease       :  http://hub-mirror.c.163.com
qiniu         :  https://reg-mirror.qiniu.com
azure         :  http://dockerhub.azk8s.cn
aliyun        :  https://2h3po24q.mirror.aliyuncs.com
daocloud      :  http://f1361db2.m.daocloud.io
docker        :  https://registry-1.docker.io

[Test]
Test Image        : library/centos:latest
Download layer    : sha256:8a29a15cefaeccf6545f7ecf11298f9672d2f0cdaf9e357a95133ac3ad3e1f07

Site Name     IPv4 address        File Size     Download Time       Download Speed
ustc          202.141.176.110                                                     
tencent                                                                           
netease       45.127.128.36       70M           6.0s                11.7MB/s      
qiniu         115.231.100.205                                                     
azure         40.73.69.67         70M           4.9s                14.3MB/s      
aliyun        118.31.232.171      70M           5.6s                12.6MB/s      
daocloud      1.1.1.1                                                             
docker        104.18.123.25       70M           5m31s               216KB/s       
```