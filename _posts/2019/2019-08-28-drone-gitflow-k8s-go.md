---
layout: post
title: "基于Drone + GitFlow + K8s的云原生语义化 CI 工作流"
date: "2019-08-29 17:00:00"
category: drone
tags: ci drone gitflow kubernetes
author: lework
---
* content
{:toc}

在进行构建CI工作流前,先了解下2个概念

## 什么是GitOps?

GitOps是一种持续交付的方式.它的核心思想是将应用系统的声明性基础架构和应用程序存放在Git的版本库中.

将Git作为交付流水线的核心,每个开发人员都可以提交拉取请求（Pull Request）并使用Git来加速和简化应用程序部署和运维任务.通过使用像Git这样的简单熟悉工具,开发人员可以更高效地将注意力集中在创建新功能而不是运维相关任务上（例如,应用系统安装,配置,迁移等）

![gitops.jpg](/assets/images/ci/gitops.jpg)

通过应用 GitOps ,应用系统的基础架构和应用程序代码可以快速查找来源——基础架构和应用程序代码都存放在 GitLab 、或者 GitHub 等版本控制系统上.这使开发团队可以提高开发和部署速度并提高应用系统可靠性.

将 GitOps 应用在持续交付流水线上,有诸多优势和特点：

1. 安全的云原生 CI/CD 管道模型
2. 更快的平均部署时间和平均恢复时间
3. 稳定且可重现的回滚（例如,根据 Git 恢复/回滚/ fork）
4. 与监控和可视化工具相结合,对已经部署的应用进行全方位的监控

在我看来 GitOps 的最大优势就是通过完善的 Git 分支管理来达到管理所有 CI/CD 管道流水线的目的,不同的环境可以对应不同分支,在该环境出现问题时候,可以直接查找对应分支代码,达到快速排查问题的目的.而对于 Git 的熟悉,更是省去学习使用一般 DevOps 工具所需的学习成本和配置时间,开发人员可以无任何培训直接上手使用,进一步降低了时间与人力成本.




## GitFlow开发模式

在更大的项目中,参与的角色更多,一般会有开发、测试、运维几种角色的划分；还会划分出开发环境、测试环境、预发布环境、生产环境等用于代码的验证和测试；同时还会有多个功能会在同一时间并行开发.可想而知 CI 的流程也会进一步复杂.

能比较好应对这种复杂性的,首选 [GitFlow 工作流](https://nvie.com/posts/a-successful-git-branching-model/), 即通过并行两个长期分支的方式规范代码的提交.而如果使用了 Github,由于有非常好用的 Pull Request 功能,可以将 GitFlow 进行一定程度的简化,最终有这样的工作流：

![gitflow.jpg](/assets/images/ci/gitflow.png)


- 以`dev` 为主开发分支,`Master`为发布分支
- 开发人员始终从`dev`创建自己的分支,如 `feature/readme`
- `feature/readme` 开发完毕后创建 PR 到 `dev` 分支,并进行 `code review`
- review 后 `feature/readme` 的新功能被合并入 `dev`,如有多个并行功能亦然
- 待当前开发周期内所有功能都合并入`dev`后,从`dev`创建 PR 到`master`
- `dev`合并入`Master`,并创建一个新的`Release`.
- 以`Release`版本为发布版本

上述是从 Git 分支角度看代码仓库发生的变化,实际在开发人员视角里,工作流程是怎样的呢.假设我是项目的一名开发人员,今天开始一期新功能的开发：

1. Clone 项目到本地,`git checkout dev`.从 dev 创建一个分支来完成新功能的开发, `git checkout -b feature/reademe`.在这个分支修改一些代码,比如将Hello World V3修改为Hello World Feature A
2. `git add .`,书写符合规范的 Commit 并提交代码, `git commit -m "feature: hello world feature A"`
3. 将代码推送到代码库的对应分支, `git push origin feature/feature/readme:feature/feature/readme`
4. 由于分支是以 feature/readme 命名的,因此 CI 会运行单元测试,并自动构建一个当前分支的镜像,发布到测试环境,并自动配置一个当前分支的域名如 test-featue-readme.test.com
5. 联系产品及测试同学在测试环境验证并完善新功能
6. 功能通过验收后发起 PR 到 dev 分支,由 Leader 进行 code review
7. Code Review 通过后,Leader 合并当前 PR,此时 CI 会运行单元测试,构建镜像,并发布到测试环境
8. 此时 dev 分支有可能已经积累了若干个功能,可以访问测试环境对应 dev 分支的域名,如 test.test.com,进行集成测试.
9. 集成测试完成后,由运维同学从 Dev 发起一个 PR 到 Master 分支,此时会 CI 会运行单元测试,构建镜像,并发布到预发布环境
10. 测试人员在预发布环境下再次验证功能,团队做上线前的其他准备工作
11. 运维同学合并 PR,CI 将为本次发布的代码及镜像自动打上版本号并书写 ChangeLog,同时发布到生产环境.

由此就完成了上文中 Checklist 所需的所有工作.虽然描述起来看似冗长,但不难发现实际作为开发人员,并没有任何复杂的操作,流程化的部分全部由 CI 完成,开发人员只需要关注自己的核心任务：按照工作流规范,写好代码,写好 Commit,提交代码即可.

接下来将介绍这个以 CI 为核心的工作流,是如何一步步搭建的.  

## 使用 Drone 构建CI工作流

为了对 CI 流程有最直观的认识,我创建了一个精简版的Github项目 [lework/ci-demo-go](https://github.com/lework/ci-demo-go) 来演示完整的流程

### 部署实验环境

#### 环境信息

| name | ip             | 配置 | 服务                                            |
| ---- | :------------- | ---- | ----------------------------------------------- |
| test | 192.168.77.133 | 2C4G | drone,gogs, register,kubernetes (kind) 测试环境 |
| prod | 192.168.77.134 | 2C4G | kubernetes (kind) 生产环境                      |

> docker环境安装和kind安装这里不在说明,关于kind安装,请看[使用Kind搭建你的本地Kubernetes集群](https://lework.github.io/2019/07/02/use-kind/)

#### 创建k8s环境

kind 集群配置文件

```yaml
#　kind-k8s.config
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta2
  kind: ClusterConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
  imageRepository: registry.aliyuncs.com/google_containers
  nodeRegistration:
    kubeletExtraArgs:
      pod-infra-container-image: registry.aliyuncs.com/google_containers/pause:3.1
  apiServer:
    certSANs:
    - localhost
    - 127.0.0.1
    - 192.168.77.133
- |
  apiVersion: kubeadm.k8s.io/v1beta2
  kind: InitConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
  imageRepository: registry.aliyuncs.com/google_containers
nodes:
- role: control-plane
  extraPortMappings:
    - containerPort: 6443
      hostPort: 6443
    - containerPort: 30080
      hostPort: 30080
    - containerPort: 30081
      hostPort: 30081
    - containerPort: 30082
      hostPort: 30082
    - containerPort: 30083
      hostPort: 30083
    - containerPort: 30084
      hostPort: 30084
    - containerPort: 30085
      hostPort: 30085
  extraMounts:
    - containerPath: /etc/docker/daemon.json
      hostPath: /etc/docker/daemon.json
      readOnly: true
```

创建k8s测试环境

```bash
# kind create cluster --name test --config kind-k8s.config
Creating cluster "test" ...
 ✓ Ensuring node image (kindest/node:v1.15.3)
 ✓ Preparing nodes
 ✓ Creating kubeadm config
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
                            Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="test")"
kubectl cluster-info

# export KUBECONFIG="$(kind get kubeconfig-path --name="test")"
# kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:05:50Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

创建k8s生产环境

```bash
# 将配置文件中的192.168.77.133地址改成192.168.77.134
#　kind create cluster --name prod --config kind-k8s.config
Creating cluster "prod" ...
 ✓ Ensuring node image (kindest/node:v1.15.3)
 ✓ Preparing nodes
 ✓ Creating kubeadm config
 ✓ Starting control-plane 
 ✓ Installing CNI
 ✓ Installing StorageClass

                            Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="prod")"
kubectl cluster-info

# export KUBECONFIG="$(kind get kubeconfig-path --name="prod")"
# kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:05:50Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

#### 部署Drone、registry和Gogs

在`test`节点上个部署drone和gogs

配置文件

```yaml
# docker-compose.yaml
version: '3'
services:
 drone-server:
   container_name: drone-server
   image: drone/drone:1.3.1
   ports:
     - 80:80
     - 443
   volumes:
     - ./drone_data:/data
     - /var/run/docker.sock:/var/run/docker.sock
     - /etc/localtime:/etc/localtime
   environment:
     - DRONE_SERVER_HOST=192.168.77.133
     - DRONE_SERVER_PROTO=http
     - DRONE_TLS_AUTOCERT=false
     - DRONE_LOGS_DEBUG=true
     - DRONE_GIT_ALWAYS_AUTH=false
     - DRONE_GOGS_SERVER=http://gogs:3000
     - DRONE_RUNNER_CAPACITY=2
     - DRONE_USER_CREATE=username:root,admin:true
   restart: always

 gogs:
   container_name: gogs
   image: gogs/gogs:0.11.91
   volumes:
     - ./gogs_data:/data
     - /etc/localtime:/etc/localtime
   ports:
     - 3000:3000
   restart: always

 registry:
   image: registry:2.7.1
   container_name: registry
   ports:
     - 5000:5000
   volumes:
     - ./registry_data:/var/lib/registry
     - /etc/localtime:/etc/localtime
   restart: always
```

使用`docker-compose`进行部署

```bash
# docker-compose -d
# docker-compose ps
    Name                  Command               State                     Ports
--------------------------------------------------------------------------------------------------
drone-server   /bin/drone-server                Up      0.0.0.0:32768->443/tcp, 0.0.0.0:80->80/tcp
gogs           /app/gogs/docker/start.sh  ...   Up      22/tcp, 0.0.0.0:3000->3000/tcp
registry       /entrypoint.sh /etc/docker ...   Up      0.0.0.0:5000->5000/tcp
```

因为我们的registry没有配置https,需要对kind里containerd配置更改,以便接受不安全的仓库请求

```bash
# test节点
docker exec test-control-plane bash -c "sed -i '56a\        [plugins.cri.registry.mirrors.\"192.168.77.133:5000\"]' /etc/containerd/config.toml"
docker exec test-control-plane bash -c "sed -i '57a\          endpoint = [\"http://192.168.77.133:5000\"]' /etc/containerd/config.toml"
docker exec test-control-plane bash -c "sed -i 's#https://registry-1.docker.io#https://docker.mirrors.ustc.edu.cn#g' /etc/containerd/config.toml"
docker exec test-control-plane bash -c "cat /etc/containerd/config.toml"
docker exec test-control-plane bash -c 'kill -s SIGHUP $(pgrep containerd)'

# prod节点
docker exec prod-control-plane bash -c "sed -i '56a\        [plugins.cri.registry.mirrors.\"192.168.77.133:5000\"]' /etc/containerd/config.toml"
docker exec prod-control-plane bash -c "sed -i '57a\          endpoint = [\"http://192.168.77.133:5000\"]' /etc/containerd/config.toml"
docker exec prod-control-plane bash -c "sed -i 's#https://registry-1.docker.io#https://docker.mirrors.ustc.edu.cn#g' /etc/containerd/config.toml"
docker exec prod-control-plane bash -c "cat /etc/containerd/config.toml"
docker exec prod-control-plane bash -c 'kill -s SIGHUP $(pgrep containerd)'
```

#### 配置Gogs

![gogs.png](/assets/images/ci/gogs.png)

创建Git仓库

![repo.png](/assets/images/ci/repo.png)

源仓库: [https://github.com/lework/ci-demo-go.git](https://github.com/lework/ci-demo-go.git)

#### 配置Drone

使用Gogs账号进行登录,激活创建的项目

![drone-active.png](/assets/images/ci/drone-active.png)

到这里,我们的实验环境已经搭建完成了,下面开始编写项目的`pipeline`

### 一步一步的编写pipeline

> 本次以go语言为例

#### 1. 定义保存缓存的目录

```bash
kind: pipeline  
name: ci-demo  

volumes:
- name: cache
  host:
    path: /tmp/cache
```

要确保drone机器上存在`/tmp/cache`这个目录

```bash
# 在test节点上执行
mkdir /tmp/cache
```

#### 2. 恢复缓存目录

我们将当前目录下`docker`目录缓存下来,这个目录用于`dind`的image存储,这样每次构建就可以直接用本地的镜像,而不用重新下载,从而节省时间.

```yaml
steps:
- name: restore-cache
  image: drillster/drone-volume-cache
  settings:
    restore: true
    mount:
      - ./docker
  volumes:
    - name: cache
      path: /cache
  when:
    ref:
      include:
        - refs/heads/feature/*
        - refs/heads/master
        - refs/heads/dev
        - refs/tags/*
    event:
      include:
        - push
        - pull_request
        - tag
```

这里需要我们将drone项目设置为truest才可以

![drone-trusted.png](/assets/images/ci/drone-trusted.png)

`when`下面表示着运行step的条件,这里设置的是`branch`分支是`feature/*`,`master`,`dev`;`event`事件是`push`的时候才会执行.

#### 3. 执行单元测试

```yaml
- name: unit-test
  image: golang:1.12.6
  environment:
    GOOS: linux
    GOARCH: amd64
    CGO_ENABLED: 0
  commands:
    - go test -v -cover
  when:
    branch:
      include:
        - feature/*
        - master
        - dev
    event:
      include:
        - push
        - pull_request
```

#### 4. 编译项目代码

简单模拟了多环境的配置管理,测试环境编译测试环境的配置,生产环境编译生产环境的配置

```yaml
- name: build-test
  image: golang:1.12.6
  environment:
    GOOS: linux
    GOARCH: amd64
    CGO_ENABLED: 0
  commands:
    - "go build -v -ldflags \"-X main.version=test -X main.build=${DRONE_BUILD_NUMBER}\" -a -o release/linux/amd64/hello"
  when:
    branch:
      include:
        - feature/*
        - master
        - dev
    event:
      include:
        - push
        - pull_request

- name: build-prod
  image: golang:1.12.6
  environment:
    GOOS: linux
    GOARCH: amd64
    CGO_ENABLED: 0
  commands:
    - "go build -v -ldflags \"-X main.version=${DRONE_TAG##v} -X main.build=${DRONE_BUILD_NUMBER}\" -a -o release/linux/amd64/hello"
  when:
    event: tag
```

`${DRONE_BUILD_NUMBER}` `${DRONE_TAG##v}` 这些都是`Drone`的环境变量

#### 5. 编译Docker镜像

项目的`Dockerfile`

```dockerfile
FROM plugins/base:multiarch

LABEL maintainer="lework <lework@yeah.net>"

ADD release/linux/amd64/hello /bin/

ENTRYPOINT ["/bin/hello"]
```

针对`gitflow`工作流中的功能分支编译对应的镜像

* `feature/a`  分支,镜像标签为 `feature-a`
* `dev`  分支,镜像标签为 `dev`
* `master`  分支,镜像标签为 `latest`
* `tag`  分支,镜像标签代码的版本号自动规划 Docker 镜像的标签, 如代码版本为`1.0.0`,将为 Docker 镜像打三个标签 `1`, `1.0`, `1.0.0`.如果代码版本号不能被解析,则镜像标签为 `latest`

```yaml
- name: build-feature-image
  image: plugins/docker
  settings:
    dockerfile: ./Dockerfile
    storage_path: /drone/src/docker
    repo: 192.168.77.133:5000/root/ci-demo-go
    registry: 192.168.77.133:5000
    mirror: https://docker.mirrors.ustc.edu.cn/
    insecure: true
    tag:
      - feature-${DRONE_BRANCH##feature/}
  when:
    branch: feature/*
    event: push

- name: build-dev-image
  image: plugins/docker
  settings:
    dockerfile: ./Dockerfile
    storage_path: /drone/src/docker
    repo: 192.168.77.133:5000/root/ci-demo-go
    registry: 192.168.77.133:5000
    mirror: https://docker.mirrors.ustc.edu.cn/
    insecure: true
    tag:
      - dev
  when:
    branch: dev
    event:
      - push

- name: build-staging-image
  image: plugins/docker
  settings:
    dockerfile: ./Dockerfile
    storage_path: /drone/src/docker
    repo: 192.168.77.133:5000/root/ci-demo-go
    registry: 192.168.77.133:5000
    mirror: https://docker.mirrors.ustc.edu.cn/
    insecure: true
    tag:
      - latest
  when:
    branch: master
    event:
      - push

- name: build-prod-image
  image: plugins/docker
  settings:
    dockerfile: ./Dockerfile
    storage_path: /drone/src/docker
    repo: 192.168.77.133:5000/root/ci-demo-go
    registry: 192.168.77.133:5000
    mirror: https://docker.mirrors.ustc.edu.cn/
    insecure: true
    auto_tag: true
    tag:
      - ${DRONE_TAG}
  when:
    event: tag
```

#### 6. kubernetes 发布

项目的`deployment`配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ci-demo-go
  labels:
    app: ci-demo-go
    environment: test
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 3
  selector:
    matchLabels:
      app: ci-demo-springboot
      environment: test
  template:
    metadata:
      labels:
        app: ci-demo-go
        environment: test
    spec:
      containers:
      - name: ci-demo-go
        image: 192.168.77.133:5000/root/ci-demo-go:dev
        imagePullPolicy: Always
        resources:
          requests:
            cpu: 10m
            memory: 10Mi
        ports:
        - name: http
          containerPort: 80
          protocol: TCP
        readinessProbe:
            httpGet:
              path: /health
              port: http
        env:
        - name: _PLEASE_REDEPLOY
          value: 'THIS_STRING_IS_REPLACED_DURING_BUILD'

---
apiVersion: v1
kind: Service
metadata:
  name: ci-demo-go
  namespace: default
  labels:
    app: ci-demo-go
    environment: test
spec:
  selector:
    app: ci-demo-go
    environment: test
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

修改deployment内容的步骤

```yaml
- name: change-deployment
  image: busybox
  commands:
    - '[ -n "$DRONE_TAG" ] && (sed -i "s/dev/${DRONE_TAG##v}/g" deployment.yml;sed -i "s/environment: test/environment: prod/g" deployment.yml)'
    - sed -i "s/THIS_STRING_IS_REPLACED_DURING_BUILD/$(date +'%Y-%m-%y %T')/g" deployment.yml
    - cat deployment.yml
  when:
    ref:
      include:
        - refs/heads/dev
        - refs/tags/*
    event:
      include:
        - push
        - tag
```

`${DRONE_TAG##v}` 是去除了v字符

发布应用

```yaml
- name: deploy-test
  image: lework/kubectl-check
  environment:
    KUBERNETES_DEPLOY: ci-demo-go
    KUBERNETES_KUBECONFIG:
      from_secret: KUBERNETES_KUBECONFIG_TEST
  when:
    branch: dev
    event: push

- name: deploy-prod
  image: lework/kubectl-check
  environment:
    KUBERNETES_DEPLOY: ci-demo-go
    KUBERNETES_KUBECONFIG:
      from_secret: KUBERNETES_KUBECONFIG_PROD
  when:
    event: tag
```

`from_secret` 是引用secret变量用的,一般带有密码含义的都建议使用drone的secrets加密存储,这样在输出日志里就不会有敏感信息存在了.

通过 Drone UI 界面中, repo -> Settings -> Secrets 添加

![drone-k8s-secrets.png](/assets/images/ci/drone-k8s-secrets.png)

`KUBERNETES_KUBECONFIG` 变量是对kuberconfig文件内容的base64加密,以下是获取方式
{% raw %}

```bash
server=https://192.168.77.133:6443   #　需改成k8s对应节点的ip
# 创建rabc权限
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deploy
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: ci-deploy
  namespace: default
rules:
  - apiGroups: ["apps", "extensions", ""]
    resources: ["pods", "deployments", "deployments/scale", "services", "replicasets"]
    verbs: ["create","get","list","patch","update"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: ci-deploy
  namespace: default
subjects:
  - kind: ServiceAccount
    name: ci-deploy
roleRef:
  kind: Role
  name: ci-deploy
  apiGroup: rbac.authorization.k8s.io
EOF

#　节点操作
kubectl get secret $(kubectl get sa ci-deploy -o go-template --template '{{range .secrets}}{{.name}}{{end}}') -o go-template --template '{{index .data "ca.crt"}}' | base64 -d > ca.crt

kubectl config set-cluster kubernetes --certificate-authority=./ca.crt  --server="$server" --embed-certs=true --kubeconfig=ci.kubeconfig

kubectl config set-credentials ci-deploy --token=$(kubectl get secret $(kubectl get sa ci-deploy -o jsonpath={.secrets[].name})  -o jsonpath={.data.token} |base64 -d) --kubeconfig=ci.kubeconfig

kubectl config set-context ci-deploy@kubernetes --cluster=kubernetes --user=ci-deploy --kubeconfig=ci.kubeconfig
kubectl config use-context ci-deploy@kubernetes --kubeconfig=ci.kubeconfig

# base64 加密
cat ci.kubeconfig | base64 -w0
```

{% endraw %}
将base64编码的kubeconfig内容设置到项目中的Secrets.

#### 7. 归档缓存

```yaml
- name: rebuild-cache
  image: drillster/drone-volume-cache
  settings:
    rebuild: true
    mount:
      - ./docker
  volumes:
    - name: cache
      path: /cache
  when:
    ref:
      include:
        - refs/heads/feature/*
        - refs/heads/master
        - refs/heads/dev
        - refs/tags/*
    event:
      include:
        - push
        - pull_request
        - tag
```

将项目生成的缓存信息,这里也就是dind的镜像文件,缓存到指定的目录中.

#### 8. 语义化发布

上面基本完成了一个支持团队协作的半自动 CI 工作流,如果不是特别苛刻的话,完全可以用上面的工作流开始干活了.

不过基于这个工作流工作一段时间,会发现仍然存在痛点,那就是每次发布都要想一个版本号,写 ChangeLog,并且人工去 release.

标记版本号涉及到上线后的回滚,追溯等一系列问题,应该是一项严肃的工作,其实如何标记版本号早已有比较好的方案,即[语义化版本](https://semver.org/lang/zh-CN/).在这个方案中,版本号一共有 3 位,形如 `1.0.0`,分别代表：

1. 主版本号：当你做了不兼容的 API 修改,
2. 次版本号：当你做了向下兼容的功能性新增,
3. 修订号：当你做了向下兼容的问题修正.

虽然有了这个指导意见,但并没有很方便的解决实际问题,每次发布要搞清楚代码的修改到底是不是向下兼容的,有哪些新的功能等,仍然要花费很多时间.

而[语义化发布 (Semantic Release)](https://semantic-release.gitbook.io/semantic-release/) 就能很好的解决这些问题.

语义化发布的原理很简单,就是让每一次 Commit 所附带的 Message 格式遵守一定规范,保证每次提交格式一致且都是可以被解析的,那么进行 Release 时,只要统计一下距离上次 Release 所有的提交,就分析出本次提交做了何种程度的改动,并可以自动生成版本号、自动生成 ChangeLog 等.

语义化发布中,Commit 所遵守的规范称为[约定式提交 (Conventional Commits)](https://www.conventionalcommits.org/zh/v1.0.0-beta.3/).比如 node.js、 Angular、Electron 等知名项目都在使用这套规范.

语义化发布首先将 Commit 进行分类,常用的分类 (Type) 有：

- **feat**: 新功能
- **fix**: BUG 修复
- **docs**: 文档变更
- **style**: 文字格式修改
- **refactor**: 代码重构
- **perf**: 性能改进
- **test**: 测试代码
- **chore**: 工具自动生成

每个 Commit 可以对应一个作用域(Scope),在一个项目中作用域一般可以指不同的模块.

当 Commit 内容较多时,可以追加正文和脚注,如果正文起始为`BREAKING CHANGE`,代表这是一个破坏性变更.

以下都是符合规范的 Commit:

```bash
feat: 增加重置密码功能
fix(邮件模块): 修复邮件发送延迟BUG

feat(API): API重构
BREAKING CHANGE: API v3上线,API v1停止支持
```

有了这些规范的 Commit,版本号如何变化就很容易确定了,目前语义化发布[默认的规则](https://github.com/semantic-release/commit-analyzer/blob/master/lib/default-release-rules.js)如下

| Commit            | 版本号变更 |
| :---------------- | :--------- |
| `BREAKING CHANGE` | 主版本号   |
| `feat`            | 次版本号   |
| `fix` / `perf`    | 修订号     |

因此在 CI 部署 semantic-release 之后,作为开发人员只需要按照规范书写 Commit 即可,其他的都由 CI 完成.

因此语义化的步骤如下:

```yaml
- name: semantic-release
  image: lework/drone-semantic-release
  settings:
    git_user_name: root
    git_user_email: root@test.com
    git_login: root
    git_password:
      from_secret: GIT_PASSWORD
  when:
    branch: master
    event: push
```

> semantic-release 没有支持gogs的创建,这里使用的本地的方式提交release

添加`GIT_PASSWORD` secret,并选中**Allow Pull Requests**

#### 9. 结果通知

```yaml
- name: notify
  image: drillster/drone-email
  settings:
    port: 25
    from: root@test.com
    host: smtp.test.com
    username: root@test.com
    password:
      from_secret: EMAIL_PASSWORD
    skip_verify: true
    recipients: [ root@test.com ]
    recipients_only: true
  when:
    status: [ success, changed, failure ]
```

但构建状态处于`success`, `changed`, `failure` 发送邮件通知,`recipients_only` 为`false`的时候,会发送给commit的用户

`EMAIL_PASSWORD`的secret信息也需要添加

![drone-k8s-secrets2.png](/assets/images/ci/drone-k8s-secrets2.png)

记得勾选**Allow Pull Requests**

这里也可以使用脚本进行添加secret,方便大家初始化项目

```
#!/bin/bash

token=MCXSO9ef0zqKoPc4cqBZ1Qkxqs5fVHxE
owner=root
repo=ci-demo-go


secret_name=GIT_PASSWORD
secret_data=12345678

curl -H "Content-Type:application/json" \
  -H "Authorization: Bearer ${token}" \
  -X POST \
  -d "{\"name\": \"${secret_name}\",\"data\": \"${secret_data}\",\"pull_request\": true}" \
  http://192.168.77.133/api/repos/${owner}/${repo}/secrets

secret_name=EMAIL_PASSWORD
secret_data=12345678

curl -H "Content-Type:application/json" \
  -H "Authorization: Bearer ${token}" \
  -X POST \
  -d "{\"name\": \"${secret_name}\",\"data\": \"${secret_data}\",\"pull_request\": true}" \
  http://192.168.77.133/api/repos/${owner}/${repo}/secrets

secret_name=KUBERNETES_KUBECONFIG_TEST
secret_data=$(cat /root/docker/ci.kubeconfig | base64 -w0)

curl -H "Content-Type:application/json" \
  -H "Authorization: Bearer ${token}" \
  -X POST \
  -d "{\"name\": \"${secret_name}\",\"data\": \"${secret_data}\",\"pull_request\": true}" \
  http://192.168.77.133/api/repos/${owner}/${repo}/secrets

secret_name=KUBERNETES_KUBECONFIG_PROD
secret_data=$(cat /root/docker/prod-ci.kubeconfig | base64 -w0)

curl -H "Content-Type:application/json" \
  -H "Authorization: Bearer ${token}" \
  -X POST \
  -d "{\"name\": \"${secret_name}\",\"data\": \"${secret_data}\",\"pull_request\": true}" \
  http://192.168.77.133/api/repos/${owner}/${repo}/secrets

curl -H "Content-Type:application/json" \
  -H "Authorization: Bearer ${token}" \
  -X PATCH \
  -d "{\"trusted\": true}" \
  http://192.168.77.133/api/repos/${owner}/${repo}
```

​至此我们的go语言以gitflow工作流的方式完成了pipeline,接下来我们就来验证下

### 模拟工作流

团队成员从 dev 分支 checkout 自己的分支 feature/readme

```bash
git checkout dev
git.exe checkout -b feature/readme dev
```

向feature/readme提交代码并 push,触发CI 运行,并构建镜像192.168.77.133:5000/root/ci-demo-go:feature-readme,最终邮件通知

```bash
git add .
git commit -m 'fix(readme): fix readme desc'
git push
```

![ci-push.png](/assets/images/ci/ci-push.png)

功能开发完成后,团队成员向 dev 分支 发起 pull request , 触发CI 运行单元测试和代码编译

![ci-pull-request1.png](/assets/images/ci/ci-pull-request1.png)

![ci-pull-request2.png](/assets/images/ci/ci-pull-request2.png)

团队组长 merge pull request, 触发CI运行,构建镜像192.168.77.133:5000/root/ci-demo-go:dev,,并且发布k8s测试环境

![ci-merge1.png](/assets/images/ci/ci-merge1.png)

![ci-merge2.png](/assets/images/ci/ci-merge2.png)

```bash
curl http://192.168.77.133:30080/version
Version: test Build: 3
```

运维人员从 dev 向 master 发起 pull request,触发CI 运行单元测试和代码编译

![ci-master-pull-request1.png](/assets/images/ci/ci-master-pull-request1.png)

![ci-master-pull-request2.png](/assets/images/ci/ci-master-pull-request2.png)

运维人员 merge pull request, 触发CI 运行,并构建镜像192.168.77.133:5000/root/ci-demo-go:latest

![ci-master-merge1.png](/assets/images/ci/ci-master-merge1.png)

![ci-master-merge2.png](/assets/images/ci/ci-master-merge2.png)

并且 release 新版本`v1.0.0`, CI 构建镜像,并且发布k8s生产环境

- 192.168.77.133:5000/root/ci-demo-go:1
- 192.168.77.133:5000/root/ci-demo-go:1.0
- 192.168.77.133:5000/root/ci-demo-go:1.0.0

![ci-tag.png](/assets/images/ci/ci-tag.png)

release 的 changlog

![changlog.png](/assets/images/ci/changlog.png)

```bash
curl http://192.168.77.134:30080/version
Version: 1.0.0 Build: 6
```

至此,我们的工作流也完成了,总的来说`drone`对使用版本管理规范的团队来说是极大的解放生产力,如果团队没有规范的使用版本管理,看到`drone`这么优秀,快速实行起来吧.

## 更多语言的Ci流程

- [ci-demo-go](https://github.com/lework/ci-demo-go.git)
- [ci-demo-django](https://github.com/lework/ci-demo-django.git)
- [ci-demo-yii2](https://github.com/lework/ci-demo-yii2.git)
- [ci-demo-vue](https://github.com/lework/ci-demo-vue.git)
- [ci-demo-springboot](https://github.com/lework/ci-demo-springboot.git)

## 参考

- https://nvie.com/posts/a-successful-git-branching-model/
- https://avnpc.com/pages/drone-gitflow-kubernetes-for-cloud-native-ci
- http://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html