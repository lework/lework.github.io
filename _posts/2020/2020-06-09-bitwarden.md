---
layout: post
title: "自建 Bitwarden 密码服务器"
date: "2020-06-09 16:40"
category: linux
tags: Bitwarden linux
author: lework
---
* content
{:toc}

## 介绍

[Bitwarden](https://bitwarden.com/) 是一个免费和开源的密码管理服务，它将一些敏感信息，比如网站证书，存储在一个加密的保险库里。Bitwarden平台提供了多种客户端应用程序，包括web界面、桌面应用程序、浏览器扩展、移动应用程序和CLI。


相关链接

- github https://github.com/bitwarden
- 社区论坛 https://community.bitwarden.com/
- 帮助中心 https://bitwarden.com/help/




## 安装

本次将在 `Centos 7.7 x64` 系统上以 `Docker` 形式部署 [Bitwarden](https://bitwarden.com/) 

### 安装 docker 环境

 ```bash
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
sed -i 's#download.docker.com#mirrors.ustc.edu.cn/docker-ce#g' /etc/yum.repos.d/docker-ce.repo

yum -y install docker-ce docker-compose bash-completion

cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/

mkdir /etc/docker
cat >> /etc/docker/daemon.json <<EOF
{
 "data-root": "/var/lib/docker",
 "log-driver": "json-file",
 "log-opts": {
  "max-size": "100m",
  "max-file": "3"
 },
 "default-ulimits": {
  "nofile": {
   "Name": "nofile",
   "Hard": 65535,
   "Soft": 65535
  },
  "nproc": {
   "Name": "nproc",
   "Hard": 65535,
   "Soft": 65535
  }
 },
 "live-restore": true,
 "max-concurrent-downloads": 10,
 "max-concurrent-uploads": 10,
 "storage-driver": "overlay2",
 "storage-opts": ["overlay2.override_kernel_check=true"],
 "exec-opts": ["native.cgroupdriver=systemd"],
 "registry-mirrors": ["https://2h3po24q.mirror.aliyuncs.com"]
}
EOF

systemctl enable docker && systemctl start docker
 ```

### 获取安装 key

打开 [https://bitwarden.com/host/](https://bitwarden.com/host/)，在网页中输入邮箱即可获得安装key，记得将此信息**保存下来**。

 

### 安装 Bitwarden

```bash
mkdir bitwarden && cd bitwarden

wget https://raw.githubusercontent.com/bitwarden/server/master/scripts/bitwarden.sh && chmod +x bitwarden.sh
```

**执行安装脚本**

```bash
# ./bitwarden.sh install
 _     _ _                         _            
| |__ (_) |___      ____ _ _ __ __| | ___ _ __  
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \ 
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2020, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

Docker version 19.03.1, build 74b1e89
docker-compose version 1.18.0, build 8dd22a9

(!) Enter the domain name for your Bitwarden instance (ex. bitwarden.example.com): # 输入域名

(!) Do you want to use Let\'s Encrypt to generate a free SSL certificate? (y/n): # 是否使用Let\'s Encrypt证书(域名需解析到安装主机的外网ip)，或者自生成。

(!) Enter your email address (Let\'s Encrypt will send you certificate expiration reminders): # 邮件地址

Using default tag: latest
latest: Pulling from certbot/certbot
Digest: sha256:568b8ebd95641a365a433da4437460e69fb279f6c9a159321988d413c6cde0ba
Status: Image is up to date for certbot/certbot:latest
docker.io/certbot/certbot:latest
Saving debug log to /etc/letsencrypt/logs/letsencrypt.log
Plugins selected: Authenticator standalone, Installer None
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for pass.leops.cn
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/pass.leops.cn/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/pass.leops.cn/privkey.pem
   Your cert will expire on 2020-09-07. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let\'s Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

1.34.0: Pulling from bitwarden/setup
Digest: sha256:e529c155e033bf3dbfb9f0a4e9c1b69eafd9becd3eba406d868173f431c01bca
Status: Image is up to date for bitwarden/setup:1.34.0
docker.io/bitwarden/setup:1.34.0

(!) Enter your installation id (get at https://bitwarden.com/host): # 安装id

(!) Enter your installation key: # 安装key

Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
...............+................................+.................................................................................+.............................................................................+................................................................................................+......++*++*++*++*
Generating key for IdentityServer.
Generating a RSA private key
...................................................................................................................................................................++++
.........................++++
writing new private key to 'identity.key'
-----

Building nginx config.
Building docker environment files.
Building docker environment override files.
Building FIDO U2F app id.
Building docker-compose.yml.

Installation complete

If you need to make additional configuration changes, you can modify
the settings in `./bwdata/config.yml` and then run:
`./bitwarden.sh rebuild` or `./bitwarden.sh update`

Next steps, run:
`./bitwarden.sh start`
```

 **注意项**

- 如果脚本没有让你输入域名，而是直接退出，可能是网络问题，试试多次执行。

- 使用Let\'s Encrypt证书时， 需将域名解析到安装主机的外网ip，以便验证。

  

### 设置 admin 和 smtp

在 `bwdata/env/global.override.env` 文件中修改下列信息

```bash
globalSettings__mail__replyToEmail=REPLACE
globalSettings__mail__smtp__host=REPLACE
globalSettings__mail__smtp__port=587
globalSettings__mail__smtp__ssl=false
globalSettings__mail__smtp__username=REPLACE
globalSettings__mail__smtp__password=REPLACE
globalSettings__disableUserRegistration=false
globalSettings__hibpApiKey=REPLACE
adminSettings__admins=REPLACE
```

**注意项**

- `replyToEmail` 要与 `smtp__username` 一致。
- `adminSettings__admins` 设置管理员用户的邮箱地址，多个使用逗号分隔。
- `globalSettings__disableUserRegistration` 可关闭用户注册。

### 启动 Bitwarden

```bash
# ./bitwarden.sh start
 _     _ _                         _            
| |__ (_) |___      ____ _ _ __ __| | ___ _ __  
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \ 
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2020, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

Docker version 19.03.1, build 74b1e89
docker-compose version 1.18.0, build 8dd22a9

Removing network docker_default
WARNING: Network docker_default not found.
Removing network docker_public
WARNING: Network docker_public not found.
Pulling mssql (bitwarden/mssql:1.34.0)...
1.34.0: Pulling from bitwarden/mssql
Digest: sha256:a16a414d242aba61a80e8e3bbcb797f1b0b6532d13d900ffd62a477946f3ea4d
Status: Image is up to date for bitwarden/mssql:1.34.0
Pulling web (bitwarden/web:2.14.0)...
2.14.0: Pulling from bitwarden/web
Digest: sha256:25db87aaf774e1555187101d509e96de5aaabe881bd4d9862f8e9aa53d8475a0
Status: Image is up to date for bitwarden/web:2.14.0
Pulling attachments (bitwarden/attachments:1.34.0)...
1.34.0: Pulling from bitwarden/attachments
Digest: sha256:528a4ca3fe691da39a919380e2aade0df17d77d1e2d4687457bcb96e1a002c7f
Status: Image is up to date for bitwarden/attachments:1.34.0
Pulling api (bitwarden/api:1.34.0)...
1.34.0: Pulling from bitwarden/api
Digest: sha256:6f61786749791d12a6795ddac736bfdffa3747ed846d130f5d1068f8d8c13077
Status: Image is up to date for bitwarden/api:1.34.0
Pulling identity (bitwarden/identity:1.34.0)...
1.34.0: Pulling from bitwarden/identity
Digest: sha256:1cf1724b24031c6a6e36352cc530f3e2047bc4b7847d1a99294ef82b35f2f4c4
Status: Image is up to date for bitwarden/identity:1.34.0
Pulling admin (bitwarden/admin:1.34.0)...
1.34.0: Pulling from bitwarden/admin
Digest: sha256:9c92de29e6c623738f038633fabc159cec8a248d303a32f8fc24be8173d62ff6
Status: Image is up to date for bitwarden/admin:1.34.0
Pulling icons (bitwarden/icons:1.34.0)...
1.34.0: Pulling from bitwarden/icons
Digest: sha256:ce7b5193f6fadccdc6b7f35495fbc463ee55931d24b66b8f62749ec9eda8b4ec
Status: Image is up to date for bitwarden/icons:1.34.0
Pulling notifications (bitwarden/notifications:1.34.0)...
1.34.0: Pulling from bitwarden/notifications
Digest: sha256:e5089384781bcf1fea5f056b31f5ed15b86c01f25e234ac3f9809d2c0d34caff
Status: Image is up to date for bitwarden/notifications:1.34.0
Pulling events (bitwarden/events:1.34.0)...
1.34.0: Pulling from bitwarden/events
Digest: sha256:46cf86da3f45f6d480e907b2a799f9ae568f3867cf608fb31d7272f26938f79c
Status: Image is up to date for bitwarden/events:1.34.0
Pulling nginx (bitwarden/nginx:1.34.0)...
1.34.0: Pulling from bitwarden/nginx
Digest: sha256:c5963667e4e98673e80b0809c992192c682bcfd2d1dd29acfb87eb0273998161
Status: Image is up to date for bitwarden/nginx:1.34.0
Using default tag: latest
latest: Pulling from certbot/certbot
Digest: sha256:568b8ebd95641a365a433da4437460e69fb279f6c9a159321988d413c6cde0ba
Status: Image is up to date for certbot/certbot:latest
docker.io/certbot/certbot:latest
Saving debug log to /etc/letsencrypt/logs/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/pass.leops.cn.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert not yet due for renewal

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

The following certs are not due for renewal yet:
  /etc/letsencrypt/live/pass.leops.cn/fullchain.pem expires on 2020-09-07 (skipped)
No renewals were attempted.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Creating directory /root/docker/bitwarden/bwdata/core
Creating directory /root/docker/bitwarden/bwdata/core/attachments
Creating directory /root/docker/bitwarden/bwdata/logs
Creating directory /root/docker/bitwarden/bwdata/logs/admin
Creating directory /root/docker/bitwarden/bwdata/logs/api
Creating directory /root/docker/bitwarden/bwdata/logs/events
Creating directory /root/docker/bitwarden/bwdata/logs/icons
Creating directory /root/docker/bitwarden/bwdata/logs/identity
Creating directory /root/docker/bitwarden/bwdata/logs/mssql
Creating directory /root/docker/bitwarden/bwdata/logs/nginx
Creating directory /root/docker/bitwarden/bwdata/logs/notifications
Creating directory /root/docker/bitwarden/bwdata/mssql/backups
Creating directory /root/docker/bitwarden/bwdata/mssql/data
Creating bitwarden-mssql ... done
Creating bitwarden-admin ... done
Creating bitwarden-nginx ... done
Creating bitwarden-icons ... 
Creating bitwarden-mssql ... 
Creating bitwarden-attachments ... 
Creating bitwarden-identity ... 
Creating bitwarden-web ... 
Creating bitwarden-notifications ... 
Creating bitwarden-events ... 
Creating bitwarden-admin ... 
Creating bitwarden-nginx ... 
1.34.0: Pulling from bitwarden/setup
Digest: sha256:e529c155e033bf3dbfb9f0a4e9c1b69eafd9becd3eba406d868173f431c01bca
Status: Image is up to date for bitwarden/setup:1.34.0
docker.io/bitwarden/setup:1.34.0


Bitwarden is up and running!
===================================================

visit https://pass.leops.cn
to update, run `./bitwarden.sh updateself` and then `./bitwarden.sh update`
```

启动的docker容器
```bash
# docker ps
CONTAINER ID        IMAGE                            COMMAND             CREATED             STATUS                         PORTS                                                 NAMES
67b1ea1b16fc        bitwarden/nginx:1.34.0           "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     80/tcp, 0.0.0.0:80->8080/tcp, 0.0.0.0:443->8443/tcp   bitwarden-nginx
81dd2acf5525        bitwarden/admin:1.34.0           "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     5000/tcp                                              bitwarden-admin
9f7286c1be93        bitwarden/web:2.14.0             "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)                                                           bitwarden-web
712fc9a405ba        bitwarden/attachments:1.34.0     "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)                                                           bitwarden-attachments
cb3576a64688        bitbetter/api:1.34.0             "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     5000/tcp                                              bitwarden-api
c4cdb436dd49        bitwarden/events:1.34.0          "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     5000/tcp                                              bitwarden-events
3caf86191670        bitwarden/icons:1.34.0           "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     5000/tcp                                              bitwarden-icons
0dee543adbab        bitwarden/notifications:1.34.0   "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     5000/tcp                                              bitwarden-notifications
5657a751b474        bitbetter/identity:1.34.0        "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)     5000/tcp                                              bitwarden-identity
99cd4bfe23d4        bitwarden/mssql:1.34.0           "/entrypoint.sh"    About an hour ago   Up About an hour (healthy)                                                           bitwarden-mssql
```

到此我们就安装好bitwarden，通过 https://pass.leops.cn 就能访问了。

![bitwarden](/assets/images/bitwarden/01.png)

## 使用

### 普通用户

通过访问 [https://pass.leops.cn](https://pass.leops.cn)，创建一个测试帐号

![bitwarden](/assets/images/bitwarden/02.png)

注册完成之后，就可以登录了。

![bitwarden](/assets/images/bitwarden/03.png)

添加项目

![bitwarden](/assets/images/bitwarden/04.png)



不仅仅通过web页面访问，还可以通过个平台的软件进行访问密码

- 桌面客户端
  - [Windows](https://vault.bitwarden.com/download/?app=desktop&platform=windows)
  - [MAC](https://vault.bitwarden.com/download/?app=desktop&platform=macos)
  - [Linux](https://vault.bitwarden.com/download/?app=desktop&platform=linux)
- 网页浏览器
  - [谷歌浏览器](https://chrome.google.com/webstore/detail/bitwarden-free-password-m/nngceckbapebfimnlniiiahkandclblb)
  - [微软 edge](https://microsoftedge.microsoft.com/addons/detail/jbkfoedolllekgbhcbcoahefnbanhhlh)
- 移动端
  - [Apple](https://itunes.apple.com/app/bitwarden-free-password-manager/id1137397744?mt=8)
  - [Google play](https://play.google.com/store/apps/details?id=com.x8bit.bitwarden)
- [命令行](https://bitwarden.com/help/article/cli/)

更多下载见 [官网](https://bitwarden.com/#download)

### Admin 后台

访问admin管理后台，可以通过 [https://pass.leops.cn/admin](https://pass.leops.cn/admin) 访问

![bitwarden](/assets/images/bitwarden/05.png)

输入管理员邮箱之后，会收到一封后台登录的链接地址，访问那个地址就能查看admin界面了。

![bitwarden](/assets/images/bitwarden/06.png)



用户界面

![bitwarden](/assets/images/bitwarden/07.png)

![bitwarden](/assets/images/bitwarden/08.png)

通过后台，只能删除用户，用户的数据是看不到的。



### 自我托管许可

在使用开源版本的bitwarden之后，用户想使用高级功能时，就需要到bitwarden官方申请许可，这个会收取一定的费用，手头宽裕的用户还是推荐申请下，支持下官方。

当我们自己托管和维护软件时，还要让企业用户去申请许可，恐怕软件很难推广下去，这里可以通过 [BitBetter](https://github.com/jakeswenson/BitBetter) 项目，为企业用户生成我们自己的个人许可和和组织许可。



#### 使用 BitBetter

下载软件

```bash
cd bitwarden
wget https://github.com/lework/BitBetter/archive/updates-1.34.0.zip
unzip BitBetter-updates-1.34.0.zip
```

> 在 bitwarden.sh 脚本同级目录中解压 BitBetter

**更新 bitwarden**

```bash
# ./update-bitwarden.sh 
Starting Bitwarden update, newest server version: 1.34.0
Enter Bitwarden base directory [/root/docker/bitwarden]: 
Rebuild docker-compose override? [Y/n]: Y
BitBetter docker-compose override created!
Rebuild BitBetter images? [y/N]: Y
Building BitBetter for BitWarden version 1.34.0
+ dotnet add package Newtonsoft.Json --version 12.0.3
  Determining projects to restore...
  Writing /tmp/tmpmU4q1R.tmp
info : Adding PackageReference for package 'Newtonsoft.Json' into project '/bitBetter/bitBetter.csproj'.
info : Restoring packages for /bitBetter/bitBetter.csproj...
info :   GET https://api.nuget.org/v3-flatcontainer/newtonsoft.json/index.json
info :   GET https://api.nuget.org/v3-flatcontainer/mono.cecil/index.json
info :   OK https://api.nuget.org/v3-flatcontainer/newtonsoft.json/index.json 163ms
info :   GET https://api.nuget.org/v3-flatcontainer/newtonsoft.json/12.0.3/newtonsoft.json.12.0.3.nupkg
info :   OK https://api.nuget.org/v3-flatcontainer/newtonsoft.json/12.0.3/newtonsoft.json.12.0.3.nupkg 68ms
info :   OK https://api.nuget.org/v3-flatcontainer/mono.cecil/index.json 624ms
info :   GET https://api.nuget.org/v3-flatcontainer/mono.cecil/0.11.2/mono.cecil.0.11.2.nupkg
info :   OK https://api.nuget.org/v3-flatcontainer/mono.cecil/0.11.2/mono.cecil.0.11.2.nupkg 67ms
info : Installing Mono.Cecil 0.11.2.
info : Installing Newtonsoft.Json 12.0.3.
info : Package 'Newtonsoft.Json' is compatible with all the specified frameworks in project '/bitBetter/bitBetter.csproj'.
info : PackageReference for package 'Newtonsoft.Json' version '12.0.3' updated in file '/bitBetter/bitBetter.csproj'.
info : Committing restore...
info : Assets file has not changed. Skipping assets file writing. Path: /bitBetter/obj/project.assets.json
log  : Restored /bitBetter/bitBetter.csproj (in 2.68 sec).
+ dotnet restore
  Determining projects to restore...
  All projects are up-to-date for restore.
+ dotnet publish
Microsoft (R) Build Engine version 16.6.0+5ff7b0c9e for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Determining projects to restore...
  All projects are up-to-date for restore.
  bitBetter -> /bitBetter/bin/Debug/netcoreapp3.1/bitBetter.dll
  bitBetter -> /bitBetter/bin/Debug/netcoreapp3.1/publish/
Sending build context to Docker daemon  2.823MB
Step 1/5 : ARG BITWARDEN_TAG
Step 2/5 : FROM ${BITWARDEN_TAG}
 ---> 86f509874ec3
Step 3/5 : COPY bin/Debug/netcoreapp3.1/publish/* /bitBetter/
 ---> 15097bfacf60
Step 4/5 : COPY ./.keys/cert.cert /newLicensing.cer
 ---> 967330bb086f
Step 5/5 : RUN set -e; set -x;     dotnet /bitBetter/bitBetter.dll &&     mv /app/Core.dll /app/Core.orig.dll &&     mv /app/modified.dll /app/Core.dll &&     rm -rf /bitBetter && rm -rf /newLicensing.cer
 ---> Running in 8689c0b8e4b9
+ dotnet /bitBetter/bitBetter.dll
Bit.Core.licensing.cer
Existing Cert Thumbprint: B34876439FCDA2846505B2EFBBA6C4A951313EBE
New Cert Thumbprint: 9945804CBA793F725FB6BABFCB67F5AA81A4D077
+ mv /app/Core.dll /app/Core.orig.dll
+ mv /app/modified.dll /app/Core.dll
+ rm -rf /bitBetter
+ rm -rf /newLicensing.cer
Removing intermediate container 8689c0b8e4b9
 ---> 8efdd493633f
Successfully built 8efdd493633f
Successfully tagged bitbetter/api:latest
Sending build context to Docker daemon  2.823MB
Step 1/5 : ARG BITWARDEN_TAG
Step 2/5 : FROM ${BITWARDEN_TAG}
 ---> 8750acc4f7f7
Step 3/5 : COPY bin/Debug/netcoreapp3.1/publish/* /bitBetter/
 ---> 42e30ebefe9d
Step 4/5 : COPY ./.keys/cert.cert /newLicensing.cer
 ---> 9a3691721506
Step 5/5 : RUN set -e; set -x;     dotnet /bitBetter/bitBetter.dll &&     mv /app/Core.dll /app/Core.orig.dll &&     mv /app/modified.dll /app/Core.dll &&     rm -rf /bitBetter && rm -rf /newLicensing.cer
 ---> Running in d7df916e2be3
+ dotnet /bitBetter/bitBetter.dll
Bit.Core.licensing.cer
Existing Cert Thumbprint: B34876439FCDA2846505B2EFBBA6C4A951313EBE
New Cert Thumbprint: 9945804CBA793F725FB6BABFCB67F5AA81A4D077
+ mv /app/Core.dll /app/Core.orig.dll
+ mv /app/modified.dll /app/Core.dll
+ rm -rf /bitBetter
+ rm -rf /newLicensing.cer
Removing intermediate container d7df916e2be3
 ---> dd111e208c45
Successfully built dd111e208c45
Successfully tagged bitbetter/identity:latest
BitBetter images updated to version: 1.34.0
 _     _ _                         _            
| |__ (_) |___      ____ _ _ __ __| | ___ _ __  
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \ 
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2020, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

Docker version 19.03.1, build 74b1e89
docker-compose version 1.18.0, build 8dd22a9

Updated self.
Patching bitwarden.sh completed...
 _     _ _                         _            
| |__ (_) |___      ____ _ _ __ __| | ___ _ __  
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \ 
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2020, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

Docker version 19.03.1, build 74b1e89
docker-compose version 1.18.0, build 8dd22a9

Bitwarden update completed!
```

**编译 bitbetter/licensegen 镜像**

```bash
# bash ./src/licenseGen/build.sh 
Sending build context to Docker daemon  23.04kB
Step 1/7 : FROM mcr.microsoft.com/dotnet/core/sdk:3.1 as build
 ---> 8c4dd5ac064a
Step 2/7 : WORKDIR /licenseGen
 ---> Using cache
 ---> da5f1d21ef91
Step 3/7 : COPY . /licenseGen
 ---> Using cache
 ---> 09954325cf87
Step 4/7 : RUN set -e; set -x;  dotnet add package Newtonsoft.Json --version 12.0.3     && dotnet restore       && dotnet publish
 ---> Using cache
 ---> 3e954a1697a8
Step 5/7 : FROM bitbetter/api
 ---> 8efdd493633f
Step 6/7 : COPY --from=build /licenseGen/bin/Debug/netcoreapp3.1/publish/* /app/
 ---> 53874154b289
Step 7/7 : ENTRYPOINT [ "dotnet", "/app/licenseGen.dll", "--core", "/app/Core.dll", "--cert", "/cert.pfx" ]
 ---> Running in af6c58006ae1
Removing intermediate container af6c58006ae1
 ---> d5fe75d1404f
Successfully built d5fe75d1404f
Successfully tagged bitbetter/licensegen:latest
 ```

**将 restart() 函数里的 dockerComposePull 注释掉**

文件 `./../bwdata/scripts/run.sh`

```bash
function restart() {
    dockerComposeDown
    #dockerComposePull
    updateLetsEncrypt
    dockerComposeUp
    printEnvironment
}
```

**重启应用**

```bash
./../bitwarden.sh restart
 _     _ _                         _            
| |__ (_) |___      ____ _ _ __ __| | ___ _ __  
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \ 
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2020, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

Docker version 19.03.1, build 74b1e89
docker-compose version 1.18.0, build 8dd22a9

Stopping bitwarden-nginx         ... done
Stopping bitwarden-admin         ... done
Stopping bitwarden-identity      ... done
Stopping bitwarden-events        ... done
Stopping bitwarden-mssql         ... done
Stopping bitwarden-attachments   ... done
Stopping bitwarden-icons         ... done
Stopping bitwarden-notifications ... done
Stopping bitwarden-api           ... done
Stopping bitwarden-web           ... done
Removing bitwarden-nginx         ... done
Removing bitwarden-admin         ... done
Removing bitwarden-identity      ... done
Removing bitwarden-events        ... done
Removing bitwarden-mssql         ... done
Removing bitwarden-attachments   ... done
Removing bitwarden-icons         ... done
Removing bitwarden-notifications ... done
Removing bitwarden-api           ... done
Removing bitwarden-web           ... done
Removing network docker_default
Removing network docker_public
Using default tag: latest
latest: Pulling from certbot/certbot
Digest: sha256:568b8ebd95641a365a433da4437460e69fb279f6c9a159321988d413c6cde0ba
Status: Image is up to date for certbot/certbot:latest
docker.io/certbot/certbot:latest
Saving debug log to /etc/letsencrypt/logs/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/pass.leops.cn.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert not yet due for renewal

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

The following certs are not due for renewal yet:
  /etc/letsencrypt/live/pass.leops.cn/fullchain.pem expires on 2020-09-07 (skipped)
No renewals were attempted.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Creating bitwarden-mssql ... done
Creating bitwarden-admin ... done
Creating bitwarden-nginx ... done
Creating bitwarden-notifications ... 
Creating bitwarden-mssql ... 
Creating bitwarden-events ... 
Creating bitwarden-api ... 
Creating bitwarden-icons ... 
Creating bitwarden-attachments ... 
Creating bitwarden-web ... 
Creating bitwarden-admin ... 
Creating bitwarden-nginx ... 
1.34.0: Pulling from bitwarden/setup
Digest: sha256:e529c155e033bf3dbfb9f0a4e9c1b69eafd9becd3eba406d868173f431c01bca
Status: Image is up to date for bitwarden/setup:1.34.0
docker.io/bitwarden/setup:1.34.0


Bitwarden is up and running!
===================================================

visit https://pass.leops.cn
to update, run `./bitwarden.sh updateself` and then `./bitwarden.sh update`
```



#### 生成用户许可

```bash
# ./src/licenseGen/run.sh interactive
Interactive license mode...
What would you like to generate, a [u]ser license or an [o]rg license? # 选择用户或组织
Okay, we will generate a user license.
Please provide the user\'s guid — refer to the Readme for details on how to retrieve this. [GUID]: # 输入用户guid, 在admin后台中可以查看用户guid
Please provide the username this license will be registered to. [username]: # 输入用户名
Please provide the email address for the user lework. [email] # 输入用户邮箱
Confirm creation of "user" license for username: "test", email: "test@test.com", User-GUID: "1abc087b-4ea5-4c85-bf2a-abd50060bf2c"? Y/n
Y
{
  "LicenseKey": "8e03d072d28147d31234c9152a2c9bcf",
  "Id": "1abc087b-4ea5-4c85-bf2a-abd50060bf2c",
  "Name": "test",
  "Email": "test@test.com",
  "MaxStorageGb": 32767,
  "Premium": true,
  "Version": 1,
  "Issued": "2020-06-09T06:10:34.4706678Z",
  "Refresh": "2120-05-09T06:10:34.4707806Z",
  "Expires": "2120-06-09T06:10:34.4709099Z",
  "Trial": false,
  "Hash": "fKhM+HzFsbxdOMD123Bke/2m5jQLevITdIt2Xt6j7I=",
  "Signature": "i8SYtI+6WYUrMJq3Ek2SnM14RdlbH/69OhGzw5mkM/54l47Jc3KwKcDQf0SgF5VoxpNo+kIi9D6E8S1lrBNHp4xWYXRQHb3gkLbhzll+yavxOJbF1WE7GThfDKYpySnDrlK5Q6wDlVJv6tv2P+KeMI1tHnT3h6gQUU+afOXLvbU/znmGro4RZRj7KxZ+bWMq8Gopt+JM823znJBrzMol4BtZDXzXQKFmCau69nB33QAYjcJ/Jnx6aSKeM3UGfPMEgr0a57emBcqbwMgU5THOQ3kv0QK8BiQsCEAlx4Qaqr1lQKB1GFP043yMVwlm0ZwZF5EylwlCgt41Af2r0Rs222+3nI1ogkFLI9d9v1JyoOsZ8FxDHgsO6S2oZEcxAHEGyQ6FKw0SLIB3N2amFukK4xxW6yI+iFZYGGCtu7DF1w6HvXigglSZI1MUK18C1r4KLa123qfFtmq3dAN7foQywsrTxodyFdaHIPnaJH94m8UJdiQgH1iC1sVi8gzgBzY2aVFeywQRsO8qH8PZvdaBljvq0hjvOeRoLSYoRoZzhhLTxYzG6dmc1JSQZMvrkt+dL2GgO9JoY5efsWwDTLrWIs+WQaC6tq1mf41Rq2JQd9wB08pndL5WldH/bhM7eRRCFhKxbxE/vy8VeFhSsHkMj+kJOcCgiS5by4nN2iBY="
}
```

> 输入的json数据，就是用户许可，记得保存哦。

#### 生成组织许可

 ```bash
# ./src/licenseGen/run.sh interactive
Interactive license mode...
What would you like to generate, a [u]ser license or an [o]rg license? # 输入o
Okay, we will generate an organization license.
Please provide your Bitwarden Install-ID — refer to the Readme for details on how to retrieve this. [Install-ID]: # 输入安装id
Please enter a business name, default is BitBetter. [Business Name]: # 输入公司名称
Please provide the username this license will be registered to. [username]: # 输入组织名称
Please provide the email address for the user test. [email] # 输入组织用户邮箱
Confirm creation of "organization" license for business name: "牛逼组织", username: "test", email: "test@test.com", Install-ID: "1be9e030-1235-4125-873f-abd5122c278e"? Y/n
Y
{
  "LicenseKey": "d14119938cd84982898c4cc60a3e3822",
  "InstallationId": "1be9e030-6215-4ce5-873f-abd5002c278e",
  "Id": "51745f00-57ce-44d5-8c35-7099b6dc2e17",
  "Name": "运维",
  "BillingEmail": "test@test.com",
  "BusinessName": "牛逼科技",
  "Enabled": true,
  "Plan": "Custom",
  "PlanType": 6,
  "Seats": 32767,
  "MaxCollections": 32767,
  "UsePolicies": true,
  "UseGroups": true,
  "UseEvents": true,
  "UseDirectory": true,
  "UseTotp": true,
  "Use2fa": true,
  "MaxStorageGb": 32767,
  "SelfHost": true,
  "UsersGetPremium": true,
  "Version": 5,
  "Issued": "2020-06-09T07:28:22.2701843Z",
  "Refresh": "2120-05-09T07:28:22.2702408Z",
  "Expires": "2120-06-09T07:28:22.2703256Z",
  "Trial": false,
  "UseApi": true,
  "Hash": "hDskkkXMfesI9EI501234hzJ0v7pqPib4yaMkffsbE=",
  "Signature": "fEHGfHTsPrH5Qkhq/s6KAhJn+8swx23aJUh63lrDikkUY7eEIiDZzAFYI+Pyqk70NGZSGeyOB7mGVonjTD0H4Ga5dBbrq4NJbvoZuVJDjqNi6eiDew/CZkpaZ9H/xYVy2iNPoHUe7moUW4FQ9DjnH7SJWF9okn/RzbxlWrAWTdbtPOK4sm9aLZI6W6UE8IaL32/0RzhpoyGffq4COixRiREBWpbZ4/pS8PDllAcdn5qXFWXeeb1i7Fysnxy3JODdrkl7a2HB035anIfpIZlYVXJIvKEKkwiA234IWN5jSHvEr37XIzFBpu4p8PA7J6bsrlDgvu31XiP7EhxwrSmc7gAqbp2hJZzsGj/DbPfU+DVCfvhc8234nUpi1ybnasqgSR/XXwh/Vsu0hyTn51mE5sLTSrrO0TIIVAry9kNk4SgwqdBAM2e9FgIX9ULC2A4n62ErYatzCN3taRIo6K8hD06zDZS07s5V4YbgLf/uT1236ncCsnUltaE4b3krAerN3XWEyOZwoQ6JOoDOJ2bbpqoeWxUKqRzdr8trwGCtDMWDB0LFvnQ4gwkSC3zuOZaBXDUyO8NuNKn84b4QelVlUBoNWxW9bPmPmZaYwIUXXgkdDX/AhfL4RTJrtv+rZ+iDFn15s06ZE/yEoH80wZLdpozTNIrtU/6Kltl8="
}
 ```

> 输入的json数据，就是组织许可，记得保存哦。

#### 使用组织许可

在用户的设置页面中点击"组织",提交刚才申请的组织许可证

![bitwarden](/assets/images/bitwarden/09.png)

许可验证通过后，即可创建一个组织

![bitwarden](/assets/images/bitwarden/10.png)

#### 使用用户许可

在用户的设置页面中点击，升级高级会员

![bitwarden](/assets/images/bitwarden/11.png)

提交刚刚申请的用户许可， 许可验证成功后，会显示许可到期时间

![bitwarden](/assets/images/bitwarden/12.png)

 

### 备份数据

将 `./bwdata` 整个目录进行备份

`./bwdata`目录中一些特别重要的部分是：

- `./bwdata/mssql/data` - 数据库数据

- `./bwdata/core/attachments` - 附件

- `./bwdata/env` - 环境变量，包括数据库和证书密码

#### 数据库备份

 Bitwarden将自动对mssql容器数据库进行每晚备份。这些数据库备份保存在`./bwdata/mssql/backups`目录中。每晚数据库备份在此目录中保留30天。万一数据丢失，您可以还原这些日常备份之一。

**恢复每晚备份**

在bitwarden-mssql容器上执行交互式bash shell 。

 ```bash
docker exec -it bitwarden-mssql /bin/bash
 ```

在每晚备份目录中记录要还原的备份文件。备份目录从主机体积映射`./bwdata/mssql/backups`到`/etc/bitwarden/mssql/backups`所述内`bitwarden-mssql`容器。

 ```bash
ls /etc/bitwarden/mssql/backups
 ```

在此示例中，我们将使用的备份名为`vault_FULL_20200302_235901.BAK`，这是2020年3月2日晚上11:53的保管库数据库的备份。容器中备份的完整路径为`/etc/bitwarden/mssql/backups/vault_FULL_20200302_235901.BAK`

sqlcmd 使用所需的身份验证执行。

 ```bash
/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P ${SA_PASSWORD}
 ```

执行SQL命令RESTORE DATABASE with your backup以恢复夜间备份，然后执行一个GO命令。

 ```bash
RESTORE DATABASE vault FROM DISK = '/etc/bitwarden/mssql/backups/vault_FULL_20200302_235901.BAK' WITH REPLACE
GO
 ```

您的Bitwarden数据库现在应该已还原。

 

### 使用 RESTful API

1. 首先要获取组织的`client_secret`, 在 **组织页面**--**设置-**-**我的组织**--**API密钥**

   ![bitwarden](/assets/images/bitwarden/13.png)

   密钥信息如下:

    ```
    client_id:
    organization.d81aa71e-1234-4850-1234-abd5006931b7

    client_secret:
    U3LEacdQ123461234NxNIL7osTbaQb

    scope:
    api.organization

    grant_type:
    client_credentials
    ```
   
2. 获取token

   ```bash
   curl -X POST \
   https://pass.leops.cn/identity/connect/token \
   -H 'Content-Type: application/x-www-form-urlencoded' \
   -d 'grant_type=client_credentials&scope=api.organization&client_id=<ID>&client_secret=<SECRET>'
   ```

   返回值

   ```json
   {
   "access_token":"eyJhbGciOiJSUzI1NiIsImtpZCI6IkVF",
   "expires_in":3600,
   "token_type":"Bearer",
   "scope":"api.organization"
   }
   ```

3. 访问接口，获取组织集合

   ```bash
   curl -X GET \
   https://pass.leops.cn/api/public/collections \
   -H 'Authorization: Bearer <TOKEN>'
   ```

   返回值

   ```json
   {"object":"list","data":[{"object":"collection","id":"5fda1df6-c1c4-49f3-97e6-abd5006931bf","groups":null,"externalId":null},{"object":"collection","id":"9245b399-47c9-4a2b-ae30-abd5006f433e","groups":null,"externalId":null},{"object":"collection","id":"7b9ade4f-d637-4cd8-bcaa-abd5006f4a7a","groups":null,"externalId":null},{"object":"collection","id":"16ad09f5-4826-407a-8540-abd5006f51d9","groups":null,"externalId":null}],"continuationToken":null}
   ```

   

更多api接口信息可查看 [接口文档](https://pass.leops.cn/api/docs/index.html)

