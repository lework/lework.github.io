---
layout: post
title: "使用国内镜像加速你的jenkins"
date: "2020-03-05 20:40"
category: jenkins
tags: jenkins
author: lework
---
* content
{:toc}

老铁们，是不是每次安装 **jenkins** 的时候，下载插件都能等上半天一天的。还在为此烦恼么，看过此篇文章立即解决你的烦恼。

国内已经有几家站点都同步了 **jenkins** 仓库和插件仓库，当你设置了**清华大学**的 [update-center.json](https://mirror.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json) 时，在满怀欣喜的等待飙升的下载速度，得到的确实一动不动的进度条，那是因为国内镜像源是原封不动的同步 **jenkins**仓库的，其 [update-center.json](https://mirror.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json) 里的插件下载地址还是 **jenkins** 的地址，当然加速不了。需要把这个文件里的下载路径更改为国内镜像源地址，才能享受飙升的下载速度。

为此，我针对国内的镜像站点一一生成了 [update-center.json](https://mirror.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json) 下面就教大家怎么使用吧！




## 测试速度

在使用国内镜像站点的时候，不妨先测试下哪个站点下载速度最快的，我们当然要选择**最靓**的那个崽了。

**镜像站点**

- tencent https://mirrors.cloud.tencent.com/jenkins/
- huawei https://mirrors.huaweicloud.com/jenkins/
- tsinghua https://mirrors.tuna.tsinghua.edu.cn/jenkins/
- ustc https://mirrors.ustc.edu.cn/jenkins/
- bit http://mirror.bit.edu.cn/jenkins/

```bash
# curl -sSL https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/speed-test.sh | bash

Jenkins mirror update center speed test

[Mirror Site]
tencent       :  https://mirrors.cloud.tencent.com/jenkins/
bit           :  http://mirror.bit.edu.cn/jenkins/
huawei        :  https://mirrors.huaweicloud.com/jenkins/
tsinghua      :  https://mirrors.tuna.tsinghua.edu.cn/jenkins/
ustc          :  https://mirrors.ustc.edu.cn/jenkins/

[Test]
Test File        : updates/current/plugin-versions.json

Site Name     IPv4 address        File Size     Download Time       Download Speed
tencent       111.231.36.190      9.2M          2.6s                3.58MB/s      
bit           202.204.80.77       9.2M          6.6s                1.39MB/s      
huawei        117.78.24.32        9.2M          0.5s                19.4MB/s      
tsinghua      101.6.8.193         9.2M          9.6s                976KB/s       
ustc          202.38.95.110       9.2M          1.2s                7.75MB/s      

```

*huawei* 就是那个**最靓**的崽



## 使用国内镜像

当我们在安装完 **jenkins** 的时候，别着急登录web，先使用下国内源



1. 先上传自定义的ca证书

   > 因为 `update-center.json`  需要证书签名，jenkins会去校验里面的数据。

```bash
[ ! -d /var/lib/jenkins/update-center-rootCAs ] && mkdir /var/lib/jenkins/update-center-rootCAs
wget https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/rootCA/update-center.crt -O /var/lib/jenkins/update-center-rootCAs/update-center.crt
chown jenkins.jenkins -R /var/lib/jenkins/update-center-rootCAs
```

2. 更改url

这里在终端里进行更改

```bash
sed -i 's#https://updates.jenkins.io/update-center.json#https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/huawei/update-center.json#' /var/lib/jenkins/hudson.model.UpdateCenter.xml
```

> 当然也可以通过web 来更改：Go to `Jenkins` → `Manage Jenkins` → `Manage Plugins` → `Advanced` → Update Site and submit URL to your `https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/huawei/update-center.json`   

然后再去 **web** 页面初始化你的 **jenkins**，享受速度飙升的快感吧。



其他镜像站点的 **update-center.json** 

> 文件会在每天utc时间1点钟更新

| Site     | Source                                                       | CDN                                                          |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| tencent  | https://raw.githubusercontent.com/lework/jenkins-update-center/master/updates/tencent/update-center.json | https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/tencent/update-center.json |
| huawei   | https://raw.githubusercontent.com/lework/jenkins-update-center/master/updates/huawei/update-center.json | https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/huawei/update-center.json |
| tsinghua | https://raw.githubusercontent.com/lework/jenkins-update-center/master/updates/tsinghua/update-center.json | https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/tsinghua/update-center.json |
| ustc     | https://raw.githubusercontent.com/lework/jenkins-update-center/master/updates/ustc/update-center.json | https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/ustc/update-center.json |
| bit      | https://raw.githubusercontent.com/lework/jenkins-update-center/master/updates/bit/update-center.json | https://cdn.jsdelivr.net/gh/lework/jenkins-update-center/updates/bit/update-center.json |

