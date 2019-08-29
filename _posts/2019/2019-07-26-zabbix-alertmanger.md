---
layout: post
title: "使用AlertManager为Zabbix做报警分组和沉默"
date: "2019-07-26 9:20"
category: monitor
tags: monitor zabbix alertmanger
author: lework
---
* content
{:toc}

因为[Zabbix]([http://www.zabbix.com](http://www.zabbix.com/))没有报警分组和沉默的功能，有时候大规模的产生报警，就会有很多的报警邮件或短信通知，导致了一些不必要的浪费，而[Prometheus](https://prometheus.io/)套件中的[AlertManager](https://prometheus.io/docs/alerting/alertmanager/)符合我们的诉求，可以将报警分组聚合进行发送，并且可以在一段时间内沉默报警.这里就以zabbix报警脚本向AlertManager发送报警信息，再由AlertManager处理后通知用户。




## 配置Zabbix

zabbix怎么安装这里不在说明，只说明有更改的地方。

### 脚本配置

```python
cat  /usr/lib/zabbix/alertscripts/zabbix-to-am.py
#!/usr/bin/env python
# -*- coding: utf-8 -*-


import os
import sys
import json
import requests
import logging.handlers
from datetime import datetime
import time


def str_time(s):
    dt = datetime.strptime(s, '%Y.%m.%d %H:%M:%S')
    tm = time.mktime(dt.timetuple())
    ut = datetime.utcfromtimestamp(tm).isoformat() + 'Z'
    return ut


if len(sys.argv) < 2:
    print("Require Arguments 1 subject")
    sys.exit()

# 设置日志
log_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'logs')
log_file = os.path.join(log_path, os.path.basename(__file__).split('.')[0] + '.log')

if not os.path.exists(log_path):
    os.mkdir(log_path)

logger = logging.getLogger()
handler = logging.handlers.RotatingFileHandler(log_file, maxBytes=(100 * 1024 * 1024 * 8), backupCount=5,
                                               encoding='utf-8')
formatter = logging.Formatter('[%(asctime)s] %(levelname)s [%(funcName)s: %(filename)s, %(lineno)d] %(message)s',
                              "%Y-%m-%d %H:%M:%S")
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

subject = sys.argv[1]

'''
Zabbix 发送的消息体

{TRIGGER.SEVERITY}
{HOST.NAME}
{HOST.IP}
{TRIGGER.HOSTGROUP.NAME}
{ITEM.NAME}
{EVENT.DATE} {EVENT.TIME}
{TRIGGER.NAME}
{ITEM.NAME}:{ITEM.VALUE}
firing/resolved
{EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}

---
subject = """
Average\r
putao-zabbix\r
172.20.9.24\r
Zabbix servers\r
Agent ping\r
2019.07.22 09:43:30\r
Zabbix agent on zabbix is unreachable for 5 minutes\r
Agent ping:Up (1)\r
resolved\r
2019.07.22 10:44:07
"""
'''

logging.info('[start].....')
logging.info('[source]: ' + subject)

data = {"labels": {}}

# 解析zabbix发送的消息体
labels = subject.split("\r\n")
data["labels"]["severity"] = labels[0].strip()
data["labels"]["name"] = labels[1].strip()
data["labels"]["instance"] = labels[2].strip()
data["labels"]["group"] = labels[3].strip()
data["labels"]["alertname"] = labels[4].strip()
data["labels"]["service"] = "zabbix"

data["annotations"] = {}
data["annotations"]["summary"] = labels[6].strip()
data["annotations"]["description"] = labels[7].strip()
data['status'] = labels[8].strip()

data['startsAt'] = str_time(labels[5].strip())

if data['status'] == 'resolved':
    data['endsAt'] = str_time(labels[9].strip())

data['generatorURL'] = "http://172.20.9.24/zabbix.php?action=dashboard.view&ddreset=1"

post_data = json.dumps([data])

# 向alertmanager发送请求
am_url = "http://172.20.6.7:30080/api/v2/alerts"
headers = {"Content-Type": "application/json", "Host": "alertmanager.monitoring.k8s.local"}

logging.info('[DATA] %s' % post_data)

r = requests.post(am_url, data=post_data, headers=headers)

logging.info('[return] %s' % r)
logging.info('[return] %s' % r.text)
logging.info('[end].....')
```

> `am_url` 更换成你的地址  `generatorURL` 设置zabbix报警信息的页面地址

进行目录授权

```bash
chown zabbix.zabbix -R /usr/lib/zabbix/alertscripts
```

### 添加报警媒介类型

在zabbix的web上，管理->报警媒介类型->创建媒体类型

| 项目     | 值              |
| -------- | --------------- |
| 名称     | zabbix-to-am    |
| 类型     | 脚本            |
| 脚本名称 | zabbix-to-am.py |
| 脚本参数 | {ALERT.MESSAGE} |
| 已启用   | ✔               |

## 添加动作

在zabbix的web上，配置->动作->创建动作

| 项目              | 值                                                                                                                                                                                                                                       | 说明                                                                  |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 名称              | 报警到alertmanger                                                                                                                                                                                                                        |                                                                       |
| 操作/消息内容     | {TRIGGER.SEVERITY}<br/>{HOST.NAME}<br/>{HOST.IP}<br/>{TRIGGER.HOSTGROUP.NAME}<br/>{ITEM.NAME}<br/>{EVENT.DATE} {EVENT.TIME}<br/>{TRIGGER.NAME}<br/>{ITEM.NAME}:{ITEM.VALUE}<br/>firing                                                   | 每个值使用换行符隔开，firing为alert的状态                             |
| 操作/操作         | 1 - 0	发送消息给用户: Admin (Zabbix Administrator) 通过 zabbix-to-am<br/>立即地	5m                                                                                                                                                       | 发送消息给admin用户, 注意前面`1-0`表示每隔`5分钟`通知报警，且次数不限 |
| 恢复操作/消息内容 | {TRIGGER.SEVERITY}<br/>{HOST.NAME}<br/>{HOST.IP}<br/>{TRIGGER.HOSTGROUP.NAME}<br/>{ITEM.NAME}<br/>{EVENT.DATE} {EVENT.TIME}<br/>{TRIGGER.NAME}<br/>{ITEM.NAME}:{ITEM.VALUE}<br/>resolved<br/>{EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME} | 每个值使用换行符隔开，resolved为alert的状态，最后一行是恢复的时间点   |
| 恢复操作/操作     | **发送消息给用户:** Admin (Zabbix Administrator) 通过 zabbix-to-am                                                                                                                                                                       | 发送消息给admin用户                                                   |

### 添加报警用户

在zabbix的web上，配置->用户-报警媒介

| 项目   | 值           |
| ------ | ------------ |
| 类型   | zabbix-to-am |
| 收件人 | 1            |

在Administrator用户上增加一个报警媒介，收件人随便填写，暂时没有用到

## 配置AlertManager

AlertManager怎么安装这里不在说明，只说明有更改的地方。

### alertmanger配置文件

{% raw %}

```bash
global:
  resolve_timeout: 10m
  smtp_smarthost: 'your_smtp_smarthost:587'
  smtp_from: 'your_smtp_from'
  smtp_auth_username: 'your_smtp_user'
  smtp_auth_password: 'your_smtp_pass'
templates:
- '*.tmpl'
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 10m
  repeat_interval: 2h
  receiver: default-receiver
  routes:
  - match:
      service: zabbix
    receiver: 'zabbix'
  - match:
      alertname: DeadMansSwitch
    receiver: 'null'
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  equal: ['alertname', 'cluster', 'service']
receivers:
- name: 'default-receiver'
  wechat_configs:
  - corp_id: '123456'
    to_party: '10'
    agent_id: '1000000'
    api_secret: 'api_secret'
    send_resolved: true
  email_configs:
  - to: 'your_alert_email_address'
    send_resolved: true
- name: 'zabbix'
  wechat_configs:
  - corp_id: '123456'
    to_user: 'Lework'
    agent_id: '1000003'
    api_secret: 'api_secret'
    send_resolved: true
    message: '{{ template "wechat-zabbix.default.message" . }}'
  email_configs:
  - to: 'your_alert_email_address'
    send_resolved: true
- name: 'null'

# corp_id: 企业微信账号唯一ID, 可以在我的企业中查看
# to_party: 需要发送的组
# to_user: 发送给用户
# agent_id: 第三方企业应用的ID, 可以在自己创建的第三方企业应用详情页面查看
# api_secret: 第三方企业应用的密钥, 可以在自己创建的第三方企业应用详情页面查看
```

> 这里使用的企业微信作为通知工作

### 企业微信消息模板

```bash
cat /etc/alertmanager/config/wechat-zabbix.tmpl
{{- define "__zabbix_text_alert_list" -}}
{{- range .Alerts.Firing -}}
告警级别: {{ .Labels.severity }}
告警项目: {{ .Labels.alertname }}
故障实例: {{ .Labels.instance }}
实例名称: {{ .Labels.name }}
实例组:   {{ .Labels.group }}
告警主题: {{ .Annotations.summary }}
告警详情: {{ .Annotations.description }}
触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
-------------------

{{ end }}
{{- range .Alerts.Resolved -}}
告警级别: {{ .Labels.severity }}
告警项目: {{ .Labels.alertname }}
故障实例: {{ .Labels.instance }}
实例名称: {{ .Labels.name }}
实例组:   {{ .Labels.group }}
告警主题: {{ .Annotations.summary }}
告警详情: {{ .Annotations.description }}
触发时间: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
恢复时间: {{ (.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
-------------------

{{ end }}
{{- end }}

{{- define "wechat-zabbix.default.message" -}}
{{- if gt (len .Alerts.Firing) 0 -}}
[Zabbix]Alerts Firing[{{ .Alerts.Firing | len }}]:
{{ template "__zabbix_text_alert_list" . }}
{{- end }}
{{- if gt (len .Alerts.Resolved) 0 -}}
[Zabbix]Alerts Resolved[{{ .Alerts.Resolved | len }}]:
{{ template "__zabbix_text_alert_list" . }}
{{- end }}
{{- end }}
```

{% endraw %}

## 已知的问题

* alertmanger 在接受到alert恢复消息时，alert的`startAt`时间+`group_interval`时间>当前时间才会发送恢复消息。
* 通知用户的管理必须使用AlertManager的yaml进行管理，比较麻烦麻烦等等。
