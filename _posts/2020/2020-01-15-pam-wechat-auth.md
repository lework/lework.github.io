---
layout: post
title: "使用Pam-Python实现SSH的企业微信双因素认证"
date: "2020-01-15 18:40"
category: linux
tags: linux ssh
author: lework
---
* content
{:toc}

现在的黑客是越来越厉害了, 单单的 key 或者 pass 认证登录 ssh，已经不怎么安全了！对于重要的服务器，又要暴露在公网中，这时该怎么保证其相对安全呢？首先要做的是：第一要更改 sshd 的服务端口，第二要设置一个复杂的密码，或者强劲的秘钥登录。当然这是第一道防护，第二道防护就是我们今天要做的，使用 pam 模块进行二次验证。常见的二次验证解决方案有短信验证码、RSA动态令牌、[Google Authenticator](/2017/07/08/Ansible-an-quan-zhi-ssh-deng-lu-er-ci-yan-zheng/)或者Duo，这些配置多多少少都有些麻烦。目前国内使用企业微信和钉钉作为企业沟通工具的公司越来越多，配置起来也非常简单，所以今天我们就以企业微信为载体，来传递二次验证的验证码。

PAM（PluggableAuthentication Module，可插拔认证模块）机制，采用模块化设计和插件功能，使用户可以轻易地在应用程序中插入新的认证模块或替换原先的组件，同时不必对应用程序做任何修改。




## 安装pam-python

**环境**

OS： centos 7.4 X64

**安装依赖和下载包**

```bash
yum -y install gcc make pam pam-devel python-devel

wget -O pam-python-1.0.7.tar.gz https://sourceforge.net/projects/pam-python/files/pam-python-1.0.7-1/pam-python-1.0.7.tar.gz/download
tar zxf pam-python-1.0.7.tar.gz 
cd pam-python-1.0.7/src
```

在 `pam_python.c` 中，将 `#include <Python.h>`  移到 `#include <security/pam_modules.h>` 上方

**编译模块**
```bash
make pam_python.so
cp pam_python.so /usr/lib64/security/
```

## 脚本文件

文件存放在： `/usr/lib64/security/pam_wechat_auth.py`

源文件： [pam_wechat_auth.py](https://github.com/lework/script/blob/master/python/pam_wechat_auth.py)

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

# @Time    : 2020-01-15
# @Author  : lework
# @Desc    : 使用Pam-Python实现SSH的企业微信双因素认证

import sys
import pwd
import json
import string
import syslog
import random
import hashlib
import httplib
import datetime
import platform


def auth_log(msg):
    """写入日志"""
    syslog.openlog(facility=syslog.LOG_AUTH)
    syslog.syslog("MultiFactors Authentication: " + msg)
    syslog.closelog()


def action_wechat(content, touser=None, toparty=None, totag=None):
    """微信通知"""
    host = "qyapi.weixin.qq.com"

    # 企业微信设置
    corpid = ""
    secret = ""
    agentid = ""

    headers = {
        'Content-Type': 'application/json'
    }

    access_token_url = '/cgi-bin/gettoken?corpid={id}&corpsecret={crt}'.format(id=corpid, crt=secret)

    try:
        httpClient = httplib.HTTPSConnection(host, timeout=10)
        httpClient.request("GET", access_token_url, headers=headers)
        response = httpClient.getresponse()
        token = json.loads(response.read())['access_token']
        httpClient.close()
    except Exception as e:
        auth_log('get wechat token error: %s' % e)
        return False

    send_url = '/cgi-bin/message/send?access_token={token}'.format(token=token)

    data = {
        "msgtype": 'text',
        "agentid": agentid,
        "text": {'content': content},
        "safe": 0
    }

    if touser:
        data['touser'] = touser
    if toparty:
        data['toparty'] = toparty
    if toparty:
        data['totag'] = totag

    try:
        httpClient = httplib.HTTPSConnection(host, timeout=10)
        httpClient.request("POST", send_url, json.dumps(data), headers=headers)
        response = httpClient.getresponse()
        result = json.loads(response.read())
        if result['errcode'] != 0:
            auth_log('Failed to send verification code using WeChat: %s' % result)
            return False
    except Exception as e:
        auth_log('Error sending verification code using WeChat: %s' % e)
        return False
    finally:
        if httpClient:
            httpClient.close()

    auth_log('Send verification code using WeChat successfully.')
    return True

def get_user_comment(user):
    """获取用户描述信息"""
    try:
        comments = pwd.getpwnam(user).pw_gecos
    except:
        auth_log("No local user (%s) found." % user)
        comments = ''

    return comments # 返回用户描述信息


def get_hash(plain_text):
    """获取PIN码的sha512字符串"""
    key_hash = hashlib.sha512()
    key_hash.update(plain_text)

    return key_hash.digest()


def gen_key(pamh, user, length):
    """生成PIN码并发送到用户"""
    pin = ''.join(random.choice(string.digits) for i in range(length))
    #msg = pamh.Message(pamh.PAM_ERROR_MSG, "The pin is: (%s)" % (pin))  # 登陆界面输出验证码，测试目的，实际使用中注释掉即可
    #pamh.conversation(msg)

    hostname = platform.node().split('.')[0]
    content = "[MFA] %s 使用 %s 正在登录 %s, 验证码为【%s】, 1分钟内有效。" % (pamh.rhost, user, hostname, pin)
    touser = get_user_comment(user) 
    result = action_wechat(content, touser=touser)

    pin_time = datetime.datetime.now()
    return get_hash(pin), pin_time


def pam_sm_authenticate(pamh, flags, argv):
    PIN_LENGTH = 6  # PIN码长度
    PIN_LIVE = 60   # PIN存活时间,超出时间验证失败
    PIN_LIMIT = 3   # 限制错误尝试次数
    EMERGENCY_HASH = '\xba2S\x87j\xedk\xc2-Jo\xf5=\x84\x06\xc6\xad\x86A\x95\xed\x14J\xb5\xc8v!\xb6\xc23\xb5H\xba\xea\xe6\x95m\xf3F\xec\x8c\x17\xf5\xea\x10\xf3^\xe3\xcb\xc5\x14y~\xd7\xdd\xd3\x14Td\xe2\xa0\xba\xb4\x13'  # 预定义验证码123456的hash, 用于紧急认证

    try:
        user = pamh.get_user()
    except pamh.exception as e:
        return e.pam_result
  
    auth_log("login_ip: %s, login_user: %s" % (pamh.rhost, user))

    if get_user_comment(user) == '':
        msg = pamh.Message(pamh.PAM_ERROR_MSG, "[Warning] You need to set the Qiyi WeChat username in the comment block for user %s." % (user))
        pamh.conversation(msg)
        return pamh.PAM_ABORT
    
    pin, pin_time = gen_key(pamh, user, PIN_LENGTH)

    for attempt in range(0, PIN_LIMIT):  # 限制错误尝试次数
        msg = pamh.Message(pamh.PAM_PROMPT_ECHO_OFF, "Verification code:")
        resp = pamh.conversation(msg)
        resp_time = datetime.datetime.now()
        input_interval = resp_time - pin_time
        if input_interval.seconds > PIN_LIVE:
            msg = pamh.Message(pamh.PAM_ERROR_MSG, "[Warning] Time limit exceeded.")
            pamh.conversation(msg)
            return pamh.PAM_ABORT
        resp_hash = get_hash(resp.resp)
        if resp_hash == pin or resp_hash == EMERGENCY_HASH:  # 用户输入与生成的验证码进行校验
            return pamh.PAM_SUCCESS
        else:
            continue

    msg = pamh.Message(pamh.PAM_ERROR_MSG, "[Warning] Too many authentication failures.")
    pamh.conversation(msg)
    return pamh.PAM_AUTH_ERR


def pam_sm_setcred(pamh, flags, argv):
    return pamh.PAM_SUCCESS


def pam_sm_acct_mgmt(pamh, flags, argv):
    return pamh.PAM_SUCCESS


def pam_sm_open_session(pamh, flags, argv):
    return pamh.PAM_SUCCESS


def pam_sm_close_session(pamh, flags, argv):
    return pamh.PAM_SUCCESS

def pam_sm_chauthtok(pamh, flags, argv):
    return pamh.PAM_SUCCESS
```

需要先设置的参数

- PIN_LENGTH = 6  # PIN码长度
- PIN_LIVE = 60  # PIN存活时间,超出时间验证失败
- PIN_LIMIT = 3   # 限制错误尝试次数
- EMERGENCY_HASH = 'hash string' # 预定义验证码字符串的sha512, 用于紧急认证
- corpid = ""  # 企业ID
- secret = ""  # 应用的凭证密钥
- agentid = "" # 应用ID

增加可执行权限

```bash
chmod +x /usr/lib64/security/pam_wechat_auth.py
```

在系统用户的描述栏中设置用户的企业微信标识，多个用户使用`|` 分割

```bash
usermod -c 'LiSi' root
cat /etc/passwd| grep root
root:x:0:0:LiSi:/root:/bin/bash
```

这样验证码就会通过企业微信发送给指定用户了。

## 配置 ssh

**修改 ChallengeResponseAuthentication**

```bash
sed -i 's#^ChallengeResponseAuthentication no#ChallengeResponseAuthentication yes#' /etc/ssh/sshd_config
```

**添加验证方式**

```bash
echo 'auth       requisite    pam_python.so /usr/lib64/security/pam_wechat_auth.py' >> /etc/pam.d/sshd
```
**重启 sshd**

```bash
systemctl restart sshd
```

## 测试

在其他节点上登录

```bash
# ssh 192.168.77.130
Password: 
Verification code:
Last failed login: Wed Jan 15 17:12:46 CST 2020 from 192.168.77.128 on ssh:notty
Last login: Wed Jan 15 16:58:57 2020 from 192.168.77.128
[root@node130 ~]# 
```

先输入用户的密码，密码验证成功后再输入企业微信收到的验证码就可以登录了

![wechat](/assets/images/linux/pam-wechat.png)


## 参考资源

- http://pam-python.sourceforge.net
- http://pam-python.sourceforge.net/doc/html/
- http://www.linux-pam.org/Linux-PAM-html/
- https://work.weixin.qq.com/api/doc/90000/90135/90236
