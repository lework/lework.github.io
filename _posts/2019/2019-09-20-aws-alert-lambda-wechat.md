---
layout: post
title: 'AWS 使用Lambda实现企业微信报警'
date: '2019-09-20 21:10:00'
category: aws
tags: aws
author: lework
---
* content
{:toc}

aws 支持邮件和短信的报警通知，考虑时效性问题和结合公司的使用情况，需要接入企业微信的告警提醒，使用 Lambda 和 sns 的结合，可以实现这个需求，利用 Lambda 接受 SNS 的告警信息，然后通过 python 脚本发送到企业微信的接口上去。由企业微信客户端接收通知。




## Lambda 的配置

> sns 的创建配置，这里不在描述

进入 lambda 服务，路径：服务->搜索，通过 lambda 关键字可以查找到

![create_lambda.png](/assets/images/aws/create_lambda.png)

### 配置触发器

创建完成后会进入函数详情页面，里面需要配置触发器，我们配置为 SNS 触发，然后选择我们设置好的主题，见下图：

![sns.png](/assets/images/aws/sns.png)

### 配置 lambda 函数

我们设置完出发器后，添加处理脚步，Lambda 可以就订阅制定的 SNS 主题，有告警发生时，会接收到告警消息，然后可以通过自己的处理脚步处理警告信息。我们在这个页面，添加我们的处理脚本：

![lambda.png](/assets/images/aws/lambda.png)

源码如下：

```
import json
from botocore.vendored import requests


def lambda_handler(event, context):
    # TODO implement
    url = "https://qyapi.weixin.qq.com"

    # 设置企业微信参数
    corpid = ""
    secret = ""
    agentid = ""
    touser = ''
    toparty = ''
    totag = ''

    headers = {
        'Content-Type': 'application/json'
    }

    access_token_url = '{url}/cgi-bin/gettoken?corpid={id}&corpsecret={crt}'.format(url=url, id=corpid, crt=secret)
    access_token_response = requests.get(url=access_token_url, headers=headers)
    token = json.loads(access_token_response.text)['access_token']

    send_url = '{url}/cgi-bin/message/send?access_token={token}'.format(url=url, token=token)

    message = event['Records'][0]['Sns']
    Timestamp = message['Timestamp']
    Subject = message['Subject']
    sns_message = json.loads(message['Message'])
    region = message['TopicArn'].split(':')[-3]

    if "ALARM" in Subject:
        title = '<font color=\"info\">[aws] 警报！！警报！！</font>'
    elif "OK" in Subject:
        title = '<font color=\"info\">[aws] 故障恢复</font>'
    else:
        title = '<font color=\"info\">[aws]</font>'

    content = title \
              + "\n> **详情信息**" \
              + "\n> 时间: " + Timestamp \
              + "\n> 内容: " + Subject \
              + "\n> 状态: <font color=\"comment\">{old}</font> => <font color=\"warning\">{new}</font>".format(
        old=sns_message['OldStateValue'], new=sns_message['NewStateValue']) \
              + "\n> " \
              + "\n> Region: " + sns_message['Region'] \
              + "\n> Namespace: " + sns_message['Trigger']['Namespace'] \
              + "\n> MetricName: " + sns_message['Trigger']['MetricName'] \
              + "\n> " \
              + "\n> AlarmName: " + sns_message['AlarmName'] \
              + "\n> AlarmDescription: " + sns_message['AlarmDescription'] \
              + "\n> " \
              + "\n> 详情请点击：[Alarm](https://{region}.console.amazonaws.cn/cloudwatch/home?region={region}#s=Alarms&alarm={alarm})".format(
        region=region, alarm=sns_message['AlarmName'])

    msg = {
        "msgtype": 'markdown',
        "agentid": agentid,
        "markdown": {'content': content},
        "safe": 0
    }

    if touser:
        msg['touser'] = touser
    if toparty:
        msg['toparty'] = toparty
    if toparty:
        msg['totag'] = totag

    response = requests.post(url=send_url, data=json.dumps(msg), headers=headers)

    errcode = json.loads(response.text)['errcode']
    if errcode == 0:
        print('Succesfully')
    else:
        print(response.json())
        print('Failed')
```

> 根据自身情况设置企业微信接口参数

### 测试 lambda

点击配置测试事件，进入配置测试事件页面，我们可以选择事件模版，这里我们选择 SNS 的通知模版，给事件命名，完成事件配置

![lambda-test.png](/assets/images/aws/lambda-test.png)

> 下面是报警和恢复的 json 数据

``` json
// 报警
{
    "Records": [
        {
            "EventSource": "aws:sns",
            "EventVersion": "1.0",
            "EventSubscriptionArn": "arn:aws-cn:sns:cn-north-1:377051231234:baojing:c4e8c500-f3c0-4d12-ad07-2191234a4ae",
            "Sns": {
                "Type": "Notification",
                "MessageId": "e05e7190-311c-5d08-acf0-2191234a4ae",
                "TopicArn": "arn:aws-cn:sns:cn-north-1:377051231234:baojing",
                "Subject": "ALARM: \"db_mem\" in China (Beijing)",
                "Message": "{\"AlarmName\":\"db_mem\",\"AlarmDescription\":\"db数据库可用内存不足5g\",\"AWSAccountId\":\"377051238643\",\"NewStateValue\":\"ALARM\",\"NewStateReason\":\"Threshold Crossed: 1 datapoint [1.50785441792E10 (12/09/19 09:06:00)] was greater than the threshold (5.0).\",\"StateChangeTime\":\"2019-09-12T09:11:07.155+0000\",\"Region\":\"China (Beijing)\",\"OldStateValue\":\"OK\",\"Trigger\":{\"MetricName\":\"FreeableMemory\",\"Namespace\":\"AWS/RDS\",\"StatisticType\":\"Statistic\",\"Statistic\":\"AVERAGE\",\"Unit\":null,\"Dimensions\":[{\"value\":\"db\",\"name\":\"DBInstanceIdentifier\"}],\"Period\":300,\"EvaluationPeriods\":1,\"ComparisonOperator\":\"GreaterThanThreshold\",\"Threshold\":5.0,\"TreatMissingData\":\"- TreatMissingData: missing\",\"EvaluateLowSampleCountPercentile\":\"\"}}",
                "Timestamp": "2019-09-12T09:11:07.222Z",
                "SignatureVersion": "1",
                "Signature": "KwapI3oT4030RnNYx9e6qjQw5UrfjeCIkO5aQ6gdCCWte1qot08QxoxI1iOusZkbuj2S9OZINFG4gyvN2oZVDEozjUVgUokrGBalBLMCJjv68DAz2FMGfiftiMY7d+N+ESpymqzNkLVFPxN5oTdbHo2P1eKiODILLDZYD31ICAmyXtl1NPfTLyDFlga+xvTnbF9Qzb8LwjzRr0VoZnnkCYp3lP/Mjr6zdT356Y7s9H8HEVp9YycoVkD2KJK5Bd2cyBuXsQc/I1RzYZbSxbAmqzSTUb0sLAuCTTUTaJRwvGXVtYH0G7fSXv6nDGB9eO8lWWkWBEHCAHcLoI8VyzPndQ==",
                "SigningCertUrl": "https://sns.cn-north-1.amazonaws.com.cn/SimpleNotificationService-3250158c6506d40f628c21ed8dad1234.pem",
                "UnsubscribeUrl": "https://sns.cn-north-1.amazonaws.com.cn/?Action=Unsubscribe&SubscriptionArn=arn:aws-cn:sns:cn-north-1:377051238643:baojing:c4e8c500-f3c0-4d12-ad07-2191234a4ae",
                "MessageAttributes": {}
            }
        }
    ]
}

// 恢复
{
    "Records": [
        {
            "EventSource": "aws:sns",
            "EventVersion": "1.0",
            "EventSubscriptionArn": "arn:aws-cn:sns:cn-north-1:377051231234:baojing:c4e8c500-f3c0-4d12-ad07-21966357a4ae",
            "Sns": {
                "Type": "Notification",
                "MessageId": "82a293cf-419e-51ba-8eac-2191234a4ae",
                "TopicArn": "arn:aws-cn:sns:cn-north-1:377051231234:baojing",
                "Subject": "OK: \"db_mem\" in China (Beijing)",
                "Message": "{\"AlarmName\":\"db_mem\",\"AlarmDescription\":\"db数据库可用内存不足5g\",\"AWSAccountId\":\"377051238643\",\"NewStateValue\":\"OK\",\"NewStateReason\":\"Threshold Crossed: 1 datapoint [1.50755950592E10 (12/09/19 09:49:00)] was not less than the threshold (5.0).\",\"StateChangeTime\":\"2019-09-12T09:54:32.108+0000\",\"Region\":\"China (Beijing)\",\"OldStateValue\":\"ALARM\",\"Trigger\":{\"MetricName\":\"FreeableMemory\",\"Namespace\":\"AWS/RDS\",\"StatisticType\":\"Statistic\",\"Statistic\":\"AVERAGE\",\"Unit\":null,\"Dimensions\":[{\"value\":\"db\",\"name\":\"DBInstanceIdentifier\"}],\"Period\":300,\"EvaluationPeriods\":1,\"ComparisonOperator\":\"LessThanThreshold\",\"Threshold\":5.0,\"TreatMissingData\":\"- TreatMissingData: missing\",\"EvaluateLowSampleCountPercentile\":\"\"}}",
                "Timestamp": "2019-09-12T09:54:32.174Z",
                "SignatureVersion": "1",
                "Signature": "FaxqVSttIR5A4jOLFE6fFrV/YjwXUYFoWaFvw6+5ItSaPJ1gsxfQOSaqWBct+X7DqBi5VBmqmH7CMhaWCXeHm8Uo3RL0nsy0gsXg4WTo6LCWqC4t+jtI53JjQfDedW3eTEVcYcRjPEcyocWvlSsIXVVE/cdoTJmt0Df9hRhgwRPMOCYC8AGMVgIZuOsOvdNvlACdUS0KJsWlKuoYtA/E0sAWugycxXiArj52vEfj7F7MLdCj+j94wGSImUlGeYc419NqBOjORZN0VtBwQ6fSAp8D1FzUBz58+zHa77UjlRMbnqm+CP2rr1cyd2Scqm8kUqwiemrQa4Ikrf0XngMLsg==",
                "SigningCertUrl": "https://sns.cn-north-1.amazonaws.com.cn/SimpleNotificationService-3250158c6506d40f628c21ed8dad1234.pem",
                "UnsubscribeUrl": "https://sns.cn-north-1.amazonaws.com.cn/?Action=Unsubscribe&SubscriptionArn=arn:aws-cn:sns:cn-north-1:377051231234:baojing:c4e8c500-f3c0-4d12-ad07-2191234a4ae",
                "MessageAttributes": {}
            }
        }
    ]
}
```

事件配置完成后，我们可以点击测试进行验证，通过红框处的日志信息，我们可以看到测试的结果是否成功，同时可以查看自己的企业微信，看是否收到了告警信息。

![lambda-test2.png](/assets/images/aws/lambda-test2.png)

### 通知效果

![wechat.png](/assets/images/aws/wechat.png)

## lambda 日志的存放点

CloudWatch->日志，选择日志组，就可以查询当前日志组的日志信息了，可以根据日志信息详情来进行问题分析排查。

![sns_log.png](/assets/images/aws/sns_log.png)
