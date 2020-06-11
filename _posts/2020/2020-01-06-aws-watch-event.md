---
layout: post
title: "使用lamdba实现产品的监控项自动创建"
date: "2020-01-06 19:00"
category: aws
tags: aws lamdba cloudwatch
author: lework
---
* content
{:toc}

aws的云产品(ec2,elb,redis,rds)的监控需要一个一个的添加，比较耗时耗力。本着偷懒的精神要实现在产品创建后，自动的创建产品对应的监控项和监控面板。




## 实现方案

1. 使用 `cloudwatch` 中的`事件`功能来监听产品的`创建`事件。
2. 产品的`创建`事件会触发 `lamdba` 函数来生成对应产品的监控项。


## 实现步骤

### 添加监控的 lambda 函数

**函数名称为:**  `addAlarm`

**函数语言：**  `python 3.7`

**函数简要介绍:**
1. 使用 `boto3` 访问 aws 接口
2. 使用 `boto3.client('cloudwatch').put_metric_alarm()` 添加监控项
3. 使用 `boto3.client('cloudwatch').put_dashboard()` 添加监控面板

**函数权限:**
```
Allow：cloudwatch:ListMetrics
Allow：cloudwatch:GetMetricStatistics
Allow：cloudwatch:Describe*
Allow：cloudwatch:*
```

**函数超时时间:**
- 3分钟

**产品事件**

| 产品  | 事件源                             | 事件                   |
| ----- | ---------------------------------- | ---------------------- |
| elb   | elasticloadbalancing.amazonaws.com | CreateLoadBalancer     |
| ec2   | ec2.amazonaws.com                  | RunInstances           |
| rds   | rds.amazonaws.com                  | CreateDBInstance       |
| redis | elasticache.amazonaws.com          | CreateReplicationGroup |


**完整脚本**

见：[addAlarm.py](https://raw.githubusercontent.com/lework/script/master/cloud/aws/lambda/addAlarm.py)

> 需要修改函数里面的`sns_arn`和`region`




### 删除监控的 lambda 函数

**函数名称为:**  `delAlarm`

**函数语言：**  `python 3.7`

**函数简要介绍:**

1. 使用 `boto3` 访问 aws 接口
2. 使用 `boto3.client('cloudwatch').delete_alarms()` 添加监控项
3. 使用 `boto3.client('cloudwatch').delete_dashboards()` 添加监控面板


**函数权限:**
```
Allow：cloudwatch:ListMetrics
Allow：cloudwatch:GetMetricStatistics
Allow：cloudwatch:Describe*
Allow：cloudwatch:*
```

**函数超时时间:**
- 3分钟

**产品事件**

| 产品  | 事件源                             | 事件                   |
| ----- | ---------------------------------- | ---------------------- |
| elb   | elasticloadbalancing.amazonaws.com | DeleteLoadBalancer     |
| ec2   | ec2.amazonaws.com                  | TerminateInstances     |
| rds   | rds.amazonaws.com                  | DeleteDBInstance       |
| redis | elasticache.amazonaws.com          | DeleteReplicationGroup |

**完整脚本**

见：[delAlarm.py](https://raw.githubusercontent.com/lework/script/master/cloud/aws/lambda/delAlarm.py)

### 添加事件

选择 "cloudwatch" ==> "事件" ==> "规则" 

针对各个产品创建添加/删除的事件通知, 下列是 `redis` 的创建事件

![event1](/assets/images/aws/event1.png)

然后创建各个产品的规则

![event2](/assets/images/aws/event2.png)

完成后，就可以创建产品，在警报和控制面板中看看lamdba创建的项目了。

> lamdba 的执行日志在"cloudwatch" ==> "日志" ==> "日志组" 中存储。 