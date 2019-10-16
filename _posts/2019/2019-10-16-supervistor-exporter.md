---
layout: post
title: "通过Supervisor exporter监控进程并报警"
date: "2019-10-16 22:01:00"
category: supervisor
tags: supervisor
author: lework
---
* content
{:toc}
## 需求

监控supervisor管理的进程状态，在进程异常退出的时候给出报警。

## 实现思路

将进程的状态信息数据发送给Prometheus，通过Prometheus进行报警。

上一篇文章[利用 Supervisor 的 Event & Listener 监控进程并报警](https://lework.github.io/2019/10/16/supervistor-event/) 已经完全满足需求了，那为啥还要这种方式么？我想有两个点吧，1. 内部环境中存在Prometheus监控报警体系，将报警统一到平台中，方便管理。2.通过平台可以看到进程的趋势图，比如进程的cpu，内存。。不过这个方案也有个缺点，就是报警不及时。有人人说了，node_exporter也可以监控supervisor进程呀。为啥非要自己开发，我只能说，方案千千万，适合自己最好。





下面介绍下这次用到的知识

### Superver RPC

通过连接supervisor的`XML-RPC`接口，可以管理并查看进程的信息

连接接口

python2 使用xmlrpclib

```
import xmlrpclib
server = xmlrpclib.Server('http://localhost:9001/RPC2')
```

 Python 3 使用`xmlrpc.client` 

```
from xmlrpc.client import ServerProxy
server = ServerProxy('http://localhost:9001/RPC2')
```

这次我们使用的是`getAllProcessInfo()` API,其他API请到[XML-RPC API](http://supervisord.org/api.html)

返回的格式如下：

```
[
 {
 'name':           'process name',
 'group':          'group name',
 'description':    'pid 18806, uptime 0:03:12'
 'start':          1200361776,
 'stop':           0,
 'now':            1200361812,
 'state':          20,
 'statename':      'RUNNING',
 'spawnerr':       '',
 'exitstatus':     0,
 'logfile':        '/path/to/stdout-log', # deprecated, b/c only
 'stdout_logfile': '/path/to/stdout-log',
 'stderr_logfile': '/path/to/stderr-log',
 'pid':            1
 }
]
```

### Prometheus client_python

client_python是Prometheus的python客户端，具体请看官方文档[github](https://github.com/prometheus/client_python)

本次用到的数据类型

- Counter Counter可以增长，并且在程序重启的时候会被重设为0，常被用于任务个数，总处理时间，错误个数等只增不减的指标。

- Gauge Gauge与Counter类似，唯一不同的是Gauge数值可以减少，常被用于温度、利用率等指标。

  


## 实现步骤

### 实现脚本

```
#!/usr/bin/python
# -*- coding: utf-8 -*-

# @Time    : 2019-10-15
# @Author  : lework
# @Desc    : 收集supervisor的进程状态信息，并将信息暴露给Prometheus。

# [program:supervisor_exporter]
# process_name=%(program_name)s
# command=/usr/bin/python /root/scripts/supervisor_exporter.py
# autostart=true
# autorestart=true
# redirect_stderr=true
# stdout_logfile=/var/log/supervisor/supervisor_exporter.log
# stdout_logfile_maxbytes=50MB
# stdout_logfile_backups=3
# buffer_size=10


from xmlrpclib import ServerProxy
from prometheus_client import Gauge, Counter, CollectorRegistry ,generate_latest, start_http_server
from time import sleep

try:
    from BaseHTTPServer import BaseHTTPRequestHandler, HTTPServer
except ImportError:
    # Python 3
    from http.server import BaseHTTPRequestHandler, HTTPServer

def is_runing(state):
    state_info = {
            # 'STOPPED': 0,
            'STARTING': 10,
            'RUNNING': 20
            # 'BACKOFF': 30,
            # 'STOPPING': 40
            # 'EXITED': 100,
            # 'FATAL': 200,
            # 'UNKNOWN': 1000
    }
    if state in state_info.values():
        return True
    return  False


def get_metrics():
    collect_reg = CollectorRegistry(auto_describe=True)

    try:
        s = ServerProxy(supervisord_url)
        data = s.supervisor.getAllProcessInfo()
    except Exception as e:
        print("unable to call supervisord: %s" % e)
        return collect_reg

    labels=('name', 'group')

    metric_state = Gauge('state', "Process State", labelnames=labels, subsystem='supervisord', registry=collect_reg)
    metric_exit_status=Gauge('exit_status', "Process Exit Status", labelnames=labels, subsystem='supervisord', registry=collect_reg)
    metric_up = Gauge('up', "Process Up", labelnames=labels, subsystem='supervisord', registry=collect_reg)
    metric_start_time_seconds=Counter('start_time_seconds', "Process start time", labelnames=labels, subsystem='supervisord', registry=collect_reg)

    for item in data:
        now = item.get('now', '')
        group = item.get('group', '')
        description = item.get('description', '')
        stderr_logfile = item.get('stderr_logfile', '')
        stop = item.get('stop', '')
        statename = item.get('statename', '')
        start = item.get('start', '')
        state = item.get('state', '')
        stdout_logfile = item.get('stdout_logfile', '')
        logfile = item.get('logfile', '')
        spawnerr = item.get('spawnerr', '')
        name = item.get('name', '')
        exitstatus = item.get('exitstatus', '')

        labels = (name, group)

        metric_state.labels(*labels).set(state)
        metric_exit_status.labels(*labels).set(exitstatus)

        if is_runing(state):
            metric_up.labels(*labels).set(1)
	    metric_start_time_seconds.labels(*labels).inc(start)
        else:
            metric_up.labels(*labels).set(0)

    return  collect_reg


class myHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type','text/plain')
        self.end_headers()
        data=""
        if self.path=="/":
            data="hello, supervistor."
	elif self.path=="/metrics":
            data=generate_latest(get_metrics())
        else:
            data="not found"
	    # Send the html message
        self.wfile.write(data)
        return

if __name__ == '__main__':
    try:
        supervisord_url = "http://localhost:9001/RPC2"
		
        PORT_NUMBER = 8000
        #Create a web server and define the handler to manage the
        #incoming request
        server = HTTPServer(('', PORT_NUMBER), myHandler)
        print 'Started httpserver on port ' , PORT_NUMBER
	
        #Wait forever for incoming htto requests
        server.serve_forever()

    except KeyboardInterrupt:
        print '^C received, shutting down the web server'
        server.socket.close()
   
```

> 这里要配置supervisor的rpc链接地址和http监听的地址

生成的`metrics`

```
# HELP supervisord_up Process Up
# TYPE supervisord_up gauge
supervisord_up{group="supervisor-exporter",name="supervisor-exporter"} 1.0
supervisord_up{group="sleep",name="sleep"} 1.0
supervisord_up{group="supervisor_event_exited",name="supervisor_event_exited"} 1.0
# HELP supervisord_state Process State
# TYPE supervisord_state gauge
supervisord_state{group="supervisor-exporter",name="supervisor-exporter"} 20.0
supervisord_state{group="sleep",name="sleep"} 20.0
supervisord_state{group="supervisor_event_exited",name="supervisor_event_exited"} 20.0
# HELP supervisord_start_time_seconds_total Process start time
# TYPE supervisord_start_time_seconds_total counter
supervisord_start_time_seconds_total{group="supervisor-exporter",name="supervisor-exporter"} 1.571219534e+09
supervisord_start_time_seconds_total{group="sleep",name="sleep"} 1.571234835e+09
supervisord_start_time_seconds_total{group="supervisor_event_exited",name="supervisor_event_exited"} 1.571219256e+09
# TYPE supervisord_start_time_seconds_created gauge
supervisord_start_time_seconds_created{group="supervisor-exporter",name="supervisor-exporter"} 1.571234883621971e+09
supervisord_start_time_seconds_created{group="sleep",name="sleep"} 1.571234883621722e+09
supervisord_start_time_seconds_created{group="supervisor_event_exited",name="supervisor_event_exited"} 1.571234883622026e+09
# HELP supervisord_exit_status Process Exit Status
# TYPE supervisord_exit_status gauge
supervisord_exit_status{group="supervisor-exporter",name="supervisor-exporter"} 0.0
supervisord_exit_status{group="sleep",name="sleep"} 0.0
supervisord_exit_status{group="supervisor_event_exited",name="supervisor_event_exited"} 0.0
```

### Supervisor配置

```
[inet_http_server]         ; inet (TCP) server disabled by default
port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for all iface)

[program:supervisor-exporter]
process_name=%(program_name)s
command=/usr/bin/python /root/scripts/supervisor_exporter.py
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/supervisor/supervisor_exporter.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=3
buffer_size=10
```

### Prometheus配置

配置job

```
  - job_name: 'supervistor-exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.77.133:8000']
```

配置alert rule
{% raw %}

```
groups:
- name: supervisord
  rules:
  - alert: JobDown
    expr: supervisord_up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Job {{ $labels.job }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
```

{% endraw %}