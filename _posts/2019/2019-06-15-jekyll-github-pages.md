---
layout: post
title:  "使用github page托管jekyll博客"
date: 2019-06-15 17:33:56
categories:  jekyll
tags: install jekyll github
auth: lework
---
* content
{:toc}
本文记录下此次博客的一些安装资源，没有一步一步的记录，只是把重要部分讲到.

**Jekyll（发音/'dʒiːk əl/，"杰克尔"）是一个静态站点生成器，它会根据网页源码生成静态文件。**它提供了模板、变量、插件等功能，所以实际上可以用来编写整个网站。jekyll使用ruby语言编写，文章头部使用yaml格式解析，内容使用jinja2模板语法。

**GitHub**是一个面向[开源](https://baike.baidu.com/item/开源/20720669)及私有[软件](https://baike.baidu.com/item/软件/12053)项目的托管平台，因为只支持git 作为唯一的版本库格式进行托管，故名GitHub。

**GitHub Pages** 是托管在GtiHub的一个仓库，仓库名称需要特别指定才能开启仓库的Github page功能。





## 本地安装jekyll编译环境

1. 安装ruby环境

   下载地址：`https://rubyinstaller.org/`

   选择带devkit的当前版本，比如：`https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-2.5.5-1/rubyinstaller-devkit-2.5.5-1-x64.exe`

   下完之后在本地windows环境进行安装即可。

2. 安装jekylly

   ```
   gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
   gem sources -l
   gem install jekyll
   ```

3. 创建一个新的项目

   ```
   jekyll new my-awesome-site
   ```

4. 启动项目

   ```
   bundle exec jekyll serve
   ```

   编译完成后，jekyll会启动一个本地的4000端口服务，http://localhost:4000通过这个端口，我们就可以看到新的项目



## jekyll资源

1. 官网 [https://jekyllrb.com/](https://jekyllrb.com/) 
2. 中文官网 [https://www.jekyll.com.cn/](https://www.jekyll.com.cn/)  [http://jekyllcn.com](http://jekyllcn.com/)
3. github [https://github.com/jekyll/jekyll](https://github.com/jekyll/jekyll) 
4. 模板资源 [http://jekyllthemes.org/](http://jekyllthemes.org/) 



## 在github上创建GitHub Pages仓库

GitHub Pages repository跟普通的repository是一样的，唯一的区别就是他的名字必须叫做username.gihub.io。这个官方教程 [GitHub Pages](https://pages.github.com/) 写的十分好懂，按这个做完之后你就有了一个你的网址 username.github.io，里面有一句 Hello World！>

>  这个仓库的名字必须为你的github的名字+github+io，即yourname.github.io

![1560432864952](/assets/images/jekyll/1560432864952.png)

创建完仓库，把jekyll项目`push`到仓库，github就会自动帮我们编译jekyll，然后就可以使用仓库名称访问，比如我的: https://lework.github.com

而我的博客用的是主题是<https://github.com/Gaohaoyang/gaohaoyang.github.io>，把这个项目clone下来，里面的内容改改，原博客文章都删除。push到仓库就可以啦。

