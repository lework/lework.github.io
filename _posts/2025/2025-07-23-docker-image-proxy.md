---
layout: post
title: 'Docker 镜像代理加速器配置指南'
date: '2025-07-23 20:00'
category: docker
tags: docker
author: lework
---
* content
{:toc}

{% raw %}

> 本文所有网址均来自网络，如有侵权，请联系删除。

Docker 镜像代理是为了解决国内用户访问 Docker Hub 等境外镜像仓库速度慢的问题而设立的代理服务。通过配置镜像加速器，可以大幅提升 Docker 镜像的下载速度。





## 一键配置 Docker 镜像加速器

如果您想快速配置，可以使用以下一键脚本：

```bash
bash <(curl -sSL https://ghfast.top/https://raw.githubusercontent.com/lework/script/refs/heads/master/shell/test/check_docker_registry.sh)
```

## 手动配置方法

### Linux 系统配置

在 Linux 系统上，您需要创建或编辑 Docker 的配置文件：

```bash
# 创建 Docker 配置目录
sudo mkdir -p /etc/docker

# 创建 daemon.json 配置文件
sudo tee /etc/docker/daemon.json <<-'EOF'
{
   "registry-mirrors": [
    "https://docker-registry.nmqu.com",
    "https://docker.1ms.run",
    "https://docker.1panel.live",
    "https://docker.1panel.top",
    "https://docker.actima.top",
    "https://docker.aityp.com",
    "https://docker.hlmirror.com",
    "https://docker.kejilion.pro",
    "https://docker.m.daocloud.io",
    "https://docker.melikeme.cn",
    "https://docker.tbedu.top",
    "https://docker.xuanyuan.me",
    "https://dockercf.jsdelivr.fyi",
    "https://dockerhub.xisoul.cn",
    "https://dockerproxy.net",
    "https://dockerpull.pw",
    "https://doublezonline.cloud",
    "https://hub.fast360.xyz",
    "https://hub.rat.dev",
    "https://hub.xdark.top",
    "https://image.cloudlayer.icu",
    "https://lispy.org"
  ]
}
EOF
```

配置完成后，重启 Docker 服务使配置生效：

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 重启 Docker 服务
sudo systemctl restart docker
```

### macOS 系统配置（Docker Desktop）

在 macOS 系统上使用 Docker Desktop 的配置步骤：

**第一步：** 点击菜单栏中的 Docker Desktop 图标

**第二步：** 选择 "Settings..." 或 "偏好设置..."

**第三步：** 在左侧菜单中选择 "Docker Engine"

**第四步：** 在 JSON 配置编辑器中添加以下镜像源配置：

```json
{
  "registry-mirrors": [
    "https://docker-registry.nmqu.com",
    "https://docker.1ms.run",
    "https://docker.1panel.live",
    "https://docker.1panel.top",
    "https://docker.actima.top",
    "https://docker.aityp.com",
    "https://docker.hlmirror.com",
    "https://docker.kejilion.pro",
    "https://docker.m.daocloud.io",
    "https://docker.melikeme.cn",
    "https://docker.tbedu.top",
    "https://docker.xuanyuan.me",
    "https://dockercf.jsdelivr.fyi",
    "https://dockerhub.xisoul.cn",
    "https://dockerproxy.net",
    "https://dockerpull.pw",
    "https://doublezonline.cloud",
    "https://hub.fast360.xyz",
    "https://hub.rat.dev",
    "https://hub.xdark.top",
    "https://image.cloudlayer.icu",
    "https://lispy.org"
  ]
}
```

**第五步：** 点击 "Apply & Restart" 按钮应用配置并重启 Docker

### Windows 系统配置（Docker Desktop）

在 Windows 系统上使用 Docker Desktop 的配置步骤：

**第一步：** 右键点击系统托盘中的 Docker Desktop 图标

**第二步：** 选择 "Settings" 或 "设置"

**第三步：** 在左侧菜单中选择 "Docker Engine"

**第四步：** 在 JSON 配置中添加以下镜像源配置：

```json
{
  "registry-mirrors": [
    "https://docker-registry.nmqu.com",
    "https://docker.1ms.run",
    "https://docker.1panel.live",
    "https://docker.1panel.top",
    "https://docker.actima.top",
    "https://docker.aityp.com",
    "https://docker.hlmirror.com",
    "https://docker.kejilion.pro",
    "https://docker.m.daocloud.io",
    "https://docker.melikeme.cn",
    "https://docker.tbedu.top",
    "https://docker.xuanyuan.me",
    "https://dockercf.jsdelivr.fyi",
    "https://dockerhub.xisoul.cn",
    "https://dockerproxy.net",
    "https://dockerpull.pw",
    "https://doublezonline.cloud",
    "https://hub.fast360.xyz",
    "https://hub.rat.dev",
    "https://hub.xdark.top",
    "https://image.cloudlayer.icu",
    "https://lispy.org"
  ]
}
```

**第五步：** 点击 "Apply & Restart" 按钮应用配置并重启 Docker

## 可用的 Docker 镜像加速器列表

> 以下是目前测试可用的镜像代理站点：

| 状态 | 镜像代理地址                     |
| ---- | -------------------------------- |
| ✅   | https://docker-registry.nmqu.com |
| ✅   | https://docker.1ms.run           |
| ✅   | https://docker.1panel.live       |
| ✅   | https://docker.1panel.top        |
| ✅   | https://docker.actima.top        |
| ✅   | https://docker.aityp.com         |
| ✅   | https://docker.hlmirror.com      |
| ✅   | https://docker.kejilion.pro      |
| ✅   | https://docker.m.daocloud.io     |
| ✅   | https://docker.melikeme.cn       |
| ✅   | https://docker.tbedu.top         |
| ✅   | https://docker.xuanyuan.me       |
| ✅   | https://dockercf.jsdelivr.fyi    |
| ✅   | https://dockerhub.xisoul.cn      |
| ✅   | https://dockerproxy.net          |
| ✅   | https://dockerpull.pw            |
| ✅   | https://doublezonline.cloud      |
| ✅   | https://hub.fast360.xyz          |
| ✅   | https://hub.xdark.top            |
| ✅   | https://image.cloudlayer.icu     |
| ✅   | https://lispy.org                |
| ✅   | https://registry.cyou            |
| ✅   | https://docker.ameke.cn          |
| ✅   | https://docker.aeko.cn           |
| ✅   | https://docker-mirror.aigc2d.com |

## 其他实用工具

### 镜像源测试脚本

您可以使用以下脚本来测试和选择最快的镜像源：

```bash
# 国内网络环境执行
bash <(curl -sSL https://gitee.com/xjxjin/scripts/raw/main/check_docker_registry.sh)

# 国际网络环境执行
bash <(curl -sSL https://github.com/xjxjin/scripts/raw/main/check_docker_registry.sh)
```

### 镜像站点状态监控

以下网站可以帮助您实时查看各个镜像站点的可用状态：

- https://mirror.kentxxq.com/image
- https://status.anye.xyz/
- https://pull.whsir.com

{% endraw %}
