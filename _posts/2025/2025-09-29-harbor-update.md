---
layout: post
title: 'Harbor 2.5.0 升级到 2.13.2'
date: '2025-09-29 20:00'
category: docker
tags: docker harbor
author: lework
---

- content
  {:toc}

{% raw %}

### 确定升级版本路径

根据 https://goharbor.io/docs/2.13.0/administration/upgrade/ 升级文档，我们可以确定升级到目标版本所需的最低版本要求。

- 升级到 2.13.0 版本，需要最低版本 2.11.0
- 升级到 2.11.0 版本，需要最低版本 2.9.0
- 升级到 2.9.0 版本，需要最低版本 2.7.0
- 升级到 2.7.0 版本，需要最低版本 2.5.0

**版本重要变动：**

- 2.8.0 版本：取消了 chartmuseum 的支持
- 2.9.0 版本：取消了 notary 和 chartmuseum 的支持，PostgreSQL 升级到 14 版本
- 2.11.0 版本：PostgreSQL 升级到 15 版本

**数据库重要变动：**

在 https://github.com/goharbor/harbor/blob/main/make/migrations/postgresql/ 目录中，可以查看每个版本升级时的数据库迁移脚本。

其中 `0120_2.9.0_schema.up.sql` 的变动最大，涉及多个表的数据迁移。

```sql
select count(*) from report_vulnerability_record;
select count(*) from vulnerability_record;
select count(*) from scan_report
```

如果这几张表的数据非常大，可以考虑只保留最近 3 个月的数据，然后再进行迁移，这样可以减少迁移时间。

**最终升级路径：**
2.5.0（当前版本）→ 2.7.0 → 2.9.0 → 2.11.0 → 2.13.2（目标版本）

**当前版本目录结构：**

```bash
/data/harbor/
├── cert/               # 证书文件目录
├── common/             # 公共文件目录
├── common.sh           # 公共脚本
├── docker-compose.yml  # 容器编排文件
├── harbor_data/        # Harbor 数据文件目录
├── harbor.yml          # 配置文件
├── harbor.yml.tmpl     # 配置文件模板
├── install.sh          # 安装脚本
├── LICENSE             # 许可证文件
└── prepare             # 准备脚本
```

### 升级到 2.7.0

#### 1. 下载离线包

```bash
cd /opt/harbor-upgrade
wget https://github.com/goharbor/harbor/releases/download/v2.7.0/harbor-offline-installer-v2.7.0.tgz
mkdir /data/harbor-v2.7.0
tar zxf harbor-offline-installer-v2.7.0.tgz -C /data/harbor-v2.7.0 --strip-components 1

docker load -i  /data/harbor-v2.7.0/harbor.v2.7.0.tar.gz
```

#### 2. 停止 Harbor 服务

```bash
cd /data/harbor
docker-compose down
```

#### 3. 备份 Harbor 数据

```bash
cp -rf /data/harbor /data/harbor-v2.5.0_backup
```

#### 4. 迁移 Harbor 配置文件

```bash
# 迁移配置文件
cp -fv /data/harbor/harbor.yml /data/harbor-v2.7.0/harbor.yml
docker run -it --rm -v /:/hostfs goharbor/prepare:v2.7.0 migrate -i /data/harbor-v2.7.0/harbor.yml

# 对比两个配置文件的差异
diff /data/harbor/harbor.yml /data/harbor-v2.7.0/harbor.yml
```

#### 5. 安装 Harbor

```bash
cd /data/harbor-v2.7.0
./install.sh --with-trivy --with-notary --with-chartmuseum

```

#### 6. 验证 Harbor 服务

```bash
docker-compose ps

docker push harbor.xxx.com/library/redis:7
docker pull harbor.xxx.com/library/redis:7
```

### 升级到 2.9.0

#### 1. 下载离线包

```bash
cd /opt/harbor-upgrade
wget https://github.com/goharbor/harbor/releases/download/v2.9.0/harbor-offline-installer-v2.9.0.tgz
mkdir /data/harbor-v2.9.0
tar zxf harbor-offline-installer-v2.9.0.tgz -C /data/harbor-v2.9.0 --strip-components 1

docker load -i /data/harbor-v2.9.0/harbor.v2.9.0.tar.gz
```

#### 2. 停止 Harbor 服务

```bash
cd /data/harbor
docker-compose down
```

#### 3. 备份 Harbor 数据

```bash
cp -rf /data/harbor /data/harbor-v2.7.0_backup
```

#### 4. 迁移 Harbor 配置文件

```bash
# 迁移配置文件
cp -fv /data/harbor-v2.7.0/harbor.yml /data/harbor-v2.9.0/harbor.yml
docker run -it --rm -v /:/hostfs goharbor/prepare:v2.9.0 migrate -i /data/harbor-v2.9.0/harbor.yml

# 对比两个配置文件的差异
diff /data/harbor-v2.7.0/harbor.yml /data/harbor-v2.9.0/harbor.yml
```

#### 5. 安装 Harbor

> **注意：** 2.9.0 版本不支持 notary 和 chartmuseum，PostgreSQL 版本升级到了 14。

```bash
cd /data/harbor-v2.9.0
./install.sh --with-trivy

# 通过 diff 来查看 PostgreSQL 13 和 14 的存储差异，判断数据是否迁移成功
du -sh /data/harbor/harbor_data/database/*
```

> **注意：** 这一步因为需要迁移数据库，所以耗时较长。

#### 6. 验证 Harbor 服务

```bash
docker-compose ps

docker push harbor.xxx.com/library/redis:7
docker pull harbor.xxx.com/library/redis:7
```

### 升级到 2.11.0

#### 1. 下载离线包

```bash
cd /opt/harbor-upgrade
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-offline-installer-v2.11.0.tgz
mkdir /data/harbor-v2.11.0
tar zxf harbor-offline-installer-v2.11.0.tgz -C /data/harbor-v2.11.0 --strip-components 1
docker load -i /data/harbor-v2.11.0/harbor.v2.11.0.tar.gz
```

#### 2. 停止 Harbor 服务

```bash
cd /data/harbor
docker-compose down
```

#### 3. 备份 Harbor 数据

```bash
cp -rf /data/harbor /data/harbor-v2.9.0_backup
```

#### 4. 迁移 Harbor 配置文件

```bash
# 迁移配置文件
cp -fv /data/harbor-v2.9.0/harbor.yml /data/harbor-v2.11.0/harbor.yml
docker run -it --rm -v /:/hostfs goharbor/prepare:v2.11.0 migrate -i /data/harbor-v2.11.0/harbor.yml

# 对比两个配置文件的差异
diff /data/harbor-v2.9.0/harbor.yml /data/harbor-v2.11.0/harbor.yml
```

#### 5. 安装 Harbor

```bash
cd /data/harbor-v2.11.0
./install.sh --with-trivy

# 通过 diff 来查看 PostgreSQL 14 和 15 的存储差异，判断数据是否迁移成功
du -sh /data/harbor/harbor_data/database/*
```

> **注意：** 这一步因为需要迁移数据库，所以耗时较长。

#### 6. 验证 Harbor 服务

```bash
docker-compose ps

docker push harbor.xxx.com/library/redis:7
docker pull harbor.xxx.com/library/redis:7
```

### 升级到 2.13.2

#### 1. 下载离线包

```bash
cd /opt/harbor-upgrade
wget https://github.com/goharbor/harbor/releases/download/v2.13.2/harbor-offline-installer-v2.13.2.tgz
mkdir /data/harbor-v2.13.2
tar zxf harbor-offline-installer-v2.13.2.tgz -C /data/harbor-v2.13.2 --strip-components 1
docker load -i /data/harbor-v2.13.2/harbor.v2.13.2.tar.gz
```

#### 2. 停止 Harbor 服务

```bash
cd /data/harbor
docker-compose down
```

#### 3. 备份 Harbor 数据

```bash
cp -rf /data/harbor /data/harbor-v2.11.0_backup
```

#### 4. 迁移 Harbor 配置文件

```bash
# 迁移配置文件
cp -fv /data/harbor-v2.11.0/harbor.yml /data/harbor-v2.13.2/harbor.yml
docker run -it --rm -v /:/hostfs goharbor/prepare:v2.13.2 migrate -i /data/harbor-v2.13.2/harbor.yml

# 对比两个配置文件的差异
diff /data/harbor-v2.11.0/harbor.yml /data/harbor-v2.13.2/harbor.yml
```

#### 5. 安装 Harbor

```bash
cd /data/harbor-v2.13.2
./install.sh --with-trivy
```

#### 6. 验证 Harbor 服务

```bash
docker-compose ps

docker push harbor.xxx.com/library/redis:7
docker pull harbor.xxx.com/library/redis:7
```

### 清理历史版本

> **注意：** 清理历史版本前，请先确保 Harbor 服务已经正常运行。

```bash
# 清理升级过程中的临时文件
rm -rf /opt/harbor-upgrade
rm -rf /data/harbor-v2.5.0*
rm -rf /data/harbor-v2.7.0*
rm -rf /data/harbor-v2.9.0*
rm -rf /data/harbor-v2.11.0*

# 清理历史版本的 Docker 镜像
docker rmi $(docker images | grep v2.5.0 | awk '{print $3}')
docker rmi $(docker images | grep v2.7.0 | awk '{print $3}')
docker rmi $(docker images | grep v2.9.0 | awk '{print $3}')
docker rmi $(docker images | grep v2.11.0 | awk '{print $3}')
```

{% endraw %}
