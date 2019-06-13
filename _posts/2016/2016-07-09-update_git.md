---
layout: post
title:  "更新git"
categories: git
tags: git update
excerpt: 更新git
auth: lework
---
* content
{:toc}

本次服务器的系统版本是：

```bash
CentOS release 6.7 (Final)
```


解决git clone时报错：`The requested URL returned error: 401 Unauthorized while accessing`

```bash
wget -O git.zip https://github.com/git/git/archive/master.zip
unzip git.zip
cd git-master
autoconf
./configure --prefix=/usr/local
make && make install
mv  /usr/bin/git /usr/bin/git_old
ln -s /usr/local/bin/git /usr/bin/git
```