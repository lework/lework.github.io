---
layout: post
title: "Devops 相关面试"
date: "2020-10-14 19:10"
category: 运维
tags: 运维 面试 devops
author: lework
---
* content
{:toc}

# Devops 相关面试 

群里一位大佬的devops 开发面试题



## Python相关

### 1、谈下python 的 GIL x
**GIL：又叫全局解释器锁**，每个线程在执行的过程中都需要先获取GIL，保证同一时刻只有一个线程在运行，目的是解决多线程同时竞争程序中的全局变量而出现的线程安全问题。

Python语言和GIL没有什么关系。仅仅是由于历史原因在Cpython虚拟机(解释器)，难以移除GIL。

**线程释放GIL锁的情况：** 在IO操作等可能会引起阻塞的system call之前,可以暂时释放GIL,但在执行完毕后,必须重新获取GIL Python 3.x使用计时器（执行时间达到阈值后，当前线程释放GIL）或Python 2.x，tickets计数达到100。
**解决GIL问题:** python使用多进程是可以利用多核的CPU资源。多线程爬取比单线程性能有提升，因为遇到IO阻塞会自动释放GIL锁。

### 2、简述面向对象中__new__ 和___init__区别
- `__init__`是初始化方法，创建对象后，就立刻被默认调用了，可接收参数.
- `__new__`至少要有一个参数cls，代表当前类，此参数在实例化时由Python解释器自动识别
- `__new__`必须要有返回值，返回实例化出来的实例，这点在自己实现`__new__`时要特别注意，可以return父类（通过super(当前类名, cls)）`__new__`出来的实例，或者直接是object的`__new__`出来的实例
- `__init__`有一个参数self，就是这个`__new__`返回的实例，`__init__`在`__new__`的基础上可以完成一些其它初始化的动作，`__init__`不需要返回值
- 如果`__new__`创建的是当前类的实例，会自动调用`__init__`函数，通过return语句里面调用的`__new__`函数的第一个参数是cls来保证是当前类实例，如果是其他类的类名，；那么实际创建返回的就是其他类的实例，其实就不会调用当前类的`__init__`函数，也不会调用其他类的`__init__`函数。

### 3、python2和 python3区别?请列举几个
- Python3 对 Unicode 字符的原生支持。
- Python2 中使用 ASCII 码作为默认编码方式导致 string 有两种类型 str 和 unicode，Python3 只支持 unicode 的 string。
- Python3 采用的是绝对路径的方式进行 import
- Python2中存在老式类和新式类的区别，Python3统一采用新式类。
- Python3 使用更加严格的缩进。Python2 的缩进机制中，1 个 tab 和 8 个 space 是等价的
- print 语句被 Python3 废弃，统一使用 print 函数
- exec 语句被 python3 废弃，统一使用 exec 函数
- 不相等操作符"<>"被 Python3 废弃，统一使用"!="
- long 整数类型被 Python3 废弃，统一使用 int
- xrange 函数被 Python3 废弃，统一使用 range
- 迭代器 iterator 的 next()函数被 Python3 废弃，统一使用 next(iterator)
- raw_input 函数被 Python3 废弃，统一使用 input 函数
- Python2，for 循环会修改外部相同名称变量的值,python3不会
- Python2，round 函数返回 float 类型值， python3 返回int 类型值
- 比较操作符: Python2 中任意两个对象都可以比较, Python3 中只有同一数据类型的对象可以比较

### 4、list和 tuple的区别

- list是一种有序的集合，可以随时添加和删除其中的元素
- tuple是一种有序列表, 一旦初始化就不能修改
- tuple没有append() insert()方法，可以获取元素但不能赋值变成另外的元素
- tuple 用( )， list用[ ]

### 5、range和xrange

- range和xrange都是在循环中使用，输出结果一样。
- range返回的是一个list对象，而xrange返回的是一个生成器对象(xrange object)。
- xrange则不会直接生成一个list，而是每次调用返回其中的一个值，内存空间使用极少，因而性能非常好。

### 6、深浅copy的区别

- 浅copy:外层内存地址改变，里边内存地址不变，共享内存地址。
  ```python
  source= [1,2,3,4]
  target = source.copy()
  ```
- 深copy:完完全全复制了一份，两个内存地址完全不同，没有任何关系。
  ```python
  import copy
  source = [[1,2],[3,4]]
  target = copy.deepcopy(source)
  ```

### 7、session和 cookie 分别是什么，有什么区别?

- Cookie实际上是一小段的文本信息。客户端请求服务器，如果服务器需要记录该用户状态，就使用response向客户端浏览器颁发一个Cookie。客户端会把Cookie保存起来。
- Session是另一种记录客户状态的机制，不同的是Cookie保存在客户端浏览器中，而Session保存在服务器上。客户端浏览器访问服务器的时候，服务器把客户端信息以某种形式记录在服务器上。这就是Session。客户端浏览器再次访问时只需要从该Session中查找该客户的状态就可以了。

1. Session保存在服务端，Cookie保存在客户端。
2. 因为Cookie存在客户端，用户可以看见进行编辑伪造，所以安全性低。
3. Cookie不是很安全，别人可以分析存放在本地的Cookie，进行Cookie欺骗；Session会在一定时间内保存在服务器上，当访问增多，会占用服务器性能。
4. Cookie保存数据不能超过4k，很多浏览器都限制一个站点最多存放20个Cookie。

### 8、请写一个装饰器
```python
def log(func):
    def wrapper(*args, **kw):
        print 'call %s():' % func.__name__
        return func(*args, **kw)
    return wrapper

@log
def now():
    print '2016-7-14'
    return "done"
```
### 9、谈下你对restfulapi的理解

- API：应用程序接口，也可以叫应用程序界面，或者简称为应用接口。应用程序的设计可以相当复杂，但最终的用户并不需要知道应用程序的内部到底是如何工作的，你只需要给用户提供一些操作接口，再告诉用户怎么用这些接口就行了。
- REST：表现层状态转移，通俗的说法是：URL定位资源，用HTTP动词（GET，POST，DELETE，PUSH等）描述操作。
- Restful：基于REST构建的API就是Restful风格
- 动作有几个类型，比如获取（GET），提交（POST），修改（PUT / PATCH），删除（DELETE）。比如你想得到一个课程列表资源，完成这项任务可以用 HTTP 的 GET 方法去请求应用提供的某个地址（接口 / API）。GET /api/v1/courses。我要删除掉 ID 号是 13 的课程资源，DELETE /api/v1/courses/13。

- 如果有人对你说，我的应用支持 REST 接口。他的意思就是，你可以通过 HTTP + 具体的动作去处理他的应用上的一些资源（Resources），比如文章，评论，文件，用户 ...

### 10、跨域有哪些解决方案(至少列举3种）

- JSONP跨域
  jsonp的原理就是利用`<script>`标签没有跨域限制，通过`<script>`标签src属性，发送带有callback参数的GET请求，服务端将接口返回数据拼凑到callback函数中，返回给浏览器，浏览器解析执行，从而前端拿到callback函数返回的数据。
- 跨域资源共享（CORS）
  CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。它允许浏览器向跨源服务器，发出XMLHttpRequest请求，从而克服了AJAX只能同源使用的限制。
- nginx代理跨域 
  nginx代理跨域，实质和CORS跨域原理一样，通过配置文件设置请求响应头Access-Control-Allow-Origin…等字段。
- nodejs中间件代理跨域
  node中间件实现跨域代理，原理大致与nginx相同，都是通过启一个代理服务器，实现数据的转发
- document.domain + iframe跨域
  此方案仅限主域相同，子域不同的跨域应用场景。实现原理：两个页面都通过js强制设置document.domain为基础主域，就实现了同域。
- location.hash + iframe跨域
  实现原理： a欲与b跨域相互通信，通过中间页c来实现。 三个页面，不同域之间利用iframe的location.hash传值，相同域之间直接js访问来通信。
- window.name + iframe跨域
  window.name属性的独特之处：name值在不同的页面（甚至不同域名）加载后依旧存在，并且可以支持非常长的 name 值（2MB）。
- postMessage跨域
  postMessage是HTML5 XMLHttpRequest Level 2中的API，且是为数不多可以跨域操作的window属性之一
- WebSocket协议跨域
  WebSocket protocol是HTML5一种新的协议。它实现了浏览器与服务器全双工通信，同时允许跨域通讯，是server push技术的一种很好的实现。
  原生WebSocket API使用起来不太方便，我们使用Socket.io，它很好地封装了webSocket接口，提供了更简单、灵活的接口，也对不支持webSocket的浏览器提供了向下兼容。

### 11、简述多进程、多线程、协程

- 并行：多个CPU核心，不同的程序就分配给不同的CPU来运行。可以让多个程序同时执行。
- 并发：单个CPU核心，在一个时间切片里一次只能运行一个程序，如果需要运行多个程序，则串行执行。
- 多进程/多线程：表示可以同时执行多个任务，进程和线程的调度是由操作系统自动完成。
- 进程：一个运行的程序（代码）就是一个进程，没有运行的代码叫程序，进程是系统资源分配的最小单位，进程拥有自己独立的内存空间，所以进程间数据不共享，开销大
- 线程：调度执行的最小单位，也叫执行路径，不能独立存在，依赖进程存在一个进程至少有一个线程，叫主线程，而多个线程共享内存（数据共享，共享全局变量），从而极大地提高了程序的运行效率。
- 协程：是一种用户态的轻量级线程，协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存在其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。

**Python的多线程：**

GIL 全局解释器锁：线程的执行权限，在Python的进程里只有一个GIL。
一个线程需要执行任务，必须获取GIL。
好处：直接杜绝了多个线程访问内存空间的安全问题。
坏处：Python的多线程不是真正多线程，不能充分利用多核CPU的资源。
但是，在I/O阻塞的时候，解释器会释放GIL。

多进程：密集CPU任务，需要充分使用多核CPU资源（服务器，大量的并行计算）的时候，用多进程。 multiprocessing
缺陷：多个进程之间通信成本高，切换开销大。


多线程：密集I/O任务（网络I/O，磁盘I/O，数据库I/O）使用多线程合适。
threading.Thread、multiprocessing.dummy
缺陷：同一个时间切片只能运行一个线程，不能做到高并行，但是可以做到高并发。

协程：又称微线程，在单线程上执行多个任务，用函数切换，开销极小。不通过操作系统调度，没有进程、线程的切换开销。genvent，monkey.patchall
多线程请求返回是无序的，那个线程有数据返回就处理那个线程，而协程返回的数据是有序的。
缺陷：单线程执行，处理密集CPU和本地磁盘IO的时候，性能较低。处理网络I/O性能还是比较高.

## 二、Linux相关

### 1、如何优化Linux系统，简单列举几项

- 在资源限制配置文件中，加大打开文件数，进程数，nofile noproc
- 调整虚拟内存swap的使用 vm.swappiness
- 关闭大内存页 echo never > /sys/kernel/mm/transparent_hugepage/enabled; echo never > /sys/kernel/mm/transparent_hugepage/defrag
- 增加文件句柄和inode缓存的大小：fs.file-max, fs.nr_open
- 加大conntrack表大小 net.nf_conntrack_max
- 加大积压套接字的最大数量 net.core.somaxconn
- 扩大允许的本地端口范围 net.ipv4.ip_local_port_range
- 减少tcp_fin_timeout连接的时间默认值 net.ipv4.tcp_fin_timeout

### 2、redis RDB和AOF的区别

- RDB持久化（原理是将Reids在内存中的数据库记录定时dump到磁盘上的RDB持久化）
- AOF持久化（原理是将Reids的操作日志以追加的方式写入文件）

- AOF持久化以日志的形式记录服务器所处理的每一个写、删除操作，查询操作不会记录，以文本的方式记录，可以打开文件看到详细的操作记录。
- RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。

### 3、Nginx有哪些用途及优点?

Nginx是一款自由、开源、高性能的HTTP服务器和反向代理服务器，可以当做IMAP，POP3,SMTP代理服务器。

1. 更快
2. 高扩展性
3. 高可靠性
4. 低内存消耗
5. 单机支持10万以上的并发连接
6. 热部署
7. 最自由的BSD许可协议
8. 支持lua脚本，可以实现waf，api网关等功能。

### 4、代码部署方案、发布流程是怎样的?

采用分支开发，主干发布的策略
从主干拉出特性分支，开发在特性分支上开发功能，开发完成后，合并到测试分支，由测试验证功能，验证通过后，合并到主干上，由测试进行回归性验证，验证通过后，提交发布请求到运维，由运维发起发布任务。

### 5、k8s主要功能有哪些，请列举几项

Kubernetes 是一个跨主机集群的 开源的容器调度平台，它可以自动化应用容器的部署、扩展和操作 , 提供以容器为中心的基础架构。

- 自愈: 重新启动失败的容器，在节点不可用时，替换和重新调度节点上的容器，对用户定义的健康检查不响应的容器会被中止，并且在容器准备好服务之前不会把其向客户端广播。
- 弹性伸缩: 通过监控容器的cpu的负载值,如果这个平均高于80%,增加容器的数量,如果这个平均低于10%,减少容器的数量
- 服务的自动发现和负载均衡: 不需要修改您的应用程序来使用不熟悉的服务发现机制，Kubernetes 为容器提供了自己的 IP 地址和一组容器的单个 DNS 名称，并可以在它们之间进行负载均衡。
- 滚动升级和一键回滚: Kubernetes 逐渐部署对应用程序或其配置的更改，同时监视应用程序运行状况，以确保它不会同时终止所有实例。 如果出现问题，Kubernetes会为您恢复更改，利用日益增长的部署解决方案的生态系统。 

### 6、calico 的 iptables 和 ipvs 的区别

- ipvs 具有调度算法的负载均衡器，例如循环，最短延迟，最少连接，权重等。iptables则只有随机调度。
- kube-proxy 的iptables 规则数目随着集群的扩大成几何数增长，更新一次规则需要很长时间。但Calico广泛的使用ipset来优化规则链。
- iptables  按顺序处理规则，而IPVS使用hash表。 随着服务的增加，ipvs的性能保持不变。随着服务数量增加到几千个，IPTables的性能变得不可预测和缓慢。
- iptables 模式中 kube-proxy 在 `NAT pre-routing` Hook 中实现它的 NAT 和负载均衡功能。而ipvs是在netfilter的INPUT链 上做dnat

### 7、calico 与 flannel 区别

flannel是overlay network, 主要是L2(VXLAN)。 calico主要是L3，用BGP路由。

- flannel vxlan和calico ipip模式都是隧道方案，但是calico的封装协议IPIP的header更小，所以性能比flannel vxlan要好一点点。
- flannel 旨在解决不同host上的容器网络互联问题，大致原理是每个 host 分配一个 subnet，容器从此 subnet 中分配 IP，这些 IP 可以在 host 间路由，容器间无需 NAT 和 port mapping 就可以跨主机通信。
- calico是一个纯三层的数据中心网络方案，实现类似于flannel host-gw,不过它没有复用docker 的docker0 bridge，而是自己实现的。
- Calico基于iptables还提供了丰富而灵活的网络Policy，保证通过各个节点上的ACLs来提供Workload的多租户隔离、安全组以及其他可达性限制等功能。flannel 无策略处理。

## 三、前端相关

### 1、简单说下 vue/react的生命周期，有哪些阶段

**vue 生命周期**

- `beforeCreate` 实例创建前：这个阶段实例的data、methods是读不到的
- `created` 实例创建后：这个阶段已经完成了数据观测(data observer)，属性和方法的运算， watch/event 事件回调。mount挂载阶段还没开始，$el 属性目前不可见，数据并没有在DOM元素上进行渲染
- `beforeMount`：在挂载开始之前被调用：相关的 render 函数首次被调用。
- `mounted`：el选项的DOM节点 被新创建的 vm.$el 替换，并挂载到实例上去之后调用此生命周期函数。此时实例的数据在DOM节点上进行渲染
- `beforeUpdate`：数据更新时调用，但不进行DOM重新渲染，在数据更新时DOM没渲染前可以在这个生命函数里进行状态处理
- `updated`：这个状态下数据更新并且DOM重新渲染，当这个生命周期函数被调用时，组件 DOM 已经更新，所以你现在可以执行依赖于 DOM 的操作。当实例每次进行数据更新时updated都会执行
- `beforeDestory`：实例销毁之前调用。
- `destroyed`：Vue 实例销毁后调用。调用后，Vue 实例指示的所有东西都会解绑定，所有的事件监听器会被移除，所有的子实例也会被销毁。

业务应用

- `created`：进行ajax请求异步数据的获取、初始化数据
- `mounted`：挂载元素内dom节点的获取
- `nextTick`：针对单一事件更新数据后立即操作dom
- `updated`：任何数据的更新，如果要做统一的业务逻辑处理
- `watch`：监听具体数据变化，并做相应的处理

**react 生命周期**

挂载

- componentWillMount（过时）
- componentDidMount

更新

- componentWillReceiveProps（过时）
- shouldComponentUpdate
- componentWillUpdate（过时）
- componentDidUpdate

卸载

- componentWillUnmount

### 2、父组件如何向子组件传递数据

1. 父组件向子组件传递数据，首先要在父组件中使用`v-bind`命令将要传递的数据绑定到子组件上。

2. 父组件中完成数据绑定之后，在子组件中的props属性接收一下父组件传递过来的数据，要接收多少数据，就在pros属性中写多少数据。

- 子组件通过`$emit`方法向父组件传递参数



## 四、综合类
### 1、什么是 SRE，什么DevOps，有何区别

SRE：全称Site reliability engineering， 网站稳定性工程师是致力于打造「高扩展、高可用系统」，并将其贯彻为原则的软件工程师。

DevOps（Development 和 Operations 的组合词）是一种重视“软件开发人员（Dev）”和“IT 运维技术人员（Ops）”之间沟通合作的文化、运动或惯例。

- 两者职能不同
  - SRE 工程师主要是提高业务系统的高可用性、可扩展性，参与业务系统设计和实施，定位、处理、管理故障，优化导致发生相关部件等。
  - DevOps 是一种文化，没有具体的职能，但DevOps 文化目的是提交交付速度，其工程师需要管理应用全生命周期，关注全流程效率提升，自动化运维平台设计和研发

DevOps 首先是一种文化，后期逐渐独立成一个职位；SRE 一开始就明确是一个职位，其核心目标是提升业务系统稳定性。

DevOps 以文化为指引，建立一种可以快速交付价值并且具有持续改进能力的现代化 IT 组织, 而实现SRE目标是少不了 DevOps 倡导的快速交付思想。

### 2、什么是分布式架构

服务系统分散部署在不同服务器组成一个整体应用，分散压力，解决高并发。

### 3、什么是微服务架构

微服务架构强调的一个重点是“业务需要彻底的组件化和服务化”，原有的单个业务系统会拆分为多个可以独立开发、设计、运行的小应用。这些小应用之间通过服务完成交互和集成。

- 从概念理解，分布式服务架构强调的是服务化以及服务的分散化，微服务则更强调服务的专业化和精细分工；

- 从实践的角度来看，微服务架构通常是分布式服务架构，反之则未必成立。

- 所以，选择微服务通常意味着需要解决分布式架构的各种难题。

- **SOA（Service Oriented Architecture）“面向服务的架构”:**他是一种设计方法，其中包含多个服务， 服务之间通过相互依赖最终提供一系列的功能。一个服务 通常以独立的形式存在于操作系统进程中。各个服务之间 通过网络调用。

- 区别

	- 架构不同：微服务的设计是为了不因为某个模块的升级和BUG影响现有的系统业务。微服务与分布式的细微差别是，微服务的应用不一定是分散在多个服务器上，他也可以是同一个服务器。　

	- 作用不同：分布式：不同模块部署在不同服务器上，分布式主要解决的是网站高并发带来问题。微服务：各服务可独立应用，组合服务也可系统应用。
  

	- 粒度不同：微服务相比分布式服务来说，它的粒度更小，服务之间耦合度更低，由于每个微服务都由独立的小团队负责，因此它敏捷性更高，分布式服务最后都会向微服务架构演化，这是一种趋势， 不过服务微服务化后带来的挑战也是显而易见的，例如服务粒度小，数量大，后期运维将会很难。