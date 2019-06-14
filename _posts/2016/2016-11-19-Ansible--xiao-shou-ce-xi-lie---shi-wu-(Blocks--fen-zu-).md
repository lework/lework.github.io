---
layout: post
title: "Ansible 小手册系列 十五（Blocks 分组）"
date: "2016-11-19 20:40:04"
categories: Ansible
excerpt: "当我们想在满足一个条件下，执行多个任务时，就需要分组了。而不再每个任务都要用when。 错误处理 block中的任务在执行中，如果有任何错误，将..."
auth: lework
---
* content
{:toc}

当我们想在满足一个条件下，执行多个任务时，就需要分组了。而不再每个任务都要用when。
```
  tasks: 
   - block:
     - command: echo 1
     - shell: echo 2
     - raw: echo 3
     when: ansible_distribution == 'CentOS'
```

**错误处理**

```
tasks:
  - block:
      - debug: msg='i execute normally'
      - command: /bin/false
      - debug: msg='i never execute, cause ERROR!'
    rescue:
      - debug: msg='I caught an error'
      - command: /bin/false
      - debug: msg='I also never execute :-('
    always:
      - debug: msg="this always executes"
```

block中的任务在执行中，如果有任何错误，将执行rescue中的任务。 无论在block和rescue中发生或没有发生错误，always部分都运行。

**发生错误后，运行`handlers`**

```
 tasks:
  - block:
      - debug: msg='i execute normally'
        notify: run me even after an error
      - command: /bin/false
    rescue:
      - name: make sure all handlers run
        meta: flush_handlers
 handlers:
   - name: run me even after an error
     debug: msg='this handler runs even on error'
```
> 测试的时候，设置meta: flush_handlers时，handlers 不执行，rescue的任务还是执行的。

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
