---
layout: post
title: 'Dify 开源版增加工作空间'
date: '2025-02-27 20:00'
category: ai
tags: ai dify
author: lework
---
* content
{:toc}

{% raw %}

[Dify](https://docs.dify.ai/zh-hans)是一款开源的大语言模型(LLM) 应用开发平台。它融合了后端即服务（Backend as Service）和 LLMOps 的理念，使开发者可以快速搭建生产级的生成式 AI 应用。即使你是非技术人员，也能参与到 AI 应用的定义和数据运营过程中。

在安装开源版本 [Dify](https://github.com/langgenius/dify.git) 时，后台只有一个工作空间，且无法在后台创建新的工作空间。为了解决这个问题，我们可以手动创建工作空间。 本次使用的版本是 `1.0.0`。





### 1. 创建工作空间

#### 1.1 生成公钥私钥

进入 dify-api 容器中， 执行下面的命令，生成公钥和私钥。

```bash
cd /tmp/
openssl genrsa -out private.pem 2048
openssl rsa -pubout -in private.pem -out public.pem

# cat public.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAhhG0d/sHgpDxReXumzu1
a+/EZgGlJQhOdT2WkjoY3JDmVAklBdJMr0nLBlgpXsyZg9eJUMSseKeSfMLegYxE
BhNaDPAFpR5AccCFF4RvBPjKU43FdR5MOTSsXCqZzWSVF8v16bDCMdv3H81TEdMd
Z2QJ1Aqm5IVHkEoSXTgpVRBx/onNVndKAA6+hvpwbn4MVpC8NSmGFteHjS959ZG+
ViCJA/rBxckZjufJm8LScKPQvsa2kwqdlCXKwPxFZ+VqGbcJIQcSogoI8UjVd8mC
L43971h/1WZDwyG4lN4EXq70FfPUQa6H9zCahy8yn+cdRgd5PgquZDuqqGI/aaLt
nQIDAQAB
-----END PUBLIC KEY-----
```

#### 1.2 创建工作空间

在数据库中插入记录

```sql
INSERT INTO "public"."tenants" ("name", "encrypt_public_key", "plan", "status",) VALUES ('dify-demo', '-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAhhG0d/sHgpDxReXumzu1
a+/EZgGlJQhOdT2WkjoY3JDmVAklBdJMr0nLBlgpXsyZg9eJUMSseKeSfMLegYxE
BhNaDPAFpR5AccCFF4RvBPjKU43FdR5MOTSsXCqZzWSVF8v16bDCMdv3H81TEdMd
Z2QJ1Aqm5IVHkEoSXTgpVRBx/onNVndKAA6+hvpwbn4MVpC8NSmGFteHjS959ZG+
ViCJA/rBxckZjufJm8LScKPQvsa2kwqdlCXKwPxFZ+VqGbcJIQcSogoI8UjVd8mC
L43971h/1WZDwyG4lN4EXq70FfPUQa6H9zCahy8yn+cdRgd5PgquZDuqqGI/aaLt
nQIDAQAB
-----END PUBLIC KEY-----', 'basic', 'normal');
```

> 注意：工作空间的名称不要重复，`encrypt_public_key` 需要使用上面生成的公钥。

查询生成的工作空间的 id

```sql
select id from tenants where name = 'dify-demo'
```

> 注意：工作空间的名称需要和上面插入的名称一致。

查询用户的 id

```sql
SELECT "id" FROM "accounts" where name ='demo'
```

添加用户到工作空间，

```sql
INSERT INTO "public"."tenant_account_joins" ("tenant_id", "account_id", "role", "invited_by", "current") VALUES ('63500927-a6e3-496c-8291-66042600006c','9a9a68c8-cfa7-417d-996c-8df3aaf26c86', 'admin', NULL, 't');
```

> `account_id` 是上面查询到的用户 id，`tenant_id` 是上面查询到的工作空间 id。

#### 1.3 上传私钥

将工作空间的私钥放到工作空间的目录下，

```bash
cd /app/api/storage/privkeys
mkdir -p 63500927-a6e3-496c-8291-66042600006c
cp /tmp/private.pem 63500927-a6e3-496c-8291-66042600006c/private.pem
```

> 注意：工作空间的目录名称需要和上面查询到的工作空间 id 一致。 /app/api/storage 是 dify 的 STORAGE_LOCAL_PATH 配置的目录。

### 2. 使用工作空间

使用刚才加入工作空间的用户登录 `dify`， 在 `dify`后台中，点击左上角的工作空间，就能看到刚添加的工作空间了。

![image-20250228103243092](\assets\images\2025\image-20250228103243092.png)

选择添加的 `dify-demo` 工作空间，进行安装插件，把常用的模型都安装上。

![image-20250228103139123](\assets\images\2025\image-20250228103139123.png)

![image-20250228103345079](\assets\images\2025\image-20250228103345079.png)

在 右上角的用户设置，添加模型的 `API Key`

![image-20250228103419577](\assets\images\2025\image-20250228103419577.png)

然后在 工作室页面，创建一个 聊天助手 应用。

![image-20250228103745388](\assets\images\2025\image-20250228103745388.png)

在调试和预览中，选择多个模型提供商，来测试下联通。

![image-20250228103819503](\assets\images\2025\image-20250228103819503.png)

到此，恭喜你，又多了一个工作空间啦。其他功能都可以正常使用。


### 已知问题

1. 在一个工作空间邀请用户后，在其他工作空间继续邀请此用户时，不会再发送邀请邮件到用户邮箱里面了，且用户在工作空间的状态是被邀请过的。 
