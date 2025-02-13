---
layout: post
title: '阿里云GPU服务器部署 DeepSeek R1 深度思考模型'
date: '2025-02-12 20:00'
category: 大模型
tags: 阿里云 GPU DeepSeek
author: lework
---
* content
{:toc}

> 本文介绍如何在阿里云 GPU 服务器上部署 DeepSeek R1 深度思考模型。

{% raw %}

## 方案说明

目前部署模型的方式有很多种，比如 Docker、Kubernetes、Ollama、vLLM 等，这里我们使用 Ollama + DeepSeek 。

**Ollama**

[Ollama](https://ollama.com/)  是一个运行大模型的工具，可以看成是大模型领域的 Docker，可以下载所需的大模型并暴露 Ollama API，极大的简化了大模型的部署。

**DeepSeek**

[DeepSeek](https://www.deepseek.com/) 系列模型是由深度求索（DeepSeek）公司推出的大语言模型。

DeepSeek-R1 模型包含 671B 参数，激活 37B，在后训练阶段大规模使用了强化学习技术，在仅有极少标注数据的情况下，极大提升了模型推理能力，尤其在数学、代码、自然语言推理等任务上。

DeepSeek-V3 为 MoE 模型，671B 参数，激活 37B，在 14.8T Token 上进行了预训练，在长文本、代码、数学、百科、中文能力上表现优秀。

DeepSeek-R1-Distill 系列模型是基于知识蒸馏技术，通过使用 DeepSeek-R1 生成的训练样本对 Qwen、Llama 等开源大模型进行微调训练后，所得到的增强型模型。




**DeepSeek 模型配置要求**

| **模型参数规模**   | **典型用途**             | **CPU 建议**                              | **GPU 建议**                                    | **内存建议 (RAM)** | **磁盘空间建议**  | **适用场景**                      |
| ------------------ | ------------------------ | ----------------------------------------- | ----------------------------------------------- | ------------------ | ----------------- | --------------------------------- |
| **1.5b (15 亿)**   | 小型推理、轻量级任务     | 4 核以上 (Intel i5 / AMD Ryzen 5)         | 可选，入门级 GPU (如 NVIDIA GTX 1650, 4GB 显存) | 8GB                | 10GB 以上 SSD     | 小型 NLP 任务、文本生成、简单分类 |
| **7b (70 亿)**     | 中等推理、通用任务       | 6 核以上 (Intel i7 / AMD Ryzen 7)         | 中端 GPU (如 NVIDIA RTX 3060, 12GB 显存)        | 16GB               | 20GB 以上 SSD     | 中等规模 NLP、对话系统、文本分析  |
| **14b (140 亿)**   | 中大型推理、复杂任务     | 8 核以上 (Intel i9 / AMD Ryzen 9)         | 高端 GPU (如 NVIDIA RTX 3090, 24GB 显存)        | 32GB               | 50GB 以上 SSD     | 复杂 NLP、多轮对话、知识问答      |
| **32b (320 亿)**   | 大型推理、高性能任务     | 12 核以上 (Intel Xeon / AMD Threadripper) | 高性能 GPU (如 NVIDIA A100, 40GB 显存)          | 64GB               | 100GB 以上 SSD    | 大规模 NLP、多模态任务、研究用途  |
| **70b (700 亿)**   | 超大规模推理、研究任务   | 16 核以上 (服务器级 CPU)                  | 多 GPU 并行 (如 2x NVIDIA A100, 80GB 显存)      | 128GB              | 200GB 以上 SSD    | 超大规模模型、研究、企业级应用    |
| **671b (6710 亿)** | 超大规模训练、企业级任务 | 服务器级 CPU (如 AMD EPYC / Intel Xeon)   | 多 GPU 集群 (如 8x NVIDIA A100, 320GB 显存)     | 256GB 或更高       | 1TB 以上 NVMe SSD | 超大规模训练、企业级 AI 平台      |

我们本次只运行模型，在 阿里云 `24G` 显存下，可以运行 `32b参数` 的模型

## 购买服务器

本次使用的是阿里云的 GPU 服务器，规格如下：

- 实例规格: `ecs.gn7i-c16g1.4xlarge`
- 操作系统: `Alibaba Cloud Linux 3.2104 LTS 64位`
- GPU: `NVIDIA Tesla A10 `
- CPU: `16 核(vCPU)`
- 内存: `60 GiB`
- 硬盘:
  - 系统盘: `40 GiB`
  - 数据盘: `500 GiB`
- 配置费用：`￥5369.81`

更多 ECS 规格请参考：[ECS 实例规格可购买地域总览](https://ecs-buy.aliyun.com/instanceTypes#/instanceTypeByRegion)

## 安装驱动

1. 安装系统内核头文件

```bash
yum install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r)
```

2. 安装 NVIDIA 驱动

```bash
wget https://cn.download.nvidia.com/tesla/470.161.03/NVIDIA-Linux-x86_64-470.161.03.run
chmod +x NVIDIA-Linux-x86_64-470.161.03.run
sh NVIDIA-Linux-x86_64-470.161.03.run
```

3. 验证 gpu 驱动是否安装成功

```bash
# nvidia-smi
Thu Feb 12 19:17:52 2025
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 470.161.03   Driver Version: 470.161.03   CUDA Version: 11.4     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A10          Off  | 00000000:00:08.0 Off |                    0 |
|  0%   29C    P8    17W / 150W |      2MiB / 22731MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

更多介绍可以参考阿里云官方文档：[安装 GPU 驱动]()

## 安装 ollama

```bash
mkdir -p /data/ollama && cd /data/ollama
wget https://ollama.com/install.sh

# 为了加速下载，可以使用代理
sed -i 's#https://ollama.com/download#https://gh-proxy.com/github.com/ollama/ollama/releases/download/v0.5.8#g' install.sh

# 修改家目录, data目录是数据盘
sed -i 's#-U -m -d /usr/share/ollama ollama#-U -m -d /data/ollama ollama#g' install.sh

# 增加ollama systemd的环境变量
sed -i '/\[Service\]/a\EnvironmentFile=/data/ollama/env' install.sh

cat > /data/ollama/env <<EOF
OLLAMA_HOST=0.0.0.0:11434
OLLAMA_ORIGINS=*
EOF

chmod +x install.sh
./install.sh

# 安装完成后，可以查看 ollama 的状态
systemctl status ollama

```

## 运行 DeepSeek 模型

下载 [DeepSeek](https://ollama.com/library/deepseek-r1) 模型

```bash
ollama pull deepseek-r1:32b
```

运行模型

```bash
# ollama run deepseek-r1:32b
>>> /set verbose
Set 'verbose' mode.
>>> 你好，给我讲个可笑的笑话  
<think>
好的，用户让我讲个可笑的笑话。首先，我得考虑笑话的类型，要适合大多数人的口味，不能太冷僻或者有冒犯性。动物类的笑话通常比较受欢迎，也比较容易让人发笑。

然后，我需要想一个有趣的场景。海滩是个不错的选择，因为它给人一种轻松愉快的感觉。接下来，确定角色，海鸥和螃蟹都是常见的海洋生物，容易让人联想到它们的特点。

接下来，构思对话。海鸥问问题，螃蟹回答，形成互动。问题要有悬念，比如为什么不能把袜子留在海滩上。这样的问题能引起听众的好奇心，想知道答案是什么。

然后，设计一个出人意料的转折。袜子会自己走？这听起来有点奇怪，但其实是因为螃蟹喜欢在水里活动，所以袜子会被带入水中，显得像是自己走了。这个解释既有趣又合理，让人觉得好笑。

最后，加上一句调侃的话，比如“蟹总”这样称呼，增加亲切感和幽默感，让用户感觉更自然，更容易发笑。

整体上，笑话的结构清晰，有开头、发展和结尾，符合一般的幽默模式，应该能引起用户的笑声。
</think>

当然可以！这是一个经典的笑话：

有一天，一只海鸥问一只螃蟹：“为什么你不能把袜子留在海滩上？”

螃蟹回答说：“因为它们会自己走！”

海鸥愣了一下，说：“哦，原来是这样！不过……你的袜子呢？”

螃蟹不好意思地说：“我还没学会穿袜子。”

哈哈，是不是有点好笑？😄

total duration:       16.75064966s
load duration:        14.872699ms
prompt eval count:    12 token(s)
prompt eval duration: 42ms
prompt eval rate:     285.71 tokens/s
eval count:           330 token(s)
eval duration:        16.693s
eval rate:            19.77 tokens/s

```

通过接口使用 DeepSeek

```bash
# curl http://localhost:11434/api/chat -d '{
  "model": "deepseek-r1:32b",
  "messages": [{ "role": "user", "content": "hi" }],
  "stream": false
}'
{"model":"deepseek-r1:32b","created_at":"2025-02-13T08:06:34.246236674Z","message":{"role":"assistant","content":"\u003cthink\u003e\n\n\u003c/think\u003e\n\nHello! How can I assist you today? 😊"},"done_reason":"stop","done":true,"total_duration":808218860,"load_duration":14494335,"prompt_eval_count":4,"prompt_eval_duration":29000000,"eval_count":16,"eval_duration":764000000}
```

**注意**: 外部访问时，需要在阿里云安全组中开放 `11434` 端口

更多的 ollama REST API 请参考：[Ollama API](https://www.postman.com/postman-student-programs/ollama-api/documentation/suc47x8/ollama-rest-api)



**Token生成对比**

用户输入：`你好，给我讲个可笑的笑话`

| deepseek-r1:14b                                              | deepseek-r1:32b                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| total duration:       16.497871632s<br/>load duration:        14.803653ms<br/>prompt eval count:    12 token(s)<br/>prompt eval duration: 574ms<br/>prompt eval rate:     20.91 tokens/s<br/>eval count:           620 token(s)<br/>eval duration:        15.907s<br/>eval rate:            38.98 tokens/s | total duration:       16.75064966s<br/>load duration:        14.872699ms<br/>prompt eval count:    12 token(s)<br/>prompt eval duration: 42ms<br/>prompt eval rate:     285.71 tokens/s<br/>eval count:           330 token(s)<br/>eval duration:        16.693s<br/>eval rate:            19.77 tokens/s |

网络上分享的一些 Token 生成数据

https://llm.aidatatools.com/results-linux.php 



## 社区集成

在放开 ollama 的 API 访问后，我们可以通过社区集成，快速的使用 DeepSeek 模型。

**WEB 项目**

- [Open WebUI](https://github.com/open-webui/open-webui)
- [Lobe Chat](https://github.com/lobehub/lobe-chat)
- [LibreChat](https://github.com/danny-avila/LibreChat)
- [Dify.AI](https://github.com/langgenius/dify)
- [RAGFlow](https://github.com/infiniflow/ragflow)

**桌面项目**

- [Chatbox](https://github.com/Bin-Huang/Chatbox)
- [Cherry Studio](https://github.com/kangfenmao/cherry-studio) (Desktop client with Ollama support)

{% endraw %}
