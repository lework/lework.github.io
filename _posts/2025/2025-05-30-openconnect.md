---
layout: post
title: ' CentOS 上安装 OpenConnect VPN Server 详细教程'
date: '2025-05-30 20:00'
category: vpn
tags: vpn openconnect
author: lework
---

- content
  {:toc}

{% raw %}

OpenConnect VPN 服务器（OCSERV）是一款开源 Linux SSL VPN 服务器，专为需要具有企业级用户管理和控制的远程访问 VPN 而设计。它遵循 OpenConnect 协议，是 OpenConnect VPN 客户端的对应物。它也与 Cisco 的 AnyConnect SSL VPN 兼容。本文档将详细介绍如何在 CentOS 上安装和配置 OpenConnect VPN 服务器。

## 环境要求

- 操作系统：CentOS 7/8 或 RHEL 7/8
- 内存：至少 512MB
- 磁盘空间：至少 2GB 可用空间
- 网络：需要公网 IP 地址
- 权限：需要 root 权限进行安装和配置

## 前置准备

在开始安装之前，请确保系统已经更新到最新版本：

```bash
# 更新系统包
yum update -y

# 安装必要的依赖工具
yum install -y epel-release
yum install -y gnutls-utils iptables-services
```

## 第一步：安装 ocserv

OpenConnect VPN Server（ocserv）是一个开源的 VPN 服务器，兼容 Cisco AnyConnect 客户端。

```bash
# 安装 ocserv 软件包
yum install -y ocserv
```

> **注意**：如果安装失败，请确保已经启用了 EPEL 仓库。

## 第二步：配置 SSL 证书

SSL 证书用于加密客户端与服务器之间的通信。这里我们创建自签名证书（生产环境建议使用正式证书）。

> **说明**：如果您已有正式的 SSL 证书，可以跳过此步骤，直接在配置文件中指定证书路径。

```bash
# 创建证书存储目录
mkdir -p /etc/ocserv/certs
cd /etc/ocserv/certs

# 创建 CA 证书模板文件
cat << EOF > ca.tmpl
cn = "My CA"
organization = "My Org"
serial = 1
expiration_days = 3650
ca
signing_key
cert_signing_key
crl_signing_key
EOF

# 生成 CA 私钥
certtool --generate-privkey --outfile ca-key.pem

# 生成 CA 证书
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem

# 创建服务器证书模板（请将域名改为您的实际域名或IP）
cat << EOF > server.tmpl
cn = "vpn-ops.example.com"
organization = "My Org"
expiration_days = 3650
signing_key
encryption_key
tls_www_server
EOF

# 生成服务器私钥
certtool --generate-privkey --outfile server-key.pem

# 生成服务器证书
certtool --generate-certificate --load-privkey server-key.pem \
    --load-ca-certificate ca-cert.pem \
    --load-ca-privkey ca-key.pem \
    --template server.tmpl \
    --outfile server-cert.pem

# 设置证书文件权限
chmod 600 *.pem
chown ocserv:ocserv *.pem
```

> **重要提示**：请将 `cn = "vpn-ops.example.com"` 中的域名修改为您服务器的实际域名或公网 IP 地址。

## 第三步：配置 ocserv 主配置文件

创建并编辑 ocserv 的主配置文件：

```bash
# 备份原配置文件
cp /etc/ocserv/ocserv.conf /etc/ocserv/ocserv.conf.backup

# 创建新的配置文件
cat << EOF > /etc/ocserv/ocserv.conf
# 认证方式：使用本地密码文件
auth = "plain[passwd=/etc/ocserv/ocpasswd]"

# 监听端口
tcp-port = 443
udp-port = 443

# 运行用户和组
run-as-user = ocserv
run-as-group = ocserv

# Socket 文件位置
socket-file = /var/run/ocserv-socket
chroot-dir = /var/lib/ocserv

# SSL 证书配置
server-cert = /etc/ocserv/certs/server-cert.pem
server-key = /etc/ocserv/certs/server-key.pem

# 工作进程隔离
isolate-workers = true

# 连接限制
max-clients = 16          # 最大同时连接用户数
max-same-clients = 2      # 同一用户最大连接数

# 速率控制
rate-limit-ms = 100

# 统计重置时间（秒）
server-stats-reset-time = 604800

# 连接保持时间（秒）
keepalive = 32400

# 死连接检测时间（秒）
dpd = 90                  # 普通设备
mobile-dpd = 1800         # 移动设备

# TCP 切换超时
switch-to-tcp-timeout = 25

# MTU 发现
try-mtu-discovery = false

# 证书用户标识
cert-user-oid = 0.9.2342.19200300.100.1.1

# TLS 加密优先级
tls-priorities = "NORMAL:%SERVER_PRECEDENCE"

# 认证超时（秒）
auth-timeout = 240

# 重新认证最小间隔（秒）
min-reauth-time = 300

# 登录失败处理
max-ban-score = 80        # 最大失败次数
ban-reset-time = 1200     # 禁用重置时间（秒）

# Cookie 超时（秒）
cookie-timeout = 300

# 禁用漫游
deny-roaming = false

# 密钥重新协商
rekey-time = 172800       # 重新协商时间（秒）
rekey-method = ssl        # 重新协商方式

# 启用 occtl 管理工具
use-occtl = true

# PID 文件
pid-file = /var/run/ocserv.pid

# 日志级别（0-9，数字越大越详细）
log-level = 1

# 虚拟网络设备
device = vpns

# 可预测的 IP 分配
predictable-ips = true

# 默认域名
default-domain = vpn-ops.example.com

# VPN 客户端 IP 地址池（请确保与现有网络不冲突）
ipv4-network = 192.168.100.0/24

# DNS 设置
tunnel-all-dns = true
dns = 233.5.5.5             # 公共 DNS
dns = 233.6.6.6             # 备用 DNS

# IP 租约检测
ping-leases = false

# 路由配置
route = 10.0.0.0/8        # 允许访问的内网段
# no-route = 192.168.5.0/255.255.255.0  # 禁止访问的网段（可选）

# 客户端兼容性
cisco-client-compat = true
dtls-legacy = true
cisco-svc-client-compat = false
client-bypass-protocol = false

# 流量伪装（可选，用于绕过网络限制）
camouflage = false
camouflage_secret = "mysecretkey"
camouflage_realm = "Restricted Content"
EOF
```

## 第四步：配置参数详细说明

以下是主要配置参数的详细说明：

| 参数                | 说明                                 | 推荐值                               |
| ------------------- | ------------------------------------ | ------------------------------------ |
| `auth`              | 认证方式，plain 表示使用密码文件认证 | `plain[passwd=/etc/ocserv/ocpasswd]` |
| `tcp-port/udp-port` | 监听端口，443 是标准 HTTPS 端口      | `443`                                |
| `max-clients`       | 最大同时连接数，根据服务器性能调整   | `16-64`                              |
| `max-same-clients`  | 同一用户最大连接数，防止滥用         | `2-4`                                |
| `keepalive`         | 连接保持时间，单位秒                 | `32400`                              |
| `dpd`               | 死连接检测时间，单位秒               | `90`                                 |
| `ipv4-network`      | VPN 客户端 IP 池，避免与现有网络冲突 | `192.168.100.0/24`                   |
| `dns`               | DNS 服务器地址                       | `233.5.5.5`                          |
| `route`             | 允许访问的网段                       | `10.0.0.0/8`                         |

## 第五步：创建 VPN 用户账号

使用 `ocpasswd` 命令创建 VPN 用户：

```bash
# 创建第一个用户（会提示输入密码）
ocpasswd -c /etc/ocserv/ocpasswd username1

# 添加更多用户（-c 参数只在第一次创建文件时使用）
ocpasswd /etc/ocserv/ocpasswd username2

# 查看已创建的用户
cat /etc/ocserv/ocpasswd
```

> **安全提示**：密码建议使用强密码，包含大小写字母、数字和特殊字符。

**用户管理常用命令：**

```bash
# 删除用户
ocpasswd -d /etc/ocserv/ocpasswd username1

# 修改用户密码
ocpasswd /etc/ocserv/ocpasswd username1

# 锁定用户账号
ocpasswd -l /etc/ocserv/ocpasswd username1

# 解锁用户账号
ocpasswd -u /etc/ocserv/ocpasswd username1
```

## 第六步：配置系统内核参数

启用 IP 转发和优化网络性能：

```bash
# 创建系统配置文件
cat << EOF > /etc/sysctl.d/99-ocserv.conf
# 启用 IP 转发（必需）
net.ipv4.ip_forward=1

# 启用 BBR 拥塞控制算法（可选，提升性能）
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

# 优化网络缓冲区
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# 禁用源路由验证（避免某些网络问题）
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
EOF

# 应用配置
sysctl -p /etc/sysctl.d/99-ocserv.conf

# 验证配置是否生效
sysctl net.ipv4.ip_forward
```

## 第七步：配置防火墙规则

### 使用 iptables（CentOS 7）

```bash
# 安装并启用 iptables 服务
systemctl stop firewalld
systemctl disable firewalld
yum install -y iptables-services
systemctl enable iptables

# 配置 iptables 规则
# 设置 MSS 钳制，避免 MTU 问题
iptables -I FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# 允许 VPN 端口
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p udp --dport 443 -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT

# 启用 NAT 转发
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -j MASQUERADE

# 保存规则
iptables-save > /etc/sysconfig/iptables

# 启动 iptables 服务
systemctl start iptables
systemctl enable iptables
```

### 使用 firewalld（CentOS 8）

```bash
# 启用 firewalld 服务
systemctl start firewalld
systemctl enable firewalld

# 开放 VPN 端口
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=443/udp

# 启用 NAT 转发
firewall-cmd --permanent --add-masquerade

# 添加 VPN 网段到信任区域
firewall-cmd --permanent --zone=trusted --add-source=192.168.100.0/24

# 重新加载防火墙规则
firewall-cmd --reload

# 查看当前规则
firewall-cmd --list-all
```

## 第八步：启动 ocserv 服务

```bash
# 启动 ocserv 服务
systemctl start ocserv

# 设置开机自启动
systemctl enable ocserv

# 检查服务状态
systemctl status ocserv

# 查看服务日志
journalctl -u ocserv -f
```

## 第九步：验证和测试

### 检查服务是否正常运行

```bash
# 检查进程是否存在
ps aux | grep ocserv

# 检查端口是否监听
netstat -tulpn | grep :443

# 或使用 ss 命令
ss -tulpn | grep :443

# 查看详细日志
tail -f /var/log/messages | grep ocserv
```

### 测试连接

```bash
# 使用 openconnect 客户端测试（如果安装了的话）
echo "your_password" | openconnect -u your_username --passwd-on-stdin https://your_server_ip:443

# 检查当前连接的用户
occtl show users
```

## 客户端下载和配置

### 官方客户端下载

- **Cisco AnyConnect**：

  - 官方下载（需要 Cisco 账号）：https://software.cisco.com/download/home/286330811/type/282364313/release/5.1.9.113
  - 备用下载链接：https://pan.quark.cn/s/5e298cead8dc

- **开源客户端 OpenConnect**：
  - Windows：https://gitlab.com/openconnect/openconnect-gui/releases
  - macOS：`brew install openconnect`
  - Linux：`yum install openconnect` 或 `apt install openconnect`

### 客户端连接配置

**连接参数：**

- 服务器地址：`https://your_server_ip:443` 或 `https://your_domain:443`
- 用户名：之前创建的用户名
- 密码：对应的密码

**Android/iOS 配置：**

1. 下载 Cisco AnyConnect 应用
2. 添加新连接
3. 输入服务器地址和认证信息
4. 连接并测试

## 故障排除

### 常见问题及解决方案

1. **无法连接到服务器**

   ```bash
   # 检查防火墙规则
   iptables -L -n | grep 443

   # 检查 SELinux 状态
   getenforce
   # 如果是 Enforcing，可以临时禁用测试
   setenforce 0
   ```

2. **连接成功但无法上网**

   ```bash
   # 检查 IP 转发是否启用
   cat /proc/sys/net/ipv4/ip_forward

   # 检查 NAT 规则
   iptables -t nat -L -n
   ```

3. **证书错误**

   ```bash
   # 重新生成证书，确保 CN 字段与服务器IP/域名匹配
   # 或在客户端选择"忽略证书错误"
   ```

4. **用户认证失败**

   ```bash
   # 检查密码文件是否存在
   ls -la /etc/ocserv/ocpasswd

   # 重新创建用户
   ocpasswd -c /etc/ocserv/ocpasswd username
   ```

### 日志分析

```bash
# 查看详细日志
tail -f /var/log/messages | grep ocserv

# 或查看 systemd 日志
journalctl -u ocserv -f

# 增加日志详细程度（修改配置文件中的 log-level）
# 0: 只记录错误
# 1: 记录警告
# 3: 记录信息
# 9: 记录所有调试信息
```

### 性能优化建议

1. **调整连接数限制**：根据服务器性能和带宽调整 `max-clients`
2. **优化加密算法**：在 `tls-priorities` 中使用更快的加密算法
3. **启用压缩**：在配置中添加 `compression = true`
4. **调整 MTU**：根据网络环境调整 MTU 大小

## 安全建议

1. **定期更新证书**：建议使用 Let's Encrypt 等免费证书服务
2. **强化用户认证**：使用复杂密码或考虑集成 LDAP/AD
3. **监控连接日志**：定期检查异常连接和登录失败记录
4. **网络分段**：合理配置路由规则，限制 VPN 用户的访问范围
5. **定期备份配置**：备份重要的配置文件和用户数据

## 相关资源

- **官方文档**：https://ocserv.openconnect-vpn.net/ocserv.8.html
- **OpenConnect 项目**：https://www.infradead.org/openconnect/

---

> **注意**：本教程适用于测试和学习环境。在生产环境中部署时，请务必考虑安全性、性能和合规性要求。

{% endraw %}
