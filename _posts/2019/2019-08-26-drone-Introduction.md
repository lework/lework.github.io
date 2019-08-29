---
layout: post
title: "Drone CI 介绍"
date: "2019-08-26 16:00:00"
category: drone
tags: ci drone
author: lework
---
* content
{:toc}

## 什么是CI/CD

**持续集成（ContinousIntergration，CI）**是一种软件开发实践，即团队开发成员经常集成它们的工作，通常每个成员每天至少集成一次，也就意味着每天可能会发生多次集成。每次集成都需要通过自动化的编译、发布、自动化回归测试来验证，从而尽快地发现集成错误。而这些自动化的操作则由CI软件进行执行。

**持续部署（ContinousDelivery，CD）**在持续集成的基础上，将集成后的代码部署到更贴近真实运行环境中。交付团队 - >版本控制 - >构建和单元测试 - >自动验收测试（e2e）- > UAT - >发布（部署，DevOps）

## 什么是Drone

[Drone](<https://drone.io/>) 是一个基于Docker容器技术的可扩展的持续集成引擎，用于自动化测试与构建，甚至发布。每个构建都在一个临时的Docker容器中执行，使开发人员能够完全控制其构建环境并保证隔离。开发者只需在项目中包含 `.drone.yml`文件，将代码推送到 git 仓库，Drone就能够自动化的进行编译、测试、发布。

官方[文档](https://docs.drone.io/)上面大量充斥着`0.8`版本的配置文件描述,而最新的版本是`1.2.3`,两个版本之间有大量的不同地方，需要大家一点一点的试错摸索，不过读者在读完本篇文章后，就可以很顺畅的编写pipeline了。




## Drone 自动化流程

![cicd.png](/assets/images/ci/cicd.png)

Drone调用代码仓库的 API 给 Repository 增加一个 webhook ，当 Repository 触发相应事件(push, tag, pull request)时，代码仓库发起 http 请求回调 drone 触发构建。

Drone 通过 OAuth 认证或账号密码登录代码仓库后，获得完整的控制权。在 Drone 的 web后台管理页面激活 Repository 后，Drone调用代码仓库的 API 给 Repository 增加一个 webhook ，当 Repository 触发相应事件(push, tag, pull request)时，代码仓库发起 http 请求回调 drone 触发构建。

Webhooks 由代码仓库发送，用于触发 pipeline。代码仓库会在下面 3 种情况下，自动发送 Webhook 请求到 Drone：

- 代码被 push 到仓库
- 新建一个合并请求（pull request）
- 新建一个tag
- ...

**跳过自动部署**

在本地用git提交代码的时候可以通过添加 `[CI SKIP]` （大小写不敏感）到提交信息（commit message）中来让 Drone 跳过某个提交。

```bash
git commit -m "updated README [CI SKIP]"
```

{% raw %}
## Drone 安装

目前Drone支持多种代码托管服务，几乎涵盖市面上主流的代码仓库，如`Github`, `GitLab`, `Gogs`, `Gitea`、`Bitbucket Server`等，且在drone里预设了对应托管服务的 API，Drone的很多功能比如拉取 git repo list/add webhook to repo 都是通过这些 API 完成的。另外Drone的账户体系依赖于托管服务的账户系统， 并不存在维护账户这个概念。例如我们使用Gogs作为代码托管服务，我们在登录 Drone 时，实际上是 Drone 把用户名密码传给了 Gogs. 因此，激活某个 Repository （仓库） 的构建(为Repository 添加webhook) 能否成功取决于该账号在 Gogs 里是不是该 Repository 的管理员。

> drone的服务配置都是以环境变量设置的。

### 单机形式

与gogs一起
```bash
docker run \
  --volume=/var/run/docker.sock:/var/run/docker.sock \
  --volume=/var/lib/drone:/data \
  --env=DRONE_GIT_ALWAYS_AUTH=false \
  --env=DRONE_GOGS_SERVER=${DRONE_GOGS_SERVER} \      # 指定gogs server地址
  --env=DRONE_RUNNER_CAPACITY=2 \
  --env=DRONE_SERVER_HOST=${DRONE_SERVER_HOST} \
  --env=DRONE_SERVER_PROTO=${DRONE_SERVER_PROTO} \
  --env=DRONE_TLS_AUTOCERT=false \
  --env=DRONE_USER_CREATE=username:root,admin:true  \  # 指定管理员
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone:1.2.3
```

与gitlab一起

```bash
docker run \
  --volume=/var/run/docker.sock:/var/run/docker.sock \
  --volume=/var/lib/drone:/data \
  --env=DRONE_GIT_ALWAYS_AUTH=false \
  --env=DRONE_GITLAB_SERVER=https://gitlab.com \
  --env=DRONE_GITLAB_CLIENT_ID={% your-gitlab-client-id %} \
  --env=DRONE_GITLAB_CLIENT_SECRET={% your-gitlab-client-secret %} \
  --env=DRONE_RUNNER_CAPACITY=2 \
  --env=DRONE_SERVER_HOST={% your-drone-server-host %} \
  --env=DRONE_SERVER_PROTO={% your-drone-server-protocol %} \
  --env=DRONE_TLS_AUTOCERT=false \
  --env=DRONE_USER_CREATE=username:root,admin:true  \  # 指定管理员
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone:1.2.3
```


drone默认使用嵌入式sqlite数据库存储，当然也可以使用其他存储

```bash
# mysql
DRONE_DATABASE_DRIVER=mysql
DRONE_DATABASE_DATASOURCE=root:password@tcp(1.2.3.4:3306)/drone?parseTime=true

# PostGress
DRONE_DATABASE_DRIVER=postgres
DRONE_DATABASE_DATASOURCE=postgres://root:password@1.2.3.4:5432/postgres?sslmode=disable
```

**日志**

日志默认由`stderr` `json`格式写入，可以通过`docker logs` 访问日志

默认日志级别为`INFO`,更改日志级别为`DEBUG` 可以看到更详细的调试日志

```bash
DRONE_LOGS_DEBUG=true
```

可以使用以下配置让日志更易于阅读

```bash
DRONE_LOGS_TEXT=true
DRONE_LOGS_PRETTY=true
DRONE_LOGS_COLOR=true
```

查看RPC通信的日志

```bash
DRONE_RPC_DEBUG=true
```

### 多主机形式

启动控制端

```bash
# openssl rand -hex 16                   # 生成用于rpc通信的密钥
bea26a2221fd8090ea38720fc445eca6

# docker run \
  --volume=/var/lib/drone:/data \
  --env=DRONE_TLS_AUTOCERT=false \
  --env=DRONE_AGENTS_ENABLED=true \ # 开启客户端，这样drone就不会自己启动一个本地客户端了，而使用其他的客户端。
  --env=DRONE_GOGS_SERVER={% your-gogs-server-url %} \
  --env=DRONE_RPC_SECRET={% your-shared-secret %} \
  --env=DRONE_SERVER_HOST={% your-drone-server-host %} \
  --env=DRONE_SERVER_PROTO={% your-drone-server-protocol %} \
  --env=DRONE_GIT_ALWAYS_AUTH=false \
  --env=DRONE_USER_CREATE=username:root,admin:true  \  # 指定管理员
  --publish=80:80 \
  --publish=443:443 \
  --restart=always \
  --detach=true \
  --name=drone \
  drone/drone:1

```

启动客户端

```bash
docker run -d \
  -e DRONE_RPC_PROTO=https \
  -e DRONE_RPC_HOST=drone.company.com \
  -e DRONE_RPC_SECRET=super-duper-secret \        # 指定用于rpc通信的密钥
  -e DRONE_RUNNER_CAPACITY=2 \
  -e DRONE_RUNNER_NAME=${HOSTNAME} \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart always \
  --name runner \
  drone/agent:1
```



客户端用三种Runners

1. Docker Runner  Dind的形式执行pipeline中的步骤。
2. Exec Runner    直接在主机上执行pipeline中的步骤，目前不怎么推荐使用了。
3. SSH Runner     使用ssh协议在远程服务器上执行pipeline中的步骤，目前不怎么推荐使用了。



### Drone cli

```bash
docker run \
  -e DRONE_SERVER=http://drone-server \   # 指定drone server
  -e DRONE_TOKEN=9JFtB5oRaKLS04FjRp92aLG7NHmfwHfg \  # 指定token
  drone/cli:alpine \
  drone list
```



## PipeLine AS Code

在项目根目录有一个 `.drone.yml` 的文件，这个是 Drone 构建脚本的配置文件，它随项目一块进行版本管理，开发者不需要额外再去维护一个配置脚本。

另外就是基于`yaml`的配置文件相对来说非常简单了，例如一个简单的构建脚本如下：

```yaml
# .drone.yml
kind: pipeline         # 定义pipeline
name: default          # pipeline的名称

clone:                 # 指定git clone的配置
  depth: 50

volumes:               # 定义存储
- name: cache
  host:
    path: /tmp/cache

steps:                           # 定义执行步骤
- name: build                    # 定义build步骤
  image: golang:1.12.6           # 定义镜像      
  commands:                      # 定义容器内的执行命令
  - make build
  volumes:                       # 定义容器挂载卷
  - name: cache
    path: /cache
  when:                          # 执行定义条件
    event:                       # 定义事件型条件
    - tag                        # tag事件或push事件
    - push
  
- name: build-branch-image
  image: plugins/docker
  settings:
    dockerfile: ./Dockerfile
    storage_path: /drone/src/docker
    repo: 192.168.77.134:5000/root/drone-go
    registry: 192.168.77.134:5000
    mirror: https://docker.mirrors.ustc.edu.cn/
    insecure: true
    tag:
      - ${DRONE_BRANCH}                          # 使用drone变量
  when:                    
    event: 
    - push
    
- name: deploy                                    # 定义deploy步骤
  image: sh4d1/drone-kubernetes                   # 定义镜像地址
  settings:                                       # 定义插件变量
    kubernetes_template: deployment.yml
    kubernetes_namespace: default
  environment:                                    # 定义容器环境变量
    KUBERNETES_SERVER:
      from_secret: kubernetes_server              # 从secret中获取信息
    KUBERNETES_CERT:
      from_secret: kubernetes_cert
    KUBERNETES_TOKEN:
      from_secret: kubernetes_token
  when:                                           # 执行定义条件
    event:
    - push
    
- name: notify
  image: drillster/drone-email
  settings:
    port: 25
    from: test@test.com
    host: smtp.test.com
    username: test@test.com
    password:
      from_secret: email_password
    skip_verify: true
    recipients: [ test@test.com ]
    recipients_only: true
  when:
    status: [ success, changed, failure ]
```


`steps`中可用的配置

```bash
name        # 步骤名称
image       # 指定镜像
commands    # 指定在容器里执行的命令
detach      # 分离pipeline步骤，次此步骤退出后，不会使pipeline失败
environment # 设置容器环境变量
privileged  # 设置容器是否拥有扩展权限
pull        # 拉取进行的策略
volumes     # 挂载卷
settings    # 指定插件的配置信息
```



更多配置请看 [官方文档](https://docs.drone.io/user-guide/pipeline/)



## 丰富的插件支持

http://plugins.drone.io/ 这个是 Drone 的插件商店，包含了常见的一些需求插件，例如：

- 构建后发送消息：[slack](http://plugins.drone.io/drone-plugins/drone-slack/), [telegram](http://plugins.drone.io/appleboy/drone-telegram/), [line](http://plugins.drone.io/appleboy/drone-line/), [facebook](http://plugins.drone.io/appleboy/drone-facebook/), [discord](http://plugins.drone.io/appleboy/drone-discord/), [gitter](http://plugins.drone.io/drone-plugins/drone-gitter/), [email](http://plugins.drone.io/drillster/drone-email/)...
- 构建成功后发布：[npm](http://plugins.drone.io/drone-plugins/drone-npm/), [docker](http://plugins.drone.io/drone-plugins/drone-docker/), [github release](http://plugins.drone.io/drone-plugins/drone-github-release/), [google container](http://plugins.drone.io/drone-plugins/drone-gcr/)...
- 构建成功后部署：[AWS](http://plugins.drone.io/devops-israel/drone-ecs-deploy/), [Kubernetes](http://plugins.drone.io/mactynow/drone-kubernetes/), [rsync](http://plugins.drone.io/drillster/drone-rsync/), [scp](http://plugins.drone.io/appleboy/drone-scp/), [ftp](http://plugins.drone.io/christophschlosser/drone-ftps/)...

pipeline 中简单的增加插件镜像配置就可以支持插件，例如：

```bash
publish:
  image: plugins/docker
  settings:
    repo: lework/hello-world
    tags: [ 1, 1.1, latest ]
    registry: index.docker.io
```

插件在后台的运行命令类似下面的

```bash
docker run --rm \
  -e PLUGIN_TAG="1, 1.1, latest" \
  -e PLUGIN_REPO=lework/hello-world \
  -e PLUGIN_REGISTRY=index.docker.io \
  -e DRONE_COMMIT_SHA=d8dbe4d94f15fe89232e0402c6e8a0ddf21af3ab \
  -v $(pwd):/drone/src \
  -w /drone/src \
  --privileged \
  plugins/docker --dry-run
```

由于 Drone 是基于 Docker 的，甚至是所有的插件也都是一个 docker 镜像。这代表着我们可以使用任何语言来开发 Drone 插件，最后只要打包成镜像就好了。

开发Drone插件也是非常简单的，容器里需要的配置信息在`.drone.yml`文件中声明就好，drone都以`PLUGIN_`开头的环境变量的形式传递给容器, 如：

drone配置

```yaml
- name: test_plugin
  image: busybox
  commands:
  - printenv
  settings:
    set_string: "hello"
    set_integer: 1
    set_float: 1.0
    set_boolean: true
    set_arry: ['1','2']
```

传递给容器的变量是

```bash
PLUGIN_SET_ARRY=1,2
PLUGIN_SET_BOOLEAN=true
PLUGIN_SET_FLOAT=1
PLUGIN_SET_INTEGER=1
PLUGIN_SET_STRING=hello
```

传递的其他变量

```bash
CI=true
CI_BUILD_CREATED=1566804461
CI_BUILD_EVENT=push
CI_BUILD_FINISHED=1566804465
CI_BUILD_LINK=http://192.168.77.132:3000/root/123/compare/1f23097c3c3ff8d87affa33eaedd2c42c80efbd8...9aa7fe62e5b5457f013f85cfcc13c2ec74d4071c
CI_BUILD_NUMBER=7
CI_BUILD_STARTED=1566804461
CI_BUILD_STATUS=success
CI_BUILD_TARGET=
CI_COMMIT_AUTHOR=root
CI_COMMIT_AUTHOR_AVATAR=https://secure.gravatar.com/avatar/10aac600a4104a338a827fc3d7c1315d?d=identicon
CI_COMMIT_AUTHOR_EMAIL=root@123.com
CI_COMMIT_AUTHOR_NAME=root
CI_COMMIT_BRANCH=master
CI_COMMIT_MESSAGE=更新 '.drone.yml'
CI_COMMIT_REF=refs/heads/master
CI_COMMIT_SHA=9aa7fe62e5b5457f013f85cfcc13c2ec74d4071c
CI_JOB_FINISHED=1566804465
CI_JOB_STARTED=1566804461
CI_JOB_STATUS=success
CI_PARENT_BUILD_NUMBER=0
CI_REMOTE_URL=http://192.168.77.132:3000/root/123.git
CI_REPO=root/123
CI_REPO_LINK=
CI_REPO_NAME=root/123
CI_REPO_PRIVATE=false
CI_REPO_REMOTE=http://192.168.77.132:3000/root/123.git
CI_WORKSPACE=/drone/src
CI_WORKSPACE_BASE=/drone/src
CI_WORKSPACE_PATH=
DOCKER_NETWORK_ID=vc7t8q0i0u9srcepz5ote72k0yyd4hwl
DRONE=true
DRONE_BRANCH=master
DRONE_BUILD_ACTION=
DRONE_BUILD_CREATED=1566804461
DRONE_BUILD_EVENT=push
DRONE_BUILD_FINISHED=1566804465
DRONE_BUILD_LINK=http://192.168.77.132/root/123/7
DRONE_BUILD_NUMBER=7
DRONE_BUILD_STARTED=1566804461
DRONE_BUILD_STATUS=success
DRONE_COMMIT=9aa7fe62e5b5457f013f85cfcc13c2ec74d4071c
DRONE_COMMIT_AFTER=9aa7fe62e5b5457f013f85cfcc13c2ec74d4071c
DRONE_COMMIT_AUTHOR=root
DRONE_COMMIT_AUTHOR_AVATAR=https://secure.gravatar.com/avatar/10aac600a4104a338a827fc3d7c1315d?d=identicon
DRONE_COMMIT_AUTHOR_EMAIL=root@123.com
DRONE_COMMIT_AUTHOR_NAME=root
DRONE_COMMIT_BEFORE=
DRONE_COMMIT_BRANCH=master
DRONE_COMMIT_LINK=http://192.168.77.132:3000/root/123/compare/1f23097c3c3ff8d87affa33eaedd2c42c80efbd8...9aa7fe62e5b5457f013f85cfcc13c2ec74d4071c
DRONE_COMMIT_MESSAGE=更新 '.drone.yml'
DRONE_COMMIT_REF=refs/heads/master
DRONE_COMMIT_SHA=9aa7fe62e5b5457f013f85cfcc13c2ec74d4071c
DRONE_DEPLOY_TO=
DRONE_DOCKER_NETWORK_ID=vc7t8q0i0u9srcepz5ote72k0yyd4hwl
DRONE_GIT_HTTP_URL=http://192.168.77.132:3000/root/123.git
DRONE_GIT_SSH_URL=git@192.168.77.132:3000:root/123.git
DRONE_JOB_FINISHED=1566804465
DRONE_JOB_STARTED=1566804461
DRONE_JOB_STATUS=success
DRONE_MACHINE=drone-agent001
DRONE_REMOTE_URL=http://192.168.77.132:3000/root/123.git
DRONE_REPO=root/123
DRONE_REPO_BRANCH=master
DRONE_REPO_LINK=
DRONE_REPO_NAME=123
DRONE_REPO_NAMESPACE=root
DRONE_REPO_OWNER=root
DRONE_REPO_PRIVATE=false
DRONE_REPO_SCM=
DRONE_REPO_VISIBILITY=public
DRONE_RUNNER_HOST=drone-agent001
DRONE_RUNNER_HOSTNAME=drone-agent001
DRONE_RUNNER_PLATFORM=linux/amd64
DRONE_SOURCE_BRANCH=master
DRONE_STAGE_ARCH=amd64
DRONE_STAGE_DEPENDS_ON=
DRONE_STAGE_FINISHED=1566804465
DRONE_STAGE_KIND=pipeline
DRONE_STAGE_MACHINE=drone-agent001
DRONE_STAGE_NAME=default
DRONE_STAGE_NUMBER=1
DRONE_STAGE_OS=linux
DRONE_STAGE_STARTED=1566804461
DRONE_STAGE_STATUS=success
DRONE_STAGE_VARIANT=
DRONE_STEP_NAME=test_plugin
DRONE_STEP_NUMBER=2
DRONE_SYSTEM_HOST=192.168.77.132
DRONE_SYSTEM_HOSTNAME=192.168.77.132
DRONE_SYSTEM_PROTO=http
DRONE_SYSTEM_VERSION=1.2.3
DRONE_TARGET_BRANCH=master
DRONE_WORKSPACE=/drone/src
DRONE_WORKSPACE_BASE=/drone/src
DRONE_WORKSPACE_PATH=
HOME=/root
HOSTNAME=20224f33e0cc
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/drone/src
SHLVL=1
```

有了这些变量，我们在插件里可做的事情就比较多了。

{% endraw %}