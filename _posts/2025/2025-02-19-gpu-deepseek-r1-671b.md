---
layout: post
title: '部署 DeepSeek R1 671B 深度思考模型'
date: '2025-02-12 20:00'
category: 大模型
tags: GPU DeepSeek
author: lework
---
* content
{:toc}

> 本文介绍如何在天翼云 GPU 服务器上部署 DeepSeek R1 深度思考模型。

{% raw %}

## 方案说明

目前部署模型的方式有很多种，比如 Docker、Kubernetes、Ollama、vLLM， SGLAng 等 。 本文主要介绍如何使用 Ollama, vLLM 和 SGLang 在单机上部署 DeepSeek R1 模型。

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

我们本次只运行模型，在  `640G` 显存下，可以运行 `671b参数` 的模型，但也是量化后的版本 `DeepSeek-R1-Q4_K_M`, `DeepSeek-R1-awq`

## 购买服务器

本次使用的是天翼云的 GPU 服务器，规格如下：

- 操作系统: `Ubuntu 22.04.5 LTS`
- GPU: `8*NvidiaH800 640G`
- CPU: `Intel(R) Xeon(R) Platinum 8480+   2路 | 56核112线程`
- 内存: `2048G`
- 硬盘:
  - 系统盘: `480GB SSD`
  - 数据盘: `3200GB NVMeSSD`



## 安装驱动

1. 安装系统内核头文件

```bash
# uname -r
5.15.0-94-generic

yum install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r)
apt-get install gcc g++ cmake ninja-build
```

这里注意下：cuda 对内核和gcc版本是有要求的，内核版本最好要是5版本以上的，具体要求请看  [CUDA系统要求](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#system-requirements)

2. 安装 NVIDIA  & CUDA

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
apt-get update
apt-get -y install cuda-toolkit-12-4 cuda-drivers

echo 'export PATH=/usr/local/cuda/bin:$PATH' >> /etc/profile
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> /etc/profile
echo 'export CUDA_HOME=/usr/local/cuda' >> /etc/profile
source /etc/profile
```

> [cuda 下载地址](https://developer.nvidia.com/cuda-12-4-1-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_network)

3. 验证 gpu 驱动是否安装成功

```bash
# nvidia-smi 
Wed Feb 19 11:20:42 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.90.12              Driver Version: 550.90.12      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H800                    Off |   00000000:18:00.0 Off |                    0 |
| N/A   37C    P0             84W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H800                    Off |   00000000:3A:00.0 Off |                    0 |
| N/A   37C    P0             83W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA H800                    Off |   00000000:4B:00.0 Off |                    0 |
| N/A   37C    P0             82W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA H800                    Off |   00000000:5C:00.0 Off |                    0 |
| N/A   37C    P0             84W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA H800                    Off |   00000000:84:00.0 Off |                    0 |
| N/A   38C    P0             83W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA H800                    Off |   00000000:AC:00.0 Off |                    0 |
| N/A   37C    P0             84W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA H800                    Off |   00000000:B3:00.0 Off |                    0 |
| N/A   37C    P0             85W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA H800                    Off |   00000000:B9:00.0 Off |                    0 |
| N/A   38C    P0             87W /  700W |       4MiB /  81559MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+

# nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2024 NVIDIA Corporation
Built on Thu_Mar_28_02:18:24_PDT_2024
Cuda compilation tools, release 12.4, V12.4.131
Build cuda_12.4.r12.4/compiler.34097967_0
```



## 安装 nvitop

[`nvitop`](https://nvitop.readthedocs.io/en/latest/) 是一款交互式的 NVIDIA GPU 设备性能、资源、进程的实时监测工具。

```bash
apt-get install python3-pip
pip install pip -U
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple
pip install nvitop

# nvitop -1
Wed Feb 19 11:44:37 2025
╒═════════════════════════════════════════════════════════════════════════════╕
│ NVITOP 1.4.2       Driver Version: 550.90.12      CUDA Driver Version: 12.4 │
├───────────────────────────────┬──────────────────────┬──────────────────────┤
│ GPU  Name        Persistence-M│ Bus-Id        Disp.A │ MIG M.   Uncorr. ECC │
│ Fan  Temp  Perf  Pwr:Usage/Cap│         Memory-Usage │ GPU-Util  Compute M. │
╞═══════════════════════════════╪══════════════════════╪══════════════════════╪════════════════════╕
│   0  H800                Off  │ 00000000:18:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   36C    P0    85W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   1  H800                Off  │ 00000000:3A:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   36C    P0    83W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   2  H800                Off  │ 00000000:4B:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   37C    P0    82W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   3  H800                Off  │ 00000000:5C:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   36C    P0    84W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   4  H800                Off  │ 00000000:84:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   37C    P0    83W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   5  H800                Off  │ 00000000:AC:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   36C    P0    84W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   6  H800                Off  │ 00000000:B3:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   37C    P0    85W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
├───────────────────────────────┼──────────────────────┼──────────────────────┼────────────────────┤
│   7  H800                Off  │ 00000000:B9:00.0 Off │ Disabled           0 │ MEM: ▏ 0.0%        │
│ N/A   37C    P0    87W / 700W │   3.50MiB / 79.65GiB │      0%      Default │ UTL: ▏ 0%          │
╘═══════════════════════════════╧══════════════════════╧══════════════════════╧════════════════════╛
[ CPU: ▏ 0.3%                                UPTIME: 22:52:42 ]  ( Load Average:  0.15  0.07  0.08 )
[ MEM: ▍ 0.8%                                   USED: 7.92GiB ]  [ SWP: ▏ 0.0%                     ]

╒══════════════════════════════════════════════════════════════════════════════════════════════════╕
│ Processes:                                                                          root@pm-93f2 │
│ GPU     PID      USER  GPU-MEM %SM  %CPU  %MEM  TIME  COMMAND                                    │
╞══════════════════════════════════════════════════════════════════════════════════════════════════╡
│  No running processes found                                                                      │
╘══════════════════════════════════════════════════════════════════════════════════════════════════╛
```



## 使用 Ollama

[Ollama](https://ollama.com/) 是一个运行大模型的工具，可以看成是大模型领域的 Docker，可以下载所需的大模型并暴露 Ollama API，极大的简化了大模型的部署。

### 安装 ollama

```bash
mkdir -p /data/ollama && cd /data/ollama
wget https://ollama.com/install.sh

# 为了加速下载，可以使用代理
sed -i 's#https://ollama.com/download#https://gh-proxy.com/github.com/ollama/ollama/releases/download/v0.5.11#g' install.sh


# 增加ollama systemd的环境变量
sed -i '/\[Service\]/a\EnvironmentFile=/data/ollama/env' install.sh

cat > /data/ollama/env <<EOF
OLLAMA_HOST=0.0.0.0:11434
OLLAMA_ORIGINS=*
OLLAMA_KEEP_ALIVE=10m
OLLAMA_FLASH_ATTENTION=1
OLLAMA_NUM_PARALLEL=1
OLLAMA_MODELS=/data/ollama/models
EOF

chmod +x install.sh
./install.sh

# 安装完成后，可以查看 ollama 的状态
systemctl status ollama

```

### 运行模型

1. 下载模型

因为 `ollama pull deepseek-r1:671b` 速度比较慢，这里选择了 `unsloth` 的 `DeepSeek-R1-Q4_K_M` 量化模型

```bash
GIT_LFS_SKIP_SMUDGE=1  git clone https://www.modelscope.cn/unsloth/DeepSeek-R1-GGUF.git
git lfs pull  --include="DeepSeek-R1-Q4_K_M/DeepSeek-R1-Q4_K_M*"
# # ls -al
total 789902780
drwxr-xr-x  2 root root         4096 Feb 19 18:43 .
drwxr-xr-x 16 root root         4096 Feb 19 16:30 ..
-rw-r--r--  1 root root  48339779936 Feb 19 18:01 DeepSeek-R1-Q4_K_M-00001-of-00009.gguf
-rw-r--r--  1 root root  49429396320 Feb 19 18:02 DeepSeek-R1-Q4_K_M-00002-of-00009.gguf
-rw-r--r--  1 root root  49527312640 Feb 19 18:02 DeepSeek-R1-Q4_K_M-00003-of-00009.gguf
-rw-r--r--  1 root root  48272509536 Feb 19 17:59 DeepSeek-R1-Q4_K_M-00004-of-00009.gguf
-rw-r--r--  1 root root  49422027488 Feb 19 18:03 DeepSeek-R1-Q4_K_M-00005-of-00009.gguf
-rw-r--r--  1 root root  48272509536 Feb 19 18:00 DeepSeek-R1-Q4_K_M-00006-of-00009.gguf
-rw-r--r--  1 root root  49429396320 Feb 19 18:01 DeepSeek-R1-Q4_K_M-00007-of-00009.gguf
-rw-r--r--  1 root root  46938773696 Feb 19 17:59 DeepSeek-R1-Q4_K_M-00008-of-00009.gguf
-rw-r--r--  1 root root  14798482144 Feb 19 18:21 DeepSeek-R1-Q4_K_M-00009-of-00009.gguf
```

可以看到，git仓库中的gguf是分割后的文件，还不能直接给ollama 使用，需要使用 `llama-gguf-split` 工具合并下文件。

```bash
# 使用代理地址
apt-get install cmake
git clone https://gh-proxy.com/github.com/ggerganov/llama.cpp
cmake -B build
cmake --build build --config Release
echo 'export PATH=/data/llama.cpp/build/bin:$PATH' >> /etc/profile
source /etc/profile

# llama-cli --version
version: 4741 (9626d935)
built with cc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0 for x86_64-linux-gnu

# 合并 gguf
llama-gguf-split --merge DeepSeek-R1-Q4_K_M-00001-of-00009.gguf DeepSeek-R1-Q4_K_M.gguf
```

创建 ollama model

```bash
# 创建描述文件
cat << EOF >DeepSeekQ1_Modelfile
FROM /data/modlescope/DeepSeek-R1-GGUF/DeepSeek-R1-Q4_K_M/DeepSeek-R1-Q4_K_M.gguf
PARAMETER num_gpu 16
PARAMETER num_ctx 8192
PARAMETER temperature 0.6
TEMPLATE "<｜User｜>{{ .System }} {{ .Prompt }}<｜Assistant｜>"
EOF

# 创建模型
ollama create DeepSeek-R1-Q4_K_M -f DeepSeekQ1_Modelfile

# ollama list
NAME                         ID              SIZE      MODIFIED     
DeepSeek-R1-Q4_K_M:latest    1d8ef98ce5e7    404 GB    14 hours ago  
```

2. 运行模型

```bash
# ollama run deepseek-r1:q4 --verbose
>>> 一个汉字具有左右结构，左边是木，右边是乞。这个字是什么？只需回答这个字即可。
<think>
好的，我现在遇到了一个问题：用户问的是一个左右结构的汉字，左边是“木”，右边是“乞”。需要找出这个字是什么。首先我得回忆一下常见的带有木字旁的字，然后再看右边的结构是不是“乞”。

首先，“木”作为偏旁通常出现在左边或右边，但题目说是左右结构，所以确定是在左边。接下来右边的部分是“乞”。“乞”本身是一个独立的汉字，读音为qǐ，比如乞丐的乞。

现在要组合这两个部分。可能的字有吗？我记得有个字是“杞”，不过右边的写法可能不同。“杞”字的右边其实是“己”，而不是“乞”。虽然看起来有点像，但“己”和“乞”在结构上还是不同的，所以这可能不是正
确答案。

那有没有其他可能性呢？“木”加“乞”？或者是否这个字存在？也许是比较少见的汉字或者是古体字？

有时候可能用户会记错右边的部分，比如是否是“气”而不是“乞”。“气”的话加上木就是“気”，但这是日文中的简化字，对应中文的“氣”。所以这也不对。

再仔细想想，“桔”字的右边是“吉”，“橘”比较复杂。有没有可能是“杚”？这个字的结构确实是左边木，右边乞。查一下读音和意思。“杚”读作gū或gài，根据不同的发音有不同的含义。比如在《说文解字》里可
能指平物的器具，或者量米时刮平斗斛的用具。

不过用户问的是现代常用的汉字吗？如果是的话，“杚”可能不太常见，但确实是存在的。另外可能需要确认输入法是否能打出来这个字，以及是否有其他可能的候选。比如“栔”，右边是契，不是乞。“桼”也不
是。那看来正确的答案应该是“杚”。
</think>

杚

total duration:       1m25.847168276s
load duration:        13.150159ms
prompt eval count:    28 token(s)
prompt eval duration: 2.535s
prompt eval rate:     11.05 tokens/s
eval count:           382 token(s)
eval duration:        1m23.298s
eval rate:            4.59 tokens/s
```

通过接口使用 DeepSeek

```bash
# curl http://localhost:11434/api/chat -d '{
  "model": "DeepSeek-R1-Q4_K_M:latest",
  "messages": [{ "role": "user", "content": "hi" }],
  "stream": false
}'
{"model":"deepseek-r1:32b","created_at":"2025-02-13T08:06:34.246236674Z","message":{"role":"assistant","content":"\u003cthink\u003e\n\n\u003c/think\u003e\n\nHello! How can I assist you today? 😊"},"done_reason":"stop","done":true,"total_duration":808218860,"load_duration":14494335,"prompt_eval_count":4,"prompt_eval_duration":29000000,"eval_count":16,"eval_duration":764000000}
```

**注意**: 外部访问时，需要在阿里云安全组中开放 `11434` 端口

更多的 ollama REST API 请参考：[Ollama API](https://www.postman.com/postman-student-programs/ollama-api/documentation/suc47x8/ollama-rest-api)



**监控数据**

```bash
# ollama  ps
NAME                         ID              SIZE      PROCESSOR          UNTIL   
DeepSeek-R1-Q4_K_M:latest    1d8ef98ce5e7    510 GB    84%/16% CPU/GPU    Forever 
```

![image-20250220103050406](\assets\images\2025\image-20250220103050406.png)

可以看到，ollama 使用了 gpu和cpu 做处理。



## 使用 vLLM 

[vLLM](https://docs.vllm.ai/en/stable/) 是一个快速且易于使用的库，专为大型语言模型 (LLM) 的推理和部署而设计。

### 安装

1. 使用 uv 来安装 vLLM

   [uv](https://docs.astral.sh/uv/) 是一个非常快的Python包和项目管理器，用Rust编写。

```bash
wget https://gh-proxy.com/github.com/astral-sh/uv/releases/latest/download/uv-installer.shexport 
# 使用代理
UV_INSTALLER_GITHUB_BASE_URL=https://gh-proxy.com/github.com
bash uv-installer.sh 
source $HOME/.local/bin/env

# 安装python3.12版本
uv python install 3.12

# 安装 vllm python 环境
cd /data/
uv venv vllm --python 3.12 --seed 
source vllm/bin/activate
pip install vllm
```

### 运行模型

1. 下载模型

```bash
# 安装 git-lfs
apt-get install git-lfs

# 国内镜像
git clone https://www.modelscope.cn/cognitivecomputations/DeepSeek-R1-awq.git
```

> **AWQ 模型** https://huggingface.co/cognitivecomputations/DeepSeek-R1-AWQ
>
> - **全称**：Activation-aware Weight Quantization
> - **原理**：根据激活值分布动态量化权重，降低模型精度（如 FP32 → INT4）。
> - **优势**：减少显存占用，提升推理速度，同时尽量保持模型性能。
>
> 

2. 启动模型

```bash
python -m vllm.entrypoints.openai.api_server \
 --served-model-name deepseek-r1 --model /data/modlescope/DeepSeek-R1-awq \
 --trust-remote-code \
 --host 0.0.0.0 \
 --port 8000 \
 --gpu-memory-utilization 0.95 \
 --tensor-parallel-size 8 \
 --enable-prefix-caching \
 --max-model-len 8192 \
 --enforce-eager \
 --dtype float16 
```

nvitop 情况

![image-20250219114746929](C:\Users\yaokl\AppData\Roaming\Typora\typora-user-images\image-20250219114746929.png)

请求模型接口

```
time curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "deepseek-r1",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "一个汉字具有左右结构，左边是木，右边是乞。这个字是什么？只需回答这个字即可。"}
        ]
    }'
```

此时的nvitop监控

![image-20250219114918872](\assets\images\2025\image-20250219114918872.png)



vllm 日志输出

```
INFO 02-19 11:49:01 chat_utils.py:332] Detected the chat template content format to be 'string'. You can set `--chat-template-content-format` to override this.
INFO 02-19 11:49:01 logger.py:39] Received request chatcmpl-592d0358a18a4297a93b4d54caf60070: prompt: '<｜begin▁of▁sentence｜>You are a helpful assistant.<｜User｜>一个汉字具有左右结构，左边是木，右边是乞。这个字是什么？只需回答这个字即可。<｜Assistant｜>', params: SamplingParams(n=1, presence_penalty=0.0, frequency_penalty=0.0, repetition_penalty=1.0, temperature=1.0, top_p=1.0, top_k=-1, min_p=0.0, seed=None, stop=[], stop_token_ids=[], bad_words=[], include_stop_str_in_output=False, ignore_eos=False, max_tokens=8159, min_tokens=0, logprobs=None, prompt_logprobs=None, skip_special_tokens=True, spaces_between_special_tokens=True, truncate_prompt_tokens=None, guided_decoding=None), prompt_token_ids: None, lora_request: None, prompt_adapter_request: None.
INFO 02-19 11:49:01 engine.py:275] Added request chatcmpl-592d0358a18a4297a93b4d54caf60070.
INFO 02-19 11:49:01 metrics.py:455] Avg prompt throughput: 4.6 tokens/s, Avg generation throughput: 0.1 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.1%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:01 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:07 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 4.4 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.2%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:07 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:12 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.3%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:12 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:17 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.3 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.3%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:17 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:22 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.4%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:22 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:27 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.3 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.5%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:27 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:32 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.3 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.6%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:32 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:37 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.3 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.7%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:37 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:42 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.8%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:42 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:47 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.9%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:47 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:53 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.9%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:53 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:49:58 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.0%, CPU KV cache usage: 0.0%.
INFO 02-19 11:49:58 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:03 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.1%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:03 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:08 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.2%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:08 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:13 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.3 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.3%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:13 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:18 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.4%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:18 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:23 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.4%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:23 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:28 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.5%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:28 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:33 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.6%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:33 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:38 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.7%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:38 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO 02-19 11:50:43 metrics.py:455] Avg prompt throughput: 0.0 tokens/s, Avg generation throughput: 6.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 1.8%, CPU KV cache usage: 0.0%.
INFO 02-19 11:50:43 metrics.py:471] Prefix cache hit rate: GPU: 0.00%, CPU: 0.00%
INFO:     127.0.0.1:41194 - "POST /v1/chat/completions HTTP/1.1" 200 OK
```



模型输出

```bash
time curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "deepseek-r1",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "一个汉字具有左右结构，左边是木，右边是乞。这个字是什么？只需回答这个字即可。"}
        ]
    }'
'
{"id":"chatcmpl-592d0358a18a4297a93b4d54caf60070","object":"chat.completion","created":1739936941,"model":"deepseek-r1","choices":[{"index":0,"message":{"role":"assistant","reasoning_content":null,"content":"<think>\n好的，我现在要解决的这个问题是：一个汉字左右结构，左边是“木”，右边是“乞”，这是什么字。我需要先仔细分析这个问题，然后思考可能的答案。\n\n首先，题目给出的结构是左右结构，左边是“木”字旁，右边是“乞”。那么我需要根据这个结构在记忆里搜索或者是查阅可能的汉字。首先，我应该回想学过的汉字中是否有这样的组合。想到的常见字可能有哪些呢？\n\n可能我会想到“树”这个字，不过“树”的结构是比较复杂的，左边是“木”，右边其实是“对”加其他结构。或者有没有其他的字。“木”作为左边，可能构成很多形声字，比如“材”、“树”、“林”等等，但是我需要右边是“乞”的组合。\n\n记得“乞”这个字有时候会和别的部首形成新的字，比如“吃”中的“口”加“乞”。“迄”是走字旁加“乞”，还有“仡”是单人旁加“乞”。而和木字旁组合的，例如“木”加“乞”会是什么字呢？\n\n想到这里，如果这个字存在的话，可能是比较生僻或者不常用的汉字。我需要确认一下是否有这样的汉字符存在。或者可能是另一个发音的字？\n\n可能这时候我需要拆解字形来分析，木字旁加“乞”是否可以组成一个合法的汉字，是否在汉字的规范中存在。例如，“木”加“乞”组合的话，可能念什么音？\n\n根据形声字的规则，右边的“乞”可能提供这个字的发音。比如“仡”读作yì，“吃”读chī，“迄”读qì。如果右边的“乞”在这里作为声旁，那么这个字可能发音为qì之类，比如“气”是qì，但“乞”本身是qǐ发音第三声或轻声可能？\n\n不过有时候发音可能不完全对应。例如，“仡”读yì，可能与“乞”的发音不同。那么这里右边的“乞”可能作为声旁，可能读qǐ或者是qì？\n\n如果存在这个字，可能就是一个木字旁加乞，比如“杚”：左边木，右边乞。那么应该这个字是存在的吗？我需要确认这一点。\n\n可能我需要查一下汉字词典或者相关资料。比如，现在的输入法里输入“muqi”会不会有这个字出现？或者查阅《现代汉语词典》中的索引。\n\n假设用户问的这个字确实存在，那么正确的答案应该是“杚”这个字对吗？那它的发音可能是gài或gǔ。这两个读音，可能根据不同的意思有不同的发音。\n\n例如，根据《现代汉语词典》，“杚”读gài时，同“概”，是平斗斛的木棍；读gǔ时，意为“摩”。但可能这个字相对生僻，不是常用字。\n\n所以，用户的问题的答案应该就是这个字：“杚”。需要确认的是这个字的结构符合题目的左右结构，木在左，乞在右。是的，结构是这样的。\n\n那么总结问题，正确回答就是这个字为“杚”。\n</think>\n\n杚","tool_calls":[]},"logprobs":null,"finish_reason":"stop","stop_reason":null}],"usage":{"prompt_tokens":33,"total_tokens":680,"completion_tokens":647,"prompt_tokens_details":null},"prompt_logprobs":null}

real    0m55.043s
user    0m0.000s
sys     0m0.005s
```



### vllm常用推理参数:

#### 1.  网络参数

| 参数                             | 类型   | 默认值    | 说明                                                         |
| -------------------------------- | ------ | --------- | ------------------------------------------------------------ |
| `--host`                         | string | localhost | API服务监听地址，生产环境建议设为`0.0.0.0`以允许外部访问     |
| `--port`                         | int    | 8000      | API服务监听端口号                                            |
| `--uvicorn-log-level`            | enum   | info      | 控制Uvicorn框架日志粒度，可选:`debug,trace,info,warning,error,critical` |
| `--allowed-origins`              | list   | 空        | 允许跨域请求的来源列表（例：`http://example.com`）           |
| `--allow-credentials`            | flag   | False     | 允许发送Cookies等凭证信息                                    |
| `--ssl-keyfile`/`--ssl-certfile` | path   | 无        | HTTPS所需的私钥和证书文件路径                                |

------

#### 2. 硬件资源管理

| 参数                       | 类型  | 默认值 | 说明                                   |
| -------------------------- | ----- | ------ | -------------------------------------- |
| `--tensor-parallel-size`   | int   | 1      | 张量并行度（必须等于物理GPU数量）      |
| `--gpu-memory-utilization` | float | 0.9    | GPU显存利用率阈值（0.9=90%显存上限）   |
| `--block-size`             | enum  | 16     | 连续Token块大小，取值`8/16/32/64/128`  |
| `--device`                 | enum  | auto   | 执行设备类型(`cuda/tpu/hpu/xpu/cpu`等) |

------

#### 3. 异构存储配置

| 参数               | 类型 | 默认值     | 说明                                                 |
| ------------------ | ---- | ---------- | ---------------------------------------------------- |
| `--swap-space`     | int  | 4          | 每个GPU的CPU换页空间大小（GiB）                      |
| `--cpu-offload-gb` | int  | 0          | 每GPU使用CPU内存扩展显存的GiB数（需高速CPU-GPU互联） |
| `--max-cpu-loras`  | int  | =max_loras | CPU内存缓存的最大LoRA适配器数量                      |

------

#### 4. 模型基础参数

| 参数               | 类型   | 默认值   | 说明                                    |
| ------------------ | ------ | -------- | --------------------------------------- |
| `--model`          | string | 必填     | 模型名称（如`gpt-3.5-turbo`）或本地路径 |
| `--dtype`          | enum   | auto     | 计算精度控制，常用`float16/bfloat16`    |
| `--max-model-len`  | int    | 自动获取 | 模型最大支持的上下文长度                |
| `--tokenizer-mode` | enum   | auto     | Tokenizer模式（`auto`自动选择快速实现） |

------

#### 5. 高级加载控制

| 参数                  | 类型 | 默认值 | 说明                                            |
| --------------------- | ---- | ------ | ----------------------------------------------- |
| `--load-format`       | enum | auto   | 权重加载协议，优先`safetensors`更安全           |
| `--config-format`     | enum | auto   | 配置格式`hf/mistral`或自动检测                  |
| `--trust-remote-code` | flag | False  | 加载HuggingFace自定义代码时必须启用，有安全风险 |
| `--hf-overrides`      | JSON | 无     | 动态覆盖HuggingFace模型配置（如调整隐藏层维度） |

------

#### 6. 推理参数限制

| 参数                                  | 类型 | 默认值   | 说明                             |
| ------------------------------------- | ---- | -------- | -------------------------------- |
| `--max-num-seqs`                      | int  | 256      | 单批次允许多少序列并行处理       |
| `--max-num-batched-tokens`            | int  | 动态调整 | 每个推理阶段处理的Token总数上限  |
| `--max-logprobs`                      | int  | 5        | 返回每个位置的概率最高前N个token |
| `--speculative-disable-by-batch-size` | int  | 无       | 排队请求超过该阈值时关闭推测解码 |

------

#### 7. 安全与许可控制

| 参数                         | 类型   | 默认值 | 说明                                               |
| ---------------------------- | ------ | ------ | -------------------------------------------------- |
| `--api-key`                  | string | 无     | API访问密钥，设置后所有请求需包含`Authorization`头 |
| `--allowed-local-media-path` | path   | 无     | 允许服务端访问的本地媒体路径（仅可信环境启用）     |
| `--cert-reqs`                | enum   | 无     | SSL证书验证级别（参考Python ssl模块）              |

------

#### 8. 量化配置

| 参数                    | 类型 | 默认值 | 说明                                |
| ----------------------- | ---- | ------ | ----------------------------------- |
| `--quantization`        | enum | 无     | 权重量化方法，如`awq/gptq/marlin`等 |
| `--kv-cache-dtype`      | enum | auto   | KV缓存量化类型（`fp8/fp8_e5m2`等）  |
| `--lora-dtype`          | enum | auto   | LoRA适配器的量化精度设置            |
| `--calculate-kv-scales` | flag | False  | 动态计算FP8量化比例                 |

------

**调优建议**：

1. **操作建议**：
   - 显存使用较高时优先调整`--gpu-memory-utilization`和`--block-size`
   - 多GPU环境严格保证`tensor-parallel-size`与GPU数量一致
   - 高安全场景必须设置`--api-key`和`--allowed-origins`白名单
2. **性能关键控制点**：
   - CPU换页大小 (`swap-space`) 与批处理容量 (`max-num-seqs`) 的平衡
   - 权重量化 (`quantization`) 和KV缓存量化 (`kv-cache-dtype`) 的组合选择
   - Token块大小 (`block-size`) 对内存利用率和吞吐量的影响

## 使用SGLang

[SGLang](https://docs.sglang.ai/index.html)（Scalable Graph Language）是一种基于图计算的AI大模型推理加速技术，通过将复杂的计算任务分解为图结构，利用图计算的高效性和并行性，显著提升推理速度。



### 安装

```bash
uv venv sglang --python 3.12 --seed
source sglang/bin/activate
  
pip install sgl-kernel --force-reinstall --no-deps
pip install "sglang[all]>=0.4.3.post2" --find-links https://flashinfer.ai/whl/cu124/torch2.5/flashinfer-python
```



### 运行模型

```bash
python -m sglang.launch_server --model-path /data2/Qwen2.5-72B-Instruct --served-model-name Qwen2.5-72B-Instruct --tp 8 --trust-remote-code  --host 0.0.0.0 --port 8000

# 启动如果以下遇到错误，需要回退 transformers 版本
ImportError: cannot import name 'is_valid_list_of_images' from 'transformers.models.mllama.image_processing_mllama' (/data/sglang/lib/python3.12/site-packages/transformers/models/mllama/image_processing_mllama.py)

# 回退 transformers 版本
pip uninstall transformers
pip install transformers==4.48.3
```





{% endraw %}
