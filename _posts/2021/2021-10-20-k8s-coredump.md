---
layout: post
title: '统一收集 kubernetes worker 节点上所产生的 coredump 文件并报警'
date: '2021-10-20 20:02'
category: kubernetes
tags: kubernetes coredump
author: lework
mermaid: true
---
* content
{:toc}

## 需求

统一收集 kubernetes worker 节点上所产生的 coredump 文件并报警。




## 可能遇到的问题

1. 进程大内存 => coredump  文件很大（100G +）

   coredump 进行时：对宿主机本地磁盘 IO 造成冲击。

   coredump 完成后：对宿主机磁盘容量造成冲击。

   ```bash
   # 通过 gzip 压缩来减少磁盘压力
   
   gzip > "/tmp/${coredump_file}"
   ```

2. 大型 coredump  文件累积产生会占用大量磁盘空间，我们应该何时清理 ? 何种策略清理 ?

   ```bash
   # 在 coredump 事件爆发后，清除60分钟外的文件
   find "${coredump_path:-/tmp/123}" -type f -mtime +60 -exec rm -rf {} \;
   ```

3. 同一应用实例频繁 coredump 怎么办？

   假设触发同一批 coredump 的原因是相同，理论上只需要留下一个 core 文件用以后续排查即可。这里我们可以加个锁机制，来处理多实例同时触发coredump的问题。

   ```bash
   # 加锁，避免多实例重复生成coredump, 锁时间5分钟。
   find "${coredump_path}" -type f -mmin +5 -name "${lock_file}" -exec rm -fv {} \;
   [ -f "${lock_file}"] && {log "lock file exits: ${lock_file}"; exit 0}
   touch "${lock_file}"
   ```

4. 如何调式 core 文件 ？
   
   **C++:** 想要调试 core 文件，需要：
   1. gdb 工具；
   2. 产生 core 文件的进程；
3. core 文件。
   
   **Java:** 专门的 core 文件分析工具和 core 文件即可。
   
5. 怎么获取容器内的进程文件？

   ```bash
   # 通过查找 /var/lib/docker 目录下的进程文件名称，这种方式对进程名称有要求，有些局限性。
   find /var/lib/docker -type f -executable -size +1M -name "${exe_name:-ABC123456}*" -exec cp -fv {} /tmp/ \;
   ```

   但是此方法会受pod删除时间的影响，pod删除比较快的时候，是获取不到的。这时我们需要修改容器的 

   `entrypoint.sh` 来接管 `pid 1` 从而延迟 pod 的删除。

   ```bash
   #!/bin/sh
   
   
   trap stop TERM INT QUIT HUP ERR
   
   app=${1}
   
   start(){
      $app
   }
   
   stop(){
     echo "$(date +'%F %T,%3N') Stopping..."
     
     kill -9 $(ps aux | grep $app | grep -v grep | awk '{print $2}')
   }
   
   start
   
   ```

   

## 实现工作机制

![feishu-coredump.png](/assets/images/kubernetes/coredump-flow.png)

## 具体操作

> 下面是在 k8s 节点上进行操作

### 配置 obs 的 fuse 挂载

> 这里我使用的是 华为云的 obs 用作 coredump 文件的公共存储。

```bash
yum install -y openssl-devel fuse fuse-devel
wget https://obs-community.obs.cn-north-1.myhuaweicloud.com/obsfs/current/obsfs_CentOS7.6_amd64.tar.gz
tar zxf obsfs_CentOS7.6_amd64.tar.gz

cd obsfs_CentOS7.6_amd64/
bash install_obsfs.sh 

echo xxx:xxxx > /etc/passwd-obsfs # obs 认证信息
chmod 600 /etc/passwd-obsfs

mkdir /var/coredump

/usr/local/bin/obsfs coredump /var/coredump -o url=obs.cn-east-3.myhuaweicloud.com -o passwd_file=/etc/passwd-obsfs -o big_writes -o max_write=131072 -o allow_other -o use_ino

echo "/usr/local/bin/obsfs coredump /var/coredump -o url=obs.cn-east-3.myhuaweicloud.com -o passwd_file=/etc/passwd-obsfs -o big_writes -o max_write=131072 -o allow_other -o use_ino" >> /etc/rc.d/rc.local

chmod +x /etc/rc.d/rc.local
```

### 配置 coredump 的内核参数

```bash
cat <<EOF > /etc/sysctl.d/coredump.conf 
kernel.core_pattern=|/usr/bin/coredump_helper.sh core_%h_exe_%e_tid_%I_pid_%p_sig_%s_time_%t.gz
kernel.core_uses_pid=1
EOF
sysctl --system

cp /etc/security/limits.conf  /etc/security/limits.conf.bak
cat << EOF >> /etc/security/limits.conf
root soft nofile 165535
root hard nofile 165535
root soft nproc 165535
root hard nproc 165535
root soft core unlimited
root hard core unlimited

* soft nofile 165535
* hard nofile 165535
* soft nproc 165535
* hard nproc 165535
* soft core unlimited
* hard core unlimited
EOF

sed -i 's#4096#165535#g' /etc/security/limits.d/20-nproc.conf

echo "DefaultLimitCORE=infinity"  >> /etc/systemd/system.conf
echo "DefaultLimitNOFILE=165535" >> /etc/systemd/system.conf
echo "DefaultLimitNPROC=165535" >>/etc/systemd/system.conf

sed -i "s#LimitCORE=.*#LimitCORE=unlimited#g" /usr/lib/systemd/system/docker.service

systemctl daemon-reload
systemctl restart docker
```

### 配置 coredump 的处理脚本

file: `/usr/bin/coredump_helper.sh`

```bash
#!/bin/bash

function log {
    echo "[$(date --rfc-3339=seconds)]: $*" >> "/tmp/coredump_$(date +"%Y-%m").log"
}

coredump_file="${1}"
coredump_path="/var/coredump"
host_ip="$(/usr/sbin/ip -4 route get 8.8.8.8 | head -1 | awk '{print $7}')"
pod_name="$(echo ${coredump_file} | awk -F'_' '{print $2}')"
exe_name="$(echo ${coredump_file} | awk -F'_exe_|_tid_' '{print $2}')"
file_time="$(date +'%Y-%m-%d %T' -d @$(echo ${coredump_file} | awk -F'_time_|.gz' '{print $2}'))"
file_url="https://xxx.com/" # coredump 文件的下载路径
notify_url="https://open.feishu.cn/open-apis/bot/v2/hook/XXXX"  # 飞书的通知API
lock_file="${coredump_path}/${exe_name}:-ABC123456.lock"

log "start."
log "host_ip: ${host_ip}"
log "pod_name: ${pod_name}"
log "exe_name: ${exe_name}"
log "file_time: ${file_time}"

[ ! -d "${coredump_path}" ] &&  mkdir -p "${coredump_path}"

# 加锁，避免多实例重复生成coredump
find "${coredump_path}" -type f -mmin +5 -name "${lock_file}" -exec rm -fv {} \;
[ -f "${lock_file}"] && {log "lock file exits: ${lock_file}"; exit 0}
touch "${lock_file}"

gzip > "/tmp/${coredump_file}" && log "coredump: /tmp/${coredump_file}"


if [ -f "/tmp/${coredump_file}" ];then

  find /var/lib/docker -type f -executable -size +1M -name "${exe_name:-ABC123456}*" -exec cp -fv {} /tmp/ \;

  tar zcvf "/tmp/${coredump_file}.tgz" -C /tmp "${coredump_file}" $(find /tmp/ -type f -executable -size +1k -name "${exe_name:-ABC123456}*")
  log "compressed file: /tmp/${coredump_file}.tgz"

  rm -fv "/tmp/${coredump_file}" "${coredump_file}" $(find /tmp/ -type f -executable -size +1k -name "${exe_name:-ABC123456}*")

  core_size="$(du -sh /tmp/${coredump_file}.tgz 2>/dev/null | awk '{print $1}')"
  log "core_size: ${core_size}"

  mv -fv "/tmp/${coredump_file}.tgz" "${coredump_path}/${coredump_file}.tgz"

  curl -X POST -H "Content-Type: application/json" \
     -d "{\"msg_type\":\"interactive\",\"card\":{\"config\":{\"wide_screen_mode\":true,\"enable_forward\":true},\"elements\":[{\"tag\":\"div\",\"text\":{\"content\":\"**Time:** ${file_time}\\n**Host:** ${host_ip}\\n**POD:** ${pod_name}\\n**EXE:** ${exe_name}\\n**Size:** ${core_size}\",\"tag\":\"lark_md\"}},{\"actions\":[{\"tag\":\"button\",\"text\":{\"content\":\"点我下载 coredump 文件\",\"tag\":\"lark_md\"},\"url\":\"${file_url}${coredump_file}.tgz\",\"type\":\"default\",\"value\":{}}],\"tag\":\"action\"}],\"header\":{\"template\":\"turquoise\",\"title\":{\"content\":\"coredump 文件生成\",\"tag\":\"plain_text\"}}}}" \
     ${notify_url}
  log "success."
else
  curl -X POST -H "Content-Type: application/json" \
     -d "{\"msg_type\":\"interactive\",\"card\":{\"config\":{\"wide_screen_mode\":true,\"enable_forward\":true},\"elements\":[{\"tag\":\"div\",\"text\":{\"content\":\"**Host:** ${host_ip}\\n**POD:** ${pod_name}\\n**File:** 未生成文件\\n**Size:** ${core_size}\",\"tag\":\"lark_md\"}}],\"header\":{\"template\": \"red\",\"title\":{\"content\":\"coredump 文件生成\",\"tag\":\"plain_text\"}}}}" \
     ${notify_url}
  log "error."
fi

find "${coredump_path:-/tmp/123}" -type f -mtime +1 -exec rm -rf {} \;
log "done."
```

```bash
chmod +x /usr/bin/coredump_helper.sh
```



测试代码

```bash
docker run -ti --rm  centos bash
yum install -y gcc 

cat <<EOF > a.c
#include<stdio.h>
int main(){
  char *p = NULL;
  *p = 0;
  return 0;
}
EOF

gcc a.c
./a.out
```



报警效果

![feishu-coredump.png](/assets/images/kubernetes/feishu-coredump.png)



## 参考

- https://blog.ppistechnology.top/2021/08/10/大型有状态服务基于-k8s-的落地实践（十五）：coredump1：/