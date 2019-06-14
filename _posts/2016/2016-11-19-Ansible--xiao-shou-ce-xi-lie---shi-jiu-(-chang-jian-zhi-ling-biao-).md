---
layout: post
title: "Ansible 小手册系列 十九（常见指令表）"
date: "2016-11-19 21:07:55"
categories: Ansible
excerpt: "Play Role Block Task 更多文章请看 Ansible 专题文章总览"
auth: lework
---
* content
{:toc}

## Play
---

|指令|说明|
|:---|:---|
|accelerate|开启加速模式|
|accelerate_ipv6|是否开启ipv6 |
|accelerate_port|加速模式的端口|
|always_run||
|any_errors_fatal|有任务错误时，立即停止 |
|become|是否提权|
|become_flags|提权命令的参数 |
|become_method|提权得方式|
|become_user|提权的用户|
|check_mode|当为True时，只检查，不做修改|
|connection|连接方式|
|environment|定义远端系统的环境变量|
|force_handlers|任务失败后，是否依然执行handlers中的任务|
|gather_facts|是否获取远端系统得facts|
|gather_subset|获取facts得哪些键值 |
|gather_timeout|获取facts的超时时间|
|handlers|定义task执行完成以后需要调用的任务 |
|hosts|指定运行得主机|
|ignore_errors|是否忽略错误 |
|max_fail_percentage|最大的错误主机数，超过则立即停止ansbile|
|name|定义任务得名称 |
|no_log|不记录日志 |
|port|定义ssh的连接端口|
|post_tasks|执行任务后要执行的任务 |
|pre_tasks|执行任务前要执行的任务|
|remote_user|远程登陆的用户|
|roles|定义角色 |
|run_once|任务只运行一次 |
|serial|任务每次执行的主机数|
|strategy|play运行的模式 |
|tags|标记标签|
|tasks|定义任务 |
|vars|定义变量|
|vars_files|包含变量文件|
|vars_prompt|要求用户输入内容 |
|vault_password|加密密码|

## Role
---

|指令|说明|
|:---|:---|
|always_run||
|become|是否提权|
|become_flags|提权命令的参数 |
|become_method|提权的方式|
|become_user|提权的用户|
|check_mode|当为True时，只检查，不做修改|
|connection|连接方式|
|delegate_facts|委托facts |
|delegate_to|任务委派 |
|environment|定义远端系统的环境变量|
|ignore_errors|是否忽略错误 |
|no_log|不记录日志 |
|port|定义ssh的连接端口|
|remote_user|远端系统的执行用户|
|run_once|只运行一次 |
|tags|标记标签|
|vars|定义变量|
|when|条件表达式结果为True则执行block|

## Block
---

|指令|说明|
|:---|:---|
|always|always里的任务总是执行|
|always_run||
|any_errors_fatal|有错误时立即中断ansbile |
|become|是否提权|
|become_flags|提权命令的参数 |
|become_method|提权的方式|
|become_user|提权的用户|
|block|分组执行 |
|check_mode|当为True时，只检查，不做修改|
|connection|连接方式|
|delegate_facts|委托facts |
|delegate_to|任务委派 |
|environment|定义远端系统的环境变量|
|ignore_errors|是否忽略错误 |
|no_log|不记录日志 |
|port|定义ssh的连接端口|
|remote_user|远端系统的执行用户|
|rescue|block中的任务在执行中，如果有任何错误，将执行rescue中的任务。|
|run_once|只运行一次 |
|tags|标记标签|
|vars|定义变量|
|when|条件表达式结果为True则执行block|


## Task
---
||说明|
|:---|:---|
|action|执行动作|
|always_run||
|any_errors_fatal|为True时，只要任务有错误，就立即停止ansible |
|args|定义任务得参数 |
|async|是否异步执行任务 |
|become|是否提权|
|become_flags|提权命令的参数 |
|become_method|提权的方式|
|become_user|提权的用户|
|changed_when|条件表达式为True时，使任务状态为changed |
|check_mode|为True时，只检查运行状态，在远端不做任何修改|
|connection|连接方式|
|delay|等待多少秒，才执行任务|
|delegate_facts|委托facts |
|delegate_to|任务委派 |
|environment|定义远端的环境变量|
|failed_when|条件表达式为True时，使任务为失败状态 |
|ignore_errors|是否忽略错误 |
|local_action|本地执行|
|loop||
|loop_args| |
|loop_control|改变循环的变量项|
|name|定义人物的名称 |
|no_log|不记录日志 |
|notify|用于任务执行完，执行handlers里的任务|
|poll|轮询时间|
|port|定义ssh的连接端口|
|register|注册变量|
|remote_user|远端系统的执行用户|
|retries|重试次数 |
|run_once|只运行一次 |
|tags|标记为标签 |
|until|直到为真时，才继续执行任务|
|vars|定义变量|
|when|条件表达式，结果为True则执行task|
|with_<lookup_plugin>|循环|

---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
