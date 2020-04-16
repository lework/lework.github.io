---
layout: page
title: About
permalink: /about/
icon: heart
type: page
---

> 目前从事运维工作，努力学习中......

请使用 Mozilla Firefox、Google Chrome 等现代浏览器浏览本博客。

本博采用 Jekyll[[1]][1] 搭建，Markdown[[2]][2] 写作，托管于 GitHub[[3]][3]。

自 2016 年 07 月 07 日起，本站已运行 <span id="days"></span> 天，截至 {{ site.time | date: "%Y 年 %m 月 %d 日" }}，写了博文 {{ site.posts.size }} 篇，{% assign count = 0 %}{% for post in site.posts %}{% assign single_count = post.content | strip_html | strip_newlines | remove: ' ' | size %}{% assign count = count | plus: single_count %}{% endfor %}{% if count > 10000 %}{{ count | divided_by: 10000 }} 万 {{ count | modulo: 10000 }}{% else %}{{ count }}{% endif %} 字。

即日起，本博客的原创内容，均采用知识共享组织（Creative Commons）的 "署名-非商业性使用 3.0 中国大陆 (CC BY-NC 3.0 CN)[[4]][4]" 许可。

## 博主

- Github: [lework](https://github.com/lework)

## 开源项目

- [Ansible-roles](https://github.com/lework/Ansible-roles) 含有运维日常部署，应用发布，系统操作等等的 ansible role，目前已有 70+。
- [kubectl-check](https://github.com/lework/kubectl-check) 用于检查 deployment 的所有 pod 是否就绪的 kubectl 插件
- [lenav](https://github.com/lework/lenav) 一个简便的公司内部网址导航站
- [leversion](https://github.com/lework/leversion) 列出开源软件的最新版本
- [lemonitor](https://github.com/lework/lemonitor) 开源软件的国内镜像站点
- [Ansible wiki](https://github.com/leops-china/ansible-wiki) Ansible Wiki站点

## 群组织

- Ansible 学习群 <a target="_blank" href="//shang.qq.com/wpa/qunwpa?idkey=76a382732441da12c7b6bc8393cdacdecd38f23a840abb8685bb55ac33f8fdd9"><img border="0" src="//pub.idqqimg.com/wpa/images/group.png" alt="Ansible学习群" title="Ansible学习群"></a>
- Ansible 学习群 2 <a target="_blank" href="//shang.qq.com/wpa/qunwpa?idkey=619146ae673362fbfa81f78d3646df3703ae324d720396a9d9c543470b0f0ff6"><img border="0" src="//pub.idqqimg.com/wpa/images/group.png" alt="Ansible学习群2" title="Ansible学习群2"></a>

## 博客历程

- 2019-06-13 使用 Jekyll 平台 写博客，托管在 Gihub 中
- 2019-06-15 博客文章从简书迁移到 Jekyll 中
- 2019-07-23 使用 gitalk 评论
- 2019-07-24 增加搜索功能
- 2019-09-04 使用 utteranc.es 评论
- 2019-10-17 增加赞赏支持功能
- 2019-12-10 文章中增加 "原文地址" 和 "字数统计"
- 2019-12-11 代码块增加 "Copy" 功能

## 赞赏奖励

若您觉得本博客所创造的内容对您有所帮助，可考虑略表心意，支持一下。

{% include reward.html %}

[1]: https://jekyllrb.com/ 'Jekyll'
[2]: http://daringfireball.net/projects/markdown/ 'Markdown'
[3]: https://github.com/ 'GitHub'
[4]: http://creativecommons.org/licenses/by-nc/3.0/cn/ '署名-非商业性使用 3.0 中国大陆'

{% include comments.html %}

<script>
var days = 0, daysMax = Math.floor((Date.now() / 1000 - {{ "2016-07-07" | date: "%s" }}) / (60 * 60 * 24));
(function daysCount(){
    if(days > daysMax){
        document.getElementById('days').innerHTML = daysMax;
        return;
    } else {
        document.getElementById('days').innerHTML = days;
        days += 10;
        setTimeout(daysCount, 1); 
    }
})();
</script>
