---
layout: post
title: "Ansible 插件 之 【CMDB】"
date: "2016-11-19 22:11:01"
categories: Ansible
excerpt: "Github地址： https://github.com/fboender/ansible-cmdb 从facts收集信息，生成主机概述 安装 ..."
auth: lework
---
* content
{:toc}

Github地址： https://github.com/fboender/ansible-cmdb

从facts收集信息，生成主机概述

## 安装
---
```
wget https://github.com/fboender/ansible-cmdb/releases/download/1.17/ansible-cmdb-1.17.tar.gz
tar zxf ansible-cmdb-1.17.tar.gz 
cd ansible-cmdb-1.17
make install
```

## 使用
---

生成所有主机得facts信息
```
ansible -m setup --tree out/ all
```

生成web页面
```
ansible-cmdb out/ > overview.html
```

![Paste_Image.png](/assets/images/Ansible/3629406-afafa21b010daca0.png)

默认模板采用html_fancy，文件存放在/usr/local/lib/ansible-cmdb/ansiblecmdb/data/tpl/html_fancy.tpl

如果facts用了本地缓存，-f指定缓存目录即可。
```
ansible-cmdb -f /path/to/facts/dir > overview.html
```

以资产列表得形式统计出ansible主机信息。
ansible-cmdb -t txt_table --columns name,os,ip,mem,cpus out/

![Paste_Image.png](/assets/images/Ansible/3629406-69d31f2d27ca04e2.png)


输出csv格式的主机信息
```
ansible-cmdb -t csv  -i hosts out/
```

![Paste_Image.png](/assets/images/Ansible/3629406-71ba789243f1db33.png)

输出sql文件，导入数据到mysql或者SQLite
```
ansible-cmdb -t sql -i hosts out/
```

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
