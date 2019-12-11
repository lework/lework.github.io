# lework.github.io ![GitHub](https://img.shields.io/github/license/lework/lework.github.io)

personal blog

## 快速启动

### 1. 安装 ruby 和 jekyll 环境

这一步和第5步主要是为了让博客系统在本地跑起来，如果不想在本地运行，可以无视这两步，但我还是强烈建议试着先在本地跑起来，没有什么问题后再推送的 GitHub 上。

Windows 用户可以直接使用 [RubyInstaller](http://rubyinstaller.org/) 安装 ruby 环境。后续的操作中可能还会提示安装 DevKit，根据提示操作即可。

建议使用 [RubyGems 镜像- Ruby China](https://gems.ruby-china.org/) 安装 jekyll。

安装 jekyll 命令如下

```
gem install jekyll
```

### 2. 复制博客主题代码

可以直接 clone 、下载 或 fork 这个仓库的代码即可

### 3. 修改参数

主要修改 `_config.yml` 中的参数和自己的网站小图`favicon.ico`

### 4. 写文章

`_posts`目录下存放文章信息，文章头部注明 layout(布局)、title、date、categories、tags、author(可选)、mathjax(可选，点击[这里](https://www.mathjax.org/)查看更多关于`Mathjax`)，如下：

```
---
layout: post
title:  "对这个 jekyll 博客主题的改版和重构"
date:   2016-03-12 11:40:18 +0800
categories: jekyll
tags: jekyll 端口 markdown Foxit RubyGems HTML CSS
author: Haoyang Gao
mathjax: true
---
```

下面这两行代码为产生目录时使用
```
* content
{:toc}
```

文章中存在的4次换行为摘要分割符，换行前的内容会以摘要的形式显示在主页Home上，进入文章页不影响。

换行符的设置见配置文件`_config.yml`的 excerpt，如下：

```yml
# excerpt
excerpt_separator: "\n\n\n\n"
```

使用 markdown 语法写文章。

代码风格与 GitHub 上 README 或 issue 中的一致。使用3个\`\`\`的方式。

### 5. 本地运行

本地执行

```
jekyll s
```

显示

```
Configuration file: E:/lework.github.io/_config.yml
            Source: E:/lework.github.io
       Destination: E:/lework.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
                    done in 9.773 seconds.
 Auto-regeneration: enabled for 'E:/lework.github.io'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```

在本地访问 localhost:4000 即可看到博客主页。
