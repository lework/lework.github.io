---
layout: post
title:  "简书博客文章转移到jekyll中"
date: 2019-06-15 17:35:56
categories:  jekyll
tags: jekyll github
auth: lework
---
* content
{:toc}
简书本身提供了一键打包下载文章的功能，下载的文章按照专题分类，文章中的图片也没有下载，跟jekyll要求的格式略微不同，所以我们需要写个脚本来转化下。




## 需求

1. 将简书下载的文件内容头部加入jekyll特定的标识信息，如：

   ```
   ---
   layout: post
   title:  "对这个 jekyll 博客主题的改版和重构"
   date:   2019-06-15 11:40:18
   categories: jekyll
   tags: jekyll 简书
   author: lework
   ---
   ```

2. 将文章文件按年份分类，并且名称改为拼音，前缀加上日期。

   >因为jekyll不支持_post目录中的文件名称为中文，我们要改成对应英文。

   ```
   File: ./user-3629406-1560475790/Ansible/Ansible-专题文章总揽.md ==> ./_posts/2016\2016-11-19-Ansible--zhuan-ti-wen-zhang-zong-lan-.md
   ```

3. 将文章中图片存储到特定目录(当前目录下的`assert/images`)，且按照主题目录进行分类，文章中的图片引入路径也要换掉

   ```
   Image: https://upload-images.jianshu.io/upload_images/3629406-51784a1683db675a.png ==> ./assets/images/kubernetes\3629406-51784a1683db675a.png 
   ```



## 实现脚本

环境: `python3`

依赖：

```
requests
bs4
xpinyin
```

脚本：

```
# -*- coding: UTF-8 -*-
import datetime
import re
from xpinyin import Pinyin
from bs4 import BeautifulSoup
import requests
import os

# 1.修改成自己的user id
user_id = 'ace85431b4bb'

# 2.修改成自己导出的简书文章目录
article_path = "./user-3629406-1560475790"
post_path = "./_posts/"
image_path = './assets/images/'

page = 1
image_count = 0
article_transform_count = 0

url = "https://www.jianshu.com/u/%s?order_by=added_at&page=%s"

# 3.修改放在makedown的头部内容
mdStr = '---\n' \
        'layout: post\n' \
        'title:\n' \
        'date:\n' \
        'categories:\n' \
        'excerpt:\n' \
        'auth: lework\n' \
        '---\n' \
        '* content\n' \
        '{:toc}\n' \
        '\n' \

headers = {"Accept": "text/html,application/xhtml+xml,application/xml;",
           "Accept-Encoding": "gzip",
           "Accept-Language": "zh-CN,zh;q=0.8",
           "Referer": "https://www.jianshu.com/u/%s" % user_id,
           "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.170 Safari/537.36"}


def archive(file, date, text):
    global image_count
    pin = Pinyin()
    table = {ord(f): ord(t) for f, t in zip(
        u'，。！？【】（）％＃＠＆１２３４５６７８９０',
        u',.!?[]()%#@&1234567890')}
    with open(file, 'r', encoding='utf-8') as context:
        file_text = context.read()
        zhpost_path = os.path.join(post_path, date.split('-')[0])
        zt = os.path.basename(os.path.dirname(file))
        file_name = date + '-' + pin.get_pinyin(os.path.basename(file)).translate(table)
        print("File: %s ==> %s" % (file, os.path.join(zhpost_path, file_name)))
        results = re.findall("(\(https?\://upload-images\.jianshu\.io.*?\))", file_text)
        if results:
            zt_image_path = os.path.join(image_path, zt)
            if not os.path.exists(zt_image_path):
                os.makedirs(zt_image_path)
            for r in results:
                r_url = r.split('?')[0][1:]
                print(r)
                file_text = file_text.replace(r, "(%s)" % os.path.join(zt_image_path, r_url.split('/')[-1]).replace('\\', '/').replace('./', '/'))
                print("Image: %s ==> %s " % (r_url, os.path.join(zt_image_path, r_url.split('/')[-1])))
                r = requests.get(r_url)
                with open(os.path.join(zt_image_path, r_url.split('/')[-1]), 'wb') as imgContent:
                    imgContent.write(r.content)
                    image_count += 1
        if not os.path.exists(zhpost_path):
            os.makedirs(zhpost_path)

        with open(os.path.join(zhpost_path, file_name), 'w+', encoding='utf-8') as new_file:
            text = text.replace('categories:', 'categories: ' + zt)
            new_context = text + file_text
            new_file.write(new_context)
    print("====================================")


file_list = []
for root, dirs, files in os.walk(article_path):
    for file in files:
        file_list.append(os.path.join(root, file).replace('\\', '/'))

article_count = len(file_list)

while True:
    pageUrl = url % (user_id, page)
    print("URL: %s" % pageUrl)
    resp = requests.get(pageUrl, headers=headers)
    bs = BeautifulSoup(resp.text, "html.parser")

    # 获取文章列表模块
    container = bs.find('div', {'id': 'list-container'})
    # 获取标题,时间,描述
    for item in container.find_all('li'):
        try:
            title = item.find('a', {'class': 'title'}).get_text()
            print("Title: %s" % title)
            time = item.find('span', {'class': 'time'})['data-shared-at']
            time2 = time.replace('T', ' ').split('+')[0].lstrip(' ')
            abstract = item.find('p', {'class': 'abstract'}).get_text()
            mdStr2 = mdStr.replace('title:', 'title: "%s"' % title).replace('date:', 'date: "%s"' % time2).replace('excerpt:',
                                                                                                           'excerpt: "%s"' % abstract.strip().replace('"', '\\"').replace('\\', '\\\\'))
            for (index, file) in enumerate(file_list):
                if title.replace(' ', '-').replace('.', '-') == file.split('/')[-1].rstrip('.md'):
                    archive(file, time2.split(' ')[0], mdStr2)
                    file_list.pop(index)
                    article_transform_count += 1
                    break
        except Exception as e:
            print("\n\n")
            print(e)
            print('Not found...\n\n')
            for f in file_list:
                mdStr2 = mdStr.replace('title:', 'title: "%s"' % os.path.basename(f).replace('.md', ''))
                archive(f, datetime.datetime.now().strftime('%Y-%m-%d'), mdStr2)
            print("\n\nResult: 文章数:%s 转换数:%s 图片数:%s" % (article_count, article_transform_count, image_count))
            for f in file_list:
                print("在网页未找到的文件: %s" % f)
            print('finish\n')
            exit()
    page += 1

```



## 使用

1. 安装python3 环境

2. 安装脚本依赖

3. 修改脚本里的内容
   1. 修改自己的user id
   2. 修改成自己导出的简书文章目录
   3. 修改放在makedown的头部内容

4. 将脚本与简书下载的压缩包同目录存放

   ```
   jianshu_to_jekyll.py
   user-3629406-1560475792
   user-3629406-1560475792.rar
   ```

5. 执行脚本

   ```
   E:\04.开发\python\venv\Scripts\python.exe E:/04.开发/python/jianshu_to_jekyll.py
   URL: https://www.jianshu.com/u/ace85431b4bb?order_by=added_at&page=1
   Title: Ansible Role 存储 之【gluster】
   File: ./user-3629406-1560475790/Ansible/Ansible-Role-存储-之【gluster】.md ==> ./_posts/2018\2018-06-22-Ansible-Role--cun-chu---zhi-[gluster].md
   ====================================
   ....
   ....
   
   
   Result: 文章数:148 转换数:144 图片数:0
   在网页未找到的文件: ./user-3629406-1560475790/kubernetes/2018-07-16.md
   在网页未找到的文件: ./user-3629406-1560475790/kubernetes/Kubernetes-v1-10-3-HA-集群手工安装教程.md
   在网页未找到的文件: ./user-3629406-1560475790/Linux性能优化/01-理解linux的“平均负载”.md
   在网页未找到的文件: ./user-3629406-1560475790/随笔/要写的role.md
   finish
   ```

   > 上述中，在网页未找到的文章，就是你还未发布的文章，或者简书封禁的文章。这些文章都会以当前日期转换。

   转换后的目录情况

   ![jianshu-to-jekyll](/assets/images/jekyll/jianshu-to-jekyll.png)

6. 将_post目录中的文件和images中的图片放入到jekyll项目，进行编译看看效果喽。

   ```
   lework.github.io
   ├── _drafts
   ├── _includes
   ├── _layouts
   ├── _sass
   ├── assets
   │   ├── images
   │      ├── Ansible
   │         ├── 3629406-0b0a38c2210d4a39.png
   │         ├── 3629406-0c630bdde0584872.png
   ├── _posts
   │   ├── 2016
   │   ├── 2016-07-07-command.md
   │      ├── 2016-07-07-gitflow..md
   │      ├── 2016-07-07-practice.md
   │      ├── 2016-07-08-centos_daemon.md
   ....
   
   ```

   

## 遇到的问题

最大的问题就是 **中文编码**

说一说`jekyll`的限制

1. `_post`文章不能用中文字符
2. 在`Windows`环境下文件名称不能有特殊字符，所以建议大家
3. 文章中如果有`jinja2`模板字符，需要加上`{% raw %} {%endraw%}` 转义jinja2内置变量