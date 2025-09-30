---
layout: post
title: '使用 Cursor 创建 MCP Server：新手友好指南'
date: '2025-03-06 20:00'
category: ai
tags: ai cursor coding
author: lework
---

- content
  {:toc}

{% raw %}

## 前言

欢迎来到这篇 Cursor IDE 使用指南！无论你是编程新手还是有经验的开发者，本教程都将帮助你了解如何利用 Cursor IDE 的强大 AI 功能来创建一个 MCP Server。

在开始之前，让我们先了解一些基本概念：

- **Cursor IDE**：一款集成了 AI 功能的代码编辑器，能够帮助你更高效地编写代码
- **Cursor 规则**：可以控制 AI 模型行为的指令集，类似于系统提示词
- **MCP (Model Context Protocol)**：一种让 AI 模型与外部服务交互的协议
- **LangGPT**：一个面向大语言模型的自然语言编程框架，帮助我们更好地构建提示词

本教程将通过三个简单步骤，带你实现一个返回 IP 地址归属地的 MCP Server：

1. 创建一个 LangGPT 风格的提示词生成助手
2. 利用这个助手生成一个 MCP 专家
3. 让 MCP 专家帮我们实现一个简单的 MCP 应用

准备好了吗？让我们开始吧！

## 详细步骤

### 步骤一：创建 LangGPT 风格提示词生成助手

首先，我们需要创建一个能够生成 LangGPT 格式提示词的助手。这将帮助我们更好地构建后续的 MCP 专家提示词。

1. 在 Cursor 中，使用`Ask 模式`（通过点击界面右下角的对话框图标进入）
2. 输入以下提示词：

```
@https://github.com/langgptai/LangGPT 学习这个LangGPT的仓库代码，帮我创建一个专门生成LangGPT格式的大模型prompt助手
```

3. AI 会生成一个 LangGPT 风格的提示词助手

![LangGPT助手生成结果](\assets\images\2025\image-20250305100900919.png)

4. 将生成的提示词保存到 Cursor 规则中：
   - 按下`Ctrl+Shift+P`打开命令面板
   - 搜索并选择`New Cursor Rule`

![打开Cursor规则](\assets\images\2025\image-20250305101005302.png)

5. 将生成的提示词粘贴进去

![创建规则](\assets\images\2025\image-20250305101113036.png)

### 步骤二：创建 MCP 专家提示词

有了 LangGPT 助手后，我们可以创建一个专门的 MCP 专家提示词。

1. 在 Cursor 中，使用 `Ask 模式`
2. 输入以下提示词：

```
@https://modelcontextprotocol.io/introduction @https://modelcontextprotocol.io/quickstart/server @https://modelcontextprotocol.io/quickstart/client 学习mcp的文档，使用LangGPT助手生成一个MCP专家
```

3. AI 会生成一个 MCP 专家的提示词

![MCP专家提示词](\assets\images\2025\image-20250305101610647.png)

4. 将这个提示词保存在你的项目中，以便后续使用

### 步骤三：利用 MCP 专家创建 IP 归属地查询服务

现在，我们可以使用 MCP 专家来帮助我们创建一个实际的应用了。

1. 在 Cursor 中，使用`Agent 模式`
2. 输入以下提示词：

```
你是一个MCP专家，帮我在当前目录下创建一个 MCP Server，用来返回ip地址的归属地，MCP Server 提供 http sse 方式传输数据，使用 https://api.ip.sb/geoip 接口来查询ip信息。可以参考@https://github.com/modelcontextprotocol/typescript-sdk，参考接口文档 @https://ip.sb/api/ 来调用接口查询ip信息。
```

3. AI 会开始生成代码和指导

![MCP应用生成](\assets\images\2025\image-20250305103617870.png)

4. 跟随 AI 的指导，点击"Accept"接受生成的代码和建议, 按照 AI 的指导进行简单调试，确保 MCP Server 能够正常运行
5. 完成所有步骤后，你将得到一个完整的工程项目

![运行结果](\assets\images\2025\image-20250305150147117.png)

### 步骤四：在 Cursor 中配置 MCP Server

最后，我们需要在 Cursor 中配置 MCP Server，以便在 agent 模式下使用它。

1. 按下`Ctrl+Shift+J`打开 Cursor 设置页面
2. 添加你刚刚创建的 MCP Server

![添加MCP Server](\assets\images\2025\image-20250305145955381.png)

![MCP Server配置](\assets\images\2025\image-20250305145840675.png)

3. 配置完成后，你就可以在 agent 模式下使用你的 MCP Server 了

![使用MCP Server](\assets\images\2025\image-20250305154436200.png)

代码放在了 https://github.com/lework/mcp-server-ip.git

## 参考资源

- [Cursor 规则官方文档](https://docs.cursor.com/context/rules-for-ai)
- [Cursor 规则 Hub](https://dotcursorrules.com/)
- [MCP 官方文档](https://modelcontextprotocol.io/)
- [LangGPT 项目](https://github.com/langgptai/LangGPT)

如果你有任何问题或建议，欢迎在评论区留言。祝你编程愉快！

{% endraw %}
