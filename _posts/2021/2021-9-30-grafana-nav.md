---
layout: post
title: '运维小需求：Grafana 面板中展示网址链接'
date: '2021-09-30 19:02'
category: 运维小需求
tags: 运维小需求 grafana
author: lework
---
* content
{:toc}

## 需求

在 Grafana 面板中展示网址链接。




## 实现

技术栈： react, grafana plugin.

大概流程：

1. 使用 `@grafana/toolkit` 工具创建插件脚手架。
2. 在 `module.ts` 文件中指定面板的配置选项, 插件入口。
3. 在 `types.ts` 文件中指定常量。
4. 在 `NavPanel.tsx` 文件中展示数据。
5. 在 `plugin.json` 文件中用来配置一些基本属性。
6. 开发完成后，即可提交到官方获取公共签名。

> https://grafana.com/docs/grafana/latest/developers/plugins/

## 安装使用

下载包：https://github.com/lework/grafana-lenav-panel/releases/download/v1.0.0/grafana-lenav-panel-v1.0.0.zip

将下载的插件包，解压到 `D:\grafana-8.1.5\data\plugins\grafana-lenav-panel` grafana 插件存储目录下，还需重启下grafana。

> 只能在 lcoalhost:3000 域名下使用, 其他域名需要公共签名。

其他域名需要修改配置, 允许未签名的插件。

```ini
[plugins]
enable_alpha = true
allow_loading_unsigned_plugins = lework-lenav-panel,
plugin_admin_enabled = true
```

## demo 

- 源码:  [https://github.com/lework/grafana-lenav-panel](https://github.com/lework/grafana-lenav-panel)

![lenav-screenshot-1](https://cdn.jsdelivr.net/gh/lework/grafana-lenav-panel@master/src/img/lenav-screenshot-1.png)
![lenav-screenshot-2](https://cdn.jsdelivr.net/gh/lework/grafana-lenav-panel@master/src/img/lenav-screenshot-2.png)
