# 第一章：Kubernetes 从入门到生产环境

提示词：

```
你是一个专业的K8S运维师。我是一个准备入门K8S进行应用部署的新手，请你把使用K8S需要用到的概念、知识和操作方法给我详细介绍。遇到比较复杂的过程或者难理解的名词，请用通俗易懂的方式去解释它。接下来你的回复我将会进行笔记，并用于我以后K8S的入门到生产环境的使用，叙述过程详细一点也没关系。
```

正文：

## 第一节：为什么会出现 Kubernetes？

学习任何技术，先知道它为什么存在。

很多新人一开始就去学 Pod、Deployment、Service，结果越学越乱。

因为他们不知道：

> Kubernetes 到底解决了什么问题？

我们先从最原始的网站部署开始。

------

# 第一代：直接部署在服务器

假设你的 ASP.NET Core 网站。

```
ASP.NET Core
```

直接运行

```
dotnet MyApi.dll
```

网站就启动了。

客户端访问：

```
浏览器
    │
    │
服务器
    │
dotnet MyApi.dll
```

这是最简单的部署方式。

问题来了。

如果：

服务器突然重启了。

你的程序没了。

怎么办？

于是大家开始使用：

Windows

```
IIS
```

Linux

```
systemd
Supervisor
```

让程序自动启动。

但是新的问题来了。

------

## 如果程序越来越多

例如：

```
API
后台管理
认证中心
消息服务
支付服务
订单服务
```

现在一台服务器上有：

```
20个程序
```

每一个都需要：

```
启动
停止
查看日志
更新
```

运维开始崩溃。

------

## 如果有很多服务器

例如：

```
Server1
Server2
Server3
Server4
...
Server50
```

每台机器都部署：

```
API
Redis
RabbitMQ
Nginx
```

更新一次版本。

你需要：

登录

```
50台服务器
```

复制文件

重启程序

检查日志

任何一台失败：

整个发布失败。

所以出现了：

> **配置管理工具**

例如：

Ansible

SaltStack

Puppet

这些工具可以批量部署。

但是。

新的问题又来了。

------

# 如果服务器坏了呢？

例如：

```
Server3

突然断电
```

你的 API 在：

```
Server3
```

用户访问：

```
500 Error
```

怎么办？

于是。

大家开始做：

```
负载均衡
```

例如：

```
        Nginx

      /   |   \

Server1
Server2
Server3
```

任何一个挂掉。

还有另外两个。

不错。

但是：

如果：

```
Server1

CPU 100%
```

另外两台：

```
CPU 5%
```

Nginx 不知道。

它还是继续转发。

资源浪费。

------

# 如果程序需要扩容呢？

例如：

双十一。

订单暴涨。

平时：

```
2个API
```

今天需要：

```
20个API
```

怎么办？

以前：

运维：

登录服务器。

复制程序。

启动。

配置Nginx。

非常慢。

------

# 如果服务器宕机呢？

例如：

```
Server2

突然烧了。
```

API 没了。

以前：

人工部署。

现在：

希望：

> 自动恢复。

------

# 如果程序升级呢？

例如：

现在版本：

```
v1
```

需要升级：

```
v2
```

不能停机。

怎么办？

于是：

需要：

```
滚动升级
```

例如：

```
10个实例

先升级1个

没问题

继续升级

直到全部完成
```

整个过程中：

用户完全无感。

------

## 如果升级失败呢？

升级：

```
v2
```

发现：

BUG。

希望：

一分钟恢复：

```
v1
```

于是需要：

```
自动回滚
```

------

## 如果程序越来越多呢？

例如：

```
1000个容器
```

谁负责：

启动？

停止？

监控？

健康检查？

调度？

资源分配？

节点故障？

全部人工？

显然不可能。

于是。

容器编排平台出现了。

------

# Kubernetes 就是干这个的

一句话：

> **Kubernetes 是一个自动管理容器的平台。**

注意。

不是：

管理程序。

而是：

**管理容器(Container)。**

为什么？

因为：

现代应用已经大量采用：

```
Docker
```

所以：

K8S 管理的是：

```
Docker镜像
↓

运行成容器

↓

组成整个应用
```

所以。

Docker 和 Kubernetes 的关系：

```
Docker
负责：

打包

↓

Kubernetes

负责：

部署
调度
扩容
恢复
升级
```

一句话总结：

> **Docker 解决"怎么把应用装进一个标准盒子（容器）"，Kubernetes 解决"怎么管理成千上万个盒子"。**

------

# 一个生活中的例子

假设。

你经营一家奶茶店。

没有 K8S：

```
老板自己：

招员工

安排岗位

有人请假

重新排班

机器坏了

自己修

忙不过来

继续招人
```

老板累死。

有了 K8S：

老板只需要说：

```
我要：

5个收银员

3个奶茶师
```

如果：

有人下班。

K8S：

自动再安排一个。

如果：

机器坏了。

自动换机器。

老板根本不用管。

这就是：

**声明式管理（Declarative Management）**。

------

# Kubernetes 最重要的一句话

以后无论学什么资源。

都围绕一句话：

> **我不告诉 Kubernetes 怎么做，我只告诉它我要什么。**

例如：

以前：

```
启动：

API

启动：

Redis

启动：

Nginx
```

这是：

命令式。

而 K8S：

只写：

```
我要：

API：

3个副本
```

K8S 自己：

创建：

```
API1
API2
API3
```

如果：

API2 挂了。

不用管。

K8S：

重新创建。

所以。

你描述的是：

**期望状态（Desired State）**。

K8S 保证：

**实际状态（Actual State）**。

始终等于：

**期望状态（Desired State）**。

这是 Kubernetes 最核心的思想，也是之后学习所有资源（Pod、Deployment、StatefulSet、DaemonSet 等）的共同基础。

------

# Kubernetes 到底负责哪些事情？

可以把 Kubernetes 看成一个超级运维机器人，它会持续观察整个集群，并自动完成很多工作：

| 功能           | 作用                   | 现实例子                         |
| -------------- | ---------------------- | -------------------------------- |
| 容器调度       | 决定容器运行在哪台机器 | 自动把新的 API 放到空闲服务器    |
| 自动恢复       | 容器崩溃自动重建       | API 崩溃后几秒内重新启动         |
| 服务发现       | 应用之间互相找到彼此   | API 自动找到 Redis、数据库等服务 |
| 负载均衡       | 请求自动分发           | 多个 API 实例共同处理请求        |
| 滚动升级       | 不停机升级版本         | 用户几乎无感知完成升级           |
| 自动回滚       | 新版本异常时恢复旧版本 | 发布失败快速恢复                 |
| 自动扩缩容     | 根据负载增加或减少实例 | 高峰期自动扩容，低峰期节省资源   |
| 配置与密钥管理 | 管理配置文件和密码     | 数据库连接串、API Key 等统一管理 |

------

# 本系列课程规划

为了让知识循序渐进，我们后续可以按下面的路线学习：

1. **Kubernetes 整体架构**：Master（控制平面）、Worker（工作节点）、API Server、Scheduler、Controller、etcd 等组件如何协同工作。
2. **容器基础与镜像**：Docker 镜像、容器生命周期、镜像仓库，以及为什么 Kubernetes 以容器为基本运行单元。
3. **Pod**：Kubernetes 最小部署单位，为什么不是直接部署容器，一个 Pod 可以包含哪些容器。
4. **Deployment**：如何声明"我要运行 3 个 API 实例"，以及滚动升级、回滚的实现方式。
5. **Service 与网络**：Pod 为什么 IP 会变化、Service 如何提供稳定访问地址、ClusterIP、NodePort、LoadBalancer、Ingress 的区别。
6. **配置管理**：ConfigMap、Secret，如何优雅管理配置和敏感信息。
7. **存储**：Volume、PersistentVolume（PV）、PersistentVolumeClaim（PVC）、StorageClass，解决数据库等有状态应用的数据持久化问题。
8. **调度与资源管理**：Requests、Limits、NodeSelector、Affinity、Taints/Tolerations 等，让应用合理使用 CPU、内存并部署到合适的节点。
9. **可观测性**：日志、监控、健康检查（Liveness/Readiness/Startup Probe）。
10. **生产环境部署**：结合你的 **ASP.NET Core + Vben Admin + Aspire** 项目，逐步完成镜像制作、私有镜像仓库、K8S 部署、Ingress、HTTPS、CI/CD，最终实现完整的生产部署。

------

这一章只有一个目标：**建立 Kubernetes 的世界观**。

如果把这一章理解透了，后面的 Pod、Deployment、Service 就不再是零散的概念，而是一个完整系统中的各个角色。下一章，我们将进入 **Kubernetes 架构**，把控制平面（Control Plane）和工作节点（Worker Node）的职责、每个核心组件的工作流程，以及一条用户请求如何从 `kubectl apply` 一直流转到容器真正启动的全过程详细讲清楚。

# 第二章：Kubernetes 整体架构（Control Plane 与 Worker Node）

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们真正进入 **Kubernetes 的核心**。

很多教程都是一上来介绍 Pod、Deployment，但实际上，**如果不知道 Kubernetes 的整体架构，就很难理解为什么 Deployment 创建出来的是 Pod、为什么 Pod 会跑到某一台机器、为什么容器挂了会自动恢复。**

所以，这一章我们不急着写 YAML，而是先建立一张完整的"脑地图"。

## 本章学习目标

学完这一章，你应该能回答下面几个问题：

- Kubernetes 集群是什么？
- 为什么需要 Master（现在称为 Control Plane）？
- Worker Node 是干什么的？
- API Server 为什么是整个 K8S 的核心？
- Scheduler 如何决定 Pod 跑到哪台机器？
- Controller 为什么能自动恢复 Pod？
- kubelet、Container Runtime、kube-proxy 分别负责什么？
- etcd 为什么说是 Kubernetes 的"大脑"？

如果这些都理解了，后面学习任何资源都会轻松很多。

------

# 第一节：什么是 Kubernetes 集群？

很多人把 Kubernetes 想成一个软件，其实它更像一个**管理系统**。

例如，你有三台服务器：

```
服务器A
服务器B
服务器C
```

以前，它们彼此没有关系。

你需要分别登录：

```
ssh serverA

ssh serverB

ssh serverC
```

分别部署程序。

而 Kubernetes 做了一件事情：

> **把很多服务器组织成一个整体。**

例如：

```
             Kubernetes Cluster（集群）

      +------------------------------------+
      |                                    |
      |   ServerA    ServerB    ServerC    |
      |                                    |
      +------------------------------------+
```

以后你面对的不是：

> 三台服务器

而是：

> 一个 Kubernetes 集群。

这是 Kubernetes 的第一个思想。

------

# 集群（Cluster）是什么？

可以理解成：

> 一群机器组成一个整体，对外表现成一台超级服务器。

例如：

以前：

```
我要部署：

API

需要：

登录A
```

现在：

你只需要告诉 Kubernetes：

```
我要部署 API
```

至于：

放在哪台机器。

不用管。

这就是：

> **资源调度。**

------

# 集群里面有哪些机器？

整个 Kubernetes 集群一般分两类：

```
              Kubernetes Cluster

            ┌──────────────────────┐
            │                      │
            │   Control Plane      │
            │                      │
            └──────────┬───────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Worker1        Worker2       Worker3
```

只有两种角色：

> **Control Plane**

和

> **Worker Node**

以后所有东西都围绕这两个角色。

------

# 第二节：什么是 Control Plane？

以前大家叫：

```
Master
```

现在官方统一叫：

> **Control Plane（控制平面）**

为什么改名字？

因为：

Master 容易让人误会：

> 它是不是运行应用？

其实不是。

Control Plane 的职责只有一个：

> **管理整个集群。**

注意：

**它一般不运行你的业务程序。**

它负责：

- 接收命令
- 调度
- 自动恢复
- 保存配置
- 监控整个集群

可以理解成：

公司的：

```
总经理
```

员工不是它。

它负责：

安排工作。

------

# Worker Node 是什么？

Worker 就是真正干活的人。

例如：

你的：

```
ASP.NET API
```

最终运行：

```
Worker Node
```

Redis：

运行：

```
Worker Node
```

RabbitMQ：

运行：

```
Worker Node
```

Nginx：

运行：

```
Worker Node
```

所以：

Worker 的职责就是：

> **运行 Pod。**

------

# 一个生活中的例子

假设：

你开了一家快递公司。

总部：

负责：

```
接订单

安排司机

规划路线

统计数据
```

但是。

总部：

并不送快递。

真正送快递的是：

```
司机
```

对应关系：

```
总部

↓

Control Plane

=================

司机

↓

Worker Node
```

------

# 第三节：Control Plane 内部有哪些组件？

这是整个 Kubernetes 最重要的一张图。

```
                kubectl

                   │

                   ▼

             API Server

      ┌────────┼─────────┐

      ▼        ▼         ▼

 Scheduler Controller   etcd

                   │

                   ▼

             Worker Node
```

以后你看到任何资源。

都会经过：

API Server。

所以：

先讲它。

------

# API Server（整个 Kubernetes 的核心）

可以说：

> **API Server 是 Kubernetes 的大门。**

所有操作。

全部经过它。

例如：

你输入：

```
kubectl apply -f api.yaml
```

其实：

kubectl 并没有：

创建 Pod。

kubectl 做的事情只有：

> 调用 API Server。

也就是说：

```
kubectl

↓

HTTP 请求

↓

API Server
```

API Server 收到：

```
Deployment
```

以后：

再交给其它组件。

所以：

API Server 就像：

政府大厅。

所有业务：

必须先来这里。

任何组件：

不能绕过它。

------

# etcd 是什么？

很多新人以为：

etcd 是数据库。

没错。

但是：

它不是业务数据库。

它保存的是：

整个 Kubernetes 的配置。

例如：

你创建：

```
Deployment

副本：

3
```

API Server：

收到以后：

第一件事：

不是创建 Pod。

而是：

先保存到：

```
etcd
```

为什么？

因为：

Kubernetes 的设计思想：

叫：

> **声明式系统。**

它必须知道：

用户：

到底想要什么。

所以。

etcd 保存的是：

> **期望状态（Desired State）**

例如：

```
Deployment

API

副本：

3
```

etcd：

保存：

```
API

Replica=3
```

不是：

保存：

程序。

而是：

保存：

配置。

所以：

etcd 可以理解成：

整个 Kubernetes 的：

> **记忆。**

------

# Scheduler（调度器）

现在。

API Server：

知道：

需要：

```
3个Pod
```

那么：

问题来了。

放哪台机器？

例如：

```
Worker1

CPU：

95%

================

Worker2

CPU：

20%

================

Worker3

CPU：

40%
```

Scheduler：

负责：

决定：

```
放：

Worker2
```

为什么？

因为：

资源最空闲。

Scheduler 并不会启动 Pod。

它只是：

做决定。

可以理解成：

> **派单系统。**

------

# Controller（控制器）

这是 Kubernetes 最聪明的地方。

假设：

用户说：

```
我要：

3个Pod
```

现在：

实际：

```
只有：

2个
```

Controller：

发现：

```
期望：

3

实际：

2
```

怎么办？

自动：

创建：

1个。

如果：

突然：

Pod：

挂了。

Controller：

再次检查：

```
期望：

3

实际：

2
```

继续：

创建。

所以：

Controller 一天到晚就在做一件事：

> **不断比较期望状态和实际状态。**

如果不一致。

就想办法：

让它一致。

------

# 为什么 Kubernetes 能自动恢复？

答案就在这里。

例如：

Deployment：

```
Replica：

3
```

突然：

Pod 删除了。

Controller：

看到：

```
Desired：

3

Actual：

2
```

于是：

马上：

创建：

新的。

所以：

不是：

Pod：

自己恢复。

而是：

Controller：

一直在：

巡逻。

像保安一样。

------

# 第四节：Worker Node 内部有哪些组件？

Worker 也不是一台普通服务器。

里面同样有几个重要组件。

```
Worker Node

│

├── kubelet

├── Container Runtime

└── kube-proxy
```

------

# kubelet

这是 Worker 最重要的软件。

可以理解成：

> **Control Plane 派驻到每台 Worker 的管理员。**

它不断问：

API Server：

```
今天：

我要干什么？
```

API Server：

回答：

```
启动：

API Pod
```

kubelet：

收到以后。

开始：

启动容器。

所以：

真正启动容器的人：

不是：

API Server。

不是：

Scheduler。

而是：

kubelet。

------

# Container Runtime（容器运行时）

kubelet 自己不会运行容器。

它会调用：

容器运行时。

以前：

大家都是：

Docker。

后来：

Kubernetes 去掉了直接依赖 Docker。

现在最常见的是：

- containerd（目前最主流）
- CRI-O
- 其他兼容 CRI（Container Runtime Interface）的运行时

所以：

```
kubelet

↓

containerd

↓

真正启动容器
```

可以理解成：

kubelet 是：

经理。

containerd：

是真正干活的人。

------

# kube-proxy

这是：

网络管理员。

它负责：

Pod 之间：

通信。

例如：

你的：

API：

访问：

Redis。

API 根本不知道：

Redis：

在哪台机器。

kube-proxy：

负责：

转发。

所以：

以后学习：

Service。

就会知道：

为什么：

Pod IP 经常变。

但是：

Service：

一直不变。

背后就是：

kube-proxy。

------

# 第五节：一次部署请求是如何完成的？

现在，把所有组件串起来。

假设：

你执行：

```
kubectl apply -f deployment.yaml
```

整个过程如下：

```
① kubectl
        │
        ▼
② API Server
        │
        ▼
③ 写入 etcd（保存期望状态）
        │
        ▼
④ Controller 发现需要新的 Pod
        │
        ▼
⑤ Scheduler 选择最合适的 Worker
        │
        ▼
⑥ kubelet 接收到任务
        │
        ▼
⑦ containerd 拉取镜像并启动容器
        │
        ▼
⑧ Pod 成功运行
```

这里有一个很重要的观察：

> **API Server 并不会直接创建容器，它只是记录用户的意图，并协调其他组件完成这件事。**

这种"多个组件各司其职、通过 API Server 协同"的设计，使 Kubernetes 具有很好的可扩展性和稳定性。

------

# 本章总结

你可以把 Kubernetes 想象成一家大型物流公司：

| Kubernetes 组件 | 现实角色      | 职责                         |
| --------------- | ------------- | ---------------------------- |
| kubectl         | 客户          | 提交需求                     |
| API Server      | 前台/总服务台 | 接收所有请求                 |
| etcd            | 档案室        | 保存集群所有期望状态         |
| Scheduler       | 调度员        | 决定任务交给哪台机器         |
| Controller      | 巡检主管      | 持续让实际状态与期望状态一致 |
| Worker Node     | 仓库          | 真正运行应用                 |
| kubelet         | 仓库管理员    | 接收任务并管理 Pod           |
| containerd      | 工人          | 创建、运行容器               |
| kube-proxy      | 配送网络      | 提供网络转发和服务访问       |

------

## 下一章预告：Pod——为什么 Kubernetes 不直接管理容器？

这是 Kubernetes 初学者最容易困惑的问题。

很多人都会问：

> **既然 containerd 能直接运行容器，为什么 Kubernetes 不直接管理容器，而是设计了一个新的概念——Pod？**

下一章我们将重点回答：

- Pod 到底是什么？
- 为什么说 **Pod 才是 Kubernetes 最小调度单位**？
- 一个 Pod 为什么可以有多个容器？
- "一个 Pod 一个 IP" 到底意味着什么？
- Pod 生命周期是怎样的？
- 为什么 Pod 会"消失"，而 Deployment 不会？

理解 Pod 后，你会真正进入 Kubernetes 的核心使用阶段。

# 第三章：Pod——Kubernetes 最小调度单位

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们正式进入 **Kubernetes 最重要的资源对象——Pod**。

很多人学习 Kubernetes 的时候，最大的困惑就是：

> **Docker 管理的是 Container（容器），为什么 Kubernetes 不直接管理 Container，而是创造了一个新的概念：Pod？**

实际上，**Pod 可以说是整个 Kubernetes 的灵魂**。

后面的 Deployment、StatefulSet、DaemonSet、Job……全部都是围绕 Pod 工作。

所以，这一章我们一定要彻底理解 Pod，而不是死记定义。

## 本章学习目标

学完这一章，你应该能回答：

- Pod 到底是什么？
- 为什么 Kubernetes 不直接管理容器？
- Pod 和 Docker Container 有什么区别？
- 一个 Pod 为什么可以运行多个容器？
- Pod 为什么只有一个 IP？
- Pod 为什么会消失？
- Pod 生命周期是什么？
- 什么情况下一个 Pod 会重建？

------

# 第一节：先忘掉 Pod，从 Docker 开始

假设我们现在只有 Docker。

你的 ASP.NET Core API：

```
MyApi.dll
```

打包镜像：

```
FROM mcr.microsoft.com/dotnet/aspnet:9.0

COPY .

ENTRYPOINT ["dotnet","MyApi.dll"]
```

然后：

```
docker build -t myapi:v1 .
```

运行：

```
docker run myapi:v1
```

现在：

Docker 创建了：

```
Container
```

整个世界只有：

```
镜像(Image)

↓

容器(Container)
```

非常简单。

------

# Kubernetes 为什么觉得这样不够？

假设：

你的 API 需要：

- 主程序
- 日志收集程序
- 监控程序

Docker：

一般就是：

```
Container1

ASP.NET Core
```

日志：

另一个容器：

```
Container2

Fluent Bit
```

监控：

又一个：

```
Container3

Exporter
```

问题来了。

这三个：

必须：

一起启动。

一起停止。

一起删除。

如果：

API 删除。

日志程序也应该删除。

Docker：

没有这种概念。

所以：

Kubernetes 发明了：

> **Pod。**

------

# Pod 到底是什么？

官方定义：

> Pod 是 Kubernetes 最小部署单位。

但是。

这句话几乎没有帮助。

真正应该理解的是：

> **Pod 就是一个"容器组（Container Group）"。**

例如：

```
Pod

│

├── ASP.NET Core

├── Fluent Bit

└── Metrics Exporter
```

这三个容器：

永远一起。

不能拆。

一起：

启动。

一起：

停止。

一起：

删除。

所以：

Pod 不是容器。

Pod：

里面可以有：

一个。

或者：

多个容器。

------

# 为什么不直接叫 Container？

因为：

Kubernetes 调度的是：

整个 Pod。

例如：

现在：

Worker1：

```
Pod A
```

里面：

```
API

+

Log
```

Scheduler：

不会：

只移动：

API。

日志：

留在原地。

它必须：

整个：

Pod：

一起移动。

所以：

最小单位：

不是：

Container。

而是：

Pod。

------

# 第二节：Pod 就像什么？

这是理解 Pod 最容易的方法。

把：

Pod：

想成：

> **一个房子。**

Container：

就是：

房间里面的人。

例如：

```
        Pod（房子）

    ┌─────────────────┐

    │                 │

    │   API           │

    │                 │

    │  Fluent Bit     │

    │                 │

    └─────────────────┘
```

房子：

有：

- 一个门牌号（IP）
- 一个网络
- 一个共享存储

里面的人：

共享：

所有资源。

所以：

Pod：

就是：

共享资源的容器集合。

------

# 第三节：为什么一个 Pod 只有一个 IP？

这是很多人第一次学 Kubernetes 时最疑惑的问题。

假设：

Pod：

里面：

```
API

Log

Monitor
```

三个容器。

它们：

IP：

不是：

三个。

而是：

一个。

例如：

```
10.0.0.15
```

为什么？

因为：

它们：

共享：

Network Namespace。

什么意思？

先看 Linux。

Linux 有一个东西：

叫：

Namespace。

它可以把：

网络：

隔离。

Docker：

就是利用：

Namespace。

而：

Pod：

里面所有容器：

共享：

同一个：

Network Namespace。

所以：

```
Pod

IP：

10.0.0.15

==================

API

127.0.0.1

==================

Log

127.0.0.1

==================

Monitor

127.0.0.1
```

注意：

三个容器：

看到的：

localhost：

都是：

同一个。

因此：

API：

可以：

直接访问：

```
localhost
```

就能找到：

Fluent Bit。

不用：

网络。

不用：

TCP。

速度非常快。

------

# 一个真实案例

例如：

ASP.NET Core：

监听：

```
5000
```

日志程序：

监听：

```
2020
```

那么：

API：

可以：

直接：

```
localhost:2020
```

访问：

日志程序。

因为：

它们：

就在：

一个：

Pod。

------

# 第四节：Pod 为什么还能共享存储？

除了：

共享网络。

Pod：

还能：

共享：

Volume。

例如：

```
Pod

│

├── API

├── Log

└── Volume
```

API：

生成：

```
/app/logs
```

日志。

Fluent Bit：

读取：

```
/app/logs
```

发送：

ElasticSearch。

它们：

共享：

同一块磁盘。

所以：

Pod：

除了：

共享网络。

还：

共享：

存储。

------

# 第五节：为什么 90% 的 Pod 都只有一个容器？

很多人看到：

Pod 可以多个容器。

于是：

什么都往里面放。

这是错误的。

事实上：

生产环境：

绝大部分：

都是：

```
Pod

↓

ASP.NET Core
```

只有：

一个容器。

为什么？

因为：

一个服务。

一个 Pod。

这是最简单。

也是：

最容易维护。

------

# 那什么时候才多个容器？

只有：

它们：

必须：

紧密合作。

例如：

日志：

Sidecar。

```
Pod

│

├── API

└── Fluent Bit
```

例如：

代理：

```
Pod

│

├── API

└── Envoy
```

例如：

Service Mesh。

Istio：

就是：

大量：

Sidecar。

所以：

多个容器：

一般都是：

Sidecar 模式。

------

# Sidecar（边车模式）

这是 Kubernetes 非常经典的设计。

为什么叫：

Sidecar？

因为：

像：

摩托车：

旁边：

挂了一个边斗。

```
      ______

     | API |

     |______|

        ||

========||======

      Sidecar
```

API：

负责：

业务。

Sidecar：

负责：

日志。

监控。

代理。

证书。

等等。

它们：

一起：

出生。

一起：

死亡。

------

# 第六节：Pod 为什么会消失？

这是 Kubernetes 和 Docker 最大区别。

Docker：

Container：

就是：

Container。

除非：

删掉。

否则：

一直存在。

但是：

Pod：

不是。

例如：

Node：

坏了。

```
Worker1

↓

断电
```

Pod：

没了。

但是：

Deployment：

发现：

```
Replica：

3

实际：

2
```

于是：

创建：

新的：

Pod。

注意：

不是：

恢复：

原来的。

而是：

创建：

一个：

新的。

所以：

Pod：

天生就是：

> **临时的（Ephemeral）。**

不要认为：

Pod：

会一直存在。

------

# 为什么说 Pod 是"一次性用品"？

这是 Kubernetes 非常重要的思想。

假设：

Pod：

名字：

```
api-7f84bc6d8d-abc12
```

升级：

以后。

新的：

```
api-8f56ffdc9-xyz99
```

旧的：

删除。

所以：

Pod：

不断：

出生。

不断：

死亡。

这是：

正常。

不是：

异常。

因此：

**千万不要把重要数据放在 Pod 本地磁盘。**

因为：

Pod 删除。

数据：

一起没了。

数据库、上传文件、用户数据等都必须放到持久化存储（Persistent Volume）或外部存储中，我们会在后续存储章节详细介绍。

------

# 第七节：Pod 生命周期（Lifecycle）

一个 Pod 从创建到结束，大致会经历下面几个阶段：

```
Pending
    │
    ▼
ContainerCreating
    │
    ▼
Running
    │
    ▼
Succeeded / Failed
    │
    ▼
Terminating
```

下面分别解释。

### 1. Pending（等待中）

Pod 已经创建了，但还没有真正运行。

常见原因：

- Scheduler 还没有完成调度
- 镜像还没开始下载
- 节点资源不足
- 等待 Volume 挂载

这时执行：

```
kubectl get pods
```

可能看到：

```
NAME      STATUS
api-xxx   Pending
```

------

### 2. ContainerCreating（创建容器）

Scheduler 已经选好了 Worker。

kubelet 开始：

- 拉取镜像
- 创建网络
- 挂载 Volume
- 创建容器

镜像越大，这个阶段可能越长。

------

### 3. Running（运行中）

所有容器启动成功。

业务开始对外提供服务。

这是我们最希望看到的状态。

------

### 4. Succeeded / Failed

通常出现在一次性任务（Job）中。

例如：

```
导入数据

执行脚本

数据库初始化
```

执行完成：

```
Succeeded
```

执行失败：

```
Failed
```

对于 Web API 来说，一般不会长期停留在这两个状态，因为 Deployment 会尝试维持它持续运行。

------

### 5. Terminating（终止中）

例如：

```
kubectl delete pod api-xxx
```

Pod 并不会立刻消失。

Kubernetes 会：

1. 通知容器准备退出（发送终止信号）
2. 等待应用完成收尾工作（例如 ASP.NET Core 完成正在处理的请求）
3. 到达宽限时间后，如果还没有退出，再强制结束

因此，一个设计良好的应用需要支持**优雅停机（Graceful Shutdown）**。

ASP.NET Core 默认已经支持这一机制，只要正确处理取消令牌（CancellationToken）即可。

------

# 本章总结

理解 Pod，只需要记住下面几句话：

1. **Pod 是 Kubernetes 的最小调度单位，而不是 Container。**
2. **Pod 可以包含一个或多个容器。**
3. **Pod 内的容器共享网络（同一个 IP）和共享存储（Volume）。**
4. **绝大多数业务 Pod 只有一个容器，多容器通常用于 Sidecar 模式。**
5. **Pod 是临时资源，不要把重要数据保存在 Pod 本地。**
6. **真正长期存在的是 Deployment、StatefulSet 等控制器，它们负责不断创建和替换 Pod。**

------

# 下一章预告：Deployment——真正管理 Pod 的资源

到目前为止，我们已经知道：

- Pod 是最小运行单位。
- Pod 会消失、会重建、名字也会变化。

那么问题来了：

> **如果 Pod 会消失，我们为什么还要直接创建 Pod？**

答案是：**生产环境几乎不会直接创建 Pod。**

下一章我们将学习 Kubernetes 中最常用、也是你部署 ASP.NET Core 应用时几乎一定会使用的资源——**Deployment**，包括：

- 为什么 Deployment 才是日常部署应用的入口？
- ReplicaSet 是什么？为什么它夹在 Deployment 和 Pod 之间？
- 如何实现自动恢复、滚动升级和版本回滚？
- 为什么执行 `kubectl apply` 后，真正创建 Pod 的其实不是 Deployment 本身？

这一章结束后，你就能真正理解 Kubernetes 是如何管理应用生命周期的。

# 第四章：Deployment —— Kubernetes 应用部署的核心

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入 **Kubernetes 真正开始"干活"的资源**。

实际上，**生产环境几乎没有人直接创建 Pod。**

原因我们上一章已经知道了：

> **Pod 是易失（Ephemeral）的，它会不断消失、不断被重新创建。**

所以 Kubernetes 又设计了一个新的资源：

> **Deployment（部署控制器）**

如果说：

- **Pod 是士兵**
- **Deployment 就是营长**

营长不会亲自打仗。

但是：

所有士兵都是它管理的。

## 本章学习目标

学完本章，你应该能够回答：

- 为什么不直接创建 Pod？
- Deployment 是什么？
- Deployment 和 Pod 的关系？
- ReplicaSet 是什么？
- 为什么 Deployment 和 Pod 中间还要多一个 ReplicaSet？
- Deployment 如何自动恢复？
- Deployment 如何滚动升级？
- Deployment 如何回滚？

------

# 第一节：为什么不要直接创建 Pod？

我们先来看一个最简单的例子。

假设：

```
apiVersion: v1
kind: Pod

metadata:
  name: my-api

spec:
  containers:
    - name: api
      image: myapi:v1
```

执行：

```
kubectl apply -f pod.yaml
```

Pod 创建成功。

看起来没有问题。

但是。

突然：

```
Pod 崩了
```

或者：

```
Pod 被删除
```

例如：

```
kubectl delete pod my-api
```

结果：

```
Pod

没了。
```

结束。

没有任何东西会重新创建它。

因为：

没有人管理它。

这就像：

公司里：

一个员工。

突然辞职。

老板：

根本不知道。

------

## Deployment 就是 Pod 的老板

如果：

不是：

直接创建 Pod。

而是：

```
kind: Deployment
```

情况完全不同。

例如：

Deployment：

要求：

```
我要：

3 个 Pod
```

实际：

```
现在：

只有2个
```

Deployment：

立即发现：

```
少了一个。
```

马上：

创建：

新的。

所以：

Deployment：

真正管理的是：

> **Pod 的数量和生命周期。**

------

# 第二节：Deployment、ReplicaSet、Pod 的关系

这是 Kubernetes 最经典的一张图。

```
Deployment

      │

      ▼

ReplicaSet

      │

      ▼

 Pod1

 Pod2

 Pod3
```

很多新人看到这里都会问：

> **为什么不直接：Deployment → Pod？**

为什么：

还要：

ReplicaSet？

答案：

因为：

ReplicaSet：

专门负责：

> **副本（Replica）管理。**

Deployment：

只负责：

升级。

回滚。

版本。

ReplicaSet：

负责：

创建 Pod。

删除 Pod。

保持数量。

这是：

职责分离。

------

# 第三节：ReplicaSet 到底是什么？

Replica：

就是：

副本。

例如：

Deployment：

写：

```
replicas: 3
```

意思：

```
我要：

3 个一样的 Pod。
```

谁负责？

ReplicaSet。

例如：

当前：

```
Pod1

Pod2
```

只有：

2个。

ReplicaSet：

发现：

```
Desired：

3

Actual：

2
```

于是：

创建：

```
Pod3
```

如果：

突然：

```
Pod2

删除
```

ReplicaSet：

继续：

创建：

新的。

所以：

ReplicaSet：

一天到晚：

只做一件事：

> **保持 Pod 数量。**

------

# 第四节：为什么 Deployment 和 ReplicaSet 要分开？

假设：

今天：

版本：

```
myapi:v1
```

ReplicaSet：

管理：

```
ReplicaSet A

↓

Pod(v1)

Pod(v1)

Pod(v1)
```

现在。

升级：

```
v2
```

Deployment：

不会：

直接修改：

Pod。

它会：

创建：

新的：

ReplicaSet。

例如：

```
Deployment

│

├── ReplicaSet A

│      │

│      ├── v1

│      ├── v1

│      └── v1

│

└── ReplicaSet B

       │

       ├── v2

       ├── v2

       └── v2
```

注意。

现在：

有：

两个：

ReplicaSet。

------

# 为什么这样设计？

因为：

以后：

可以：

回滚。

例如：

升级：

以后：

发现：

BUG。

Deployment：

直接：

把：

ReplicaSet A：

重新：

放大。

ReplicaSet B：

缩小。

整个过程：

不用：

重新部署。

所以：

ReplicaSet：

其实：

保存了：

历史版本。

------

# 第五节：Deployment 是如何滚动升级的？

假设：

当前：

```
3 个 Pod

全部：

v1
Pod1

Pod2

Pod3
```

升级：

```
v2
```

很多新人认为：

是不是：

全部：

删掉。

再：

重新创建？

不是。

那样：

网站：

直接：

挂掉。

Deployment：

真正做的是：

> **滚动升级（Rolling Update）**

例如：

第一步：

```
v1

v1

v1
```

↓

创建：

一个：

v2

```
v1

v1

v1

v2
```

确认：

健康。

然后：

删除：

一个：

v1

```
v1

v1

v2
```

继续：

创建：

一个：

v2

```
v1

v1

v2

v2
```

继续：

删除：

v1

......

最后：

全部：

变成：

```
v2

v2

v2
```

整个过程。

网站：

一直：

可以访问。

------

# 第六节：滚动升级为什么不会停机？

因为：

Deployment：

默认：

遵循：

两个原则：

## ① 不能一下子删光

例如：

副本：

```
3
```

不会：

直接：

删：

3个。

否则：

网站：

没了。

------

## ② 新 Pod 必须 Ready

这里：

有一个：

非常重要的概念。

不是：

Running。

而是：

Ready。

什么意思？

例如：

ASP.NET Core：

启动：

需要：

30秒。

虽然：

Container：

已经：

Running。

但是：

数据库：

还没连接。

Redis：

还没连接。

API：

还不能：

提供服务。

如果：

马上：

删除：

旧版本。

用户：

就会：

500。

所以：

Deployment：

等待：

```
Ready=True
```

以后。

才：

继续：

升级。

> **这也是为什么后面我们一定要学习 Readiness Probe（就绪探针）。**

没有 Ready。

滚动升级：

可能：

出事故。

------

# 第七节：Deployment 如何自动恢复？

假设：

当前：

```
Replica：

3
```

现在：

Node：

坏了。

```
Worker1

断电
```

Pod：

没了。

ReplicaSet：

发现：

```
Desired：

3

Actual：

2
```

于是：

另一台：

Worker：

启动：

新的：

Pod。

所以：

Deployment：

真正做到：

> **自愈（Self Healing）**

------

# 第八节：Deployment YAML（第一次阅读）

先不要急着背。

我们先认识每一部分。

```
apiVersion: apps/v1

kind: Deployment

metadata:
  name: my-api

spec:

  replicas: 3

  selector:
    matchLabels:
      app: my-api

  template:

    metadata:

      labels:
        app: my-api

    spec:

      containers:

      - name: api

        image: myapi:v1

        ports:

        - containerPort: 8080
```

第一次看：

是不是：

很多字段？

不要怕。

我们一点一点解释。

------

## apiVersion

表示：

使用：

哪个：

Kubernetes API。

例如：

Deployment：

现在：

就是：

```
apps/v1
```

------

## kind

资源类型。

例如：

```
kind: Deployment
```

告诉：

API Server：

我要：

创建：

Deployment。

------

## metadata

资源信息。

例如：

```
metadata:

  name: my-api
```

以后：

执行：

```
kubectl get deployment
```

看到：

就是：

```
my-api
```

------

## replicas

这个：

最重要。

```
replicas: 3
```

意思：

不是：

创建：

三个。

而是：

> **一直保持三个。**

这是：

声明式。

不是：

命令式。

例如：

删除：

一个：

ReplicaSet：

继续：

补。

------

## selector

这是：

初学者：

最容易：

困惑。

```
selector:

  matchLabels:

    app: my-api
```

什么意思？

Deployment：

必须知道：

哪些：

Pod：

归我管理。

否则。

Pod：

那么多。

怎么知道：

哪个：

属于：

哪个 Deployment？

答案：

看：

Label。

------

# Label（标签）

Label：

就是：

给 Pod：

贴标签。

例如：

```
Pod1

↓

app=my-api
```

Pod2：

```
app=my-api
```

Pod3：

```
app=my-api
```

Deployment：

说：

```
selector:

 app=my-api
```

于是：

知道：

这些：

都是：

我的。

------

# template

这里：

其实：

就是：

Pod。

也就是说：

Deployment：

里面：

嵌套了：

一个：

Pod 模板。

以后：

ReplicaSet：

就是：

根据：

这个模板：

创建：

Pod。

可以理解成：

```
Deployment

↓

复制：

这个模板

↓

生成：

Pod
```

------

# 第九节：Deployment、ReplicaSet、Pod 的完整关系

现在。

把：

整个流程。

连起来。

```
kubectl apply deployment.yaml

        │

        ▼

API Server

        │

        ▼

Deployment

        │

        ▼

ReplicaSet

        │

        ▼

Pod1

Pod2

Pod3

        │

        ▼

containerd

        │

        ▼

真正运行 ASP.NET Core
```

这里有一个非常重要的事实：

> **Deployment 从来不会直接创建 Pod。**

它只是声明："我要这样的应用、这样的版本、这样的副本数。"

真正负责创建 Pod 的是 ReplicaSet，而真正启动容器的是每个节点上的 kubelet 调用 containerd。

这就是 Kubernetes **控制器模式（Controller Pattern）** 的体现：每一层只负责自己的职责，而不是包办所有事情。

------

# 本章总结（建议记住）

整个 Kubernetes 应用部署链路可以浓缩成一句话：

```
Deployment
      │
负责应用生命周期（升级、回滚）
      │
      ▼
ReplicaSet
      │
负责维持 Pod 数量
      │
      ▼
Pod
      │
运行你的应用容器
      │
      ▼
Container（containerd）
```

请牢记下面几点：

1. **生产环境不要直接创建 Pod，而是创建 Deployment。**
2. **Deployment 管理的是应用，Pod 管理的是容器。**
3. **ReplicaSet 是 Deployment 和 Pod 之间的桥梁，负责维持副本数量。**
4. **滚动升级实际上是"新 ReplicaSet 逐渐接管旧 ReplicaSet"的过程。**
5. **Pod 被删除并不可怕，只要 Deployment 还在，它就会自动重新创建。**

------

# 下一章预告：Service——为什么 Pod 可以随时重建，但应用还能稳定访问？

现在我们已经知道：

- Pod 会不断创建、删除。
- Pod 的名字会变。
- Pod 的 IP 也会变。

那么新的问题来了：

> **如果 Pod 的 IP 一直变化，前端、其他服务、数据库连接应该访问谁？**

这就是 **Service** 要解决的问题。

下一章我们将重点讲解：

- 为什么不能直接访问 Pod IP？
- Service 到底是什么？
- ClusterIP、NodePort、LoadBalancer 有什么区别？
- kube-proxy 在其中扮演什么角色？
- Service 如何实现负载均衡？
- 为什么 Service 能做到"Pod 可以换，但访问地址永远不变"？

这一章结束后，你就会真正理解 Kubernetes 网络模型的第一层，也是部署 Web 应用时最常接触的资源之一。

# 第五章：Service —— Kubernetes 网络通信的核心

提示词：

```
开始下一章
```

正文：

非常好。

到目前为止，我们已经知道：

> Deployment 管理 Pod。

但是新的问题来了。

假设：

Deployment：

有三个 Pod：

```
Pod A

IP：10.244.1.3

----------------

Pod B

IP：10.244.2.5

----------------

Pod C

IP：10.244.3.9
```

你的前端（Vue）、或者另一个微服务，要访问 API。

它应该访问：

```
10.244.1.3 ?
```

还是：

```
10.244.2.5 ?
```

还是：

```
10.244.3.9 ?
```

更麻烦的是：

**明天这些 IP 全都会变。**

为什么？

因为上一章我们说过：

> **Pod 是一次性的。**

今天：

```
Pod A

10.244.1.3
```

明天：

```
Pod D

10.244.5.11
```

原来的 Pod 已经不存在了。

那么：

> **应用如何找到正确的 Pod？**

答案就是：

> **Service。**

## 本章学习目标

学完这一章，你应该能够回答：

- 为什么不能直接访问 Pod？
- Service 到底是什么？
- Service 为什么有一个永远不会变的 IP？
- Service 是如何找到 Pod 的？
- kube-proxy 在其中做了什么？
- ClusterIP、NodePort、LoadBalancer 有什么区别？
- Kubernetes 为什么自带负载均衡？

------

# 第一节：为什么不能直接访问 Pod？

假设：

我们有一个 Deployment。

副本：

```
3
```

于是：

```
Deployment

↓

Pod1

10.244.1.3

↓

Pod2

10.244.2.5

↓

Pod3

10.244.3.9
```

现在：

Vue：

调用：

```
10.244.1.3
```

结果：

Pod1：

挂了。

Deployment：

自动创建：

```
Pod4

10.244.8.16
```

原来的：

```
10.244.1.3
```

不存在了。

于是：

Vue：

全部：

报错。

所以：

> **Pod IP 天生就是不可靠的。**

Kubernetes 官方明确建议：

**永远不要把 Pod IP 写死到代码里。**

------

# 第二节：Service 到底是什么？

很多教程说：

> Service 是服务。

这句话容易让初学者误解。

实际上。

Service：

不是：

程序。

不是：

容器。

不是：

Pod。

它更像：

> **一个固定的入口（Stable Endpoint）。**

例如：

```
                 Service

              10.96.0.20

                   │

       ┌───────────┼───────────┐

       │           │           │

     Pod1        Pod2       Pod3
```

用户：

永远访问：

```
10.96.0.20
```

至于：

请求：

最终：

到了：

哪个：

Pod。

Service：

负责。

------

# 一个现实生活例子

假设：

一家银行。

里面：

有：

20 个柜员。

客户：

不会：

直接：

找：

柜员。

而是：

先到：

```
取号机
```

然后：

系统：

安排：

柜员。

这里：

```
取号机

↓

Service
柜员

↓

Pod
```

柜员：

今天：

请假。

明天：

来了新人。

客户：

完全：

不知道。

因为：

他：

永远：

面对：

取号机。

------

# 第三节：Service 为什么不会变？

Deployment：

创建：

Pod。

Pod：

会：

消失。

但是：

Service：

不会。

例如：

创建：

```
Service

名字：

api-service
```

它：

获得：

IP：

```
10.96.0.20
```

以后：

Pod：

全部：

删掉。

重新：

创建。

Service：

还是：

```
10.96.0.20
```

所以：

Service：

真正提供的是：

> **稳定地址。**

------

# 第四节：Service 怎么知道有哪些 Pod？

这是 Kubernetes 最巧妙的设计之一。

还记得上一章：

Deployment：

使用：

Label：

```
app=my-api
```

例如：

Pod：

```
Pod1

Label

app=my-api
```

Pod2：

```
app=my-api
```

Pod3：

```
app=my-api
```

Service：

也有：

Selector。

例如：

```
selector:

  app: my-api
```

意思：

> **所有 Label = my-api 的 Pod，都属于我。**

于是：

```
Service

↓

Selector

↓

app=my-api

↓

Pod1

Pod2

Pod3
```

是不是：

和：

Deployment：

非常像？

没错。

Deployment：

也是：

通过：

Label：

找到：

Pod。

所以：

**Label 是 Kubernetes 最重要的设计之一。**

以后：

Deployment。

Service。

NetworkPolicy。

Ingress。

全部：

依赖：

Label。

------

# 第五节：Service 如何负载均衡？

假设：

现在：

三个 Pod：

```
Pod1

Pod2

Pod3
```

用户：

不断：

访问：

```
Service
```

Service：

每收到：

一个：

请求。

都会：

选择：

一个：

Pod。

例如：

```
请求1

↓

Pod1

----------------

请求2

↓

Pod2

----------------

请求3

↓

Pod3

----------------

请求4

↓

Pod1
```

这就是：

> **负载均衡（Load Balancing）**

所以：

你：

不用：

安装：

Nginx。

不用：

写：

轮询。

Service：

已经：

做了。

> **需要注意：Service 本身并不是一个真正运行的"代理进程"，它是一组网络规则和抽象。真正负责把流量转发到后端 Pod 的，是每个节点上的 kube-proxy（或使用 eBPF 等实现的网络插件）。**

------

# 第六节：kube-proxy 到底做了什么？

上一章。

我们提到：

Worker：

里面：

有：

```
kube-proxy
```

现在：

终于：

派上：

用场。

例如：

请求：

```
10.96.0.20
```

实际上：

没有：

任何：

程序：

监听：

这个：

IP。

那么：

为什么：

还能：

访问？

答案：

因为：

kube-proxy：

提前：

写好了：

Linux：

网络规则。

例如：

```
访问：

10.96.0.20

↓

自动：

转发

↓

10.244.2.5
```

或者：

```
↓

10.244.3.9
```

整个：

过程：

应用：

根本：

不知道。

所以：

很多人：

误认为：

Service：

就是：

代理。

其实：

不是。

真正：

转发：

请求：

的是：

kube-proxy（或者某些 CNI 网络插件提供的等效实现）。

------

# 第七节：Service 类型

这里：

开始：

出现：

生产环境：

最重要：

几个：

概念。

Service：

有：

很多：

类型。

最常见：

四种：

```
ClusterIP

NodePort

LoadBalancer

ExternalName
```

其中：

真正：

每天：

使用：

主要：

前三个。

------

# ClusterIP（默认）

这是：

默认：

类型。

例如：

```
type: ClusterIP
```

特点：

> **只能集群内部访问。**

例如：

```
Vue

↓

API

↓

Redis
```

它们：

都：

在：

Kubernetes：

里面。

使用：

ClusterIP。

例如：

```
API

↓

redis-service
```

不需要：

公网。

------

# ClusterIP 网络结构

```
                Cluster

        +---------------------+

        |                     |

        |  Service            |

        | 10.96.0.20          |

        |                     |

        +----------+----------+

                   |

          ┌────────┼────────┐

          │        │        │

        Pod1     Pod2     Pod3
```

外网：

访问：

不了。

只有：

Cluster：

里面：

才能：

访问。

------

# NodePort

如果：

浏览器：

想：

访问：

怎么办？

于是：

出现：

NodePort。

例如：

```
type:

NodePort
```

Kubernetes：

会：

在：

每台：

Node：

打开：

一个：

端口。

例如：

```
30080
```

于是：

浏览器：

访问：

```
http://NodeIP:30080
```

例如：

```
http://192.168.1.20:30080
```

请求：

进入：

Node。

然后：

转发：

Service。

再：

转发：

Pod。

网络：

变成：

```
浏览器

↓

NodeIP:30080

↓

Service

↓

Pod
```

------

# NodePort 的缺点

虽然：

可以：

访问。

但是：

有：

很多：

问题。

例如：

端口：

```
30080

30081

30082
```

越来越多。

还有：

每个：

服务：

都要：

暴露：

一个：

高位端口。

对于用户来说：

```
www.xxx.com:30080
```

体验：

很差。

因此：

生产环境：

几乎不会直接让用户访问 NodePort。

NodePort 更多用于测试环境，或者作为 Ingress Controller 的入口。

------

# LoadBalancer

如果：

Kubernetes：

运行：

在：

云平台。

例如：

- AWS
- Azure
- GCP

当你创建：

```
type: LoadBalancer
```

云平台：

会：

自动：

创建：

一个：

真正：

云负载均衡。

例如：

```
Internet

↓

Cloud LoadBalancer

↓

Service

↓

Pod
```

你：

不用：

自己：

配置：

负载均衡。

这也是 Kubernetes 与云平台集成的重要能力。

------

# 那 Ingress 呢？

你可能发现：

如果：

三个：

Service：

```
API

Admin

Auth
```

全部：

NodePort。

需要：

```
30080

30081

30082
```

非常：

难看。

于是：

大家：

想到：

为什么：

不能：

只有：

一个：

80。

然后：

根据：

URL：

决定：

去哪？

例如：

```
/

↓

API

------------------

/admin

↓

Admin

------------------

/auth

↓

Auth
```

或者：

```
api.example.com

↓

API

----------------

admin.example.com

↓

Admin
```

答案：

有。

它：

就是：

> **Ingress。**

Ingress 可以理解为 Kubernetes 世界里的 **七层反向代理入口**，类似于 Nginx 的路由能力。由于它涉及 HTTP、HTTPS、域名、TLS 等内容，我们会单独用一章详细讲解。

------

# 第八节：Service YAML

先认识最简单的 Service。

```
apiVersion: v1
kind: Service

metadata:
  name: my-api-service

spec:
  selector:
    app: my-api

  ports:
    - port: 80
      targetPort: 8080

  type: ClusterIP
```

我们逐项解释。

------

## selector

告诉 Service：

我要：

找：

谁？

例如：

```
selector:

 app: my-api
```

所有：

Label：

```
app=my-api
```

全部：

加入。

------

## port

例如：

```
port: 80
```

表示：

Service：

监听：

80。

------

## targetPort

例如：

```
targetPort: 8080
```

表示：

Pod：

真正：

监听：

8080。

所以：

请求：

```
80

↓

8080
```

这也是为什么 Service 可以作为统一入口，而后端容器可以保持自己的监听端口。

------

# 第九节：一个完整请求是怎样流动的？

假设：

浏览器：

访问：

```
http://api.example.com
```

整个过程可以简化为：

```
浏览器
    │
    ▼
Ingress（或云负载均衡）
    │
    ▼
Service（稳定入口）
    │
    ▼
kube-proxy（网络转发）
    │
    ▼
Pod（ASP.NET Core）
```

如果：

Pod：

坏了。

Deployment：

重新：

创建：

新的：

Pod。

Service：

自动：

更新：

后端：

列表。

浏览器：

完全：

不知道。

这就是 Kubernetes 网络模型最核心的思想：

> **客户端永远连接 Service，而不是直接连接 Pod。**

------

# 本章总结（建议牢记）

| 对象       | 职责              | 是否稳定         |
| ---------- | ----------------- | ---------------- |
| Pod        | 运行应用          | ❌ 会变化         |
| Service    | 提供固定访问入口  | ✅ 稳定           |
| Label      | 给资源分类        | ✅ 稳定           |
| Selector   | 根据 Label 找资源 | ✅ 稳定           |
| kube-proxy | 实现网络转发      | ✅ 持续运行于节点 |
| Deployment | 管理 Pod 生命周期 | ✅ 稳定           |

请记住 Kubernetes 网络最重要的一句话：

> **Pod 可以变化，Service 永远稳定。**

------

# 下一章预告：Ingress——真正对外提供 HTTP/HTTPS 服务

到目前为止，我们已经知道：

- Deployment 管理应用。
- Service 提供稳定入口。
- NodePort 可以让外部访问。

但是新的问题来了：

假设你的项目包含：

- ASP.NET Core API
- Vben Admin 前端
- 身份认证中心
- 文件服务

如果全部使用 NodePort，用户就要访问：

```
http://example.com:30080
http://example.com:30081
http://example.com:30082
```

这显然不是现代 Web 应用的访问方式。

下一章，我们将学习 **Ingress**，包括：

- 为什么 Ingress 被称为 Kubernetes 的"网关"？
- Ingress 与 Service 的关系是什么？
- Ingress Controller 是什么？为什么 Ingress 本身不会转发流量？
- 如何配置域名、HTTPS、TLS 证书？
- 如何实现：
  - `api.example.com`
  - `admin.example.com`
  - `example.com/api`
  - `example.com/admin`
- 为什么几乎所有生产环境都会使用 Ingress？

这一章结束后，你就能理解 Kubernetes 中 Web 应用对外发布的完整链路，也离部署你的 **ASP.NET Core + Vben Admin** 项目更近一步。

# 第六章 Ingress —— Kubernetes 的 HTTP 网关

提示词：

```
开始下一章
```

正文：

很好。

到这里，我们已经建立了一条完整的链路：

```
Deployment
      │
创建
      ▼
Pod
      │
运行应用
      ▼
Service
      │
提供稳定地址
```

但是现在还有最后一个问题。

假设你有一个项目：

```
Vue 前端

ASP.NET Core API

IdentityServer

MinIO

Grafana
```

每一个都有一个 Service。

例如：

```
frontend-service

api-service

auth-service

minio-service

grafana-service
```

如果全部暴露：

```
NodePort

30080

30081

30082

30083

30084
```

访问就变成：

```
http://192.168.1.20:30080

http://192.168.1.20:30081

http://192.168.1.20:30082
```

现实世界的网站会这样吗？

不会。

真正的网站都是：

```
https://www.xxx.com

https://api.xxx.com

https://admin.xxx.com
```

或者：

```
https://www.xxx.com/api

https://www.xxx.com/admin
```

那么 Kubernetes 怎么做到这一点？

答案就是：

> **Ingress。**

## 本章学习目标

学完这一章，你应该能够回答：

- 为什么需要 Ingress？
- Ingress 和 Service 有什么区别？
- Ingress 为什么不能单独工作？
- 什么是 Ingress Controller？
- Ingress 如何根据域名转发？
- Ingress 如何根据路径转发？
- HTTPS 是怎么配置的？
- 为什么生产环境几乎都使用 Ingress？

------

# 第一节：先理解什么是"网关（Gateway）"

先不要讲 Kubernetes。

先看现实生活。

假设：

你住在一个大型小区。

里面：

```
A栋

B栋

C栋

D栋
```

快递员来了。

不会：

直接：

进入：

某一栋。

而是：

先来到：

```
小区大门
```

保安问：

```
送给谁？
```

快递员说：

```
A栋201
```

保安：

放行。

如果：

```
B栋305
```

去：

B栋。

这里：

```
小区大门

↓

Ingress
各栋楼

↓

Service
```

Ingress：

就是：

整个 Kubernetes：

唯一入口。

------

# 第二节：Ingress 和 Service 的区别

很多新人容易混。

下面一句话一定记住：

> **Service 负责找到 Pod，Ingress 负责找到 Service。**

画出来就是：

```
Internet

      │

      ▼

Ingress

      │

      ▼

Service

      │

      ▼

Pod
```

也就是说：

Ingress：

根本不知道：

Pod。

它：

只认识：

Service。

------

# 第三节：完整访问过程

例如：

浏览器：

访问：

```
https://api.example.com
```

整个过程：

```
浏览器

↓

DNS

↓

Ingress

↓

api-service

↓

Pod

↓

ASP.NET Core
```

注意。

Ingress：

不会：

直接：

访问：

Pod。

而是：

永远：

访问：

Service。

------

# 第四节：Ingress 为什么可以根据域名转发？

例如：

公司：

有：

三个系统。

```
api.example.com

admin.example.com

auth.example.com
```

Ingress：

收到：

请求。

首先：

看：

Host。

例如：

```
Host:

api.example.com
```

于是：

转发：

```
api-service
```

如果：

```
Host:

admin.example.com
```

转发：

```
admin-service
```

整个流程：

```
                 Ingress

          api.example.com

                  │

         api-service

────────────────────────

       admin.example.com

                  │

       admin-service

────────────────────────

        auth.example.com

                  │

       auth-service
```

是不是：

很像：

Nginx？

没错。

Ingress：

就是：

HTTP：

七层：

反向代理。

------

# 第五节：除了域名，还能根据路径转发

例如：

只有：

一个域名。

```
www.example.com
```

但是：

不同：

路径：

不同：

服务。

```
/api

↓

API

----------------

/admin

↓

后台

----------------

/auth

↓

认证
```

于是：

Ingress：

配置：

```
www.example.com/api

↓

api-service

────────────────

www.example.com/admin

↓

admin-service

────────────────

www.example.com/auth

↓

auth-service
```

所以：

Ingress：

支持：

- Host
- Path

两种：

路由。

------

# 第六节：Ingress 到底是什么？

这里：

很多新人：

第一次：

都会：

理解错。

他们认为：

Ingress：

就是：

一个：

程序。

其实：

不是。

Ingress：

只是：

一个：

规则。

例如：

```
Host:

api.example.com

↓

api-service
```

它：

只是：

告诉：

Kubernetes：

```
以后：

如果：

访问：

api.example.com

就：

去：

api-service
```

真正：

执行：

这些：

规则。

不是：

Ingress。

而是：

Ingress Controller。

------

# 第七节：什么是 Ingress Controller？

这是整个章节：

最重要：

一句话。

> **Ingress 只是配置，Ingress Controller 才是真正工作的程序。**

例如：

你写：

```
kind: Ingress
```

只是：

保存：

规则。

真正：

监听：

80。

监听：

443。

解析：

HTTPS。

处理：

TLS。

转发：

请求。

全部：

都是：

Ingress Controller。

可以理解成：

```
Ingress

↓

配置文件

====================

Ingress Controller

↓

真正工作的程序
```

是不是：

和：

Deployment：

很像？

Deployment：

也是：

配置。

真正：

创建：

Pod。

是：

Controller。

Kubernetes：

大量：

采用：

这种：

设计思想。

------

# 第八节：最常见的 Ingress Controller

生产环境：

目前：

最流行：

几个：

## ① NGINX Ingress Controller

最经典。

也是：

学习：

Kubernetes：

基本都会：

接触。

它：

本质：

就是：

Nginx。

只是：

自动：

读取：

Ingress：

配置。

然后：

生成：

Nginx：

配置文件。

因此：

很多：

Nginx：

经验。

可以：

直接：

使用。

------

## ② Traefik

特点：

配置：

简单。

自动：

发现：

Service。

适合：

中小项目。

------

## ③ HAProxy Ingress

性能：

非常高。

金融：

行业：

很多。

------

## ④ 云厂商 Ingress

例如：

AKS

EKS

GKE

很多：

直接：

提供：

官方：

Ingress。

------

# 第九节：HTTPS 怎么来的？

现实：

网站：

都是：

```
https://
```

不是：

```
http://
```

HTTPS：

需要：

证书。

例如：

```
example.com

证书
```

以前：

Nginx：

需要：

写：

```
ssl_certificate

ssl_certificate_key
```

Ingress：

也是：

一样。

只不过：

证书：

保存：

到：

Secret。

例如：

```
TLS Secret

↓

Ingress

↓

HTTPS
```

以后：

浏览器：

访问：

```
https://api.example.com
```

Ingress Controller：

读取：

Secret。

完成：

TLS：

握手。

所以：

你的：

ASP.NET Core：

其实：

完全：

不用：

配置：

HTTPS。

很多生产环境中，HTTPS 会在 Ingress 终止（TLS Termination），Ingress 与后端 Service 通常使用 HTTP 通信。当然，也可以配置端到端 HTTPS，但维护成本更高。

------

# 第十节：一个完整 YAML

下面：

第一次：

认识：

Ingress。

不用：

全部：

记住。

```
apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:
  name: my-ingress

spec:

  rules:

  - host: api.example.com

    http:

      paths:

      - path: /

        pathType: Prefix

        backend:

          service:

            name: api-service

            port:

              number: 80
```

读：

中文：

就是：

```
如果：

Host：

api.example.com

↓

访问：

api-service

↓

80端口
```

是不是：

非常：

容易。

------

# 第十一节：Ingress 的完整网络结构

终于。

我们：

把：

整个：

Kubernetes：

部署：

流程。

全部：

串起来。

```
Internet

        │

        ▼

DNS

        │

        ▼

Ingress Controller

        │

        ▼

Ingress Rule

        │

        ▼

Service

        │

        ▼

Pod

        │

        ▼

ASP.NET Core
```

注意：

真正：

监听：

80。

443。

是：

Ingress Controller。

Ingress：

只是：

规则。

------

# 第十二节：为什么生产环境几乎都使用 Ingress？

原因：

非常：

简单。

如果：

没有：

Ingress。

假设：

二十个：

微服务。

需要：

二十个：

NodePort。

例如：

```
30080

30081

30082

...

30100
```

维护：

困难。

用户：

体验：

极差。

有了：

Ingress。

永远：

只有：

```
80

443
```

所有：

请求：

根据：

域名。

路径。

自动：

分发。

因此：

几乎所有：

生产环境：

都是：

```
Internet

↓

LoadBalancer

↓

Ingress Controller

↓

Ingress

↓

Service

↓

Pod
```

这里补充一个生产环境常见架构：

- **云平台**：通常是 `Internet → 云负载均衡（LoadBalancer）→ Ingress Controller`
- **自建机房**：通常是 `Internet → 硬件负载均衡/F5 或 Nginx → Ingress Controller`

Ingress Controller 很少直接裸露在公网，而是由上层负载均衡保护和分流。

------

# 本章总结（建议牢记）

整个 Kubernetes 对外访问链路可以概括为：

| 组件               | 职责                                |
| ------------------ | ----------------------------------- |
| DNS                | 域名解析到入口 IP                   |
| Ingress Controller | 真正接收 HTTP/HTTPS 请求并执行转发  |
| Ingress            | 定义域名、路径到 Service 的路由规则 |
| Service            | 提供稳定的服务入口并选择后端 Pod    |
| Pod                | 运行你的应用                        |

请记住两句话：

> **Service 面向集群内部，Ingress 面向 HTTP/HTTPS 外部流量。**

以及：

> **Ingress 是规则，Ingress Controller 才是真正工作的程序。**

------

# 到这里，我们已经完成了 Kubernetes 最核心的四大对象

```
Deployment
      │
管理
      ▼
Pod
      │
运行
      ▼
Service
      │
提供稳定入口
      ▼
Ingress
      │
对外发布网站
```

这四者构成了绝大多数 Web 应用部署的核心骨架。

------

# 下一章预告：ConfigMap 与 Secret——配置为什么不能写死在镜像里？

现在假设你的 ASP.NET Core API 有如下配置：

```
{
  "ConnectionStrings": {
    "Default": "Server=db;Database=App;User Id=sa;Password=123456;"
  },
  "Redis": {
    "Host": "redis:6379"
  }
}
```

如果你把这些直接打进 Docker 镜像，会遇到很多问题：

- 测试环境和生产环境配置不同怎么办？
- 数据库密码变了，要重新构建镜像吗？
- API Key、JWT 密钥应该放在哪里？
- 为什么 Kubernetes 要把普通配置和敏感配置分成 **ConfigMap** 与 **Secret** 两种资源？
- ASP.NET Core 如何读取它们？

下一章我们将学习 Kubernetes 配置管理，这是生产环境部署中几乎每天都会接触的内容，也是部署 **ASP.NET Core + Vben Admin** 项目时必不可少的一环。

# 第七章：ConfigMap 与 Secret —— 配置管理

提示词：

```
开始下一章
```

正文：

很好。

到这里，我们已经能够部署一个完整的网站了。

整个流程已经变成：

```
Deployment
      │
创建
      ▼
Pod
      │
运行
      ▼
Service
      │
稳定访问
      ▼
Ingress
      │
对外提供 HTTP/HTTPS
```

但是，当真正开始部署 ASP.NET Core 项目时，你马上会遇到一个非常现实的问题。

例如你的 `appsettings.json`：

```
{
  "ConnectionStrings": {
    "Default": "Server=192.168.1.100;Database=MyApp;User Id=sa;Password=123456;"
  },
  "Redis": {
    "Host": "192.168.1.101:6379"
  },
  "Jwt": {
    "Key": "MySecretKey"
  }
}
```

如果你把这些配置直接打进 Docker 镜像，会发生什么？

假设：

测试环境数据库：

```
192.168.1.100
```

生产环境数据库：

```
10.0.10.25
```

是不是意味着：

**每切换一个环境，就要重新打一次 Docker 镜像？**

显然，这是错误的。

于是 Kubernetes 引入了：

> **配置与应用分离（Configuration as Data）**

这就是：

- **ConfigMap**
- **Secret**

存在的意义。

## 本章学习目标

学习完本章，你应该能回答：

- 为什么配置不能写死到镜像？
- ConfigMap 是什么？
- Secret 是什么？
- ConfigMap 与 Secret 的区别？
- Kubernetes 如何把配置交给 Pod？
- 为什么一个 Pod 可以不用修改镜像，就运行不同环境？

------

# 第一节：为什么配置不能写进镜像？

假设：

我们有一个 Docker 镜像：

```
myapi:v1
```

里面：

```
/app

    appsettings.json
```

内容：

```
{
    "Database":"192.168.1.100"
}
```

部署到测试环境。

没问题。

但是：

生产环境数据库：

```
10.10.10.25
```

怎么办？

以前很多人会：

修改：

```
appsettings.json
```

重新：

```
docker build
```

再：

```
docker push
```

再：

部署。

是不是：

很麻烦？

更重要的是：

镜像：

本来应该：

表示：

> **程序代码。**

而不是：

环境配置。

所以：

现代部署：

遵循：

> **Build Once，Deploy Anywhere（一次构建，到处部署）**

意思：

Docker 镜像：

永远：

只有：

一份。

不同环境：

只修改：

配置。

------

# 第二节：什么是 ConfigMap？

一句话：

> **ConfigMap 用来保存普通配置。**

例如：

```
数据库地址

Redis 地址

日志等级

ASP.NET 环境变量

第三方接口地址
```

这些：

都不是：

秘密。

所以：

放：

ConfigMap。

例如：

```
ConfigMap

│

├── Database=10.0.0.20

├── Redis=10.0.0.30

└── LogLevel=Information
```

Pod：

启动：

以后。

读取：

这些：

配置。

------

# 第三节：什么是 Secret？

有些配置：

不能：

公开。

例如：

```
数据库密码

JWT Key

API Key

TLS 证书

OAuth Secret

Azure Key

AWS Secret
```

这些：

全部：

属于：

Secret。

例如：

```
Secret

│

├── Password

├── JwtKey

├── TLS

└── APIKey
```

所以：

一句话：

> **ConfigMap 保存普通配置，Secret 保存敏感配置。**

------

# 第四节：为什么要分开？

很多新人：

会问：

"为什么不能全部放 ConfigMap？"

当然可以。

但是：

生产环境：

最好不要。

因为：

Secret：

有一些：

额外能力。

例如：

- 可以单独控制访问权限（RBAC）
- 可以存储 TLS 证书
- 可以更方便与云密钥管理系统集成（如 Azure Key Vault、AWS Secrets Manager）
- Kubernetes 会将 Secret 以特定方式处理，而不是作为普通配置对象

> **需要说明一点：Kubernetes 中的 Secret 默认只是 Base64 编码，并不是加密。**
>
> 如果没有启用 **etcd Encryption at Rest（静态数据加密）**，拥有 etcd 访问权限的人仍然可以读取 Secret 内容。
>
> 因此，不要把 Secret 理解为"绝对安全"，它只是"敏感数据专用对象"。

这是很多初学者容易误解的地方。

------

# 第五节：Pod 如何读取 ConfigMap？

Kubernetes：

提供：

三种方式。

```
ConfigMap

↓

环境变量

────────────────

挂载文件

────────────────

命令参数
```

生产环境：

最常见：

前两种。

------

# 第一种：环境变量（最常用）

例如：

ConfigMap：

```
apiVersion: v1

kind: ConfigMap

metadata:

  name: my-config

data:

  Database: 10.0.0.20

  Redis: redis-service
```

然后：

Deployment：

写：

```
env:

- name: Database

  valueFrom:

    configMapKeyRef:

      name: my-config

      key: Database
```

于是：

Pod：

里面：

出现：

```
Database=10.0.0.20
```

ASP.NET Core：

自动：

读取：

环境变量。

这就是：

为什么：

ASP.NET：

在 Kubernetes：

里面：

几乎不用：

修改：

代码。

------

# ASP.NET Core 为什么天然支持？

例如：

ASP.NET Core：

配置来源：

默认：

就是：

```
appsettings.json

↓

appsettings.Environment.json

↓

Environment Variable

↓

Command Line
```

后面的来源会覆盖前面的来源。

因此：

Kubernetes：

注入：

环境变量。

ASP.NET：

自动：

覆盖：

配置。

例如：

原来：

```
{
    "ConnectionStrings": {
        "Default":"localhost"
    }
}
```

环境变量：

```
ConnectionStrings__Default

=

db-service
```

ASP.NET：

最终：

读取：

```
db-service
```

这里有一个非常重要的规则：

> **ASP.NET Core 使用双下划线（`__`）表示配置层级。**

例如：

```
ConnectionStrings__Default
```

等价于：

```
ConnectionStrings:Default
```

这个技巧在 Kubernetes 部署 ASP.NET Core 时几乎每天都会用到。

------

# 第二种：挂载文件（Volume）

有些程序：

不是：

读取：

环境变量。

而是：

必须：

读取：

文件。

例如：

```
nginx.conf

application.yml

config.json

prometheus.yml
```

怎么办？

ConfigMap：

可以：

直接：

挂载：

```
Pod

↓

/config
```

例如：

```
/config/appsettings.json
```

程序：

直接：

读取：

文件。

完全：

不知道：

这是：

ConfigMap。

------

# 第六节：Secret 怎么使用？

Secret：

和：

ConfigMap：

几乎：

一样。

例如：

```
apiVersion: v1

kind: Secret
```

然后：

Deployment：

读取：

```
secretKeyRef
```

例如：

```
env:

- name: DB_PASSWORD

  valueFrom:

    secretKeyRef:

      name: db-secret

      key: password
```

于是：

Pod：

里面：

得到：

```
DB_PASSWORD=******
```

------

# 第七节：TLS 为什么使用 Secret？

上一章：

Ingress：

HTTPS：

需要：

证书。

例如：

```
server.crt

server.key
```

Kubernetes：

要求：

保存：

到：

```
Secret

Type:

kubernetes.io/tls
```

Ingress：

引用：

这个：

Secret。

于是：

HTTPS：

成功。

所以：

TLS：

本质：

也是：

Secret。

------

# 第八节：ConfigMap 更新后，Pod 会自动更新吗？

这是：

生产环境：

最容易：

踩坑：

的问题。

答案：

**分情况。**

## 情况一：环境变量

如果：

ConfigMap：

修改：

```
Database
```

Pod：

不会：

自动：

变化。

为什么？

因为：

环境变量：

是在：

容器启动：

那一刻：

复制：

进去。

以后：

不会：

改变。

必须：

重新：

创建：

Pod。

例如：

```
kubectl rollout restart deployment my-api
```

------

## 情况二：挂载文件

如果：

ConfigMap：

作为：

Volume：

挂载。

Kubernetes：

通常会在几十秒到几分钟内，把新的内容同步到挂载文件（具体时间取决于 kubelet 的同步周期）。

但是：

> **文件更新了，不代表你的程序会重新读取。**

例如：

ASP.NET Core：

如果：

启用：

```
reloadOnChange=true
```

它：

可以：

自动：

重新：

加载。

否则：

还是：

需要：

重启。

------

# 第九节：ConfigMap、Secret、Deployment 的关系

整个流程：

```
ConfigMap

        │

        │

Secret

        │

        ▼

Deployment

        │

        ▼

Pod

        │

        ▼

ASP.NET Core
```

Deployment：

负责：

把：

ConfigMap：

Secret：

注入：

Pod。

Pod：

运行：

ASP.NET。

ASP.NET：

读取：

配置。

------

# 第十节：针对 ASP.NET Core 的最佳实践

结合你的技术栈（ASP.NET Core + Vben Admin），建议采用下面的方式：

### 普通配置（ConfigMap）

例如：

```
ASPNETCORE_ENVIRONMENT

Redis__Host

RabbitMQ__Host

ConnectionStrings__ReadOnly

Logging__LogLevel__Default
```

放：

ConfigMap。

------

### 敏感配置（Secret）

例如：

```
ConnectionStrings__Default

Jwt__Key

Redis__Password

RabbitMQ__Password

SMTP__Password

OAuth__ClientSecret
```

放：

Secret。

这样：

同一个：

Docker：

镜像。

可以：

部署：

```
开发

测试

预发布

生产
```

全部：

不用：

重新：

构建。

------

# 本章总结（建议牢记）

| 对象       | 用途             | 是否敏感 |
| ---------- | ---------------- | -------- |
| ConfigMap  | 普通配置         | ❌        |
| Secret     | 密钥、密码、证书 | ✅        |
| Deployment | 将配置注入 Pod   | -        |
| Pod        | 读取配置运行应用 | -        |

请记住 Kubernetes 配置管理最重要的三句话：

1. **镜像只保存程序，不保存环境配置。**
2. **普通配置放 ConfigMap，敏感配置放 Secret。**
3. **ASP.NET Core 可以直接通过环境变量读取 Kubernetes 配置，几乎不需要额外代码。**

------

# 下一章预告：Volume、PV、PVC——为什么数据库不能把数据放在 Pod 里？

到这里，你已经可以部署一个无状态（Stateless）的 ASP.NET Core API。

但是如果部署：

- PostgreSQL
- MySQL
- SQL Server
- MinIO
- Elasticsearch

新的问题来了：

> **Pod 删除以后，数据库数据去哪了？**

下一章，我们将进入 Kubernetes 中另一个非常重要的主题——**存储（Storage）**，包括：

- 为什么 Pod 本地磁盘不能保存重要数据？
- Volume 是什么？
- 什么是 PV（PersistentVolume）？
- 什么是 PVC（PersistentVolumeClaim）？
- StorageClass 又解决了什么问题？
- 动态供给（Dynamic Provisioning）是如何工作的？
- 如何部署一个真正可以长期保存数据的数据库？

这一章结束后，你会真正理解 Kubernetes 中**有状态应用（Stateful Application）**的部署方式，也是从开发环境走向生产环境必须掌握的知识。

# 第八章：Volume、PV、PVC、StorageClass —— Kubernetes 存储体系

提示词：

```
开始下一章
```

正文：

很好。

到目前为止，我们已经完成了 Kubernetes 最重要的几个对象：

```
Deployment
Pod
Service
Ingress
ConfigMap
Secret
```

如果部署的是：

- ASP.NET Core API
- Vue 前端
- 身份认证中心

那么到这里其实已经够了。

因为这些都属于：

> **无状态应用（Stateless Application）**

但是。

真正的生产环境，不可能只有无状态应用。

你一定还会部署：

- SQL Server
- PostgreSQL
- MySQL
- Redis（持久化）
- MinIO
- RabbitMQ
- Elasticsearch

这些服务有一个共同特点：

> **数据必须永久保存。**

于是，我们进入 Kubernetes 最容易理解错误，也是最重要的一章。

## 本章学习目标

学完这一章，你应该能回答：

- 为什么 Pod 不能保存数据？
- Volume 是什么？
- emptyDir、hostPath 是什么？
- PV、PVC 分别是什么？
- StorageClass 是什么？
- 动态创建存储是什么意思？
- StatefulSet 为什么离不开 PVC？
- 生产环境数据库为什么一定要使用 Persistent Volume？

------

# 第一节：为什么 Pod 不能保存数据？

我们先来看一个最简单的例子。

假设：

你的 ASP.NET Core API：

上传了一张图片。

保存到：

```
/app/upload
```

图片成功保存。

现在：

Deployment：

升级。

旧 Pod：

删除。

新 Pod：

创建。

结果：

```
/app/upload
```

里面：

空了。

为什么？

因为：

Pod：

本质：

就是：

容器。

容器：

删除。

文件：

一起：

删除。

所以：

Pod：

本地磁盘：

属于：

> **临时存储（Ephemeral Storage）**

------

# 一个生活例子

假设：

你住酒店。

酒店房间：

就是：

Pod。

你：

把：

重要合同：

放：

酒店抽屉。

第二天：

退房。

房间：

清空。

合同：

没了。

为什么？

因为：

酒店房间：

不是：

保险柜。

Pod：

也是：

一样。

------

# 第二节：那数据应该放哪里？

应该：

放：

一个：

独立：

磁盘。

例如：

```
Pod

│

├── API

└── Disk
```

Pod：

删除。

磁盘：

还在。

新的：

Pod：

继续：

挂载。

数据：

继续：

存在。

这就是：

> **Volume（卷）**

------

# 第三节：Volume 到底是什么？

一句话：

> **Volume 是 Pod 可以挂载的存储。**

注意：

Volume：

不是：

硬盘。

它：

只是：

一种：

"把存储接入 Pod"的方式。

例如：

```
Pod

↓

Volume

↓

真正磁盘
```

因此：

Volume：

更像：

插座。

真正：

供电：

的是：

后面的：

磁盘。

------

# 第四节：Volume 有很多类型

Kubernetes：

支持：

很多：

Volume。

例如：

```
emptyDir

hostPath

PersistentVolume

NFS

Azure Disk

AWS EBS

Ceph

CSI
```

但是。

学习：

只需要：

先掌握：

几个：

最重要。

------

# 第五节：emptyDir（临时目录）

这是：

最简单：

一种。

例如：

```
volumes:

- name: cache

  emptyDir: {}
```

什么意思？

Pod：

启动。

自动：

创建：

一个：

空目录。

例如：

```
/tmp/cache
```

Pod：

删除。

目录：

一起：

删除。

所以：

emptyDir：

适合：

```
缓存

临时文件

中间结果

解压目录
```

千万：

不要：

保存：

数据库。

------

# 第六节：hostPath（宿主机目录）

例如：

Worker：

有：

```
/data
```

Pod：

挂载：

```
/data
```

于是：

Pod：

看到：

```
/app/data
```

其实：

就是：

宿主机：

```
/data
```

这样：

Pod：

删除。

数据：

还在。

听起来：

很好。

是不是：

数据库：

都可以：

hostPath？

答案：

生产：

不要。

为什么？

例如：

Pod：

原来：

运行：

```
Worker1
```

数据：

在：

```
Worker1:/data
```

后来：

Scheduler：

重新：

调度：

```
Worker2
```

Worker2：

没有：

数据。

数据库：

坏了。

所以：

hostPath：

一般：

只适合：

单节点：

测试。

------

# 第七节：Persistent Volume（PV）

终于：

进入：

真正：

生产环境。

PV：

官方：

名字：

叫：

Persistent Volume。

中文：

永久卷。

一句话：

> **PV 表示 Kubernetes 集群中的一块存储资源。**

例如：

管理员：

准备：

100GB：

磁盘。

告诉：

Kubernetes：

```
这里：

有：

100GB
```

于是：

产生：

```
PV
```

画出来：

```
PV

100GB

SSD
```

注意。

PV：

不是：

Pod：

创建。

而是：

集群：

已有：

资源。

就像：

停车场：

已经：

修好：

100个：

车位。

------

# 第八节：PVC（PersistentVolumeClaim）

现在。

开发：

来了。

他说：

```
我要：

20GB
```

怎么办？

开发：

不能：

自己：

找：

PV。

于是：

创建：

PVC。

PVC：

全称：

PersistentVolumeClaim。

翻译：

永久卷：

申请。

注意：

它不是磁盘。

它是：

> **申请书。**

例如：

```
我要：

20GB

ReadWriteOnce
```

然后：

Kubernetes：

自动：

找：

一个：

符合：

条件：

PV。

于是：

关系：

变成：

```
PVC

↓

绑定

↓

PV

↓

真正磁盘
```

------

# 一个生活中的例子

停车场：

有：

100个：

车位。

```
PV
```

你：

来了。

拿：

停车票。

```
PVC
```

管理员：

安排：

一个：

车位。

以后：

你：

永远：

停：

那里。

所以：

PVC：

不是：

车位。

而是：

申请。

------

# 第九节：Pod 为什么不用直接使用 PV？

很多新人：

都会：

问：

为什么：

不直接：

```
Pod

↓

PV
```

答案：

解耦。

如果：

Pod：

知道：

PV。

那么：

PV：

换了。

所有：

YAML：

修改。

PVC：

就是：

中间层。

于是：

Pod：

永远：

只认识：

PVC。

例如：

```
Pod

↓

PVC

↓

PV

↓

Disk
```

以后：

磁盘：

升级。

PVC：

不用：

改。

Pod：

也：

不用：

改。

------

# 第十节：StorageClass（真正现代 Kubernetes）

到这里。

还有：

一个：

问题。

假设：

管理员：

每天：

收到：

请求。

```
20GB

50GB

100GB
```

每次：

都：

创建：

PV。

是不是：

太慢？

于是：

StorageClass：

出现。

一句话：

> **StorageClass 是自动创建 PV 的模板。**

例如：

开发：

创建：

PVC。

写：

```
storageClassName: fast-ssd
```

Kubernetes：

发现：

没有：

PV。

于是：

自动：

调用：

StorageClass。

创建：

新的：

磁盘。

然后：

绑定。

整个过程：

开发：

根本：

不知道。

这：

叫：

> **动态供给（Dynamic Provisioning）**

------

# 第十一节：完整流程

现代 Kubernetes：

几乎：

都是：

下面：

这个：

流程。

```
Pod

↓

PVC

↓

StorageClass

↓

自动创建

↓

PV

↓

真正云磁盘
```

这就是：

生产环境：

最常见：

方式。

------

# 第十二节：ReadWriteOnce、ReadOnlyMany、ReadWriteMany

除了：

容量。

PVC：

还要：

声明：

访问方式。

最常见：

三个。

## ReadWriteOnce（RWO）

意思：

一个：

节点：

读写。

例如：

SQL Server。

MySQL。

PostgreSQL。

这是：

最常见。

------

## ReadOnlyMany（ROX）

多个：

节点：

只读。

例如：

共享：

静态：

文件。

------

## ReadWriteMany（RWX）

多个：

节点：

同时：

读写。

例如：

NFS。

CephFS。

Azure Files。

很多：

共享：

文件系统：

支持。

但是：

普通：

云盘：

一般：

不支持。

------

# 第十三节：为什么数据库通常使用 StatefulSet？

现在：

再：

思考：

一个：

问题。

Deployment：

删除：

Pod。

重新：

创建。

名字：

变。

例如：

```
sql-abc123

↓

sql-xyz789
```

数据库：

希望：

名字：

永远：

一样。

于是：

Kubernetes：

设计：

StatefulSet。

例如：

```
sql-0

sql-1

sql-2
```

永远：

不会：

变化。

每个：

都有：

自己的：

PVC。

例如：

```
sql-0

↓

PVC-0

↓

PV-0

────────────

sql-1

↓

PVC-1

↓

PV-1
```

所以：

Deployment：

适合：

无状态。

StatefulSet：

适合：

数据库。

我们后面会专门讲 StatefulSet，因为它是数据库、中间件部署的核心。

------

# 第十四节：ASP.NET Core 项目需要 Volume 吗？

这是很多开发者都会问的问题。

答案：

分情况。

## 不需要

如果你的 API：

只是：

```
业务接口

JWT

Redis

SQL Server
```

数据库：

在：

外部。

那么：

API：

其实：

不用：

PV。

Deployment：

即可。

------

## 需要

例如：

你的 API：

保存：

```
上传图片

Excel

PDF

日志

缓存
```

如果：

这些：

不能：

丢。

那么：

应该：

挂：

PVC。

例如：

```
/app/upload
```

对应：

PVC。

这样：

Pod：

升级。

图片：

还在。

------

# 本章总结（建议牢记）

下面这张表，是 Kubernetes 存储体系中最重要的一张。

| 对象         | 职责               | 可以理解为                 |
| ------------ | ------------------ | -------------------------- |
| Volume       | Pod 挂载存储的方式 | 插座                       |
| emptyDir     | 临时目录           | 一次性缓存                 |
| hostPath     | 宿主机目录         | 本机磁盘（仅适合特定场景） |
| PV           | 实际存储资源       | 停车位                     |
| PVC          | 对存储的申请       | 停车票                     |
| StorageClass | 自动创建存储的模板 | 自动停车管理系统           |
| StatefulSet  | 管理有状态应用     | 数据库专用 Deployment      |

请记住 Kubernetes 存储最重要的四句话：

1. **Pod 本地磁盘是临时的，不能保存重要数据。**
2. **Pod 不直接使用 PV，而是使用 PVC。**
3. **StorageClass 让 PV 可以自动创建，这是现代 Kubernetes 的主流方式。**
4. **Deployment 适合无状态应用，StatefulSet 更适合数据库等有状态应用。**

------

# 现在，我们已经掌握了 Kubernetes 的核心资源

```
Deployment
      │
      ▼
Pod
      │
      ▼
Service
      │
      ▼
Ingress

ConfigMap
Secret

Volume
PVC
PV
StorageClass
```

到这里，你已经理解了 Kubernetes 最核心的一套资源模型。

------

# 下一章预告：StatefulSet——为什么数据库不能使用 Deployment？

虽然我们已经提到了 StatefulSet，但还有很多关键问题没有解释：

- 为什么数据库不能直接用 Deployment？
- StatefulSet 和 Deployment 到底有哪些区别？
- Headless Service 是什么？
- 为什么 Pod 名字必须固定？
- 为什么每个 Pod 都会自动拥有自己的 PVC？
- MySQL、PostgreSQL、Redis、RabbitMQ 在 Kubernetes 中是如何部署的？

这一章结束后，你就能真正理解 **有状态应用（Stateful Application）** 在 Kubernetes 中的设计思想，也能够读懂绝大多数数据库 Helm Chart 的结构。

# 第九章：StatefulSet —— Kubernetes 中的有状态应用

提示词：

```
开始下一章
```

正文：

很好。

到这里，我们已经学习了：

```
Deployment
Service
Ingress
ConfigMap
Secret
PV
PVC
StorageClass
```

但是还有一个非常重要的问题没有解决。

假设我们部署 MySQL。

如果使用 Deployment：

```
Deployment

↓

mysql-6fd8f89df8-k8j2s
```

Pod 崩了。

重新创建：

```
mysql-7bd7d7d4d9-l9v2q
```

名字变了。

IP 变了。

如果你部署的是 **MySQL 主从集群**、**Redis Cluster**、**Kafka**、**RabbitMQ Cluster**，这些节点之间是靠名字互相通信的。

名字一变。

整个集群就乱了。

于是 Kubernetes 专门设计了：

> **StatefulSet（有状态控制器）**

这一章，我们会彻底理解 StatefulSet 的设计思想。

## 本章学习目标

学完这一章，你应该能够回答：

- Deployment 和 StatefulSet 到底有什么区别？
- 为什么数据库不能随便改名字？
- StatefulSet 为什么 Pod 名字固定？
- 什么是 Headless Service？
- StatefulSet 如何自动创建 PVC？
- StatefulSet 如何保证启动顺序？
- 哪些应用应该使用 StatefulSet？

------

# 第一节：什么叫"有状态"？

很多初学者看到"有状态"都会想到 HTTP Session。

这里不是那个意思。

在 Kubernetes 中：

> **状态（State） = 应用依赖长期存在且唯一的身份或数据。**

例如：

ASP.NET Core API。

今天：

```
api-abc123
```

明天：

```
api-def456
```

没有关系。

用户不会在意。

因为：

真正的数据：

在数据库。

所以：

API：

属于：

> **无状态（Stateless）**

------

但是：

MySQL 呢？

假设：

今天：

```
mysql-0
```

里面：

保存：

500GB：

数据。

明天：

变成：

```
mysql-x8a91
```

集群里的其它节点：

还能找到它吗？

不能。

因此：

数据库：

必须：

拥有：

稳定身份。

------

## 一个生活例子

想象两种职业。

### 外卖员

今天：

小王。

明天：

小李。

顾客：

无所谓。

因为：

只要有人送餐就行。

这就是：

Deployment。

------

### 银行柜台

柜台：

```
1号窗口

2号窗口

3号窗口
```

每天：

都必须：

在那里。

否则：

预约系统。

叫号系统。

全部：

混乱。

这就是：

StatefulSet。

------

# 第二节：Deployment 最大的问题

Deployment：

创建：

Pod。

名字：

随机。

例如：

```
api-7b94dc9fd5-7r4tb
```

升级：

以后：

```
api-8d76cf4f88-kx2jq
```

完全：

不同。

为什么？

因为：

Deployment：

认为：

Pod：

可以：

随时：

替换。

对于：

Web API：

没问题。

对于：

数据库：

就是：

灾难。

------

# 第三节：StatefulSet 最大特点

StatefulSet：

最大的特点：

就是：

> **Pod 名字永远固定。**

例如：

副本：

3。

Deployment：

可能：

```
api-a9d8c

api-83hd2

api-b7c1f
```

StatefulSet：

永远：

```
mysql-0

mysql-1

mysql-2
```

即使：

Pod：

删除。

重新：

创建。

还是：

```
mysql-1
```

不会：

变。

------

# 第四节：为什么固定名字这么重要？

例如：

MySQL：

主从复制。

主节点：

```
mysql-0
```

从节点：

配置：

```
连接：

mysql-0
```

如果：

Deployment：

升级。

名字：

变成：

```
mysql-a92cd
```

从节点：

全部：

失联。

但是：

StatefulSet：

永远：

```
mysql-0
```

所以：

整个：

集群：

一直：

稳定。

------

# 第五节：固定的不只是名字，还有网络身份

还记得上一章：

Service：

提供：

固定 IP。

但是：

Service：

对应：

整个：

Deployment。

例如：

```
api-service

↓

三个 Pod
```

它：

不会：

告诉你：

是哪一个：

Pod。

数据库：

不行。

例如：

Redis Cluster：

需要：

知道：

每一个：

节点。

所以：

StatefulSet：

每个：

Pod：

都有：

自己的：

DNS。

例如：

```
mysql-0.mysql

mysql-1.mysql

mysql-2.mysql
```

注意：

这里：

不是：

IP。

而是：

DNS。

即使：

IP：

改变。

DNS：

仍然：

正确。

------

# 第六节：Headless Service 是什么？

这里：

是 StatefulSet：

最难理解：

也是：

最重要：

一个概念。

我们先回忆：

普通：

Service。

```
api-service

↓

Pod1

Pod2

Pod3
```

访问：

```
api-service
```

Service：

随机：

选择：

一个：

Pod。

例如：

```
↓

Pod2
```

但是：

数据库：

需要：

指定：

某一个：

节点。

怎么办？

于是：

出现：

Headless Service。

------

## Headless Service 的作用

Headless：

意思：

就是：

> **没有 ClusterIP。**

例如：

普通：

Service。

有：

```
10.96.0.20
```

Headless：

没有：

IP。

配置：

只有：

一句：

```
clusterIP: None
```

是不是：

很奇怪？

没有：

IP。

那：

怎么：

访问？

答案：

DNS。

例如：

查询：

```
mysql
```

普通：

Service：

返回：

```
10.96.0.20
```

Headless：

返回：

```
mysql-0

mysql-1

mysql-2
```

或者更准确地说，会返回这些 Pod 对应的 IP（DNS A 记录），因此客户端可以直接访问每个 Pod。

------

# 第七节：为什么 StatefulSet 必须配合 Headless Service？

StatefulSet：

要求：

每个：

Pod：

都有：

稳定：

DNS。

DNS：

是谁：

生成？

Headless Service。

例如：

```
StatefulSet

↓

mysql-0

↓

Headless Service

↓

mysql-0.mysql.default.svc.cluster.local
```

同理：

```
mysql-1.mysql.default.svc.cluster.local

mysql-2.mysql.default.svc.cluster.local
```

这样：

MySQL。

Kafka。

Redis。

全部：

可以：

互相：

找到。

所以：

**StatefulSet 几乎总是和一个 Headless Service 配套使用。**

------

# 第八节：StatefulSet 如何管理 PVC？

上一章：

我们：

手动：

创建：

PVC。

StatefulSet：

不用。

例如：

```
volumeClaimTemplates:
```

什么意思？

可以理解成：

> **每创建一个 Pod，就自动创建一个对应的 PVC。**

例如：

副本：

```
3
```

自动：

得到：

```
mysql-0

↓

PVC-mysql-0

↓

PV

────────────

mysql-1

↓

PVC-mysql-1

↓

PV

────────────

mysql-2

↓

PVC-mysql-2

↓

PV
```

每个：

Pod：

永远：

使用：

自己的：

磁盘。

即使：

Pod：

删除。

PVC：

仍然：

保留。

新：

Pod：

重新：

挂载。

数据：

不会：

丢。

------

# 第九节：StatefulSet 的启动顺序

Deployment：

启动：

三个：

Pod。

通常：

一起：

启动。

例如：

```
Pod1

Pod2

Pod3
```

同时：

Running。

------

StatefulSet：

不是。

它：

必须：

按顺序。

例如：

```
mysql-0

↓

Ready

↓

mysql-1

↓

Ready

↓

mysql-2
```

为什么？

假设：

MySQL：

主从。

必须：

先：

主节点。

再：

从节点。

否则：

初始化：

失败。

------

删除：

也是：

反过来。

例如：

```
mysql-2

↓

mysql-1

↓

mysql-0
```

这叫：

**有序创建（Ordered Create）**

和：

**有序删除（Ordered Termination）**

------

# 第十节：哪些应用必须使用 StatefulSet？

下面这张表，非常重要。

| 应用             | 推荐控制器  | 原因                       |
| ---------------- | ----------- | -------------------------- |
| ASP.NET Core API | Deployment  | 无状态                     |
| Vue 前端         | Deployment  | 无状态                     |
| IdentityServer   | Deployment  | 无状态（状态通常存数据库） |
| MySQL            | StatefulSet | 固定身份 + 独立存储        |
| PostgreSQL       | StatefulSet | 固定身份                   |
| SQL Server       | StatefulSet | 固定存储                   |
| Redis Cluster    | StatefulSet | 节点互相发现               |
| Kafka            | StatefulSet | Broker 身份固定            |
| RabbitMQ Cluster | StatefulSet | 节点名称固定               |
| Elasticsearch    | StatefulSet | 节点发现                   |

------

# 第十一节：ASP.NET Core 项目什么时候需要 StatefulSet？

对于你的技术栈：

- ASP.NET Core
- Vben Admin

答案：

**通常不需要。**

因为：

你的：

API：

属于：

无状态。

真正：

需要：

StatefulSet：

的是：

```
SQL Server

Redis

MinIO

RabbitMQ
```

这些：

基础设施。

你的：

业务：

API：

仍然：

Deployment。

------

# 第十二节：Deployment 与 StatefulSet 对比

这是整个章节最重要的一张表。

| 特性     | Deployment    | StatefulSet              |
| -------- | ------------- | ------------------------ |
| Pod 名称 | 随机变化      | 固定编号（如 `mysql-0`） |
| Pod 身份 | 可替换        | 唯一且稳定               |
| 启动方式 | 并行          | 默认有序                 |
| 删除方式 | 并行          | 默认逆序                 |
| 存储     | 共享或无存储  | 每个 Pod 独立 PVC        |
| 典型应用 | Web API、前端 | 数据库、中间件、消息队列 |

------

# 本章总结（建议牢记）

请记住下面几句话：

1. **Deployment 适合无状态应用，StatefulSet 适合有状态应用。**
2. **StatefulSet 最大特点是稳定身份：固定名称、稳定网络标识、独立存储。**
3. **StatefulSet 通常配合 Headless Service 使用，为每个 Pod 提供稳定的 DNS 名称。**
4. **StatefulSet 会通过 `volumeClaimTemplates` 为每个 Pod 自动创建独立的 PVC。**
5. **数据库、消息队列、分布式存储几乎都使用 StatefulSet。**

------

# 到这里，我们已经掌握了 Kubernetes 最核心的资源对象

```
Deployment
StatefulSet
Pod
Service
Ingress
ConfigMap
Secret
PV
PVC
StorageClass
```

理解这些对象之后，你已经具备阅读绝大多数 Kubernetes YAML 和 Helm Chart 的基础。

------

# 下一章预告：Namespace、Label、Selector——Kubernetes 是如何管理成百上千个资源的？

现在假设你的集群中有：

- 300 个 Deployment
- 1200 个 Pod
- 400 个 Service
- 80 个 Ingress

新的问题来了：

- Kubernetes 怎么区分开发环境、测试环境、生产环境？
- 为什么所有资源都有 Label？
- Label 和 Selector 到底是什么关系？
- Namespace 是不是 Kubernetes 的"文件夹"？
- 为什么 Helm Chart 都要求指定 Namespace？
- 一个 Pod 为什么可以同时拥有十几个 Label？

下一章，我们将学习 Kubernetes 的**资源组织与管理体系**。这一章虽然不像 Deployment 那样显眼，但它几乎贯穿所有 Kubernetes 日常运维和应用部署工作，也是阅读生产环境 YAML 时必备的基础知识。

# 第十章：Namespace、Label、Selector —— Kubernetes 的资源管理体系

提示词：

```
开始下一章
```

正文：

很好。

到这里，我们已经学习了 Kubernetes 的各种资源。

现在你可能会有一个疑问：

假设一家公司只有一个 Kubernetes 集群。

里面同时部署了：

```
开发环境（Dev）

测试环境（Test）

预发布环境（Staging）

生产环境（Prod）
```

还有：

```
订单系统

支付系统

会员系统

库存系统

日志系统

监控系统
```

加起来可能有：

```
300+ Deployment

5000+ Pod

800+ Service
```

如果全部放在一起。

会发生什么？

例如：

```
kubectl get pod
```

看到：

```
payment-api

payment-api

payment-api

payment-api

payment-api

payment-api

payment-api

......
```

你根本不知道：

哪个是开发环境。

哪个是生产环境。

于是 Kubernetes 设计了一整套资源组织体系。

其中最重要的三个概念：

> **Namespace + Label + Selector**

## 本章学习目标

学习完本章，你应该能够回答：

- Namespace 是什么？
- 为什么要使用 Namespace？
- Namespace 和 Linux 文件夹有什么区别？
- Label 是什么？
- 为什么 Kubernetes 几乎所有资源都有 Label？
- Selector 如何工作？
- Label 为什么是 Kubernetes 的核心设计？
- Kubernetes 如何利用 Label 管理几千个 Pod？

------

# 第一节：Namespace 是什么？

一句话：

> **Namespace（命名空间）就是 Kubernetes 中的资源隔离空间。**

很多人会说：

Namespace 就是文件夹。

这个比喻：

有一点像。

但并不完全正确。

更准确一点：

Namespace 更像：

> **一个独立的办公区域。**

例如：

一家公司：

有：

```
研发部

测试部

财务部

行政部
```

大家：

都在：

同一栋楼。

但是：

彼此：

互不影响。

Kubernetes：

也是：

一样。

例如：

```
Cluster

├── dev

├── test

├── staging

└── prod
```

这些：

就是：

Namespace。

------

# 第二节：为什么需要 Namespace？

假设：

没有 Namespace。

你：

创建：

```
api
```

同事：

也：

创建：

```
api
```

马上：

冲突。

因为：

资源名字：

必须：

唯一。

但是：

有了：

Namespace。

开发：

```
dev

↓

api
```

生产：

```
prod

↓

api
```

是不是：

就没有：

冲突？

因为：

真正：

资源名称：

其实：

可以理解成：

```
dev/api

prod/api
```

它们：

属于：

两个：

不同：

命名空间。

------

# 第三节：Kubernetes 默认有哪些 Namespace？

安装好 Kubernetes。

一般都会看到：

```
kubectl get ns
```

输出：

```
NAME              STATUS
default           Active
kube-system       Active
kube-public       Active
kube-node-lease   Active
```

下面分别介绍。

------

## default

这是：

默认：

Namespace。

如果：

你：

没有：

指定：

Namespace。

例如：

```
kubectl apply -f api.yaml
```

默认：

进入：

```
default
```

所以：

很多教程：

都是：

在：

default。

但是。

生产环境：

**不要把所有业务都部署到 default。**

建议按环境或团队划分 Namespace，例如：

```
dev
test
staging
prod
monitoring
logging
```

------

## kube-system

这是：

Kubernetes：

系统组件。

例如：

```
CoreDNS

kube-proxy

metrics-server

Ingress Controller（很多安装方式会部署在这里或单独命名空间）
```

全部：

都在：

这里。

一般：

不要：

修改。

------

## kube-public

用于：

公开：

共享：

一些：

集群：

信息。

普通开发者：

很少：

使用。

------

## kube-node-lease

用于：

Node：

心跳。

Kubernetes：

内部：

使用。

一般：

不用：

管理。

------

# 第四节：如何查看指定 Namespace？

例如：

查看：

生产：

Pod。

```
kubectl get pod -n prod
```

查看：

开发：

```
kubectl get deployment -n dev
```

查看：

全部：

```
kubectl get pod -A
```

或者：

```
kubectl get pod --all-namespaces
```

输出：

例如：

```
NAMESPACE     NAME

dev           api-xxx

prod          api-yyy

monitoring    prometheus

logging       loki
```

第一列：

就是：

Namespace。

------

# 第五节：Namespace 能隔离什么？

这是：

一个：

容易：

误解：

的问题。

很多新人认为：

Namespace：

就是：

虚拟机。

不是。

Namespace：

主要：

隔离：

Kubernetes：

资源。

例如：

可以隔离：

```
Pod

Deployment

Service

ConfigMap

Secret

Ingress
```

但是：

Node：

不能。

例如：

整个：

Cluster：

只有：

三台：

Worker。

所有：

Namespace：

共享：

这：

三台。

所以：

Namespace：

不是：

物理隔离。

而是：

逻辑隔离。

------

# 第六节：什么是 Label？

现在：

进入：

Kubernetes：

真正：

最重要：

设计。

一句话：

> **Label（标签）就是给资源贴标签。**

例如：

Pod：

```
labels:

  app: api

  env: prod

  version: v2

  team: payment
```

是不是：

很像：

快递：

标签？

例如：

一个：

包裹。

贴：

```
易碎

上海

顺丰

加急
```

以后：

就：

可以：

根据：

标签：

管理。

------

# 第七节：一个 Pod 可以有很多 Label

例如：

ASP.NET Core：

API。

可以：

同时：

拥有：

```
labels:

  app: api

  env: prod

  version: v2

  team: payment

  region: taiwan

  tier: backend
```

注意：

不是：

只能：

一个。

可以：

几十个。

以后：

各种：

资源。

都：

通过：

Label：

找到：

它。

------

# 第八节：Selector 是什么？

Label：

只是：

贴：

标签。

谁：

使用？

答案：

Selector。

例如：

Deployment：

写：

```
selector:

  matchLabels:

    app: api
```

意思：

找到：

所有：

Label：

```
app=api
```

Pod。

Service：

也是：

一样。

例如：

```
selector:

  app: api
```

于是：

Service：

知道：

应该：

转发：

哪些：

Pod。

------

# 第九节：Label 与 Selector 的关系

这是：

整本 Kubernetes：

最重要：

的一张图。

```
Pod

Label：

app=api

env=prod

version=v2

        ▲

        │

Selector

app=api

        │

        ▼

Deployment

Service

NetworkPolicy

...
```

记住：

**Label 是数据。**

**Selector 是查询条件。**

就像数据库：

```
SELECT *

FROM Pod

WHERE app='api'
```

是不是：

一下：

就理解了？

------

# 第十节：为什么 Label 如此重要？

假设：

集群：

里面：

1000 个：

Pod。

如果：

没有：

Label。

Deployment：

怎么知道：

哪些：

Pod：

属于：

自己？

Service：

怎么知道：

流量：

转发：

给谁？

NetworkPolicy：

怎么知道：

限制：

哪些：

Pod？

答案：

全部：

依赖：

Label。

可以说：

> **Kubernetes 的大多数资源之间，并不是通过"名字"关联，而是通过 Label + Selector 关联。**

这也是 Kubernetes 与很多传统运维系统最大的区别之一。

------

# 第十一节：Label 的最佳实践

生产环境中，不要只写：

```
labels:

  app: api
```

建议：

至少：

包含：

这些：

维度。

```
labels:

  app: order-api

  env: prod

  version: v2.1.0

  team: payment

  tier: backend
```

这样：

以后：

无论：

查询。

监控。

日志。

权限。

都：

方便。

例如：

查看：

所有：

生产：

API。

```
env=prod

tier=backend
```

查看：

支付：

团队：

资源。

```
team=payment
```

------

# 第十二节：Namespace 与 Label 的区别

很多新人：

第一次：

都会：

混。

下面：

这张表：

一定：

记住。

| Namespace                      | Label                       |
| ------------------------------ | --------------------------- |
| 一个资源只能属于一个 Namespace | 一个资源可以有多个 Label    |
| 用于逻辑隔离                   | 用于分类和筛选              |
| 例如：`prod`、`dev`            | 例如：`env=prod`、`app=api` |
| 主要用于组织资源               | 主要用于资源关联            |

例如：

一个：

Pod：

可以：

是：

```
Namespace：

prod
```

同时：

拥有：

```
app=api

team=payment

version=v2

region=taiwan
```

两者：

不是：

互相：

替代。

而是：

互补。

------

# 第十三节：实际项目应该如何规划？

以你的 **ASP.NET Core + Vben Admin** 项目为例，一个比较合理的规划可以是：

```
Namespace: dev
    ├── order-api
    ├── vben-admin
    └── redis

Namespace: prod
    ├── order-api
    ├── vben-admin
    └── redis
```

其中：

API：

可以：

拥有：

```
labels:

  app: order-api

  env: prod

  team: backend

  version: v1.0.0
```

Redis：

可以：

拥有：

```
labels:

  app: redis

  env: prod

  tier: cache
```

Service：

通过：

```
selector:

  app: order-api
```

找到：

对应：

Pod。

------

# 第十四节：kubectl 中最常用的 Namespace 操作

查看：

所有：

Namespace：

```
kubectl get ns
```

创建：

Namespace：

```
kubectl create namespace dev
```

删除：

Namespace：

```
kubectl delete namespace dev
```

> ⚠️ 注意：删除 Namespace 会删除其中绝大多数命名空间级资源（如 Deployment、Pod、Service、ConfigMap、Secret 等）。生产环境一定要谨慎操作。

如果经常操作同一个 Namespace，可以切换 `kubectl` 当前上下文的默认 Namespace，这样就不用每次都写 `-n prod`。这是日常运维中非常常见的做法。

------

# 本章总结（建议牢记）

请记住 Kubernetes 资源管理最重要的几句话：

1. **Namespace 用于资源的逻辑隔离，一个资源只能属于一个 Namespace。**
2. **Label 用于给资源打标签，一个资源可以拥有很多 Label。**
3. **Selector 根据 Label 找资源，是 Deployment、Service 等控制器工作的基础。**
4. **Kubernetes 中大量资源之间的关联都是通过 Label + Selector 完成，而不是通过资源名称。**
5. **生产环境建议按环境、团队或业务划分 Namespace，并建立统一的 Label 规范。**

------

# 到目前为止，你已经掌握了 Kubernetes 的核心对象和组织方式

```
Namespace
      │
管理资源范围
      ▼
Deployment / StatefulSet
      │
创建
      ▼
Pod
      │
通过 Label 标识
      ▼
Service（Selector 选择 Pod）
      │
对内提供稳定入口
      ▼
Ingress
      │
对外提供 HTTP/HTTPS 服务
```

这已经覆盖了绝大多数 Kubernetes 应用部署的基础知识。

------

# 下一章预告：Probe（探针）——Kubernetes 如何判断你的应用是否真的"活着"？

假设你的 ASP.NET Core API 已经启动：

```
Pod：Running
```

但是实际上：

- 数据库还没连接完成；
- Redis 正在重连；
- 程序已经发生死锁；
- HTTP 服务虽然还在监听，但所有请求都返回 500。

Kubernetes 怎么知道：

- 应该什么时候开始对外提供流量？
- 什么时候应该自动重启这个 Pod？
- 如何避免滚动升级时把流量切给一个还没准备好的新版本？

下一章我们将学习 **Liveness Probe、Readiness Probe、Startup Probe**，这是 Kubernetes 实现**高可用、自愈和零停机发布**的关键机制，也是生产环境中最容易配置错误、但最值得掌握的内容。

# 第十一章：Probe（探针）—— Kubernetes 如何判断应用是否真的健康？

提示词：

```
开始下一章
```

正文：

很好。

到目前为止，我们已经可以部署一个完整的应用：

```
Deployment
      │
      ▼
Pod
      │
      ▼
Service
      │
      ▼
Ingress
```

Deployment 会自动创建 Pod。

Pod 运行 ASP.NET Core。

Service 对外提供稳定地址。

Ingress 提供 HTTP/HTTPS。

看起来一切都很好。

但是。

真正的生产环境，很快就会遇到下面这个问题。

假设：

你的 ASP.NET Core：

启动：

需要：

```
35 秒
```

原因：

```
连接 SQL Server

初始化 EF Core

连接 Redis

加载配置

预热缓存
```

但是：

Pod：

刚启动：

3 秒。

状态：

已经：

```
Running
```

是不是：

说明：

应用：

已经：

可以：

提供服务？

**不是。**

这里：

很多新人都会误解。

## 本章学习目标

学完本章，你应该能够回答：

- Running 为什么不代表应用已经可用？
- 什么是 Probe（探针）？
- Liveness Probe 是什么？
- Readiness Probe 是什么？
- Startup Probe 是什么？
- 三种 Probe 有什么区别？
- ASP.NET Core 如何配置健康检查？
- 为什么 Probe 是滚动升级和自动恢复的基础？

------

# 第一节：Running ≠ 应用正常

这是 Kubernetes 最容易踩坑的一点。

假设：

Pod：

启动。

执行：

```
dotnet MyApi.dll
```

容器：

成功：

启动。

于是：

Pod：

状态：

```
Running
```

但是：

你的程序：

还在：

```
等待数据库

等待 Redis

等待 RabbitMQ
```

用户：

现在：

访问：

```
/api/login
```

返回：

```
500
```

为什么？

因为：

程序：

还没：

准备好。

所以：

> **Running 只表示容器进程已经启动，不代表应用已经可以提供服务。**

------

# 一个现实生活例子

假设：

一家餐厅。

老板：

刚刚：

开门。

灯：

亮了。

员工：

已经：

进店。

但是：

厨师：

还在：

洗菜。

米饭：

没煮。

锅：

没热。

这时候：

餐厅：

算：

营业了吗？

显然：

不算。

容器：

Running。

就像：

灯：

亮了。

真正：

营业。

应该：

是：

> **Ready。**

------

# 第二节：Probe 是什么？

一句话：

> **Probe（探针）就是 Kubernetes 主动检查应用健康状况的方法。**

可以理解成：

Kubernetes：

隔几秒：

就会：

问：

```
你：

还好吗？
```

如果：

回答：

正常。

继续：

运行。

如果：

回答：

异常。

根据：

不同：

Probe。

采取：

不同：

动作。

------

# 第三节：Kubernetes 有三种 Probe

这是：

整章：

最重要：

的一张图。

```
Startup Probe

↓

应用

有没有

启动成功？

──────────────

Readiness Probe

↓

应用

现在

能不能

接收请求？

──────────────

Liveness Probe

↓

应用

是不是

已经

死掉了？
```

很多人：

第一次：

会：

混。

下面：

分别：

讲。

------

# 第四节：Readiness Probe（就绪探针）

这是：

生产环境：

最常用：

Probe。

一句话：

> **告诉 Service：现在可以把流量转给我了吗？**

假设：

Deployment：

有：

三个：

Pod。

```
Pod1

Ready

────────

Pod2

Ready

────────

Pod3

Starting...
```

此时：

Service：

会：

把：

请求：

发送：

给：

```
Pod1

Pod2
```

不会：

发送：

给：

```
Pod3
```

直到：

Readiness：

返回：

成功。

于是：

Pod3：

加入：

负载均衡。

------

## 为什么 Readiness 如此重要？

假设：

滚动升级。

原来：

```
v1

v1

v1
```

开始：

升级：

v2。

Deployment：

创建：

新的：

Pod。

如果：

没有：

Readiness。

Pod：

刚：

Running。

Service：

立即：

把：

请求：

发送：

过去。

但是：

数据库：

还没：

连接。

于是：

用户：

全部：

500。

Readiness：

就是：

避免：

这种：

事故。

------

# 第五节：Liveness Probe（存活探针）

现在：

假设：

程序：

已经：

运行：

2 天。

突然：

发生：

死锁。

例如：

```
CPU：

0%

没有：

异常

没有：

退出

但是：

所有：

请求：

卡死
```

容器：

还在。

Pod：

还是：

Running。

Deployment：

认为：

正常。

但是：

网站：

打不开。

怎么办？

Liveness：

出现。

它：

不停：

检查：

```
应用：

还活着吗？
```

如果：

失败：

连续：

很多次。

Kubernetes：

直接：

重启：

容器。

所以：

Liveness：

真正：

作用：

是：

> **自动恢复（Self Healing）**

------

# 一个生活例子

Readiness：

像：

餐厅：

门口：

挂牌：

```
营业中
```

如果：

没：

挂牌。

客人：

不会：

进去。

------

Liveness：

像：

老板：

发现：

厨师：

晕倒。

直接：

叫：

120。

然后：

换：

新：

厨师。

------

# 第六节：Startup Probe（启动探针）

这里：

很多：

新人：

不知道：

为什么：

还要：

第三个。

例如：

ASP.NET Core：

第一次：

启动：

需要：

90 秒。

Liveness：

每：

10 秒：

检查。

连续：

失败：

3 次。

于是：

30 秒：

Kubernetes：

认为：

程序：

死了。

直接：

重启。

结果：

程序：

永远：

启动：

不了。

这：

就是：

经典：

启动：

死循环。

于是：

Startup Probe：

出现。

它：

告诉：

Kubernetes：

```
先：

别：

检查：

Liveness。

等：

我：

真正：

启动：

成功。

再：

开始：

检查。
```

所以：

Startup：

主要：

解决：

> **启动慢的问题。**

------

# 第七节：三种 Probe 的关系

这是：

生产环境：

必须：

理解：

的一张图。

```
容器启动

      │

      ▼

Startup Probe

      │

成功

      ▼

Readiness Probe

      │

Ready=True

      ▼

Service

开始

转发流量

────────────

同时

Liveness Probe

持续检查

应用是否存活
```

注意：

如果：

配置：

Startup Probe。

那么：

在：

Startup：

成功：

之前。

Liveness：

Readiness：

都：

不会：

生效。

------

# 第八节：Probe 如何检查？

Kubernetes：

支持：

三种：

检查方式。

```
HTTP

TCP

Command
```

------

## HTTP（最常见）

例如：

ASP.NET Core：

提供：

```
/health
```

Kubernetes：

访问：

```
GET

/health
```

返回：

```
200
```

说明：

健康。

否则：

失败。

------

## TCP

例如：

数据库。

检查：

```
3306
```

端口：

能否：

连接。

能：

说明：

服务：

还在。

------

## Command

执行：

Linux：

命令。

例如：

```
cat

/tmp/healthy
```

如果：

成功。

说明：

正常。

这种：

一般：

用于：

特殊：

程序。

------

# 第九节：ASP.NET Core 如何配置 Health Check？

ASP.NET Core：

天生：

支持。

例如：

Program.cs

```
builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/health");
```

运行：

以后。

浏览器：

访问：

```
/health
```

返回：

```
200 OK
```

Kubernetes：

直接：

拿来：

做：

Probe。

如果你的应用依赖数据库、Redis 等，还可以把它们加入 Health Check。这样只有真正依赖都正常时，Readiness Probe 才会返回成功。

------

# 第十节：Deployment 如何配置 Probe？

例如：

```
containers:
- name: api

  image: myapi:v1

  readinessProbe:

    httpGet:

      path: /health

      port: 8080

    initialDelaySeconds: 5

    periodSeconds: 10

  livenessProbe:

    httpGet:

      path: /health

      port: 8080

    initialDelaySeconds: 30

    periodSeconds: 20
```

这里只需要理解：

Readiness：

负责：

接流量。

Liveness：

负责：

重启。

后面：

我们：

讲 YAML：

会：

详细：

解释：

每个：

参数。

------

# 第十一节：Probe 与滚动升级

现在：

终于：

理解：

为什么：

Deployment：

不会：

停机。

升级：

过程：

其实：

就是：

```
创建：

新 Pod

↓

Readiness

成功

↓

Service

加入

↓

删除：

旧 Pod
```

如果：

Readiness：

失败。

Deployment：

不会：

继续。

这就是：

为什么：

Probe：

是：

零停机：

部署：

基础。

------

# 第十二节：ASP.NET Core 项目的最佳实践

结合你的 ASP.NET Core 项目，可以遵循以下建议：

### Readiness Probe

检查：

```
/health/ready
```

确认：

- 数据库连接正常
- Redis 可连接
- RabbitMQ 可连接（如果依赖）
- 必要配置加载完成

只有满足这些条件，才返回成功。

------

### Liveness Probe

检查：

```
/health/live
```

只确认：

```
程序主循环正常

没有死锁

Web 服务仍然响应
```

**不要**在 Liveness 中检查数据库。

为什么？

如果：

数据库：

临时：

网络：

抖动。

Liveness：

失败。

Pod：

不停：

重启。

问题：

反而：

更严重。

因此：

**外部依赖是否可用，更适合作为 Readiness 的检查内容，而不是 Liveness。**

------

### Startup Probe

如果：

你的：

应用：

启动：

超过：

30 秒。

建议：

配置。

避免：

启动：

一直：

被：

误杀。

------

# 第十三节：三种 Probe 对比

这是整章最重要的一张表。

| Probe           | 作用                 | 失败后会怎样                | 常见检查内容            |
| --------------- | -------------------- | --------------------------- | ----------------------- |
| Startup Probe   | 判断应用是否启动完成 | 启动失败达到阈值后重启容器  | 应用初始化完成          |
| Readiness Probe | 判断是否可以接收流量 | 从 Service 后端移除，不重启 | 数据库、Redis、必要依赖 |
| Liveness Probe  | 判断应用是否仍然存活 | 重启容器                    | 应用主循环、HTTP 响应   |

------

# 本章总结（建议牢记）

请记住 Kubernetes 健康检查最重要的五句话：

1. **Running 不代表应用已经可以提供服务。**
2. **Readiness 决定 Service 是否把流量发送给这个 Pod。**
3. **Liveness 决定 Kubernetes 是否需要重启这个 Pod。**
4. **Startup Probe 用于保护启动较慢的应用，避免被误判。**
5. **对于 ASP.NET Core，建议分别提供 `/health/live` 和 `/health/ready` 两个健康检查接口，而不是共用一个。**

------

# 到这里，你已经掌握了 Kubernetes 应用部署的完整生命周期

```
Deployment
      │
创建
      ▼
Pod
      │
Startup Probe（启动完成？）
      ▼
Readiness Probe（可以接流量？）
      ▼
Service
      │
Ingress
      ▼
用户访问

（运行期间）

Liveness Probe（还活着吗？）
      │
      ▼
失败则自动重启
```

可以看到，**Probe 是 Kubernetes 实现高可用、自愈和零停机升级的重要机制**。

------

# 下一章预告：Resource（Requests、Limits）——为什么 Kubernetes 不会让一个 Pod 吃光整台服务器？

现在假设：

你的集群有一台 Worker：

```
CPU：8 Core

内存：32 GB
```

你部署了：

- ASP.NET Core API
- Redis
- RabbitMQ
- PostgreSQL
- Prometheus
- Grafana

如果其中一个 API 因为程序 Bug：

```
CPU：100%

内存：28GB
```

其它所有服务都会受到影响，甚至整个节点可能因为内存耗尽而出现 **OOM（Out Of Memory）**。

Kubernetes 是如何防止这种情况的？

下一章，我们将学习：

- `requests` 与 `limits` 的区别
- Scheduler 为什么只看 `requests`
- CPU 为什么可以"超卖"，内存为什么不能
- OOMKilled 是怎么发生的
- 如何为 ASP.NET Core 设置合理的资源限制
- 为什么资源配置会影响自动扩缩容（HPA）

这一章结束后，你会真正理解 Kubernetes 如何实现资源隔离和容量管理，这也是生产环境稳定运行的关键。

# 第十二章：Resources（Requests、Limits）—— Kubernetes 如何管理 CPU 和内存？

提示词：

```
开始下一章
```

正文：

很好。

到目前为止，我们已经学会了：

- 如何部署应用（Deployment）
- 如何暴露服务（Service、Ingress）
- 如何管理配置（ConfigMap、Secret）
- 如何管理存储（PV、PVC）
- 如何部署数据库（StatefulSet）
- 如何判断应用健康（Probe）

现在，我们要学习 Kubernetes 中最容易导致生产事故、也是运维工程师每天都会关注的话题：

> **资源管理（Resource Management）**

很多线上故障，其实都不是代码 Bug，而是资源配置不合理导致的。

例如：

- Pod 一直 `Pending`
- Pod 被 `OOMKilled`
- CPU 一直 100%
- 节点突然不可用
- 自动扩容（HPA）不生效

这些问题，几乎都和今天这一章有关。

## 本章学习目标

学完本章，你应该能够回答：

- 为什么 Kubernetes 要限制资源？
- CPU 和内存是如何分配的？
- `requests` 是什么？
- `limits` 是什么？
- 为什么 Scheduler 只看 `requests`？
- 什么是 OOMKilled？
- CPU 为什么可以超卖，内存为什么不能？
- 如何为 ASP.NET Core 设置合理的资源？

------

# 第一节：为什么需要资源管理？

假设：

你的 Worker 节点：

```
CPU：8 Core

Memory：32 GB
```

现在：

部署：

```
API-A

API-B

Redis

RabbitMQ

SQL Server
```

都运行正常。

突然：

API-A：

出现：

Bug。

例如：

```
while(true)
{
}
```

CPU：

马上：

变成：

```
800%
```

（8 个核心全部跑满）

或者：

程序：

不停：

创建对象：

```
new byte[1024 * 1024];
```

一分钟：

内存：

占用：

28GB。

结果：

整个：

Worker：

资源：

耗尽。

其它：

所有：

Pod：

开始：

异常。

是不是：

很危险？

所以：

Kubernetes：

必须：

限制：

每个：

Pod：

最多：

使用：

多少：

资源。

------

# 第二节：Kubernetes 管理哪些资源？

最常见：

只有：

两个。

```
CPU

Memory
```

当然：

还有：

```
GPU

Ephemeral Storage（临时存储）
HugePages
```

但是。

对于：

ASP.NET Core：

项目。

99%：

情况下：

只需要：

理解：

CPU。

Memory。

------

# 第三节：Requests 是什么？

一句话：

> **Requests = 我至少需要这么多资源才能正常运行。**

例如：

你的：

API：

至少：

需要：

```
requests:

  cpu: "500m"

  memory: "512Mi"
```

什么意思？

就是：

告诉：

Scheduler：

> **请至少给我半个 CPU 和 512MB 内存。**

这里：

注意：

Scheduler：

不是：

运行：

Pod。

Scheduler：

负责：

找：

Node。

例如：

集群：

```
Node1

Node2

Node3
```

Scheduler：

会：

计算：

哪个：

Node：

还有：

足够：

资源。

然后：

放：

进去。

------

# 一个生活例子

假设：

一家：

电影院。

一排：

只有：

10 个：

座位。

你：

买票：

时：

告诉：

售票员：

```
我要：

2 个：

座位。
```

售票员：

必须：

保证：

还有：

2 个：

空位。

否则：

不能：

卖。

Requests：

就是：

预留：

座位。

------

# 第四节：Limits 是什么？

一句话：

> **Limits = 我最多只能使用这么多资源。**

例如：

```
limits:

  cpu: "1"

  memory: "1Gi"
```

意思：

最多：

```
CPU：

1 Core

Memory：

1GB
```

超过：

怎么办？

CPU：

和：

Memory：

处理：

完全：

不同。

这一点：

非常：

重要。

------

# 第五节：CPU Limit

假设：

Limit：

```
cpu: "1"
```

程序：

突然：

需要：

```
3 Core
```

怎么办？

Kubernetes：

不会：

杀死：

Pod。

而是：

限制：

CPU。

例如：

只能：

使用：

```
1 Core
```

这：

叫：

> **CPU Throttling（CPU 限流）**

程序：

还能：

运行。

只是：

变慢。

------

# 第六节：Memory Limit

内存：

不一样。

例如：

Limit：

```
memory: "1Gi"
```

程序：

突然：

需要：

```
1.3Gi
```

怎么办？

不能：

像：

CPU：

一样：

慢一点。

因为：

内存：

不能：

凭空：

增加。

于是：

Linux：

直接：

杀死：

进程。

Pod：

状态：

变成：

```
OOMKilled
```

这是：

Kubernetes：

最常见：

的问题：

> **OOM（Out Of Memory）**

------

# 第七节：为什么 CPU 可以超卖，而内存不能？

这是：

Kubernetes：

资源管理：

最重要：

的知识点。

假设：

CPU：

只有：

1 Core。

两个：

程序：

都需要：

1 Core。

Linux：

可以：

轮流：

执行。

例如：

```
A

B

A

B

A

B
```

虽然：

速度：

慢。

但是：

都：

能：

运行。

所以：

CPU：

可以：

共享。

------

但是：

内存：

不是。

假设：

电脑：

只有：

8GB。

程序：

需要：

10GB。

怎么办？

Linux：

没有：

办法：

把：

8GB：

变成：

10GB。

所以：

只能：

杀掉：

某个：

程序。

因此：

> **CPU 可以限流，内存只能 OOM。**

------

# 第八节：CPU 为什么使用 m？

很多新人：

第一次：

看到：

```
cpu: 500m
```

都会：

问：

m：

是什么？

不是：

MB。

这里：

m：

表示：

> **millicore（毫核）**

换算：

```
1000m = 1 Core

500m = 0.5 Core

250m = 0.25 Core

100m = 0.1 Core
```

例如：

8 Core：

机器。

总共：

```
8000m
```

------

# 第九节：Memory 为什么使用 Mi、Gi？

例如：

```
memory: 512Mi
```

不是：

512MB。

Mi：

表示：

Mebibyte。

计算：

方式：

```
1Mi = 1024 × 1024 Byte

1024Mi = 1Gi
```

Kubernetes 推荐使用二进制单位（Mi、Gi），与 Linux 内核和容器运行时保持一致。

------

# 第十节：Requests 和 Limits 的关系

最经典：

配置：

例如：

```
resources:

  requests:

    cpu: "500m"

    memory: "512Mi"

  limits:

    cpu: "1"

    memory: "1Gi"
```

意思：

至少：

需要：

```
0.5 Core

512MB
```

最多：

使用：

```
1 Core

1GB
```

Scheduler：

只看：

Requests。

运行：

过程中：

Limit：

才：

生效。

------

# 第十一节：为什么 Scheduler 不看 Limits？

假设：

Node：

```
8 Core
```

两个：

Pod。

都是：

```
request:

1 Core

limit:

8 Core
```

Scheduler：

只会：

计算：

```
Request：

2 Core
```

所以：

还能：

继续：

调度。

为什么？

因为：

Kubernetes：

认为：

大家：

不会：

一直：

跑满。

这：

叫：

> **资源超分（Overcommit）**

这是：

现代：

云平台：

普遍：

采用：

的方法。

但也意味着，如果所有 Pod 同时冲到各自的 CPU Limit，节点上的应用都会变慢；如果内存同时逼近限制，则可能触发 OOM。

------

# 第十二节：ASP.NET Core 如何设置？

对于：

一般：

Web API。

可以：

作为：

起点：

参考：

```
resources:

  requests:

    cpu: "250m"

    memory: "256Mi"

  limits:

    cpu: "1"

    memory: "512Mi"
```

**但请注意：这不是通用标准值。**

真正的配置应该根据：

- 应用压测结果
- 实际 CPU、内存使用情况
- 并发量
- GC 行为

不断调整。

例如：

高并发：

API。

可能：

需要：

```
2 Core

2Gi
```

而：

后台：

任务。

可能：

只要：

```
100m

128Mi
```

------

# 第十三节：如何查看资源使用情况？

安装 Metrics Server 后，可以：

```
kubectl top pod
```

例如：

```
NAME          CPU   MEMORY

api-xxx       80m   210Mi

redis         30m   90Mi
```

查看：

Node：

```
kubectl top node
```

例如：

```
NODE

worker-1

CPU

35%

Memory

60%
```

这是日常排查资源问题最常用的命令之一。

------

# 第十四节：常见资源问题

下面这些状态，你以后一定会遇到。

| 现象           | 常见原因                          | 解决思路                           |
| -------------- | --------------------------------- | ---------------------------------- |
| Pod Pending    | Requests 太大，没有节点满足       | 调整 Requests 或扩容节点           |
| OOMKilled      | 超过 Memory Limit                 | 增加内存、优化程序、分析内存泄漏   |
| CPU Throttling | 超过 CPU Limit                    | 提高 CPU Limit 或优化 CPU 密集逻辑 |
| 节点资源耗尽   | Requests 配置不合理或节点容量不足 | 重新规划资源或增加节点             |

------

# 第十五节：为什么资源配置会影响自动扩缩容（HPA）？

我们下一章会详细介绍 **HPA（Horizontal Pod Autoscaler）**。

这里先理解一个概念。

例如：

HPA：

目标：

```
CPU：

70%
```

如果：

你的：

Requests：

配置：

严重：

偏小。

例如：

实际：

需要：

```
500m
```

却：

配置：

```
100m
```

那么：

CPU 使用率很快就会显示为远超 100%，HPA 会频繁扩容。

反过来：

如果：

Requests：

配置：

过大。

HPA：

可能：

迟迟：

不扩容。

因此：

**Requests 不仅影响调度，还直接影响 HPA 的扩缩容判断。**

------

# 本章总结（建议牢记）

请记住 Kubernetes 资源管理最重要的六句话：

1. **Requests 表示调度时保证的最小资源，Scheduler 主要依据它进行节点选择。**
2. **Limits 表示运行时允许使用的最大资源。**
3. **CPU 超过 Limit 会被限流（Throttling），不会直接杀死容器。**
4. **Memory 超过 Limit 会触发 OOMKilled。**
5. **CPU 可以超分，内存通常不能依赖超分来保证稳定运行。**
6. **合理设置 Requests 和 Limits，是稳定运行和自动扩缩容的基础。**

------

# 到这里，你已经掌握了 Kubernetes 的资源管理体系

```
Deployment
      │
创建
      ▼
Pod
      │
Resources
      │
├── Requests（调度保证）
└── Limits（运行上限）
      │
      ▼
Node
```

------

# 下一章预告：HPA（Horizontal Pod Autoscaler）——Kubernetes 如何根据流量自动扩容？

现在假设：

你的 ASP.NET Core API：

平时：

```
2 个 Pod
```

双十一：

开始。

CPU：

从：

```
20%

↓

95%
```

难道：

运维：

半夜：

登录：

执行：

```
kubectl scale deployment api --replicas=20
```

当然不是。

下一章，我们将学习 Kubernetes 的**自动扩缩容（HPA）**，包括：

- HPA 的工作原理
- Metrics Server 的作用
- 根据 CPU、内存自动扩容
- 自定义指标（如 QPS、队列长度）
- HPA 与 Requests 的关系
- HPA 与 Cluster Autoscaler 的区别
- ASP.NET Core API 的自动扩容最佳实践

这一章结束后，你就能让 Kubernetes 根据业务负载自动增加或减少 Pod 数量，真正发挥云原生平台的弹性能力。

# 第十三章：HPA（Horizontal Pod Autoscaler）—— Kubernetes 如何自动扩容？

提示词：

```
开始下一章
```

正文：

非常好。

到这里，我们已经学习了 Kubernetes 的资源管理：

```
Requests（至少需要多少资源）
Limits（最多可以使用多少资源）
```

现在，我们来看一个真实的生产场景。

------

假设你的 ASP.NET Core API：

平时只有：

```
2 个 Pod
```

每天凌晨：

CPU：

```
15%
```

运行得很好。

但是。

每天晚上：

8 点。

大量用户同时登录。

CPU：

瞬间：

```
95%
```

请求开始排队。

接口越来越慢。

用户投诉：

```
网页打不开

订单提交失败

登录超时
```

怎么办？

传统运维：

一般这样做：

```
kubectl scale deployment api --replicas=10
```

流量过去以后。

再：

```
kubectl scale deployment api --replicas=2
```

是不是：

每天都要：

人工操作？

当然不是。

于是 Kubernetes 提供了：

> **HPA（Horizontal Pod Autoscaler）**

也就是：

**水平自动扩缩容。**

## 本章学习目标

学习完本章，你应该能够回答：

- 什么是 HPA？
- 为什么叫 Horizontal（水平）？
- HPA 如何工作？
- Metrics Server 是什么？
- HPA 为什么依赖 Requests？
- HPA 如何根据 CPU 自动扩容？
- HPA 可以根据内存扩容吗？
- HPA 能根据 QPS、队列长度扩容吗？
- HPA 与 Cluster Autoscaler 有什么区别？

------

# 第一节：什么叫 Horizontal？

很多新人：

第一次：

都会问：

为什么：

叫：

Horizontal？

因为：

Kubernetes：

有：

两种：

扩容。

------

第一种：

增加：

Pod。

例如：

原来：

```
API

↓

2 个 Pod
```

变成：

```
API

↓

8 个 Pod
```

是不是：

数量：

变多了？

这：

叫：

> **Horizontal（水平扩容）**

------

第二种：

增加：

Pod：

资源。

例如：

原来：

```
CPU：

500m
```

改成：

```
2 Core
```

Pod：

没有：

增加。

只是：

变胖了。

这：

叫：

> **Vertical（垂直扩容）**

Kubernetes：

今天：

讲：

第一种。

------

# 一个生活例子

一家餐厅。

原来：

只有：

2 个：

服务员。

客人：

越来越多。

方案一：

再招聘：

8 个：

服务员。

这：

就是：

HPA。

------

方案二：

不给：

增加：

服务员。

而是：

让：

原来：

两个：

服务员：

跑得：

更快。

显然：

有限。

------

# 第二节：HPA 到底是什么？

一句话：

> **HPA 根据监控指标自动调整 Pod 副本数。**

例如：

Deployment：

```
Replica：

2
```

CPU：

一直：

95%。

HPA：

发现：

超过：

目标。

自动：

改：

```
Replica：

4
```

如果：

还是：

95%。

继续：

```
Replica：

8
```

流量：

下降。

CPU：

20%。

HPA：

又：

自动：

缩回：

```
Replica：

2
```

整个：

过程。

不需要：

人工。

------

# 第三节：HPA 的工作流程

这是：

整章：

最重要：

的一张图。

```
用户流量

      │

      ▼

Pod CPU

      │

      ▼

Metrics Server

      │

      ▼

HPA Controller

      │

      ▼

Deployment

replicas++

      │

      ▼

新的 Pod
```

是不是：

其实：

很简单？

------

# 第四节：Metrics Server 是什么？

很多新人：

创建：

HPA。

结果：

一直：

报错：

```
metrics not available
```

为什么？

因为：

Kubernetes：

默认：

不知道：

CPU：

是多少。

它：

只知道：

Pod：

Running。

不知道：

用了：

多少：

CPU。

于是：

需要：

安装：

Metrics Server。

一句话：

> **Metrics Server 负责收集 Pod 和 Node 的资源使用情况。**

例如：

它：

每隔：

几十秒：

收集：

```
CPU

Memory
```

然后：

提供：

给：

HPA。

------

# 第五节：HPA 为什么依赖 Requests？

这里：

非常：

重要。

假设：

Pod：

CPU：

实际：

用了：

```
500m
```

Requests：

配置：

```
cpu: 500m
```

CPU：

利用率：

就是：

```
100%
```

HPA：

认为：

很忙。

开始：

扩容。

------

但是。

如果：

Requests：

写：

```
2 Core
```

实际：

还是：

```
500m
```

CPU：

利用率：

只有：

```
25%
```

HPA：

认为：

一点：

都：

不忙。

不会：

扩容。

所以：

**HPA 判断的是相对于 Requests 的利用率，而不是节点 CPU 的绝对百分比。**

这就是为什么上一章说：

> Requests 配不好。

HPA：

一定：

不准。

------

# 第六节：一个完整例子

假设：

Deployment：

```
Replica：

2
```

HPA：

目标：

```
CPU：

70%
```

现在：

两个：

Pod。

CPU：

都是：

```
90%
```

平均：

```
90%
```

超过：

70%。

于是：

HPA：

自动：

变成：

```
Replica：

3
```

新的：

三个：

Pod。

CPU：

平均：

变成：

```
60%
```

达到：

目标。

停止：

扩容。

------

# 第七节：HPA 能缩容吗？

当然。

例如：

晚上：

12 点。

用户：

都：

睡觉。

CPU：

```
5%
```

HPA：

发现：

远低于：

目标。

于是：

```
8 Pod

↓

6

↓

4

↓

2
```

自动：

缩回。

所以：

HPA：

不仅：

扩容。

也：

缩容。

------

# 第八节：HPA 可以根据什么扩容？

很多新人：

以为：

只有：

CPU。

其实：

不是。

最常见：

有：

下面：

几种。

------

## CPU

最常见。

例如：

```
70%
```

------

## Memory

例如：

```
80%
```

如果：

内存：

持续：

很高。

也：

可以：

扩容。

不过对于 ASP.NET Core 等应用，内存通常不会像 CPU 一样随着请求量线性变化，因此很多团队更倾向于用 CPU 或业务指标。

------

## 自定义指标

例如：

```
QPS

HTTP 请求数

RabbitMQ 队列长度

Kafka Lag

Redis Queue

Prometheus 指标
```

例如：

RabbitMQ：

消息：

积压：

10000 条。

HPA：

自动：

增加：

消费者。

这是：

生产：

非常：

常见：

方案。

------

# 第九节：为什么 HPA 不直接看流量？

有人：

会：

问：

为什么：

不用：

```
1000 QPS
```

直接：

扩容？

因为：

不同：

程序。

1000：

QPS。

意义：

不同。

例如：

查询：

接口。

1000：

QPS。

CPU：

20%。

没事。

------

图片：

处理。

100：

QPS。

CPU：

100%。

已经：

爆了。

所以：

CPU：

通常：

更：

通用。

当然。

真正：

大型：

互联网。

一般：

都会：

使用：

Prometheus：

自定义：

指标。

------

# 第十节：HPA 会无限扩容吗？

不会。

例如：

可以：

设置：

```
最少：

2

最多：

20
```

所以：

即使：

CPU：

一直：

100%。

最多：

也是：

20。

例如：

```
minReplicas: 2

maxReplicas: 20
```

------

# 第十一节：如果 Node 没资源怎么办？

假设：

HPA：

已经：

决定：

```
20 个 Pod
```

但是。

Node：

已经：

满了。

新的：

Pod：

全部：

```
Pending
```

怎么办？

这里：

出现：

另一个：

概念。

> **Cluster Autoscaler**

它：

不是：

增加：

Pod。

而是：

增加：

Node。

流程：

变成：

```
HPA

↓

Pod++

↓

Node 不够

↓

Cluster Autoscaler

↓

Node++
```

很多云厂商（如 GKE、EKS、AKS）都提供 Cluster Autoscaler 或类似能力。

------

# 第十二节：HPA 与 Deployment 的关系

注意：

HPA：

不能：

直接：

管理：

Pod。

它：

修改：

Deployment：

里面：

的：

```
replicas
```

于是：

Deployment：

再：

创建：

Pod。

所以：

关系：

如下：

```
HPA

↓

Deployment

↓

Replica++

↓

ReplicaSet

↓

Pod
```

------

# 第十三节：ASP.NET Core 最佳实践

对于：

ASP.NET Core API。

建议：

目标：

CPU：

```
60~70%
```

原因：

留：

一点：

余量。

避免：

突然：

流量：

上涨。

另外：

建议：

至少：

```
minReplicas：

2
```

为什么？

如果：

只有：

一个：

Pod。

升级：

或者：

Node：

异常。

服务：

容易：

中断。

两个：

Pod。

才：

真正：

高可用。

------

# 第十四节：HPA 的局限性

HPA：

不是：

万能。

例如：

数据库。

MySQL。

不能：

因为：

CPU：

高。

自动：

变成：

10 个。

所以：

HPA：

主要：

适用于：

```
Web API

Worker

Consumer

无状态服务
```

数据库：

通常：

依赖：

StatefulSet：

和：

数据库：

自己的：

扩展：

机制。

------

# 本章总结（建议牢记）

请记住 HPA 最重要的几句话：

1. **HPA（Horizontal Pod Autoscaler）通过增加或减少 Pod 数量来应对负载变化。**
2. **HPA 通常依赖 Metrics Server 提供 CPU、内存等资源指标。**
3. **HPA 的 CPU 利用率是相对于 Requests 计算的，因此 Requests 配置非常重要。**
4. **HPA 可以扩容，也可以缩容，但会受到 `minReplicas` 和 `maxReplicas` 的限制。**
5. **如果 Pod 能扩但节点资源不足，需要配合 Cluster Autoscaler 扩容节点。**

------

# 到这里，你已经理解了 Kubernetes 的弹性能力

```
用户流量增加
        │
        ▼
CPU 使用率上升
        │
        ▼
Metrics Server
        │
        ▼
HPA
        │
修改 Deployment 副本数
        ▼
Deployment
        │
创建更多 Pod
        ▼
Service 自动分流
        ▼
系统恢复稳定
```

可以看到，**Deployment、Service、Resources、HPA** 共同组成了 Kubernetes 的自动扩缩容体系。

------

# 下一章预告：Helm——为什么生产环境几乎没有人手写 Kubernetes YAML？

到目前为止，我们一直在学习各种 YAML：

- Deployment.yaml
- Service.yaml
- Ingress.yaml
- ConfigMap.yaml
- Secret.yaml
- HPA.yaml

如果一个项目有：

```
20 个微服务
```

每个：

6 个：

YAML。

就是：

```
120 个 YAML 文件
```

再加上：

开发环境。

测试环境。

生产环境。

不同配置。

YAML 数量会迅速增长，维护成本非常高。

下一章，我们将学习 Kubernetes 的**包管理器——Helm**，包括：

- Helm 是什么？
- 为什么 Helm 被称为 Kubernetes 的 apt/yum？
- Chart 是什么？
- values.yaml 的作用
- Template 模板如何工作
- 如何部署 MySQL、Redis、Prometheus 等官方 Helm Chart
- 如何为 ASP.NET Core 项目制作自己的 Helm Chart

这一章结束后，你就会进入**生产环境最常见的 Kubernetes 应用交付方式**。

# 第十四章：Helm —— Kubernetes 的包管理器与模板引擎

提示词：

```
开始下一章
```

正文：

很好。

到这里，我们已经学习了 Kubernetes 最核心的对象。

如果你回顾一下，会发现我们已经写了很多 YAML。

例如：

```
deployment.yaml

service.yaml

ingress.yaml

configmap.yaml

secret.yaml

hpa.yaml

pvc.yaml
```

假设。

现在你的项目有：

```
订单系统

支付系统

用户系统

库存系统

消息系统
```

一共：

5 个微服务。

每个服务：

都有：

```
Deployment

Service

Ingress

ConfigMap

Secret

HPA
```

那么：

就是：

```
5 × 6

=

30 个 YAML
```

如果：

还有：

```
开发环境

测试环境

生产环境
```

马上：

变成：

```
90 个 YAML
```

如果：

修改：

镜像版本：

例如：

```
v1.0.0

↓

v1.0.1
```

你：

可能：

需要：

修改：

几十个：

YAML。

是不是：

非常：

痛苦？

于是。

Helm：

诞生了。

它解决的问题可以总结为一句话：

> **不要复制几十份 YAML，而是写一份模板，再通过配置生成不同环境的 YAML。**

## 本章学习目标

学习完本章，你应该能够回答：

- Helm 是什么？
- 为什么 Helm 被称为 Kubernetes 的 apt/yum？
- Chart 是什么？
- values.yaml 是什么？
- Helm Template 是什么？
- Release 是什么？
- Helm 如何安装、升级、回滚应用？
- 如何为 ASP.NET Core 项目制作 Helm Chart？

------

# 第一节：为什么 Kubernetes 需要 Helm？

先回忆一下 Docker。

安装 Nginx。

我们：

执行：

```
docker run nginx
```

是不是：

很简单？

为什么？

因为：

Docker Hub：

已经：

有：

官方：

镜像。

------

Kubernetes：

呢？

如果：

部署：

MySQL。

是不是：

要：

写：

```
Deployment

Service

PVC

Secret

ConfigMap
```

甚至：

几十页：

YAML。

是不是：

很麻烦？

于是：

Helm：

出现。

一句话：

> **Helm 帮助我们管理 Kubernetes 应用。**

很多人把 Helm 比作 Linux 的包管理器，这个比喻很贴切：

| Linux             | Kubernetes           |
| ----------------- | -------------------- |
| apt install nginx | helm install nginx   |
| yum install mysql | helm install mysql   |
| apt remove nginx  | helm uninstall nginx |

所以：

Helm：

就是：

Kubernetes：

的：

包管理器。

------

# 第二节：Helm 到底是什么？

一句话：

> **Helm = 模板 + 配置 + 生命周期管理。**

它：

做：

三件：

事情。

```
① 模板化 YAML

② 安装应用

③ 升级、回滚、卸载
```

很多新人只记住了"安装软件"。

其实：

真正：

强大的：

是：

模板。

------

# 第三节：Chart 是什么？

Helm：

里面：

最重要：

概念。

就是：

Chart。

一句话：

> **Chart 就是一份 Kubernetes 应用模板。**

例如：

MySQL。

官方：

已经：

写好：

所有：

YAML。

打包：

以后。

就是：

一个：

Chart。

里面：

可能：

包含：

```
Deployment

Service

PVC

Secret

ConfigMap

HPA

Ingress
```

用户：

不用：

自己：

写。

------

## 一个生活中的例子

想象：

装修。

如果：

没有：

设计图。

每次：

装修：

都要：

重新：

画。

Helm Chart：

就是：

设计图。

房子：

不同。

颜色：

不同。

家具：

不同。

但是：

设计：

模板：

一样。

------

# 第四节：Helm 的几个核心概念

这是整个 Helm 最重要的一张表。

| 名称        | 可以理解为         |
| ----------- | ------------------ |
| Helm        | 包管理器           |
| Chart       | 软件安装包（模板） |
| values.yaml | 安装参数           |
| Release     | 一次安装后的实例   |
| Repository  | Chart 仓库         |

这几个词，以后每天都会看到。

一定要理解。

------

# 第五节：Repository（仓库）

还记得：

Docker：

有：

Docker Hub。

Helm：

也：

一样。

例如：

官方：

社区：

提供：

很多：

Chart。

例如：

```
MySQL

PostgreSQL

Redis

RabbitMQ

Prometheus

Grafana

Nginx
```

都可以：

直接：

安装。

很多公司也会维护自己的 Helm Repository，存放内部应用的 Chart。

------

# 第六节：values.yaml 是什么？

这是 Helm 最重要：

也是：

最容易理解：

的文件。

例如：

你：

部署：

ASP.NET Core。

Deployment：

里面：

写：

```
replicas: 2

image:

  repository: my-api

  tag: v1
```

以后：

生产：

变成：

```
replicas=5
```

是不是：

要：

改：

YAML？

Helm：

不用。

它：

把：

所有：

可变：

配置。

放：

values.yaml。

例如：

```
replicaCount: 2

image:

  repository: my-api

  tag: v1.0.0

service:

  type: ClusterIP

  port: 80
```

以后：

生产：

只需要：

改：

```
replicaCount: 5
```

不用：

修改：

模板。

------

# 第七节：Template（模板）

真正：

神奇：

地方。

例如：

Deployment：

原来：

写：

```
replicas: 2
```

Helm：

改成：

```
replicas: {{ .Values.replicaCount }}
```

什么意思？

运行：

Helm。

读取：

```
replicaCount: 3
```

最终：

生成：

```
replicas: 3
```

所以：

**Helm 本身并不会运行 Deployment，它只是先把模板渲染成普通 YAML，然后再交给 Kubernetes。**

------

# 第八节：一个完整流程

假设：

values.yaml

```
replicaCount: 3

image:

  tag: v2
```

Deployment：

模板：

```
replicas: {{ .Values.replicaCount }}

image:

  myapi:{{ .Values.image.tag }}
```

Helm：

渲染：

结果：

```
replicas: 3

image:

  myapi:v2
```

最后：

发送：

给：

API Server。

整个：

流程：

如下：

```
Chart

↓

Template

↓

Values

↓

生成 YAML

↓

kubectl Apply

↓

Deployment
```

------

# 第九节：Release 是什么？

很多新人：

最容易：

混淆。

Chart：

只是：

模板。

例如：

```
MySQL Chart
```

你：

可以：

安装：

三次。

例如：

```
mysql-dev

mysql-test

mysql-prod
```

这：

三个：

安装：

实例。

就是：

三个：

Release。

所以：

一句话：

> **Chart 是模板，Release 是安装后的实例。**

------

# 第十节：Helm 最常用命令

这里只讲最常用的几个。

安装：

```
helm install my-api .
```

意思：

```
安装

Release：

my-api
```

升级：

```
helm upgrade my-api .
```

删除：

```
helm uninstall my-api
```

查看：

```
helm list
```

查看：

历史：

```
helm history my-api
```

------

# 第十一节：为什么 Helm 可以回滚？

假设：

今天：

升级：

```
v1

↓

v2
```

结果：

线上：

Bug。

怎么办？

Helm：

保存：

历史。

例如：

```
Revision1

v1

────────

Revision2

v2
```

执行：

```
helm rollback my-api 1
```

马上：

恢复：

v1。

这也是 Helm 在生产环境中非常受欢迎的原因之一。

------

# 第十二节：ASP.NET Core 项目如何使用 Helm？

一个典型的 Helm Chart 目录结构如下：

```
my-api/

├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── secret.yaml
    └── hpa.yaml
```

其中：

**Chart.yaml**

保存：

Chart：

基本：

信息。

例如：

```
名称

版本

描述
```

------

**templates**

保存：

所有：

模板。

例如：

Deployment：

里面：

大量：

使用：

```
{{ .Values.xxx }}
```

------

**values.yaml**

保存：

所有：

环境：

配置。

例如：

开发：

```
replicaCount: 1
```

生产：

```
replicaCount: 5
```

这样：

一套：

模板。

多个：

环境。

------

# 第十三节：为什么生产环境都喜欢 Helm？

假设：

没有：

Helm。

开发：

测试：

生产。

三套：

Deployment。

例如：

```
deployment-dev.yaml

deployment-test.yaml

deployment-prod.yaml
```

以后：

修改：

镜像。

三份：

一起：

改。

容易：

漏。

容易：

出错。

------

Helm：

只有：

一份：

模板。

例如：

```
deployment.yaml
```

不同：

环境：

只是：

不同：

values。

例如：

```
values-dev.yaml

values-test.yaml

values-prod.yaml
```

升级：

模板：

一次。

所有：

环境：

共享。

------

# 第十四节：Helm 的优点与局限

## 优点

- 减少重复 YAML
- 支持模板化配置
- 安装、升级、回滚方便
- 社区拥有大量现成 Chart
- 适合多环境部署

## 局限

Helm 并不是 Kubernetes 的一部分，而是一个额外工具。

另外，Helm 使用的是 **Go Template** 模板语言。对于复杂模板，可读性可能会下降，因此建议模板保持简单，把复杂逻辑放在应用或 CI/CD 中，而不是模板里。

------

# 第十五节：Helm 与 kubectl 的关系

很多新人会问：

> 有了 Helm，还需要 kubectl 吗？

答案：

**需要。**

两者分工不同。

| 工具    | 职责                                           |
| ------- | ---------------------------------------------- |
| kubectl | 与 Kubernetes 集群交互（查看、调试、管理资源） |
| Helm    | 管理应用的安装、升级、回滚和模板渲染           |

可以理解为：

- **kubectl** 是管理 Kubernetes 集群的工具。
- **Helm** 是管理 Kubernetes 应用的工具。

生产环境通常两者都会使用。

------

# 本章总结（建议牢记）

请记住 Helm 最重要的六句话：

1. **Helm 是 Kubernetes 最常用的包管理器。**
2. **Chart 是应用模板，Release 是安装后的实例。**
3. **values.yaml 保存环境配置，Template 保存通用模板。**
4. **Helm 会先渲染模板，再把生成的 YAML 提交给 Kubernetes。**
5. **Helm 支持安装、升级、回滚和卸载，是生产环境主流的应用交付方式。**
6. **Helm 不会取代 kubectl，两者是互补关系。**

------

# 到这里，你已经掌握了 Kubernetes 应用交付的主流方式

```
Helm Chart
      │
      ▼
values.yaml（环境配置）
      │
      ▼
Template（模板）
      │
渲染
      ▼
普通 Kubernetes YAML
      │
提交
      ▼
API Server
      │
创建 Deployment / Service / Ingress ...
```

这也是绝大多数公司部署业务系统的标准流程。

------

# 📚 第一阶段学习完成（Kubernetes 核心基础）

恭喜你，到这里已经完成了 Kubernetes 入门到应用部署的核心知识。

你已经掌握了：

- Kubernetes 架构（Control Plane、Node）
- Pod、Deployment、ReplicaSet
- Service、Ingress
- ConfigMap、Secret
- Volume、PV、PVC、StorageClass
- StatefulSet
- Namespace、Label、Selector
- Probe（健康检查）
- Resources（Requests、Limits）
- HPA（自动扩缩容）
- Helm（应用交付）

这些内容已经足以理解和部署绝大多数 Kubernetes 应用。

------

# 第二阶段预告：走向生产环境运维

接下来，我们建议进入真正的生产运维主题，这部分更偏向 Kubernetes 运维工程师和平台工程师的日常工作，包括：

1. **Helm 实战**：从零编写一个 ASP.NET Core + Vben Admin 的 Helm Chart，并支持 Dev/Test/Prod 多环境部署。
2. **日志体系**：理解 `kubectl logs`、容器日志、EFK/Loki 等集中式日志方案。
3. **监控体系**：Prometheus、Grafana、Alertmanager 的工作原理与部署。
4. **网络体系**：深入学习 CNI、NetworkPolicy、Service Mesh（如 Istio）的作用。
5. **CI/CD**：结合 GitHub Actions、GitLab CI、Jenkins 或 Azure DevOps，实现从代码提交到 Kubernetes 自动发布。
6. **生产故障排查**：掌握 `kubectl describe`、`kubectl events`、`kubectl exec`、`kubectl debug` 等常见排障手段。
7. **安全体系**：RBAC、ServiceAccount、Pod Security、镜像安全、Secret 管理。
8. **生产最佳实践**：高可用集群、备份恢复、滚动发布、金丝雀发布、蓝绿发布等。

> 如果你的目标是能够**独立负责 ASP.NET Core 项目在 Kubernetes 上的生产部署与运维**，那么第二阶段将是从"会用 Kubernetes"迈向"能够维护生产集群"的关键一步。

# 第二阶段 第一章：Helm 实战（一）—— 从零开始编写 Helm Chart

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入真正的**生产环境实战**。

如果说前面的内容都是在学习 Kubernetes 的"语法"。

那么从现在开始，我们学习的是：

> **企业到底是怎么部署项目的。**

很多新手学习 Kubernetes 时，会有一种误区：

> **学完 Deployment、Service、Ingress 就可以上线了。**

实际上。

真正的企业部署流程，几乎都是：

```
Git
   │
   ▼
CI（GitHub Actions / GitLab CI / Azure DevOps）
   │
   ▼
Docker Image
   │
   ▼
Helm Chart
   │
   ▼
Dev
   │
   ▼
Test
   │
   ▼
Staging
   │
   ▼
Production
```

这里面最重要的一环。

就是：

> **Helm Chart**

上一章我们已经知道 Helm 是什么。

这一章。

我们要亲手写一个真正可以部署的 Helm Chart。

而且。

不是 Hello World。

而是：

> **ASP.NET Core Web API + Vben Admin(Vue3) + Redis**

这也是很多公司最典型的一套架构。

## 本章学习目标

学完本章，你应该能够回答：

- 一个 Helm Chart 到底长什么样？
- Chart.yaml 是干什么的？
- values.yaml 为什么这么重要？
- templates 目录里面应该放什么？
- 一个企业级 Helm Chart 应该如何组织？
- 如何做到一套模板，多套环境？

------

# 第一节：为什么企业几乎都用 Helm？

假设。

你有一个项目。

```
Order.Api
```

开发环境：

镜像：

```
order-api:v1.0.0-dev
```

副本：

```
1
```

数据库：

```
Dev SQL Server
```

------

测试环境：

镜像：

```
order-api:v1.0.0-test
```

副本：

```
2
```

数据库：

```
Test SQL Server
```

------

生产环境：

镜像：

```
order-api:v1.0.0
```

副本：

```
6
```

数据库：

```
Production SQL Server
```

如果不用 Helm。

你可能会有：

```
deployment-dev.yaml

deployment-test.yaml

deployment-prod.yaml
```

Service：

又三份。

Ingress：

又三份。

HPA：

又三份。

很快。

整个仓库：

变成：

```
deployment-dev.yaml
deployment-test.yaml
deployment-prod.yaml

service-dev.yaml
service-test.yaml
service-prod.yaml

ingress-dev.yaml
ingress-test.yaml
ingress-prod.yaml

...
```

修改一次端口。

可能要改九个文件。

这就是 Helm 要解决的问题。

------

# 第二节：Helm Chart 的目录结构

我们先不要急着写模板。

先认识目录。

假设：

我们的项目叫：

```
order-api
```

执行：

```
helm create order-api
```

Helm 会自动生成：

```
order-api/

├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── hpa.yaml
├── serviceaccount.yaml
├── NOTES.txt
│
└── _helpers.tpl
```

第一次看到。

很多人都会懵。

其实。

真正重要的只有几个文件。

------

# 第三节：Chart.yaml

这是 Helm Chart 的身份证。

例如：

```
apiVersion: v2

name: order-api

description: ASP.NET Core Order API

type: application

version: 1.0.0

appVersion: "1.0.0"
```

下面解释每个字段。

------

## apiVersion

不是 Kubernetes API。

而是：

Helm Chart 格式。

目前一般都是：

```
apiVersion: v2
```

基本不用改。

------

## name

Chart 名称。

例如：

```
name: order-api
```

以后：

安装：

就是：

```
order-api
```

------

## description

项目描述。

例如：

```
description: Order API
```

只是说明文字。

不会影响部署。

------

## version

注意。

很多新人：

都会搞错。

这里：

不是：

应用版本。

而是：

> **Chart 自己的版本。**

例如：

模板：

修改了一次。

Chart：

版本：

可以：

```
1.0.0

↓

1.0.1
```

即使：

你的程序：

还是：

```
v8.0
```

Chart：

也可以：

升级。

------

## appVersion

真正：

程序：

版本。

例如：

```
appVersion: "2.1.5"
```

它一般对应：

Docker Image：

```
order-api:2.1.5
```

注意：

很多团队会把镜像 Tag 放在 `values.yaml` 中，而 `appVersion` 主要用于描述和展示，不一定直接参与部署。

------

# 第四节：values.yaml

这是：

整个 Helm：

最重要：

的文件。

可以理解成：

> **所有可以修改的配置，都放这里。**

例如：

```
replicaCount: 2
```

部署：

模板：

不用：

写：

```
replicas: 2
```

而是：

写：

```
replicas: {{ .Values.replicaCount }}
```

以后。

开发：

```
replicaCount: 1
```

生产：

```
replicaCount: 6
```

模板：

完全：

不用：

修改。

------

# 第五节：一个企业 values.yaml 应该长什么样？

下面是一份比较典型的结构：

```
replicaCount: 2

image:
  repository: mycompany/order-api
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: api.example.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

是不是：

几乎：

所有：

可变：

内容。

都：

集中：

在这里。

------

# 第六节：为什么 values.yaml 要这样设计？

假设。

镜像：

变成：

```
1.0.1
```

以前。

改：

Deployment。

现在。

只需要：

```
image:

  tag: "1.0.1"
```

即可。

例如：

CI/CD：

最后一步：

甚至：

直接：

修改：

这个：

Tag。

然后：

执行：

```
helm upgrade
```

完成：

部署。

很多公司的自动发布流水线就是这样工作的。

------

# 第七节：templates 目录

这里。

是真正：

放 Kubernetes YAML 模板的地方。

例如：

```
templates/

deployment.yaml

service.yaml

ingress.yaml

hpa.yaml
```

这些：

其实：

就是：

普通：

Kubernetes YAML。

唯一：

区别：

就是：

里面：

可以：

使用：

模板变量。

例如：

普通：

Deployment：

```
replicas: 2
```

Helm：

Deployment：

```
replicas: {{ .Values.replicaCount }}
```

是不是：

只有：

这一点：

不同？

------

# 第八节：_helpers.tpl 是什么？

这是很多新人最害怕的文件。

其实。

它不是 Kubernetes。

而是 Helm 模板里的**公共函数**。

例如。

多个文件都需要生成统一的资源名称：

```
order-api

order-api-service

order-api-config
```

如果：

每个文件：

都自己拼。

以后：

改名字。

全部：

都要：

修改。

于是：

Helm：

允许：

把：

公共：

逻辑。

放：

这里。

例如：

（这里只理解概念，不要求记模板语法。）

```
生成统一名称

生成统一 Label

生成统一 Selector
```

以后。

Deployment。

Service。

Ingress。

全部：

调用：

同一个：

公共模板。

这和你熟悉的 C# 方法、公共类有点类似：**避免重复代码**。

------

# 第九节：企业 Helm Chart 推荐目录

随着项目变大，通常会增加更多模板：

```
templates/

deployment.yaml

service.yaml

ingress.yaml

configmap.yaml

secret.yaml

hpa.yaml

pdb.yaml

networkpolicy.yaml

serviceaccount.yaml
```

这里先简单介绍几个新文件：

- **PodDisruptionBudget（PDB）**：保证节点维护、升级时，不会一下子把所有 Pod 都驱逐掉。
- **NetworkPolicy**：限制哪些 Pod 可以访问哪些 Pod，实现网络隔离。
- **ServiceAccount**：Pod 在 Kubernetes 中使用的身份，后续学习 RBAC 时会详细介绍。

这些都是生产环境非常常见的资源。

------

# 第十节：一套模板，多套环境

这是 Helm 最大的价值。

例如：

目录：

```
values.yaml

values-dev.yaml

values-test.yaml

values-prod.yaml
```

开发：

```
replicaCount: 1
```

测试：

```
replicaCount: 2
```

生产：

```
replicaCount: 8
```

部署：

开发：

```
helm upgrade --install order-api . -f values-dev.yaml
```

部署：

测试：

```
helm upgrade --install order-api . -f values-test.yaml
```

部署：

生产：

```
helm upgrade --install order-api . -f values-prod.yaml
```

**模板完全一样，只是配置不同。**

这就是 Helm 在企业里最常见的使用方式。

------

# 本章总结（建议牢记）

请记住 Helm Chart 最重要的几点：

1. **Chart.yaml 是 Chart 的元信息，不是业务配置。**
2. **values.yaml 存放所有可变配置，是不同环境差异的核心。**
3. **templates 目录存放 Kubernetes YAML 模板。**
4. **模板通过 `{{ .Values.xxx }}` 读取配置。**
5. **推荐采用"一套模板 + 多个 values 文件"管理开发、测试和生产环境。**
6. **随着项目成熟，可以逐步加入 HPA、PDB、NetworkPolicy 等生产资源。**

------

# 下一章预告：Helm 实战（二）——编写第一个 Deployment 模板

下一章，我们将真正开始写 Helm 模板，而不是只讲概念。

我们会一步一步完成一个可以直接部署的 **ASP.NET Core Web API Helm Chart**，包括：

- 将 Deployment 改造成 Helm 模板
- 使用 `{{ .Values.xxx }}` 替换固定值
- 引用镜像、端口、副本数等配置
- 编写统一的 Labels 和 Selectors
- 使用 `helm template` 查看最终渲染出来的 Kubernetes YAML
- 理解 Helm 渲染过程与 Kubernetes 部署过程的关系

从这一章开始，你会逐渐具备**独立编写企业级 Helm Chart** 的能力，而不仅仅是会使用别人写好的 Chart。

# 第二阶段 第二章：Helm 实战（二）—— 编写第一个 Deployment 模板

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们正式进入 **Helm 模板开发**。

如果说上一章只是认识 Helm。

这一章，我们要真正开始写模板。

而且不是简单地讲语法。

而是按照**企业真实项目**来写。

整个系列我们都会围绕下面这个项目：

```
Order.Api（ASP.NET Core Web API）
```

部署以后，最终会得到：

```
Deployment
        │
        ▼
Pod
        │
        ▼
Service
        │
        ▼
Ingress
```

不过。

所有固定配置都会变成：

```
{{ .Values.xxx }}
```

这就是 Helm。

## 本章学习目标

学习完本章，你应该能够回答：

- Helm Template 到底是怎么工作的？
- `{{ }}` 是什么意思？
- `.Values` 是什么？
- 如何把普通 Deployment 改造成 Helm 模板？
- 为什么企业很少直接写固定值？
- 什么是 `_helpers.tpl`？
- Labels 为什么要抽出来？

------

# 第一节：先回忆普通 Deployment

假设。

以前。

我们写：

Deployment：

```
apiVersion: apps/v1

kind: Deployment

metadata:
  name: order-api

spec:

  replicas: 2

  selector:
    matchLabels:
      app: order-api

  template:

    metadata:
      labels:
        app: order-api

    spec:

      containers:

      - name: order-api

        image: mycompany/order-api:v1.0.0

        ports:

        - containerPort: 8080
```

是不是：

全部：

都是：

固定值。

例如：

```
order-api

2

8080

v1.0.0
```

如果：

生产：

变成：

```
8

v2.0.0
```

全部：

要改。

于是。

Helm：

出现。

------

# 第二节：Helm Template 到底是什么？

一句话：

> **Template（模板）就是一份带变量的 Kubernetes YAML。**

例如。

以前：

写：

```
replicas: 2
```

Helm：

改成：

```
replicas: {{ .Values.replicaCount }}
```

什么意思？

假设：

values.yaml：

```
replicaCount: 3
```

Helm：

运行：

以后。

生成：

```
replicas: 3
```

然后。

再：

发送：

给：

API Server。

注意。

Kubernetes：

**完全不知道 Helm 的存在。**

它最终收到的依然是普通 YAML。

这一点非常重要。

------

# 第三节：理解 {{ }}

很多新人：

第一次：

看到：

```
{{ }}
```

都会：

害怕。

其实。

如果你是后端开发者。

可以把它理解成：

C#：

字符串模板。

例如：

C#：

```
var name = "Andy";

Console.WriteLine($"Hello {name}");
```

输出：

```
Hello Andy
```

Helm：

也是：

一样。

例如：

```
image:

  tag: {{ .Values.image.tag }}
```

假设：

```
image:

  tag: "2.0.0"
```

最终：

生成：

```
image:

  tag: 2.0.0
```

所以。

可以理解成：

> **{{ }} = 把变量放进 YAML。**

------

# 第四节：什么是 .Values？

这是 Helm 用得最多的对象。

```
.Values
```

表示：

> **读取 values.yaml。**

例如：

values.yaml：

```
replicaCount: 2

image:

  repository: mycompany/order-api

  tag: "1.0.0"

service:

  port: 8080
```

Deployment：

模板：

可以：

写：

```
replicas: {{ .Values.replicaCount }}
```

镜像：

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

端口：

```
containerPort: {{ .Values.service.port }}
```

最终：

Helm：

自动：

替换。

------

# 第五节：一步一步改造 Deployment

现在。

开始。

真正：

模板化。

原来：

```
replicas: 2
```

改成：

```
replicas: {{ .Values.replicaCount }}
```

------

原来：

```
image: mycompany/order-api:v1.0.0
```

改成：

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

------

原来：

```
containerPort: 8080
```

改成：

```
containerPort: {{ .Values.service.port }}
```

是不是。

其实：

没有：

那么：

复杂？

------

# 第六节：metadata.name 为什么也要模板化？

很多新人：

喜欢：

写：

```
metadata:

  name: order-api
```

其实。

企业：

一般：

不会。

而是：

写：

```
metadata:

  name: {{ include "order-api.fullname" . }}
```

第一次：

看到：

是不是：

懵？

不用：

急。

这里只需要理解：

它不是 Kubernetes。

而是：

Helm 的一个**公共函数调用**。

作用：

就是：

统一：

生成：

资源名称。

例如：

以后：

Release：

名字：

叫：

```
prod
```

生成：

可能：

就是：

```
prod-order-api
```

以后：

测试：

环境：

```
test-order-api
```

这样：

不同环境部署到同一个集群，也不会因为资源重名而冲突。

------

# 第七节：为什么 Label 不直接写？

以前：

写：

```
labels:

  app: order-api
```

企业：

通常：

写：

```
labels:

  {{- include "order-api.labels" . | nindent 4 }}
```

是不是：

更：

看不懂？

其实。

原因：

只有：

一个。

避免：

重复。

例如：

Deployment：

需要：

Labels。

Service：

需要：

Labels。

Pod：

需要：

Labels。

Ingress：

可能：

也：

需要。

如果：

全部：

自己：

写。

以后：

改：

Label。

改：

十几处。

于是。

统一：

抽：

出来。

------

# 第八节：_helpers.tpl 到底做什么？

例如：

里面：

可能：

有：

一个：

公共模板：

```
生成应用名称

生成 Labels

生成 Selector Labels
```

Deployment：

调用。

Service：

调用。

Ingress：

调用。

以后：

全部：

保持：

一致。

你可以把 `_helpers.tpl` 理解为：

```
C#

↓

Helper.cs
```

里面：

放：

公共：

方法。

而：

不是：

业务：

代码。

------

# 第九节：完整的 Deployment 模板（简化版）

下面是一份简化后的 Helm Deployment 模板。

请重点关注 **哪些地方变成了变量**。

```
apiVersion: apps/v1

kind: Deployment

metadata:

  name: {{ include "order-api.fullname" . }}

spec:

  replicas: {{ .Values.replicaCount }}

  selector:

    matchLabels:

      app: {{ include "order-api.name" . }}

  template:

    metadata:

      labels:

        app: {{ include "order-api.name" . }}

    spec:

      containers:

      - name: order-api

        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

        ports:

        - containerPort: {{ .Values.service.port }}
```

是不是。

除了：

变量。

剩下：

全部：

还是：

普通：

Deployment。

所以。

学习 Helm。

真正：

学习：

只有：

模板：

语法。

Kubernetes：

知识：

还是：

原来的。

------

# 第十节：Helm 如何渲染？

假设：

values.yaml：

```
replicaCount: 3

image:

  repository: mycompany/order-api

  tag: "2.0.0"

service:

  port: 8080
```

执行：

```
helm template order-api .
```

Helm：

输出：

```
replicas: 3

image:

  mycompany/order-api:2.0.0

containerPort: 8080
```

然后：

这些：

普通：

YAML。

才：

真正：

发送：

到：

Kubernetes。

因此：

一个非常重要的调试技巧就是：

> **先用 `helm template` 看生成的 YAML，再部署到集群。**

很多模板错误，在这一步就能发现，而不用等到 Kubernetes 创建失败。

------

# 第十一节：企业里的开发流程

真实项目中，通常不会直接修改线上 Chart，而是：

```
修改 values.yaml
        │
        ▼
helm template（检查渲染结果）
        │
        ▼
helm lint（检查 Chart 是否规范）
        │
        ▼
helm upgrade --install
        │
        ▼
Deployment 更新
        │
        ▼
Rolling Update
```

这里出现了一个新命令：

```
helm lint
```

作用：

就是：

检查：

Chart：

有没有：

明显：

错误。

它类似于：

```
C#

↓

编译器检查
```

虽然：

不能：

保证：

业务：

正确。

但是：

可以：

发现：

很多：

模板：

问题。

------

# 第十二节：模板化时容易踩的坑

很多初学者都会遇到下面这些问题：

### ① 忘记给字符串加引号

例如：

```
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

有时会因为特殊字符导致 YAML 解析问题。

更稳妥的写法是：

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

------

### ② values.yaml 和模板字段不一致

例如：

模板：

```
{{ .Values.images.tag }}
```

而：

values.yaml：

却是：

```
image:

  tag: "1.0.0"
```

由于：

```
images
```

不存在。

渲染：

结果：

就是：

错误。

因此：

保持字段命名一致非常重要。

------

### ③ 过度模板化

不是所有内容都必须放进 `values.yaml`。

例如：

```
apiVersion: apps/v1

kind: Deployment
```

通常：

都是：

固定：

值。

没有：

必要：

模板化。

经验原则：

> **只有会因环境或版本变化而改变的内容，才放进 values.yaml。**

------

# 本章总结（建议牢记）

请记住 Helm 模板最重要的几点：

1. **Helm Template 本质上还是 Kubernetes YAML，只是加入了模板变量。**
2. **`{{ }}` 表示模板表达式，用于把变量或函数结果插入 YAML。**
3. **`.Values` 用于读取 `values.yaml` 中的配置。**
4. **`include` 常用于调用 `_helpers.tpl` 中定义的公共模板，减少重复。**
5. **先使用 `helm template` 和 `helm lint`，再部署，是企业最佳实践。**
6. **不要为了模板化而模板化，把真正会变化的配置放到 `values.yaml` 即可。**

------

# 下一章预告：Helm 实战（三）——Service、Ingress、ConfigMap 模板化

下一章，我们会继续完成整个 Helm Chart，包括：

- 编写 `Service` 模板
- 编写 `Ingress` 模板
- 编写 `ConfigMap` 模板
- 使用 `Secret` 管理数据库连接字符串
- 使用 `if`、`with`、`range` 等 Helm 常用模板语法
- 支持开发、测试、生产三套配置共用一套模板

到那时，你将拥有一个**完整可用于 ASP.NET Core 项目**的 Helm Chart，而不仅仅是一个 Deployment 模板。

# 第二阶段 第三章：Helm 实战（三）—— Service、Ingress、ConfigMap 与 Helm 模板语法

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始。

我们要把整个 Helm Chart 写完整。

上一章。

我们已经完成了：

```
Deployment
```

但是。

一个真正的 ASP.NET Core 项目，仅仅有 Deployment 是远远不够的。

至少还需要：

```
Service
Ingress
ConfigMap
Secret
```

除此之外。

我们还要学习 Helm 最常用的模板语法：

```
if

with

range
```

因为几乎所有企业 Helm Chart 都会用到它们。

## 本章学习目标

学习完本章，你应该能够回答：

- 如何把 Service 改造成 Helm 模板？
- 如何让 Ingress 可以开启或关闭？
- ConfigMap 如何读取 values.yaml？
- Secret 为什么不能直接写进 values.yaml？
- Helm 的 if、with、range 分别有什么作用？
- 企业 Helm Chart 为什么大量使用条件判断？

------

# 第一节：继续完成我们的项目

目前。

我们的 Chart 已经有：

```
templates/

deployment.yaml
```

现在。

继续增加：

```
templates/

deployment.yaml

service.yaml

ingress.yaml

configmap.yaml

secret.yaml
```

以后。

执行：

```
helm install
```

这些资源。

会一起创建。

------

# 第二节：Service 模板化

先看普通 Service：

```
apiVersion: v1

kind: Service

metadata:

  name: order-api

spec:

  type: ClusterIP

  selector:

    app: order-api

  ports:

  - port: 80

    targetPort: 8080
```

里面。

哪些：

应该：

放到：

values？

一般：

有：

这些：

```
Service 类型

Port

TargetPort
```

于是。

改成：

```
spec:

  type: {{ .Values.service.type }}

  ports:

  - port: {{ .Values.service.port }}

    targetPort: {{ .Values.service.targetPort }}
```

对应：

values.yaml

```
service:

  type: ClusterIP

  port: 80

  targetPort: 8080
```

以后。

如果：

改成：

NodePort。

只需要：

```
service:

  type: NodePort
```

模板：

不用：

修改。

------

# 第三节：Ingress 为什么要支持开关？

不是：

所有：

环境：

都需要：

Ingress。

例如：

开发：

环境：

可能：

直接：

```
kubectl port-forward
```

或者：

```
NodePort
```

所以。

Ingress：

应该：

可以：

关闭。

Helm：

通常：

这样：

写：

```
ingress:

  enabled: true
```

然后。

模板：

写：

```
{{- if .Values.ingress.enabled }}

apiVersion: networking.k8s.io/v1

kind: Ingress

...

{{- end }}
```

什么意思？

如果：

```
enabled: true
```

生成：

Ingress。

如果：

```
enabled: false
```

整个：

Ingress.yaml：

不会：

生成。

------

# 一个生活例子

可以把：

```
if
```

理解成：

C#：

```
if(enable)
{
    创建Ingress();
}
```

是不是：

一模一样？

Helm：

只是：

换了一种：

写法。

------

# 第四节：Ingress 如何读取 Host？

例如：

以前：

写：

```
host: api.example.com
```

现在：

改成：

```
host: {{ .Values.ingress.host }}
```

values：

```
ingress:

  enabled: true

  host: api.example.com
```

开发：

环境：

```
host: api-dev.example.com
```

生产：

环境：

```
host: api.example.com
```

是不是：

完全：

不用：

修改：

模板？

------

# 第五节：ConfigMap 模板

假设。

ASP.NET Core：

有：

配置：

```
{
    "LogLevel":"Information",

    "CacheTime":300
}
```

以前：

可能：

写死。

现在。

可以：

放：

values：

```
config:

  logLevel: Information

  cacheTime: 300
```

ConfigMap：

模板：

```
apiVersion: v1

kind: ConfigMap

metadata:

  name: {{ include "order-api.fullname" . }}

data:

  LogLevel: "{{ .Values.config.logLevel }}"

  CacheTime: "{{ .Values.config.cacheTime }}"
```

是不是：

所有：

配置：

都：

来自：

values？

------

# 第六节：为什么 Secret 不建议直接放 values.yaml？

很多新人：

第一次：

都会：

这样：

写：

```
database:

  password: 123456
```

这是：

非常：

危险：

的。

为什么？

因为：

values.yaml：

一般：

都会：

进入：

Git。

于是。

密码：

上传：

GitHub。

生产：

事故。

------

企业：

一般：

有：

几种：

方案：

```
方案一：

CI/CD 注入

────────────

方案二：

External Secret

────────────

方案三：

云平台 Secret Manager

────────────

方案四：

独立 values-prod.yaml（不提交 Git）
```

如果只是学习，可以把 Secret 放在独立的 `values-local.yaml` 中，并确保它不会提交到代码仓库。

真正生产环境，更推荐使用专门的密钥管理方案。

------

# 第七节：Helm 的 with

这是：

第二个：

最常见：

语法。

例如：

以前：

一直：

写：

```
{{ .Values.image.repository }}

{{ .Values.image.tag }}

{{ .Values.image.pullPolicy }}
```

是不是：

很长？

可以：

写：

```
{{- with .Values.image }}

repository: {{ .repository }}

tag: {{ .tag }}

pullPolicy: {{ .pullPolicy }}

{{- end }}
```

这里：

```
with
```

相当于：

进入：

一个：

对象。

以后：

不用：

一直：

写：

```
.Values.image
```

了。

------

一个 C# 类比

例如：

原来：

```
Console.WriteLine(user.Name);

Console.WriteLine(user.Age);

Console.WriteLine(user.City);
```

如果：

进入：

对象。

是不是：

就：

不用：

一直：

写：

user？

Helm：

就是：

这个：

意思。

------

# 第八节：Helm 的 range

这是：

第三个：

最重要：

语法。

例如。

Ingress：

可能：

有：

多个：

Host。

values：

```
hosts:

- api.example.com

- admin.example.com

- test.example.com
```

模板：

可以：

写：

```
{{- range .Values.hosts }}

- host: {{ . }}

{{- end }}
```

生成：

```
- host: api.example.com

- host: admin.example.com

- host: test.example.com
```

所以。

> **range = 循环。**

是不是：

很像：

C#：

```
foreach(...)
{
}
```

------

# 第九节：企业 Chart 为什么大量使用 if？

例如：

开发：

环境：

不要：

HPA。

```
autoscaling:

  enabled: false
```

模板：

```
{{- if .Values.autoscaling.enabled }}

HPA

{{- end }}
```

生产：

环境：

```
enabled: true
```

自动：

生成：

HPA。

同样的思路还常用于：

- 是否创建 Ingress
- 是否创建 ServiceAccount
- 是否挂载额外 Volume
- 是否开启 PodDisruptionBudget（PDB）

这样一套模板可以适配多种部署场景。

------

# 第十节：values-dev、values-prod

企业：

最常见：

目录：

```
values.yaml

values-dev.yaml

values-test.yaml

values-prod.yaml
```

例如。

开发：

```
replicaCount: 1

ingress:

  enabled: false

autoscaling:

  enabled: false
```

生产：

```
replicaCount: 6

ingress:

  enabled: true

autoscaling:

  enabled: true
```

部署：

开发：

```
helm install order-api . -f values-dev.yaml
```

生产：

```
helm install order-api . -f values-prod.yaml
```

模板：

只有：

一份。

------

# 第十一节：真实企业项目通常还会有哪些 values？

很多公司都会把下面这些内容放进 values.yaml：

```
image:

resources:

nodeSelector:

tolerations:

affinity:

env:

volumeMounts:

volumes:

probes:

autoscaling:
```

为什么？

因为：

这些：

都是：

可能：

因环境而变化的配置。

例如：

测试环境：

不用：

HPA。

生产：

需要：

HPA。

GPU：

节点：

需要：

NodeSelector。

普通：

节点：

不用。

------

# 第十二节：一个完整的渲染流程

最终。

整个：

Helm：

流程：

其实：

只有：

四步。

```
values-prod.yaml

        │

        ▼

templates/

Deployment

Service

Ingress

ConfigMap

Secret

        │

        ▼

Helm Template

        │

        ▼

普通 YAML

        │

        ▼

Kubernetes API Server
```

请记住：

> **Helm 负责"生成"，Kubernetes 负责"运行"。**

两者职责非常明确。

------

# 第十三节：Helm 常见模板语法总结

| 语法                | 作用           | 类比                        |
| ------------------- | -------------- | --------------------------- |
| `{{ .Values.xxx }}` | 读取配置       | `config.Value`              |
| `if`                | 条件判断       | `if (...) {}`               |
| `with`              | 进入对象作用域 | `var image = config.Image;` |
| `range`             | 遍历列表       | `foreach`                   |
| `include`           | 调用公共模板   | 调用公共方法                |

如果你有 C# 开发经验，这些语法会很容易理解。

------

# 本章总结（建议牢记）

请记住 Helm 模板开发最重要的几点：

1. **Service、Ingress、ConfigMap 都应该尽量通过 `values.yaml` 配置可变参数。**
2. **`if` 用于按需创建资源，是企业 Helm Chart 中最常见的模板语法。**
3. **`with` 可以减少重复路径，提高模板可读性。**
4. **`range` 用于遍历列表，非常适合生成多个 Host、环境变量或端口。**
5. **不要把生产密码直接写进会提交到 Git 的 `values.yaml`。**
6. **推荐采用 `values-dev.yaml`、`values-test.yaml`、`values-prod.yaml` 管理不同环境。**

------

# 到这里，你已经能够编写一个完整的 Helm Chart

```
Chart.yaml
        │
        ▼
values-prod.yaml
        │
        ▼
templates/
    │
    ├── Deployment
    ├── Service
    ├── Ingress
    ├── ConfigMap
    └── Secret
        │
        ▼
helm template
        │
        ▼
Kubernetes YAML
        │
        ▼
API Server
```

这已经接近企业中 Helm Chart 的基本形态。

------

# 下一章预告：Helm 实战（四）——部署 ASP.NET Core + Vben Admin 完整项目

下一章，我们将把前几章学到的内容整合起来，完成一个接近生产环境的完整案例，包括：

- 一个 ASP.NET Core Web API
- 一个 Vben Admin（Vue3）前端
- Redis
- ConfigMap 与 Secret
- Ingress 路由配置
- HPA 自动扩缩容
- 多环境（Dev/Test/Prod）部署
- 从 `helm install` 到应用成功访问的完整流程

这一章结束后，你将拥有一个可以作为实际项目模板的 Helm Chart，也会理解企业是如何组织和交付 Kubernetes 应用的。

# 第二阶段 第四章：Helm 实战（四）——部署一个 ASP.NET Core + Vben Admin 完整项目

提示词：

```
开始下一章
```

正文：

非常好。

从这一章开始，我们不再单独学习某个 Kubernetes 对象，而是把前面所有知识串联起来。

这是整个教程中**第一个真正接近企业生产环境的完整案例**。

很多人在学习 Kubernetes 时有一个问题：

> 「我知道 Deployment、Service、Ingress、ConfigMap、Secret、HPA，但不知道它们怎么组合起来。」

这一章，就是回答这个问题。

> **本章是一个完整案例。建议反复阅读，并画出自己的架构图。**

------

# 本章学习目标

学完本章，你应该能够回答：

- 一个完整的 Kubernetes 项目包含哪些资源？
- 前端、后端、Redis 是如何协作的？
- 为什么前后端通常会拆成两个 Deployment？
- ConfigMap 和 Secret 应该如何组织？
- Helm 如何统一管理整个项目？
- 一次部署到底创建了哪些 Kubernetes 对象？

------

# 第一节：先认识整个系统

假设，我们要部署一个企业常见的系统：

```
Vben Admin（Vue3）
        │
        ▼
ASP.NET Core API
        │
        ▼
Redis
```

对应到 Kubernetes 中，大致会变成：

```
                Internet
                    │
                    ▼
              Ingress Controller
                    │
      ┌─────────────┴─────────────┐
      ▼                           ▼
 Frontend Service            Backend Service
      │                           │
      ▼                           ▼
Vue3 Deployment          ASP.NET Core Deployment
                                  │
                                  ▼
                            Redis Service
                                  │
                                  ▼
                           Redis StatefulSet
```

> **这是企业最常见的三层结构之一。**

------

# 第二节：为什么前后端要分开部署？

很多新人第一次接触 Kubernetes，会问：

> 为什么不把 Vue 和 ASP.NET Core 放进一个 Pod？

答案是：**职责不同，生命周期也不同。**

例如：

前端：

```
v1.0.0
```

后端：

```
v1.0.3
```

今天：

后端修复了一个 Bug。

需要升级。

如果：

放在一个 Pod。

意味着：

```
Vue
+
API
```

一起：

重新部署。

实际上：

前端根本没有变化。

因此：

企业一般采用：

```
Frontend Deployment

Backend Deployment
```

分别管理。

这样：

两边可以独立扩容、独立升级、独立回滚。

------

# 第三节：整个项目会有哪些 Kubernetes 资源？

一个典型项目可能包含：

| 资源                   | 数量 | 作用                   |
| ---------------------- | ---- | ---------------------- |
| Namespace              | 1    | 隔离整个项目           |
| Deployment（Frontend） | 1    | Vue 前端               |
| Deployment（Backend）  | 1    | ASP.NET Core API       |
| StatefulSet（Redis）   | 1    | Redis 服务             |
| Service                | 3    | 对外或内部访问         |
| Ingress                | 1    | HTTP 路由              |
| ConfigMap              | 2    | 非敏感配置             |
| Secret                 | 2    | 数据库密码、JWT 密钥等 |
| HPA                    | 2    | 前后端自动扩缩容       |
| PVC                    | 1    | Redis 持久化           |

所以：

一次：

```
helm install
```

实际上：

可能：

创建：

十几个：

Kubernetes 资源。

------

# 第四节：为什么 Redis 用 StatefulSet？

回忆我们前面学过的内容。

Redis：

需要：

```
数据
```

不能：

随便：

删除。

因此：

不能：

用：

```
Deployment
```

而是：

```
StatefulSet
```

因为：

Redis：

需要：

```
固定名称

固定存储

PVC
```

如果只是开发环境，也可以使用 Deployment 部署 Redis，但生产环境通常推荐 StatefulSet。

------

# 第五节：ASP.NET Core Deployment

它主要负责：

```
接收 HTTP 请求

执行业务逻辑

访问数据库

访问 Redis

返回 JSON
```

通常包含：

```
Deployment
        │
        ▼
Pod
        │
        ▼
Container
```

Container 内部运行：

```
dotnet Order.Api.dll
```

镜像：

例如：

```
mycompany/order-api:1.0.0
```

------

# 第六节：Vue Deployment

Vue 项目通常已经编译完成。

镜像里面一般运行的是：

```
Nginx
```

里面放着：

```
dist/
```

例如：

```
/usr/share/nginx/html
```

所以：

Vue Deployment：

实际上：

不是运行 Node.js。

而是：

运行：

```
Nginx
```

提供：

静态文件。

这是绝大多数 Vue、React、Angular 项目的生产部署方式。

------

# 第七节：ConfigMap 应该放什么？

例如：

ASP.NET Core：

```
Logging

CacheTime

FeatureFlags

ApiUrl
```

Vue：

```
API_BASE_URL

APP_NAME

Theme
```

这些：

都：

适合：

ConfigMap。

因为：

它们：

不是：

秘密。

------

# 第八节：Secret 应该放什么？

企业里面：

一般：

放：

```
SQL Password

Redis Password

JWT Secret

OAuth Client Secret

SMTP Password

第三方 API Key
```

一句话：

> **任何泄露后可能造成安全问题的内容，都应该放入 Secret 或更专业的密钥管理系统。**

------

# 第九节：Ingress 如何路由？

假设：

域名：

```
app.company.com
```

访问：

首页：

```
/
```

进入：

Vue。

API：

```
/api
```

进入：

ASP.NET Core。

整个：

路由：

如下：

```
                app.company.com

                       │

          ┌────────────┴────────────┐

          ▼                         ▼

          /                      /api

          │                         │

          ▼                         ▼

Frontend Service         Backend Service
```

> 也就是说，用户看到的是一个域名，但 Ingress 会根据路径把请求分发到不同的 Service。

------

# 第十节：整个 Helm Chart 应该如何组织？

企业里面，通常会有两种组织方式。

## 方案一：一个总 Chart（推荐中小型项目）

```
my-system/

Chart.yaml

values.yaml

templates/

    frontend-deployment.yaml

    backend-deployment.yaml

    redis-statefulset.yaml

    ingress.yaml

    services.yaml
```

适合：

中小项目。

一次：

部署：

整个系统。

------

## 方案二：多个 Chart（推荐大型微服务）

```
frontend-chart/

backend-chart/

redis-chart/

gateway-chart/
```

每个：

独立：

发布。

大型公司：

基本：

都是：

这种。

原因：

每个：

微服务：

都有：

自己的：

生命周期。

------

# 第十一节：一次 helm install 到底发生了什么？

假设：

执行：

```
helm upgrade --install order-system .
```

实际上：

Helm：

做了：

下面：

这些：

事情：

```
读取 values.yaml

        │

        ▼

渲染 Deployment

渲染 Service

渲染 ConfigMap

渲染 Secret

渲染 Ingress

渲染 HPA

渲染 StatefulSet

        │

        ▼

生成普通 YAML

        │

        ▼

提交 API Server

        │

        ▼

Scheduler 调度

        │

        ▼

Pod 创建

        │

        ▼

Service 建立

        │

        ▼

Ingress 生效

        │

        ▼

用户访问成功
```

你会发现：

Helm 只是**开始**。

真正执行资源创建和调度的，依然是 Kubernetes。

------

# 第十二节：部署后的检查顺序（企业运维经验）

很多新手部署成功后，只会执行：

```
kubectl get pods
```

其实，生产环境更推荐按照下面的顺序检查：

1. **Pod 是否正常运行**

```
kubectl get pods
```

确认状态为 `Running`，没有频繁重启（`RESTARTS` 不断增加）。

------

1. **Deployment 是否全部就绪**

```
kubectl get deployment
```

检查 `READY` 是否达到期望副本数，例如 `3/3`。

------

1. **Service 是否创建成功**

```
kubectl get svc
```

确认 ClusterIP、NodePort 或 LoadBalancer 信息是否符合预期。

------

1. **Ingress 是否获取地址**

```
kubectl get ingress
```

如果使用云平台，还需要等待外部 IP 或域名生效。

------

1. **查看事件**

```
kubectl get events --sort-by=.metadata.creationTimestamp
```

很多调度失败、镜像拉取失败、PVC 挂载失败，都能在这里看到线索。

------

1. **查看日志**

```
kubectl logs <pod-name>
```

如果 Pod 正常启动但应用异常，日志通常是第一手信息。

> 后面的"生产故障排查"章节，我们会系统学习这些命令。

------

# 第十三节：为什么企业喜欢这种部署方式？

因为它带来了几个非常重要的能力：

- **标准化**：所有项目都遵循相同的部署结构。
- **可重复**：开发、测试、生产只需要切换 `values` 文件。
- **可回滚**：Helm 保留版本历史，升级失败可快速恢复。
- **可扩展**：后续增加 Prometheus、Grafana、Kafka 等组件，只需新增 Chart 或模板。

这也是 Helm 成为 Kubernetes 主流交付方式的重要原因。

------

# 本章总结（建议牢记）

请记住一个完整 Kubernetes 项目的核心组成：

1. **前端（Vue/Nginx）和后端（ASP.NET Core）通常分别部署为不同的 Deployment。**
2. **Redis、MySQL 等有状态服务优先使用 StatefulSet。**
3. **ConfigMap 保存普通配置，Secret 保存敏感信息。**
4. **Ingress 负责把一个域名的不同路径路由到不同 Service。**
5. **一次 `helm install` 往往会创建十几个 Kubernetes 资源，而不是只有一个 Deployment。**
6. **部署完成后，应按 Pod → Deployment → Service → Ingress → Events → Logs 的顺序检查。**

------

# 到这里，你已经具备了部署完整业务系统的知识

你现在已经能够理解一个典型企业项目从 Helm 到 Kubernetes 的完整流程，也理解了各类资源之间的协作关系。

不过，一个项目部署成功，并不意味着工作结束。

真正的生产环境里，运维工程师每天做得最多的事情其实是：

- 服务变慢了怎么办？
- Pod 为什么不断重启？
- 用户反馈接口报错，如何定位？
- 如何知道 CPU、内存是否达到瓶颈？
- 日志应该去哪里看？

这些问题都离不开**可观测性（Observability）**。

------

# 下一章预告：生产运维第一课——日志体系（Logging）

下一章，我们将正式进入生产运维三大支柱之一：**日志（Logging）**。

我们会详细讲解：

- 为什么 `kubectl logs` 不够用？
- Kubernetes 日志是如何产生的？
- 容器日志为什么不能直接写到文件？
- 什么是 stdout / stderr？
- 什么是节点日志？
- 如何收集整个集群的日志？
- 为什么生产环境几乎都会使用 **Loki + Grafana** 或 **EFK（Elasticsearch + Fluent Bit + Kibana）**？
- 如何为 ASP.NET Core 配置结构化日志（如 Serilog），让日志更容易排查问题？

这一章结束后，你将开始真正掌握**生产环境运维**所需的第一项核心能力。

# 第二阶段 第五章：生产运维第一课——Kubernetes 日志体系（Logging）

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入 **Kubernetes 生产运维三大支柱**。

整个现代运维（包括 Kubernetes、Docker、云原生）都有一个非常重要的概念：

> **Observability（可观测性）**

可观测性主要由三部分组成：

```
Observability（可观测性）

├── Logging（日志）
├── Metrics（指标）
└── Tracing（链路追踪）
```

以后你无论去腾讯、阿里、字节、美团，还是国外公司，这三个概念都会反复出现。

今天，我们先学习第一部分：

> **Logging（日志）**

## 本章学习目标

学习完本章，你应该能够回答：

- Kubernetes 日志到底存在哪里？
- 为什么容器不能像以前一样直接写日志文件？
- stdout、stderr 到底是什么？
- kubectl logs 的原理是什么？
- 为什么 kubectl logs 不适合生产环境？
- 企业如何收集整个集群日志？
- ASP.NET Core 如何正确输出日志？
- 什么是结构化日志？

------

# 第一节：先回忆一下传统服务器

以前。

没有 Docker。

没有 Kubernetes。

部署 ASP.NET Core：

一般都是：

```
Windows IIS

或者

Linux + Nginx
```

日志通常写到：

例如：

```
D:\Logs\api.log
```

或者：

```
/var/log/order-api.log
```

出了问题。

SSH：

进入服务器：

```
tail -f api.log
```

结束。

是不是很简单？

------

# 第二节：为什么 Kubernetes 不建议写日志文件？

现在。

你的项目运行在：

Pod。

例如：

```
Node1

┌────────────────────┐

Pod

Order.API

└────────────────────┘
```

突然。

Pod：

挂了。

Deployment：

重新创建：

```
Node2

┌────────────────────┐

新的 Pod

Order.API

└────────────────────┘
```

请问。

旧 Pod：

里面：

日志文件：

还在吗？

答案：

> **没了。**

因为：

Pod：

本身：

就是：

临时的。

这就是云原生中非常重要的一条原则：

> **容器应该是无状态（Stateless）的。**

因此：

日志不能依赖容器本地磁盘。

------

# 第三节：那日志写到哪里？

答案：

很多新人第一次知道都会觉得很奇怪：

> **直接输出到控制台（Console）。**

也就是：

```
Console.WriteLine(...)
```

或者：

ASP.NET Core：

```
logger.LogInformation(...)
```

最终：

都会输出：

```
stdout
```

或者：

```
stderr
```

------

# 第四节：什么是 stdout 和 stderr？

这是 Linux 非常基础，但又非常重要的概念。

每个进程启动时，默认都有三个标准流：

| 名称   | 作用         |
| ------ | ------------ |
| stdin  | 标准输入     |
| stdout | 标准输出     |
| stderr | 标准错误输出 |

例如：

C#：

```
Console.WriteLine("Hello");
```

其实：

就是：

输出到：

```
stdout
```

例如：

抛异常：

```
NullReferenceException
```

通常：

输出：

```
stderr
```

Docker：

就是收集：

这两个：

输出。

------

## 一个生活中的例子

想象你在办公室工作：

- **stdout**：你正常汇报工作。
- **stderr**：你举手报告："出问题了！"

领导（Docker）会把两种信息都记录下来。

------

# 第五节：Docker 如何保存日志？

很多人以为：

Docker：

不会：

保存。

其实：

会。

流程：

如下：

```
ASP.NET Core

        │

Console.WriteLine()

        │

stdout

        │

Docker Engine

        │

json log

        │

磁盘
```

例如：

Docker：

默认：

会保存：

JSON：

日志。

所以：

执行：

```
docker logs 容器ID
```

其实：

就是：

读取：

这些：

日志。

------

# 第六节：kubectl logs 到底做了什么？

现在。

Docker：

变成：

Kubernetes。

流程：

变成：

```
ASP.NET Core

        │

stdout

        │

Container Runtime（containerd 等）

        │

Node

        │

kubectl logs
```

所以：

执行：

```
kubectl logs order-api-xxxxx
```

其实：

就是：

读取：

容器：

stdout。

而：

不是：

读取：

某个：

日志文件。

这一点：

很多新人：

都会：

误解。

------

# 第七节：kubectl logs 常用命令

查看：

日志：

```
kubectl logs pod-name
```

持续：

查看：

```
kubectl logs -f pod-name
```

查看：

上一轮崩溃前的日志：

```
kubectl logs --previous pod-name
```

查看：

Deployment：

```
kubectl logs deployment/order-api
```

如果：

Deployment：

有：

多个：

Pod。

默认：

读取：

其中：

一个。

如果要查看指定 Pod，建议先：

```
kubectl get pods
```

再查看对应 Pod。

------

# 第八节：为什么 kubectl logs 不适合生产环境？

假设：

你的：

API：

有：

```
20

Pod
```

分布：

```
Node1

Node2

Node3

Node4
```

用户：

报错。

你：

怎么办？

难道：

一个：

一个：

执行：

```
kubectl logs
```

二十次？

显然：

不现实。

更大的问题是：

Pod 删除后，本地日志最终也会消失，因此很难做长期分析。

------

# 第九节：企业如何收集日志？

生产环境。

都会：

部署：

日志：

采集器。

例如：

```
Node

────────────

Pod A

Pod B

Pod C

────────────

Fluent Bit
```

Fluent Bit：

负责：

读取：

Node：

所有：

Container：

日志。

然后：

发送：

到：

日志平台。

------

整个：

流程：

如下：

```
ASP.NET Core

        │

stdout

        │

Container Runtime

        │

Fluent Bit

        │

Loki / Elasticsearch

        │

Grafana / Kibana
```

以后。

查看：

日志。

不用：

SSH。

不用：

kubectl。

直接：

打开：

网页。

搜索。

即可。

------

# 第十节：目前企业主流日志方案

目前最常见的有两种。

------

## 方案一：Loki + Grafana（越来越流行）

```
ASP.NET Core

        │

stdout

        │

Fluent Bit

        │

Loki

        │

Grafana
```

优点：

- 资源占用较低
- 与 Grafana 集成非常好
- 运维成本相对较低
- 特别适合 Kubernetes

近年来，越来越多的新项目会优先考虑这一方案。

------

## 方案二：EFK

也就是：

```
Elasticsearch

Fluent Bit（或 Fluentd）

Kibana
```

流程：

```
stdout

↓

Fluent Bit

↓

Elasticsearch

↓

Kibana
```

优点：

- 搜索能力强
- 生态成熟
- 很多老项目仍在使用

缺点：

- Elasticsearch 占用资源较高
- 运维复杂度相对更高

------

# 第十一节：ASP.NET Core 应该如何写日志？

很多新人：

喜欢：

```
Console.WriteLine("开始执行");
```

可以。

但是。

企业：

一般：

不用。

而是：

使用：

```
ILogger<OrderService>
```

例如：

```
_logger.LogInformation("Order {OrderId} created successfully.", orderId);
```

为什么？

因为：

日志：

会：

自动：

包含：

```
时间

日志级别

类别

消息
```

更重要的是，它支持**结构化日志**。

------

# 第十二节：什么是结构化日志？

很多新手会写：

```
_logger.LogInformation("订单创建成功：" + orderId);
```

虽然能工作。

但：

更推荐：

```
_logger.LogInformation(
    "订单创建成功，订单号：{OrderId}",
    orderId);
```

为什么？

因为：

日志平台：

能够：

识别：

```
OrderId
```

以后。

Grafana。

Kibana。

可以：

直接：

搜索：

```
OrderId=10001
```

这就是：

结构化日志。

> **不要把所有信息拼成字符串，而是把关键字段作为独立属性记录。**

这是现代日志系统最重要的理念之一。

------

# 第十三节：日志级别

ASP.NET Core 常见日志级别如下：

| 级别        | 说明         | 是否常用于生产 |
| ----------- | ------------ | -------------- |
| Trace       | 最详细       | 很少开启       |
| Debug       | 调试信息     | 开发环境常用   |
| Information | 正常业务流程 | ✅ 最常用       |
| Warning     | 潜在问题     | ✅ 常用         |
| Error       | 错误         | ✅ 必须记录     |
| Critical    | 严重故障     | ✅ 必须记录     |

一般建议：

- 开发环境：`Debug` 或 `Information`
- 测试环境：`Information`
- 生产环境：通常以 `Information` 为主，必要时临时提高日志级别排查问题。

------

# 第十四节：日志最佳实践

结合 Kubernetes 和 ASP.NET Core，推荐遵循以下原则：

1. **不要把业务日志写入容器内固定文件。**
2. **统一输出到 stdout / stderr。**
3. **使用 `ILogger<T>` 而不是 `Console.WriteLine()`。**
4. **使用结构化日志，不要简单拼接字符串。**
5. **通过 Fluent Bit 收集日志，再集中存储到 Loki 或 Elasticsearch。**
6. **使用 Grafana 或 Kibana 查询日志，而不是频繁登录节点。**
7. **日志不要记录密码、Token、身份证号等敏感信息。**

------

# 本章总结（建议牢记）

请记住 Kubernetes 日志体系最重要的几点：

1. **Pod 是临时资源，因此不要依赖容器内日志文件。**
2. **Docker/Kubernetes 默认收集 stdout 和 stderr。**
3. **`kubectl logs` 实际上查看的是容器标准输出，而不是某个日志文件。**
4. **生产环境需要集中式日志平台，而不是逐个 Pod 查看日志。**
5. **推荐使用 `ILogger<T>` 和结构化日志，提高检索和分析能力。**
6. **Loki + Grafana 和 EFK 都是主流日志方案，各有适用场景。**

------

# 到这里，你已经掌握了生产环境日志体系

你已经知道：

```
ASP.NET Core
        │
ILogger<T>
        │
stdout / stderr
        │
Container Runtime
        │
Fluent Bit
        │
Loki / Elasticsearch
        │
Grafana / Kibana
```

这条链路，就是现代 Kubernetes 日志系统的核心工作流程。

------

# 下一章预告：生产运维第二课——监控体系（Prometheus + Grafana）

日志告诉我们：

> **发生了什么。**

下一章学习的**监控（Metrics）**则回答另一个问题：

> **系统现在运行得怎么样？**

我们将深入学习：

- 什么是 Metrics（指标）？
- 为什么仅有日志还不够？
- Prometheus 是如何采集 Kubernetes 指标的？
- 什么是 Exporter？
- 什么是 Alertmanager？
- Grafana 为什么几乎成为监控平台的标准？
- 如何监控 ASP.NET Core API、Redis、Node、Pod、Ingress？
- 如何配置 CPU、内存、响应时间、错误率等告警？

这一章结束后，你将具备搭建和理解**企业级 Kubernetes 监控体系**的能力。

# 第二阶段 第六章：生产运维第二课——Prometheus + Grafana 监控体系

提示词：

```
开始下一章
```

正文：

非常好。

这一章，是整个 Kubernetes 运维里面**最重要的一章之一**。

很多新人都会认为：

> 有日志，不就够了吗？

实际上。

真正的生产环境。

**日志（Logging）和监控（Metrics）解决的是两个完全不同的问题。**

例如：

凌晨 3 点。

你的手机收到一条告警：

```
Order API CPU 持续超过 95%
```

这时候。

你还没有看日志。

就已经知道：

系统出问题了。

这就是：

> **Metrics（监控指标）**

它负责：

**提前发现问题。**

而日志负责：

**定位问题原因。**

> **这是 Kubernetes 运维中必须掌握的内容。**

------

# 本章学习目标

学习完本章，你应该能够回答：

- 什么是 Metrics？
- Prometheus 为什么会成为 Kubernetes 的事实标准？
- Prometheus 是如何采集数据的？
- Exporter 是什么？
- Grafana 为什么几乎人人都在用？
- Alertmanager 有什么作用？
- 如何监控 ASP.NET Core？
- 企业一般监控哪些指标？
- 什么是 RED、USE 监控方法？

------

# 第一节：什么是 Metrics（指标）？

先来看一个例子。

假设：

你的 API：

目前运行情况如下：

```
CPU：28%

Memory：412MB

QPS：560

Response Time：35ms

Error Rate：0.03%
```

这些数字。

就是：

Metrics。

一句话：

> **Metrics 是用数字描述系统运行状态。**

它最大的特点是：

- 可以画曲线
- 可以做统计
- 可以设置告警
- 可以观察趋势

例如：

CPU：

```
10%

20%

30%

95%

100%
```

是不是：

一眼就能发现问题？

------

# 第二节：为什么日志不够？

假设：

凌晨。

系统：

突然：

很慢。

如果：

只有：

日志。

你需要：

```
打开日志

↓

搜索 Error

↓

搜索 Exception

↓

分析原因
```

但是。

如果：

先看：

Grafana：

你可能：

一分钟：

就发现：

```
CPU：

99%
```

或者：

```
Memory：

95%
```

或者：

```
Redis：

连接数爆满
```

是不是：

快很多？

所以。

一句话：

> **Metrics 告诉你哪里出了问题，Logs 告诉你为什么出了问题。**

------

# 第三节：Prometheus 是什么？

一句话：

> **Prometheus 是一个时序数据库（Time Series Database）+ 监控系统。**

它专门保存：

```
时间

+

指标
```

例如：

```
09:00 CPU 25%

09:01 CPU 28%

09:02 CPU 30%

09:03 CPU 95%
```

这些数据。

都会保存。

以后：

Grafana：

画：

曲线。

就是：

读取：

这些：

历史数据。

------

# 第四节：Prometheus 如何采集数据？

很多新人：

以为：

Prometheus：

主动进入 Pod。

其实：

不是。

绝大多数情况下，它采用的是：

> **Pull（主动拉取）模式。**

流程：

```
Prometheus

        │

HTTP GET /metrics

        │

        ▼

ASP.NET Core

返回指标
```

例如：

每：

15 秒。

Prometheus：

访问：

```
http://pod-ip:8080/metrics
```

然后：

保存：

结果。

------

## 一个生活例子

想象老师点名：

- **Pull**：老师主动叫每个同学回答。
- **Push**：每个同学自己跑去告诉老师。

Prometheus 默认采用前者。

------

# 第五节：Exporter 是什么？

很多程序。

不会：

直接：

输出：

监控数据。

怎么办？

于是：

出现：

Exporter。

一句话：

> **Exporter 是把各种系统的数据转换成 Prometheus 能理解的格式。**

例如：

```
Node

↓

Node Exporter
```

采集：

```
CPU

Memory

Disk

Network
```

------

Redis：

```
Redis

↓

Redis Exporter
```

采集：

```
Memory

Hit Rate

Connected Clients

Commands
```

------

MySQL：

```
MySQL

↓

MySQL Exporter
```

采集：

```
Slow Query

Connections

TPS
```

所以：

Exporter：

可以理解成：

> **翻译官。**

------

# 第六节：ASP.NET Core 如何提供指标？

ASP.NET Core：

通常：

通过：

OpenTelemetry 或 Prometheus 相关库暴露：

```
/metrics
```

例如：

访问：

```
http://localhost:8080/metrics
```

可能：

返回：

```
http_requests_total 18231

http_request_duration_seconds 0.032

process_cpu_seconds_total 5.2
```

Prometheus：

就是：

读取：

这些：

数据。

> 近年来，越来越多团队会采用 **OpenTelemetry** 统一输出 Metrics、Logs 和 Traces，再由 Prometheus 等系统采集。

------

# 第七节：Prometheus 整个工作流程

这是必须记住的一张图。

```
ASP.NET Core

        │

/metrics

        │

        ▼

Prometheus

        │

保存指标

        │

        ▼

Grafana

        │

画图

        │

        ▼

Alertmanager

        │

企业微信

邮件

Slack

Teams
```

是不是：

其实：

很简单？

------

# 第八节：Grafana 是什么？

一句话：

> **Grafana 是监控数据可视化平台。**

Prometheus：

保存：

数字。

Grafana：

负责：

展示。

例如：

CPU：

```
██████████
```

内存：

```
██████
```

QPS：

```
────────╮
        │
────────╯
```

Grafana：

可以：

做：

Dashboard。

例如：

```
Order API Dashboard

CPU

Memory

QPS

Response Time

Error Rate
```

企业里面。

几乎：

每天：

都会：

打开。

------

# 第九节：Alertmanager 是什么？

Prometheus：

负责：

采集。

Grafana：

负责：

展示。

那么：

谁：

负责：

报警？

答案：

Alertmanager。

例如：

规则：

```
CPU > 90%

持续：

5 分钟
```

Alertmanager：

发送：

```
企业微信

邮件

Slack

Microsoft Teams
```

所以。

凌晨：

手机：

响。

就是：

它：

发的。

------

# 第十节：企业一般监控哪些内容？

一个典型的 ASP.NET Core + Kubernetes 项目，通常会监控四个层面：

| 层面       | 典型指标                                  |
| ---------- | ----------------------------------------- |
| Node       | CPU、内存、磁盘、网络                     |
| Kubernetes | Pod 数量、重启次数、Pending Pod、HPA 状态 |
| 应用       | QPS、响应时间、错误率、并发数             |
| 中间件     | Redis、MySQL、RabbitMQ、Kafka 等          |

这些组合起来，基本能覆盖绝大多数生产问题。

------

# 第十一节：RED 方法（应用监控）

Google 提出了很多监控理念。

其中。

最经典：

就是：

RED。

```
R

Rate

请求数量

────────────

E

Errors

错误率

────────────

D

Duration

响应时间
```

例如：

你的 API：

应该：

至少：

监控：

```
QPS

500

────────

Error Rate

0.02%

────────

Latency

35ms
```

如果：

突然：

```
Latency

35ms

↓

1200ms
```

不用：

看：

日志。

已经：

知道：

有：

问题。

------

# 第十二节：USE 方法（基础设施监控）

除了应用。

Node。

也：

需要：

监控。

这里：

最经典：

的是：

USE。

```
Utilization

利用率

────────────

Saturation

饱和度

────────────

Errors

错误
```

例如：

CPU：

```
95%
```

利用率：

很高。

磁盘：

IO：

排队：

很多。

说明：

饱和。

网卡：

丢包。

说明：

错误。

这套方法非常适合分析服务器、Kubernetes 节点和存储性能。

------

# 第十三节：ASP.NET Core 最值得监控的指标

如果你主要开发 ASP.NET Core，建议重点关注：

- HTTP 请求数（Requests / QPS）
- HTTP 响应时间（Latency）
- HTTP 错误率（4xx / 5xx）
- CPU 使用率
- 内存使用量
- GC 次数与暂停时间
- 数据库连接池使用情况
- Redis 命中率
- 后台任务队列长度

这些指标足以帮助你发现大部分性能问题。

------

# 第十四节：监控最佳实践

生产环境建议：

1. **所有应用都暴露 Metrics 接口（推荐使用 OpenTelemetry）。**
2. **使用 Prometheus 定时采集。**
3. **Grafana 建立统一 Dashboard。**
4. **Alertmanager 配置关键告警。**
5. **不要只监控基础设施，也要监控业务指标。**
6. **告警不要过多，避免"告警疲劳"。真正影响业务的问题才应立即通知。**

------

# 第十五节：Logging、Metrics、Tracing 的关系

最后。

一定要理解：

这三者：

不是：

替代。

而是：

互补。

```
Logging

↓

为什么报错？

────────────

Metrics

↓

什么时候开始变慢？

────────────

Tracing

↓

到底是哪一个服务慢？
```

三者：

合起来。

才是：

真正：

完整：

的：

可观测性。

------

# 本章总结（建议牢记）

请记住监控体系最重要的几点：

1. **Metrics 用数字描述系统状态，适合画图、统计和告警。**
2. **Prometheus 是 Kubernetes 最主流的监控系统，默认采用 Pull 模式采集数据。**
3. **Exporter 用于把各种系统的数据转换为 Prometheus 可识别的格式。**
4. **Grafana 负责展示监控面板，Alertmanager 负责发送告警。**
5. **应用建议采用 RED 方法监控，基础设施建议采用 USE 方法监控。**
6. **日志、指标和链路追踪共同组成现代可观测性体系。**

------

# 到这里，你已经理解了企业级监控架构

```
ASP.NET Core
        │
   /metrics
        │
        ▼
Prometheus
        │
        ├── 存储时序数据
        │
        ├── Alertmanager（告警）
        │
        ▼
Grafana（可视化）
```

这也是目前 Kubernetes 社区和绝大多数企业采用的标准方案。

------

# 下一章预告：生产运维第三课——链路追踪（Tracing）

到目前为止：

- **日志（Logs）**告诉我们：发生了什么。
- **指标（Metrics）**告诉我们：什么时候开始变慢。

但是，还有一个问题没有解决：

> **到底是哪一个服务、哪一个数据库查询、哪一次 RPC 调用导致了请求变慢？**

下一章，我们将学习可观测性的最后一块拼图——**分布式链路追踪（Tracing）**，包括：

- 什么是 Trace、Span？
- OpenTelemetry 是什么？
- Jaeger、Tempo 的作用
- 如何追踪一次请求经过的所有微服务
- 如何为 ASP.NET Core 接入 OpenTelemetry
- 如何快速定位微服务性能瓶颈

学完这一章，你就会完整掌握现代 Kubernetes 生产环境的三大可观测性能力：**Logging、Metrics、Tracing**。

很好。

这一章，是现代 Kubernetes 和微服务体系中最容易让新人觉得抽象，但一旦理解后又会觉得非常自然的一章。

前两章，我们已经学习了：

```
Logs（日志）

回答：

为什么出错？
Metrics（监控）

回答：

什么时候开始变慢？
```

但是。

还有一个问题。

假设你的系统已经拆成了几十个微服务。

例如：

```
用户下单
      │
      ▼
Order API
      │
      ▼
Inventory API
      │
      ▼
Payment API
      │
      ▼
Redis
      │
      ▼
MySQL
```

用户告诉你：

> 下单接口响应了 **8 秒**。

请问：

到底是哪一个服务慢？

Redis？

MySQL？

库存？

支付？

还是网络？

Logs 很难告诉你。

Metrics 也只能告诉你：

> **确实变慢了。**

但是：

到底是谁慢？

不知道。

于是。

第三块拼图出现了：

> **Tracing（链路追踪）**

------

# 第二阶段 第七章：生产运维第三课——分布式链路追踪（Tracing）

> **这是现代微服务排障的核心能力。**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，是现代 Kubernetes 和微服务体系中最容易让新人觉得抽象，但一旦理解后又会觉得非常自然的一章。

前两章，我们已经学习了：

```
Logs（日志）

回答：

为什么出错？
Metrics（监控）

回答：

什么时候开始变慢？
```

但是。

还有一个问题。

假设你的系统已经拆成了几十个微服务。

例如：

```
用户下单
      │
      ▼
Order API
      │
      ▼
Inventory API
      │
      ▼
Payment API
      │
      ▼
Redis
      │
      ▼
MySQL
```

用户告诉你：

> 下单接口响应了 **8 秒**。

请问：

到底是哪一个服务慢？

Redis？

MySQL？

库存？

支付？

还是网络？

Logs 很难告诉你。

Metrics 也只能告诉你：

> **确实变慢了。**

但是：

到底是谁慢？

不知道。

于是。

第三块拼图出现了：

> **Tracing（链路追踪）**

# 本章学习目标

学习完本章，你应该能够回答：

- 什么是 Trace？
- 什么是 Span？
- Trace ID 有什么作用？
- OpenTelemetry 为什么越来越重要？
- Jaeger、Tempo 是什么？
- 如何追踪一次 ASP.NET Core 请求？
- 企业为什么一定会做链路追踪？
- Logging、Metrics、Tracing 如何协同工作？

------

# 第一节：为什么需要链路追踪？

先看一个单体应用。

```
Browser

    │

    ▼

ASP.NET Core

    │

    ▼

MySQL
```

这里只有：

一个 API。

定位问题：

很简单。

但是。

如果系统变成：

```
Browser

    │

    ▼

Gateway

    │

    ▼

Order API

    │

    ├───────────────┐

    ▼               ▼

Inventory API   User API

    │               │

    ▼               ▼

Redis          MySQL
```

现在。

一次请求。

可能经过：

五六个服务。

如果：

用户说：

```
下单用了：

6 秒
```

请问：

哪一步：

最慢？

不知道。

------

# 第二节：什么是 Trace？

一句话：

> **Trace 就是一次完整请求的生命周期。**

例如：

用户点击：

```
立即下单
```

整个过程：

```
浏览器

    │

    ▼

Gateway

    │

    ▼

Order API

    │

    ▼

Inventory API

    │

    ▼

Redis

    │

    ▼

MySQL
```

这一整条路径。

就叫：

```
Trace
```

所以。

可以理解为：

> **Trace = 一条完整的请求路线。**

------

# 第三节：什么是 Span？

如果说：

Trace：

是一趟旅行。

那么：

Span：

就是：

旅程中的每一站。

例如：

```
Trace

──────────────

Gateway

30ms

──────────────

Order API

120ms

──────────────

Inventory API

800ms

──────────────

Redis

5ms

──────────────

MySQL

1200ms
```

这里：

每一个：

矩形。

都是：

```
Span
```

一句话：

> **Span 是一次具体操作的耗时记录。**

例如：

一个 Span 可以表示：

- 一次 HTTP 请求
- 一次数据库查询
- 一次 Redis 操作
- 一次消息队列发送
- 一次调用第三方接口

------

## 一个生活中的例子

假设你从台北去高雄：

```
台北

↓

台中

↓

嘉义

↓

台南

↓

高雄
```

整趟旅行：

就是：

```
Trace
```

每一段高铁：

就是：

```
Span
```

如果：

嘉义到台南：

堵车：

3 小时。

是不是：

马上知道：

哪里：

慢？

------

# 第四节：什么是 Trace ID？

假设：

同一时间：

有：

10000 个用户。

大家：

都：

在：

下单。

系统：

怎么知道：

哪些日志。

属于：

同一次：

请求？

答案：

每一次请求都会生成：

```
Trace ID
```

例如：

```
Trace ID

4fdc7a98...
```

以后：

所有：

服务：

都会：

携带：

这个：

ID。

例如：

```
Gateway

TraceID=A123

↓

Order API

TraceID=A123

↓

Inventory API

TraceID=A123

↓

Redis

TraceID=A123
```

于是。

日志。

监控。

链路。

全部：

关联：

起来。

------

# 第五节：OpenTelemetry 是什么？

以前。

每家公司。

都有：

自己的：

监控 SDK。

后来。

行业发现：

太乱了。

于是。

诞生了：

> **OpenTelemetry（OTel）**

一句话：

> **OpenTelemetry 是可观测性的统一标准。**

它统一了：

```
Logs

Metrics

Tracing
```

现在。

ASP.NET Core。

Java。

Go。

Python。

Node.js。

都：

支持：

OpenTelemetry。

所以。

越来越多企业：

都会：

直接：

接入：

OTel。

------

# 第六节：ASP.NET Core 如何接入 OpenTelemetry？

ASP.NET Core 本身已经很好地支持 OpenTelemetry。

典型流程如下：

```
HTTP 请求

      │

      ▼

OpenTelemetry SDK

      │

生成：

Trace

Span

Metrics

Logs

      │

      ▼

OTLP Exporter

      │

      ▼

Jaeger

Tempo

Prometheus

Loki
```

你会发现：

OpenTelemetry 更像一个"采集标准"，真正负责存储和展示的是后面的系统。

------

# 第七节：Jaeger 是什么？

Jaeger：

是：

最经典：

的：

Trace：

平台。

例如：

打开：

Jaeger：

可能：

看到：

```
Order API

2.8s

──────────────

Gateway

20ms

──────────────

Inventory

300ms

──────────────

Redis

2ms

──────────────

MySQL

2400ms
```

是不是：

一眼：

就知道：

数据库：

最慢？

------

# 第八节：Tempo 又是什么？

近年来。

Grafana 推出了：

```
Tempo
```

作用：

也是：

保存：

Trace。

区别：

大致如下：

| 平台   | 特点                                                  |
| ------ | ----------------------------------------------------- |
| Jaeger | 成熟、功能丰富、学习资料多                            |
| Tempo  | 与 Grafana、Loki、Prometheus 集成更紧密，资源占用较低 |

很多新建 Kubernetes 平台，会采用：

```
Grafana

+

Prometheus

+

Loki

+

Tempo
```

形成统一的可观测性平台。

------

# 第九节：一次请求是如何被追踪的？

假设：

浏览器：

访问：

```
POST /api/order
```

整个过程：

```
Browser

        │

        ▼

Gateway

        │

        ▼

Order API

        │

        ▼

Inventory API

        │

        ▼

Redis

        │

        ▼

MySQL
```

每一步：

都会：

记录：

```
开始时间

结束时间

耗时

状态

Trace ID

Span ID
```

最后：

Jaeger：

画成：

时间线。

例如：

```
Gateway

████

Order API

████████████

Inventory

██████████████████████

Redis

█

MySQL

██████████████████████████
```

是不是：

马上：

知道：

MySQL：

慢？

------

# 第十节：Logging、Metrics、Tracing 如何协同？

这是整个可观测性最重要的一张图。

假设：

用户：

投诉：

```
订单：

很慢
```

你的排查步骤：

第一步。

Grafana：

看：

Metrics。

发现：

```
Latency

40ms

↓

1800ms
```

说明：

真的：

变慢。

------

第二步。

Jaeger：

查看：

Trace。

发现：

```
Inventory API

耗时：

1300ms
```

已经：

定位：

到：

服务。

------

第三步。

Loki：

搜索：

```
TraceID=A123
```

马上：

找到：

所有：

日志。

例如：

```
SQL Timeout

Redis Retry

Network Timeout
```

最终。

定位：

问题。

整个过程：

```
Metrics

↓

发现问题

↓

Tracing

↓

定位服务

↓

Logging

↓

定位代码
```

这就是企业排障的标准流程。

------

# 第十一节：为什么要把 Trace ID 写进日志？

如果日志里没有 Trace ID：

```
Error

Timeout

NullReferenceException
```

你根本不知道：

它属于哪个请求。

因此。

企业一般都会让日志自动带上：

```
TraceId

SpanId

RequestId

UserId（按需）
```

这样：

当你在日志平台搜索某个 Trace ID 时，就能看到整个请求相关的所有日志。

这也是结构化日志和链路追踪结合的重要价值。

------

# 第十二节：ASP.NET Core 最佳实践

对于 ASP.NET Core 项目，推荐逐步采用以下方案：

- 使用 `ILogger<T>` 输出结构化日志。
- 使用 OpenTelemetry 自动采集 HTTP、数据库等 Trace。
- 使用 OTLP Exporter 将数据发送到可观测性平台。
- 使用 Prometheus 采集 Metrics。
- 使用 Loki 收集日志。
- 使用 Grafana 展示 Metrics、Logs、Traces。
- 为关键业务（如下单、支付、登录）建立专门的 Dashboard。

------

# 第十三节：现代企业的可观测性架构

一个典型架构如下：

```
                Browser
                    │
                    ▼
             ASP.NET Core API
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     Logs        Metrics      Traces
        │           │           │
        ▼           ▼           ▼
      Loki     Prometheus     Tempo
           └──────┬────────────┘
                  ▼
              Grafana
```

你会发现：

Grafana 已经不仅仅是一个监控工具，而是很多企业统一的可观测性入口。

------

# 第十四节：生产环境最佳实践

建议遵循以下原则：

1. **所有服务都开启 OpenTelemetry。**
2. **所有日志都携带 Trace ID。**
3. **优先使用自动埋点，再针对核心业务增加手动埋点。**
4. **不要采集所有细节，避免产生过多 Trace 数据。**
5. **关注慢请求（Slow Trace），而不是每一个普通请求。**
6. **结合 Metrics、Tracing、Logging，而不是单独依赖其中一种。**

------

# 本章总结（建议牢记）

请记住链路追踪最重要的几点：

1. **Trace 表示一次完整请求，Span 表示请求中的一个具体步骤。**
2. **Trace ID 是关联日志、指标和链路的关键。**
3. **OpenTelemetry 已成为现代可观测性的统一标准。**
4. **Jaeger 和 Tempo 都是主流的 Trace 存储与查询系统。**
5. **链路追踪最适合定位微服务之间的性能瓶颈。**
6. **企业排障通常遵循：Metrics 发现问题 → Tracing 定位服务 → Logging 定位代码。**

------

# 到这里，你已经掌握了现代 Kubernetes 可观测性的完整体系

```
                 Application
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     Logging       Metrics       Tracing
        │             │             │
      Loki      Prometheus       Tempo
        └─────────────┬─────────────┘
                      ▼
                  Grafana
```

这是目前云原生领域最主流、也是最值得掌握的一套可观测性架构。

------

# 下一章预告：Kubernetes 生产故障排查（Troubleshooting）

到目前为止，你已经会：

- 部署应用
- 编写 Helm Chart
- 收集日志
- 查看监控
- 分析链路

下一章，我们将进入真正的**生产运维实战**。

你将学习：

- Pod 一直是 `Pending` 怎么办？
- Pod 一直 `CrashLoopBackOff` 怎么办？
- `ImagePullBackOff` 如何排查？
- Service 无法访问怎么办？
- Ingress 不生效怎么办？
- 为什么探针（Liveness/Readiness）会导致应用不断重启？
- 如何建立一套系统化的 Kubernetes 故障排查思路？

这一章开始，你将从"会部署"真正迈向"会运维生产环境"。

# 第三阶段 第一章：Kubernetes 生产故障排查（Troubleshooting）

> **这一章建议反复阅读。以后工作中，你会经常用到这里的方法。**

------

提示词：

```
开始下一章
```

正文：

非常好。

从这一章开始，我们进入整个 Kubernetes 学习过程中**最有价值的一部分**。

前面学习的 Deployment、Service、Ingress、Helm、监控、日志……

都是基础能力。

真正能体现一个 Kubernetes 工程师水平的，是下面这件事情：

> **生产环境出故障时，能否快速定位并恢复服务。**

很多新手遇到 Kubernetes 故障时，第一反应是：

```
删除 Pod，再试一次。
```

有时候确实能恢复。

但如果这是线上生产环境，这种做法既不可靠，也可能掩盖真正的问题。

企业真正需要的是：

> **建立一套标准化、可重复的故障排查流程。**

# 本章学习目标

学习完本章，你应该能够回答：

- Kubernetes 故障排查的正确顺序是什么？
- 为什么不能一上来就看日志？
- 如何分析 Pod 一直 Pending？
- CrashLoopBackOff 是什么？
- ImagePullBackOff 怎么解决？
- 如何系统排查 Service、Ingress、DNS 和网络问题？
- 如何利用 kubectl 工具快速定位问题？

------

# 第一节：建立正确的排查思维

很多新人排查问题像这样：

```
Pod 启动失败

↓

看日志

↓

没看懂

↓

重启

↓

还是失败
```

这是**没有方向**的排查。

企业更推荐使用"由外到内、由整体到局部"的方法。

一套简单有效的流程如下：

```
① 集群正常吗？
        │
        ▼
② Node 正常吗？
        │
        ▼
③ Pod 创建了吗？
        │
        ▼
④ Pod 为什么没运行？
        │
        ▼
⑤ 容器为什么退出？
        │
        ▼
⑥ Service 正常吗？
        │
        ▼
⑦ Ingress 正常吗？
        │
        ▼
⑧ 应用日志怎么说？
```

注意：

**日志通常是最后几步，而不是第一步。**

因为如果 Pod 根本没有启动，日志可能什么都没有。

------

# 第二节：第一步——先看集群

永远先执行：

```
kubectl get nodes
```

例如：

```
NAME      STATUS
node-1    Ready
node-2    Ready
node-3    NotReady
```

如果：

Node 已经 `NotReady`。

那么：

Pod 无法正常调度、网络异常、存储异常，都有可能发生。

先解决 Node，再看应用。

------

# 第三节：第二步——查看 Pod 状态

这是最重要的命令：

```
kubectl get pods
```

例如：

```
NAME                 STATUS
order-api-xxx        Running
redis-0              Running
frontend-xxx         Pending
```

这里的 **STATUS** 非常关键。

不同状态代表不同问题。

| 状态             | 含义                 |
| ---------------- | -------------------- |
| Running          | 正常运行             |
| Pending          | 等待调度             |
| CrashLoopBackOff | 启动后不断崩溃       |
| ImagePullBackOff | 镜像拉取失败         |
| ErrImagePull     | 拉取镜像出错         |
| Completed        | 已完成（常见于 Job） |
| Terminating      | 正在删除             |

很多时候，仅仅看到状态，就已经能缩小排查范围。

------

# 第四节：Pending 怎么排查？

很多新人看到：

```
Pending
```

就去：

```
kubectl logs
```

其实：

这是没有意义的。

因为：

Pod 还没启动。

没有日志。

正确做法：

```
kubectl describe pod <pod-name>
```

重点看：

```
Events:
```

例如：

```
0/3 nodes are available:
Insufficient memory.
```

这表示：

**集群内没有足够内存。**

或者：

```
persistentvolumeclaim not found
```

说明：

PVC 没有准备好。

再例如：

```
node(s) had taint
```

说明：

Node 有污点（Taint），当前 Pod 没有对应的容忍（Toleration）。

> **经验：Pending 问题，80% 都能在 `kubectl describe pod` 的 Events 中找到线索。**

------

# 第五节：CrashLoopBackOff 是什么？

这是生产环境最常见的问题之一。

状态：

```
CrashLoopBackOff
```

意思不是：

> Pod 创建失败。

而是：

> **Pod 能启动，但是应用启动后马上退出，然后 Kubernetes 不断重启它。**

例如：

```
启动

↓

崩溃

↓

重启

↓

再次崩溃

↓

再次重启
```

于是：

进入：

```
CrashLoopBackOff
```

------

## 常见原因

例如：

ASP.NET Core：

```
数据库连接字符串错误

Redis 无法连接

配置文件缺失

端口配置错误

程序启动异常
```

这时候。

就应该：

查看：

```
kubectl logs <pod-name>
```

如果已经重启很多次：

```
kubectl logs --previous <pod-name>
```

查看上一次崩溃前的日志。

------

# 第六节：ImagePullBackOff

状态：

```
ImagePullBackOff
```

表示：

**镜像没有拉下来。**

常见原因：

- 镜像名称写错
- Tag 不存在
- 私有镜像仓库没有登录
- 网络无法访问镜像仓库

查看：

```
kubectl describe pod <pod-name>
```

例如：

```
Failed to pull image
```

或者：

```
manifest unknown
```

通常已经说明问题。

------

# 第七节：Pod Running，但访问失败怎么办？

很多新人看到：

```
Running
```

就认为：

没问题。

其实：

未必。

例如：

```
Browser

↓

Ingress

↓

Service

↓

Pod
```

任何一层出问题。

都会导致：

访问失败。

因此。

要按链路逐层检查。

------

## 第一步：检查 Service

查看：

```
kubectl get svc
```

然后：

```
kubectl describe svc order-api
```

重点关注：

```
Selector
```

是否正确。

------

## 第二步：检查 Endpoint

执行：

```
kubectl get endpoints
```

例如：

```
NAME         ENDPOINTS
order-api    10.244.1.15:8080
```

如果：

显示：

```
<none>
```

说明：

Service 没有找到 Pod。

通常原因：

**Label 和 Selector 不匹配。**

这是非常经典的问题。

------

# 第八节：Ingress 无法访问怎么办？

排查顺序建议如下：

① Ingress 是否创建：

```
kubectl get ingress
```

② 查看详情：

```
kubectl describe ingress
```

③ 检查：

- Host 是否正确？
- Path 是否正确？
- Backend Service 是否存在？

④ 最后确认：

Ingress Controller 是否正常运行。

例如：

```
kubectl get pods -n ingress-nginx
```

如果 Controller 本身异常，所有 Ingress 都不会工作。

------

# 第九节：DNS 问题如何排查？

假设：

API：

访问：

Redis：

```
redis.default.svc.cluster.local
```

结果：

报错：

```
Name or service not known
```

先进入：

Pod：

```
kubectl exec -it <pod-name> -- sh
```

然后：

测试：

```
nslookup redis
```

如果：

解析失败。

说明：

可能是：

- Service 不存在
- CoreDNS 异常
- Namespace 写错

------

# 第十节：探针导致不断重启

很多生产事故，并不是应用崩了。

而是：

探针配置错误。

例如：

```
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

但是：

应用真正提供的是：

```
/healthz
```

于是：

Kubernetes：

一直认为：

应用：

"不健康"。

不断：

重启。

所以。

如果 Pod 不断重启，一定要检查：

```
kubectl describe pod
```

看看：

Events：

里面：

有没有：

Probe failed。

------

# 第十一节：生产环境最常用的排查命令

下面这些命令，建议熟练掌握。

| 命令                      | 用途                                  |
| ------------------------- | ------------------------------------- |
| `kubectl get pods`        | 查看 Pod 状态                         |
| `kubectl describe pod`    | 查看详细信息和 Events                 |
| `kubectl logs`            | 查看容器日志                          |
| `kubectl logs --previous` | 查看上一次崩溃日志                    |
| `kubectl get svc`         | 查看 Service                          |
| `kubectl get endpoints`   | 查看 Service 是否找到 Pod             |
| `kubectl get ingress`     | 查看 Ingress                          |
| `kubectl get events`      | 查看集群事件                          |
| `kubectl exec`            | 进入容器排查                          |
| `kubectl top pod`         | 查看资源使用（需安装 Metrics Server） |

------

# 第十二节：建立自己的排查流程

建议形成固定思维：

```
用户反馈异常
        │
        ▼
Pod 是否 Running？
        │
        ▼
查看 describe
        │
        ▼
查看 Events
        │
        ▼
查看 Logs
        │
        ▼
检查 Service
        │
        ▼
检查 Endpoints
        │
        ▼
检查 Ingress
        │
        ▼
检查 DNS
        │
        ▼
检查应用代码
```

这样。

即使面对陌生项目，也能有条不紊地分析。

------

# 第十三节：一个完整案例

假设用户反馈：

```
https://api.company.com/api/order
```

返回：

```
502 Bad Gateway
```

推荐排查顺序：

1. `kubectl get pods` —— Pod 是否 Running？
2. `kubectl logs` —— 应用是否启动成功？
3. `kubectl get svc` —— Service 是否存在？
4. `kubectl get endpoints` —— 是否有后端 Pod？
5. `kubectl describe ingress` —— 是否指向正确 Service？
6. 查看 Ingress Controller 日志 —— 是否转发失败？
7. 如有需要，再进入 Pod 测试应用监听端口。

这个流程比"想到什么查什么"更高效，也更适合团队协作。

------

# 本章总结（建议牢记）

请记住 Kubernetes 故障排查最重要的几点：

1. **先看资源状态，再看日志，不要一开始就盲目查看日志。**
2. **`kubectl describe pod` 和 `Events` 是定位 Pending、调度失败、探针失败的重要工具。**
3. **`CrashLoopBackOff` 表示应用不断崩溃重启，`ImagePullBackOff` 表示镜像拉取失败。**
4. **Service 无法访问时，重点检查 Selector 和 Endpoints。**
5. **Ingress 问题需要同时检查 Ingress 资源、Ingress Controller 和后端 Service。**
6. **建立固定的排查流程，比记忆大量命令更重要。**

------

# 到这里，你已经具备了基础生产故障排查能力

你已经学会了如何从：

```
用户报错
    │
    ▼
Pod
    │
    ▼
Service
    │
    ▼
Ingress
    │
    ▼
日志
    │
    ▼
根因定位
```

一步步缩小问题范围，而不是靠猜测。

------

# 下一章预告：Kubernetes 生产最佳实践（Best Practices）

下一章，我们将总结企业中最常见的 Kubernetes 使用规范，包括：

- 为什么不能使用 `latest` 标签？
- 为什么一定要设置 `resources.requests` 和 `resources.limits`？
- 为什么必须配置 Readiness/Liveness Probe？
- 为什么 Deployment 副本数建议至少为 2？
- 为什么生产环境尽量不要直接使用 `NodePort`？
- 为什么需要 PodDisruptionBudget（PDB）？
- 如何设计适合生产环境的 Helm Chart？

这一章会把前面所有知识串联成一套**企业级 Kubernetes 使用规范**，帮助你从"会用 Kubernetes"迈向"正确地用 Kubernetes"。

# 第三阶段 第二章：Kubernetes 生产最佳实践（Best Practices）

> **建议把这一章作为以后每个项目上线前的检查清单。**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，是很多工作 **3~5 年的 Kubernetes 工程师** 和 **刚入门的新手** 最大的区别。

因为前面我们学习的是：

> **Kubernetes 能做什么。**

而这一章学习的是：

> **生产环境应该怎么做。**

很多线上事故，并不是 Kubernetes 本身的问题，而是**没有遵循最佳实践**。

例如：

- 使用 `latest` 标签导致版本不可追踪
- 没有限制内存，结果一个 Pod 吃光整台 Node
- 没有配置 Readiness Probe，导致流量进入尚未启动完成的应用
- 所有 Pod 都部署在同一个 Node，上线时整个服务一起挂掉

这些问题，在企业里都是真实发生过的。

# 本章学习目标

学习完本章，你应该能够回答：

- 为什么不能使用 `latest` 镜像标签？
- 为什么一定要配置资源限制？
- 为什么 Readiness Probe 比 Liveness Probe 更重要？
- 为什么 Deployment 通常至少部署 2 个副本？
- 为什么需要滚动更新？
- 为什么生产环境推荐使用 HPA？
- 为什么 Helm Chart 要保持"可配置，而不是过度配置"？
- 如何设计一个适合生产环境的 Kubernetes 应用？

------

# 第一节：不要使用 latest 标签

很多新人喜欢：

```
image: mycompany/order-api:latest
```

看起来：

很方便。

实际上：

这是生产环境的大忌。

为什么？

假设：

昨天：

```
latest
↓

v1.0.0
```

今天：

```
latest
↓

v1.0.1
```

明天：

```
latest
↓

v1.1.0
```

请问：

线上现在运行的是哪个版本？

不知道。

如果回滚：

也不知道应该回滚到哪里。

------

## 正确做法

始终使用明确版本：

```
image:
  repository: mycompany/order-api
  tag: "1.0.8"
```

或者：

```
20260804.3
```

甚至：

直接使用：

Git Commit SHA：

```
4b7d2c1
```

这样：

任何时候。

都知道：

运行的是哪个版本。

------

# 第二节：一定要配置 resources

很多新人：

Deployment：

里面：

没有：

```
resources:
```

例如：

```
containers:

- name: api
```

结束。

这是非常危险的。

------

为什么？

假设：

你的程序：

突然：

内存泄漏。

没有限制。

最终：

```
Pod

↓

8GB

↓

16GB

↓

32GB
```

最后。

整个：

Node：

OOM（内存耗尽）。

其它：

Pod：

一起：

受影响。

------

## 正确做法

至少：

配置：

```
resources:

  requests:

    cpu: 200m

    memory: 256Mi

  limits:

    cpu: 1000m

    memory: 1Gi
```

理解：

### requests

表示：

> **至少给我这么多资源。**

Scheduler：

调度：

Pod：

时。

会：

参考：

requests。

------

### limits

表示：

> **最多只能使用这么多。**

超过：

限制。

可能会被限速（CPU）或因内存不足而被终止（Memory）。

------

## 一个生活例子

可以把：

Node：

理解成：

酒店。

requests：

就是：

提前订房。

limits：

就是：

最多：

住：

几个人。

------

# 第三节：Readiness Probe 比 Liveness Probe 更重要

很多教程：

只介绍：

Liveness。

实际上。

企业更关注：

Readiness。

------

为什么？

假设：

ASP.NET Core：

启动：

需要：

20 秒。

但是：

Pod：

刚创建：

2 秒。

Service：

已经：

把：

流量：

转过来。

结果：

用户：

收到：

```
502
```

应用：

其实：

没坏。

只是：

没准备好。

------

正确：

配置：

```
readinessProbe:

  httpGet:

    path: /health

    port: 8080
```

只有：

Readiness：

成功。

Service：

才会：

把：

请求：

发送：

给：

这个：

Pod。

------

Liveness：

则负责：

判断：

程序：

是不是：

已经：

"卡死"。

两者作用完全不同。

------

# 第四节：Deployment 至少两个副本

很多新人：

生产：

环境：

这样：

写：

```
replicas: 1
```

如果：

Pod：

升级。

或者：

Node：

故障。

服务：

立即：

中断。

------

推荐：

```
replicas: 2
```

甚至：

```
3

5

7
```

这样：

Rolling Update：

过程中。

始终：

有人：

提供：

服务。

------

# 第五节：使用 Rolling Update，而不是 Recreate

Deployment：

默认：

策略：

就是：

Rolling Update。

意思：

不是：

全部：

删掉。

再：

创建。

而是：

一个一个：

替换。

例如：

```
旧 Pod1

旧 Pod2

旧 Pod3
```

升级：

以后：

```
新 Pod1

旧 Pod2

旧 Pod3
```

再：

```
新 Pod1

新 Pod2

旧 Pod3
```

最后：

全部：

完成。

整个：

过程：

业务：

基本：

不会：

中断。

------

# 第六节：配置 HPA，而不是固定副本数

假设：

平时：

每天：

1000：

请求。

晚上：

双十一。

变成：

100000：

请求。

如果：

一直：

```
replicas: 2
```

很可能：

撑不住。

所以：

生产：

推荐：

HPA：

```
CPU

70%

↓

自动：

扩容
```

例如：

```
2

↓

4

↓

8
```

流量：

下降：

以后：

自动：

缩回：

2。

------

# 第七节：Pod 不要全部放在同一个 Node

假设：

三个：

Pod：

全部：

调度：

到了：

```
Node1
```

突然：

Node1：

断电。

整个：

服务：

全部：

消失。

------

推荐：

通过：

**Pod Anti-Affinity（反亲和性）**：

让：

Pod：

尽量：

分散。

例如：

```
Node1

Pod1

──────────

Node2

Pod2

──────────

Node3

Pod3
```

这样：

即使：

一个：

Node：

故障。

服务：

仍然：

可用。

> 我们后面会专门讲解 Affinity（亲和性）和 Anti-Affinity（反亲和性）的配置。

------

# 第八节：不要把所有配置写死

错误：

例如：

```
host: api.company.com

replicas: 2

cpu: 500m
```

全部：

固定。

以后：

开发：

测试：

生产：

全部：

改：

模板。

------

正确：

全部：

放：

values：

```
replicaCount:

image:

resources:

ingress:

autoscaling:
```

这样：

模板：

永远：

只有：

一份。

------

# 第九节：不要把密码放进 Git

例如：

```
database:

  password: 123456
```

千万：

不要：

提交。

Git。

正确：

做法：

生产环境可采用：

- Kubernetes Secret
- 外部密钥管理（如云厂商 Secret Manager）
- External Secrets Operator 等方案

这样既能避免敏感信息泄露，也方便统一轮换密码。

------

# 第十节：健康检查一定要有

建议：

至少：

提供：

```
/health
```

ASP.NET Core：

可以：

实现：

```
Healthy

Unhealthy
```

然后：

```
livenessProbe

readinessProbe
```

都：

可以：

使用。

不要：

使用：

首页：

```
/
```

作为：

健康检查。

因为：

首页：

可能：

包含：

大量：

业务：

逻辑。

------

# 第十一节：建立统一的标签（Labels）

建议：

所有：

资源：

统一：

Labels。

例如：

```
labels:

  app: order-api

  version: v1

  env: prod

  team: backend
```

这样：

以后：

查询：

非常：

方便。

例如：

```
kubectl get pods -l env=prod
```

或者：

```
kubectl get pods -l team=backend
```

标签不仅用于查询，也是很多策略（NetworkPolicy、监控、成本统计等）的基础。

------

# 第十二节：不要忽略 Namespace

很多新人：

全部：

部署：

```
default
```

以后：

几十：

项目。

全部：

一起。

非常：

混乱。

推荐：

```
dev

test

staging

prod
```

甚至：

```
order-system

payment-system
```

不同系统、不同环境进行隔离。

------

# 第十三节：生产环境上线前检查清单（Checklist）

建议每次上线前确认：

| 检查项                               | 是否完成 |
| ------------------------------------ | -------- |
| 镜像使用固定版本 Tag                 | ✅        |
| 设置 `resources.requests` / `limits` | ✅        |
| 配置 Readiness Probe                 | ✅        |
| 配置 Liveness Probe                  | ✅        |
| Deployment 至少 2 个副本             | ✅        |
| 使用 Rolling Update                  | ✅        |
| 使用 ConfigMap 管理普通配置          | ✅        |
| 使用 Secret 管理敏感信息             | ✅        |
| 设置 Labels                          | ✅        |
| 使用独立 Namespace                   | ✅        |
| 配置日志与监控                       | ✅        |
| HPA（适用时）已配置                  | ✅        |

这份清单可以作为团队 Code Review 或上线评审的基础。

------

# 第十四节：企业部署一个 ASP.NET Core 服务的推荐架构

综合前面的内容，一个推荐的部署方案如下：

```
                Internet
                    │
                    ▼
             Ingress Controller
                    │
                    ▼
                Service
                    │
                    ▼
          Deployment（3 Replicas）
          ┌────────┼────────┐
          ▼        ▼        ▼
        Pod1     Pod2     Pod3
          │        │        │
          ├────────┼────────┤
          ▼
      ConfigMap + Secret
          │
          ▼
      ASP.NET Core API
          │
          ▼
     Redis / MySQL 等后端服务
```

在这个架构基础上，再结合：

- Helm 管理部署
- Prometheus + Grafana 监控
- Loki 收集日志
- OpenTelemetry + Tempo 链路追踪

就已经非常接近现代企业生产环境。

------

# 本章总结（建议牢记）

请记住 Kubernetes 生产最佳实践中最重要的几点：

1. **永远不要使用 `latest` 作为生产镜像标签。**
2. **始终配置 `resources.requests` 和 `resources.limits`。**
3. **Readiness Probe 决定是否接收流量，Liveness Probe 决定是否需要重启。**
4. **生产环境 Deployment 建议至少 2 个副本，并采用 Rolling Update。**
5. **敏感信息不要提交到 Git，应使用 Secret 或专业密钥管理方案。**
6. **统一使用 Namespace、Labels 和 Helm 管理应用。**
7. **上线前使用固定的 Checklist，可以显著降低生产事故风险。**

------

# 到这里，你已经具备了企业级 Kubernetes 部署思维

现在，你已经不仅知道**如何部署**，更知道**如何安全、稳定地部署**。

这是从"能跑起来"到"能长期稳定运行"的关键一步。

------

# 下一章预告：Kubernetes 网络原理（深度篇）

前面我们一直在使用：

- Pod 可以互相访问
- Service 可以负载均衡
- Ingress 可以对外暴露

但是，我们还没有真正回答这些问题：

- 为什么每个 Pod 都有自己的 IP？
- 为什么 Pod 跨 Node 还能直接通信？
- Service 为什么会有一个虚拟 IP（ClusterIP）？
- kube-proxy 到底做了什么？
- iptables 和 IPVS 有什么区别？
- CNI（Container Network Interface）到底是什么？
- Calico、Flannel、Cilium 分别解决什么问题？

下一章开始，我们将深入学习 **Kubernetes 网络原理**。这一部分是理解 Kubernetes 底层机制、排查复杂网络问题和学习 Service Mesh（如 Istio）的基础，也是中高级 Kubernetes 工程师必须掌握的核心知识。

# 第三阶段 第三章：Kubernetes 网络原理（上）

> **本章重点：理解 Kubernetes 网络设计思想，而不是死记配置。**

------

提示词：

```
开始下一章
```

正文：

非常好。

从这一章开始，我们进入 **Kubernetes 底层原理**。

这一章，也是很多 Kubernetes 初学者觉得最难的一章。

因为前面我们一直在"使用 Kubernetes"，而现在开始，我们要学习：

> **Kubernetes 为什么能够做到这些？**

如果你真正理解了 Kubernetes 网络，那么以后学习：

- Istio
- Service Mesh
- Gateway API
- NetworkPolicy
- Cilium
- eBPF

都会轻松很多。

这一章内容会比较长，我会尽量用大量生活中的例子来解释。

# 本章学习目标

学习完本章，你应该能够回答：

- Kubernetes 网络为什么这么复杂？
- 为什么每个 Pod 都有自己的 IP？
- Pod 为什么可以直接互相访问？
- Pod 跨 Node 为什么还能通信？
- CNI 到底是什么？
- Flannel、Calico、Cilium 分别负责什么？
- Kubernetes 网络模型到底是什么？

------

# 第一节：为什么 Kubernetes 网络这么复杂？

先来看传统服务器。

假设只有两台机器：

```
Client

    │

    ▼

Server
```

服务器：

只有：

一个：

IP。

例如：

```
192.168.1.100
```

所有程序：

都运行在：

这个：

IP：

里面。

例如：

```
Nginx

ASP.NET Core

Redis

MySQL
```

大家：

共享：

服务器网络。

是不是很简单？

------

后来。

Docker 出现了。

Docker 为了隔离容器。

给每个容器：

创建：

独立网络。

例如：

```
Docker Host

172.17.0.2

Container A

────────────

172.17.0.3

Container B
```

容器之间：

已经：

拥有：

自己的：

IP。

------

但是。

Kubernetes：

规模更大。

例如：

```
Node1

Pod A

Pod B

────────────

Node2

Pod C

Pod D
```

现在：

问题来了。

Pod A：

如何：

访问：

Pod C？

它们：

已经：

不在：

同一台：

服务器。

这就是：

Kubernetes：

要解决：

的：

核心问题。

------

# 第二节：Kubernetes 网络模型

Kubernetes 从诞生开始，就提出了一条非常重要的设计原则：

> **任何 Pod 都应该可以直接访问任何其他 Pod，不需要 NAT。**

第一次看到的人，通常都会觉得：

"真的吗？"

答案：

是的。

整个 Kubernetes 网络，就是围绕这一原则设计的。

官方网络模型可以总结为三条规则：

1. **每个 Pod 都拥有唯一的 IP。**
2. **任何 Pod 都可以直接访问任何其他 Pod。**
3. **Node 和 Pod 之间可以直接通信。**

请把这三条牢记。

以后学习 CNI、Calico、Service Mesh 都会围绕它们展开。

------

# 第三节：为什么每个 Pod 都有自己的 IP？

很多新人都会问：

"为什么不让所有 Pod 共用 Node 的 IP？"

例如：

```
Node

192.168.1.100

↓

Pod A

Pod B

Pod C
```

原因很简单。

如果所有 Pod 共用 IP。

那就只能依靠端口区分。

例如：

```
192.168.1.100:8080

192.168.1.100:8081

192.168.1.100:8082
```

这样会带来很多问题：

- 端口容易冲突
- 调度困难
- 服务迁移复杂
- Service 很难实现统一抽象

于是 Kubernetes 做了一个大胆的决定：

> **把每个 Pod 都当成一台独立的主机。**

例如：

```
Pod A

10.244.1.5

────────────

Pod B

10.244.1.6

────────────

Pod C

10.244.2.8
```

从应用程序角度来看：

你甚至可以认为：

Pod 就是一台小型服务器。

------

## 一个生活中的例子

假设一栋办公楼：

错误设计：

```
公司A

电话：

10086-1

公司B

电话：

10086-2

公司C

电话：

10086-3
```

所有人：

共用：

一个号码。

管理很麻烦。

更好的方式：

```
公司A

10001

──────────

公司B

10002

──────────

公司C

10003
```

每家公司：

都有：

自己的号码。

Pod IP：

就是：

这个意思。

------

# 第四节：Pod IP 是谁分配的？

很多新人以为：

是：

Kubernetes。

其实：

不是。

真正负责分配 IP 的，是：

> **CNI 插件（Container Network Interface）。**

Kubernetes：

只负责：

告诉：

CNI：

```
我要创建：

一个：

Pod
```

然后。

CNI：

完成：

下面：

这些：

工作：

```
创建网络设备

↓

分配 IP

↓

配置路由

↓

加入网络
```

所以。

可以理解为：

```
Kubernetes

↓

提出需求

────────────

CNI

↓

真正干活
```

这也是为什么：

安装 Kubernetes 后，通常还需要安装一个网络插件。

------

# 第五节：什么是 CNI？

CNI 全称：

**Container Network Interface**。

一句话：

> **CNI 是 Kubernetes 与网络插件之间的一套标准接口。**

它并不是某一个软件。

更像是：

USB 接口。

例如：

电脑：

有：

USB。

你可以插：

```
鼠标

键盘

U盘
```

接口：

一样。

设备：

不同。

Kubernetes：

也是一样。

它只要求：

> **"只要你符合 CNI 标准，就可以接入。"**

因此：

可以选择不同的网络实现。

例如：

```
Flannel

Calico

Cilium

Weave
```

这些：

都是：

CNI 插件。

------

# 第六节：最常见的 CNI 插件

目前企业最常见的有三个。

------

## Flannel

一句话：

> **最简单，最容易上手。**

特点：

- 安装简单
- 学习成本低
- 功能相对较少
- 适合学习、小型集群

很多教学环境都会选择 Flannel。

------

## Calico

一句话：

> **企业使用最广泛的网络方案之一。**

特点：

- 性能优秀
- 支持 NetworkPolicy
- 路由能力强
- 大规模集群表现稳定

很多生产环境都会选择 Calico。

------

## Cilium

一句话：

> **基于 eBPF 的新一代 Kubernetes 网络方案。**

特点：

- 性能非常高
- 支持高级网络策略
- 可观测性强
- 与 Service Mesh、安全能力结合紧密

近年来越来越多新项目开始采用 Cilium。

------

# 第七节：Pod 为什么能跨 Node 通信？

来看下面这个例子：

```
Node1

Pod A

10.244.1.2

────────────

Node2

Pod B

10.244.2.3
```

Pod A：

要访问：

Pod B。

它并不知道：

Pod B 在哪台 Node。

它只知道：

```
10.244.2.3
```

于是：

数据包：

发送：

出去。

接下来：

由：

**CNI 配置的路由**负责找到正确的 Node。

可以把它理解成：

快递系统。

你只写：

```
收件地址
```

并不用关心：

快递：

经过：

哪些：

中转站。

Kubernetes 网络也是如此。

------

# 第八节：Node 为什么也能访问 Pod？

很多运维命令都依赖这一能力。

例如：

```
kubectl exec
```

或者：

```
kubectl logs
```

Node 必须能够直接与 Pod 通信。

否则：

健康检查、日志采集、监控采集等都会受到影响。

因此：

Kubernetes 网络模型要求：

> **Node 和 Pod 必须互通。**

------

# 第九节：Pod IP 是固定的吗？

答案：

**不是。**

例如：

今天：

```
Order API

10.244.1.5
```

明天：

Pod 重建：

可能变成：

```
10.244.3.18
```

所以：

千万不要：

在代码里：

写：

```
10.244.1.5
```

正确做法：

永远：

访问：

Service。

例如：

```
order-api.default.svc.cluster.local
```

或者：

```
http://order-api
```

Pod IP 是临时的。

Service 才是稳定入口。

------

# 第十节：为什么不能直接依赖 Pod IP？

假设：

Deployment：

升级：

```
旧 Pod

↓

删除
```

新的：

Pod：

创建：

```
新 IP
```

如果客户端：

保存：

旧 IP。

立即：

失败。

因此：

Kubernetes：

规定：

> **Pod 可以变化，Service 保持稳定。**

这也是为什么 Kubernetes 中，大多数服务之间都通过 Service 名称通信，而不是直接记住 Pod 地址。

------

# 第十一节：本章知识关系图

请把下面这张图记住：

```
                Kubernetes
                     │
                     ▼
          创建 Pod 的请求
                     │
                     ▼
                 CNI 插件
                     │
      ┌──────────────┼──────────────┐
      ▼              ▼              ▼
  分配 Pod IP     配置网络      设置路由
                     │
                     ▼
            Pod 之间互相通信
```

以后遇到网络问题，可以先问自己：

- Pod 有没有 IP？
- CNI 是否正常？
- 路由是否正确？
- 是否通过 Service 通信？

------

# 第十二节：本章总结（建议牢记）

请记住 Kubernetes 网络最重要的几点：

1. **Kubernetes 的核心设计目标是：任何 Pod 都可以直接访问任何其他 Pod。**
2. **每个 Pod 都拥有独立 IP，把 Pod 看成一台"小服务器"最容易理解。**
3. **Pod IP 由 CNI 插件分配，不是 Kubernetes 自己分配。**
4. **CNI 是标准接口，Flannel、Calico、Cilium 都是它的实现。**
5. **Pod IP 是临时的，不应该写死在配置或代码中。**
6. **服务之间应通过 Service 名称访问，而不是直接访问 Pod IP。**

------

# 到这里，你已经理解了 Kubernetes 网络设计思想

这是学习 Kubernetes 网络的第一步，也是最重要的一步。

后面的内容，我们将开始真正进入底层实现。

------

# 下一章预告：Kubernetes 网络原理（下）——Service、kube-proxy、iptables 与 IPVS

下一章，我们将回答 Kubernetes 网络中最经典的几个问题：

- 为什么 **Service 明明没有 Pod，却可以访问？**
- ClusterIP 到底是不是一个真实存在的 IP？
- kube-proxy 究竟做了什么？
- iptables 是如何把请求转发到 Pod 的？
- IPVS 为什么性能更高？
- 一个请求从浏览器进入集群，到达 Pod，中间究竟经历了哪些步骤？

学完下一章，你将真正理解 **Service 的底层实现原理**，也会明白为什么 Kubernetes 能实现稳定的服务发现与负载均衡。这也是排查网络故障和理解 Ingress、Service Mesh 的重要基础。

# 第三阶段 第四章：Kubernetes 网络原理（下）——Service、kube-proxy、iptables 与 IPVS

> **这一章是 Kubernetes 网络的核心原理。**

------

提示词：

```
开始下一章
```

正文：

非常好。

这一章，是整个 Kubernetes 网络中**最重要的一章**。

很多 Kubernetes 面试，都会问下面几个问题：

> **Service 为什么有 IP？**

> **ClusterIP 到底是真实存在的吗？**

> **kube-proxy 到底干了什么？**

> **为什么 Pod 重建以后，Service 不需要修改？**

如果这一章真正理解了，你对 Kubernetes 网络的理解就已经超过大多数只会部署应用的人。

# 本章学习目标

学习完本章，你应该能够回答：

- Service 为什么可以访问？
- ClusterIP 是什么？
- 为什么 ClusterIP 不是一个真实的 IP？
- kube-proxy 到底负责什么？
- iptables 和 IPVS 的区别是什么？
- Service 是如何实现负载均衡的？
- 一个请求从客户端到 Pod 的完整路径是什么？

------

# 第一节：先回顾一个问题

假设：

你的 Deployment：

有三个 Pod：

```
Order API

Pod1

10.244.1.2

────────────

Pod2

10.244.2.8

────────────

Pod3

10.244.3.15
```

现在：

客户端：

应该访问哪个 IP？

Pod1？

Pod2？

还是 Pod3？

答案：

**都不应该。**

因为：

Pod 随时可能：

- 删除
- 重建
- 扩容
- 缩容

IP 都会变化。

所以 Kubernetes 引入了一个中间层：

```
Client

    │

    ▼

Service

    │

    ▼

Pods
```

------

# 第二节：Service 到底是什么？

很多新人认为：

Service 是一个程序。

其实不是。

也不是一个容器。

一句话：

> **Service 是 Kubernetes 提供的一个稳定访问入口。**

例如：

```
Service

名称：

order-api

IP：

10.96.0.15
```

客户端：

永远：

访问：

```
10.96.0.15
```

或者：

```
http://order-api
```

至于：

后面：

到底：

是哪一个：

Pod。

客户端：

不用关心。

------

## 一个生活中的例子

假设一家银行：

有三个柜台：

```
柜台1

柜台2

柜台3
```

如果客户直接找柜台：

柜台换位置就麻烦了。

于是银行设置：

```
取号机
```

客户：

永远：

先到：

取号机。

然后：

取号机：

安排：

去：

哪个：

柜台。

这里：

Service：

就是：

取号机。

Pod：

就是：

真正办理业务的柜台。

------

# 第三节：ClusterIP 是什么？

创建：

Service：

以后：

例如：

```
kind: Service

spec:

  type: ClusterIP
```

Kubernetes：

会：

分配：

一个：

IP。

例如：

```
10.96.0.15
```

这个：

IP：

就是：

ClusterIP。

以后：

整个：

集群：

都可以：

访问：

```
10.96.0.15
```

------

# 第四节：ClusterIP 是真实存在的吗？

这是 Kubernetes 面试经典问题。

很多新人回答：

> "当然是真实存在。"

实际上：

**不是。**

ClusterIP：

并没有：

真正：

绑定：

到：

某个：

网卡。

没有：

哪个：

Pod：

拥有：

它。

没有：

哪个：

Node：

真正：

监听：

它。

它更像是：

一个：

**虚拟 IP（VIP）**。

------

## 一个生活中的例子

例如：

客服电话：

```
400-800-1234
```

请问：

这个号码：

属于：

哪个：

客服？

没有。

真正接电话的：

可能：

1000：

个人。

客服电话：

只是：

一个：

统一入口。

ClusterIP：

就是：

这个意思。

------

# 第五节：既然不存在，那为什么还能访问？

答案：

因为：

有：

```
kube-proxy
```

------

# 第六节：kube-proxy 是什么？

一句话：

> **kube-proxy 是 Kubernetes 的 Service 转发器。**

每一个：

Node：

都会：

运行：

一个：

kube-proxy。

例如：

```
Node1

────────────

kube-proxy

────────────

Pods
```

它：

负责：

监听：

Service：

变化。

例如：

新增：

```
Service

order-api
```

kube-proxy：

马上：

更新：

自己的：

转发表。

------

# 第七节：请求到底发生了什么？

假设：

客户端：

访问：

```
10.96.0.15
```

实际上：

流程：

如下：

```
Client

        │

访问：

10.96.0.15

        │

        ▼

iptables / IPVS

        │

选择：

Pod

        │

        ▼

10.244.2.8
```

所以：

真正：

收到：

请求：

的是：

Pod。

不是：

ClusterIP。

ClusterIP：

只是：

入口。

------

# 第八节：iptables 是什么？

Linux：

本身：

就有：

一个：

网络规则系统：

```
iptables
```

可以理解成：

一本：

交通规则。

例如：

```
如果：

访问：

10.96.0.15

↓

转发：

到：

10.244.2.8
```

再例如：

```
下一次：

↓

转发：

10.244.3.15
```

这些：

规则：

都是：

kube-proxy：

写进去：

的。

所以：

真正：

干活：

的是：

Linux。

不是：

Kubernetes。

------

## 一个生活中的例子

想象一个大型商场。

顾客来到总服务台（ClusterIP）。

服务台不会亲自卖东西。

而是根据规则：

```
今天：

去：

柜台 A

下一位：

去：

柜台 B

再下一位：

去：

柜台 C
```

这些规则：

就像：

iptables。

------

# 第九节：Service 如何实现负载均衡？

假设：

Deployment：

三个：

Pod：

```
Pod1

10.244.1.2

────────────

Pod2

10.244.2.8

────────────

Pod3

10.244.3.15
```

iptables：

规则：

可能：

如下：

```
请求1

↓

Pod1

────────────

请求2

↓

Pod2

────────────

请求3

↓

Pod3

────────────

请求4

↓

Pod1
```

这就是：

最基本：

的：

负载均衡。

需要注意的是，**并不是严格轮询（Round Robin）**。在 iptables 模式下，本质上是通过 Linux 内核的规则和连接跟踪（conntrack）实现概率分配和连接保持，因此实际流量可能不会完全平均。

------

# 第十节：为什么后来出现了 IPVS？

随着集群越来越大。

例如：

```
5000

Service
```

iptables：

规则：

会：

越来越：

多。

例如：

```
100000

规则
```

查找：

效率：

下降。

于是：

Linux：

推出：

```
IPVS
```

一句话：

> **IPVS 是 Linux 内核提供的高性能四层负载均衡。**

它不像 iptables 那样逐条匹配规则，而是使用更高效的数据结构来管理转发表，因此在大量 Service 和连接的场景下性能更好。

------

# 第十一节：iptables 与 IPVS 的区别

| 对比项         | iptables       | IPVS               |
| -------------- | -------------- | ------------------ |
| 实现方式       | Netfilter 规则 | Linux IPVS 模块    |
| 数据结构       | 规则链         | 哈希表等高效结构   |
| 大规模集群性能 | 一般           | 更好               |
| 配置复杂度     | 较低           | 略高               |
| 历史使用       | 较早默认方案   | 长期作为高性能方案 |

> **补充说明：** 从较新的 Kubernetes 版本开始，社区越来越多地推荐使用 **eBPF**（例如 Cilium）作为现代数据平面。在一些新建集群中，甚至可以完全绕过 kube-proxy（称为 *kube-proxy replacement*），获得更好的性能和可观测性。我们会在后续讲解 Cilium 和 eBPF 时详细介绍。

------

# 第十二节：Service 为什么不用修改？

假设：

今天：

Deployment：

升级：

```
旧 Pod

↓

10.244.1.2
```

删除。

新：

Pod：

创建：

```
10.244.6.20
```

发生：

什么？

其实：

Service：

不用：

改。

为什么？

因为：

Kubernetes：

自动：

更新：

```
Endpoints（或 EndpointSlice）
```

kube-proxy：

同步：

新的：

Pod。

客户端：

继续：

访问：

```
10.96.0.15
```

完全：

不知道：

后面：

发生：

什么。

这就是：

Service：

最大的价值。

------

# 第十三节：一次请求完整流程

假设：

浏览器：

访问：

```
http://order-api
```

整个：

过程：

如下：

```
Browser
    │
    ▼
DNS
    │
解析为：
order-api → 10.96.0.15
    │
    ▼
ClusterIP
    │
    ▼
kube-proxy
    │
    ▼
iptables / IPVS
    │
    ▼
选择一个 Pod
    │
    ▼
Order API
```

请把这张图牢牢记住。

这是 Kubernetes 网络最核心的一条链路。

------

# 第十四节：Endpoint 与 EndpointSlice

前面我们提到：

Service：

真正连接的是：

```
Pods
```

但：

Service：

并不会：

直接：

保存：

Pod。

它：

关联：

的是：

```
Endpoint
```

现代 Kubernetes 中，更推荐使用：

```
EndpointSlice
```

为什么？

假设：

有：

```
10000

Pods
```

如果：

全部：

放：

一个：

Endpoint。

对象：

会：

非常：

大。

于是：

拆分：

多个：

EndpointSlice。

例如：

```
EndpointSlice A

100 Pods

────────────

EndpointSlice B

100 Pods

────────────

EndpointSlice C

100 Pods
```

这样：

性能：

更好。

> 从 Kubernetes 较新的版本开始，**EndpointSlice 已经成为默认和推荐的后端实现**，Service、kube-proxy 等组件都会优先使用它。

------

# 第十五节：本章知识关系图

```
                Browser
                    │
                    ▼
               Service 名称
                    │
                    ▼
                 ClusterIP
                    │
                    ▼
              kube-proxy
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
    iptables                 IPVS
        │                       │
        └───────────┬───────────┘
                    ▼
              EndpointSlice
                    │
                    ▼
                  Pod
```

这张图建议你画在笔记里，后续很多知识都会围绕它展开。

------

# 第十六节：本章总结（建议牢记）

请记住 Service 网络最重要的几点：

1. **Service 是稳定访问入口，不是真正提供服务的程序。**
2. **ClusterIP 是虚拟 IP（VIP），不是绑定在某块网卡上的真实地址。**
3. **真正接收请求的是后端 Pod。**
4. **kube-proxy 负责维护 Service 到 Pod 的转发规则。**
5. **iptables 和 IPVS 都可以实现 Service 转发，IPVS 更适合大规模集群。**
6. **现代 Kubernetes 默认使用 EndpointSlice 管理 Service 后端。**
7. **Pod 可以随时变化，而 Service 地址保持稳定，这正是 Kubernetes 服务发现的核心思想。**

------

# 到这里，你已经掌握了 Kubernetes Service 的底层原理

现在你已经理解了：

```
Service
    │
ClusterIP（虚拟 IP）
    │
kube-proxy
    │
iptables / IPVS
    │
EndpointSlice
    │
Pod
```

这条链路是 Kubernetes 网络最核心的组成部分。

------

# 下一章预告：Kubernetes 网络原理（终章）——DNS、CoreDNS、Service Discovery 与完整请求链路

下一章，我们将把网络体系的最后一块拼图补上，包括：

- 为什么 `http://order-api` 就能访问？
- Kubernetes DNS 是如何工作的？
- CoreDNS 的职责是什么？
- FQDN（完全限定域名）是什么？
- 不同 Namespace 为什么会影响服务发现？
- 一个请求从浏览器到 Ingress，再到 Service，最终到 Pod 的完整网络链路是什么？

学完下一章，你将完整掌握 **Kubernetes 网络体系**，真正理解一次请求在集群中的完整流转过程，为后续学习 **Ingress Controller、Gateway API、Service Mesh（Istio）** 打下坚实基础。

# 第三阶段 第五章：Kubernetes 网络原理（终章）——DNS、CoreDNS、Service Discovery

> **本章重点：理解 Kubernetes 如何实现服务发现（Service Discovery）。**

------

提示词：

```
开始下一章
```

正文：

非常好。

这一章，我们将完成 **Kubernetes 网络** 的最后一块拼图。

前面两章，我们已经知道：

```
Pod
有自己的 IP
```

知道：

```
Service
提供稳定入口
```

知道：

```
kube-proxy
负责转发
```

但是，还有一个问题一直没有解释：

> **为什么代码里面写 `http://order-api` 就能访问？**

甚至很多人一直使用：

```
http://redis
```

却不知道：

为什么能找到 Redis。

答案就是：

> **Kubernetes DNS（CoreDNS）**

这一章学完以后，你就真正理解了 Kubernetes 网络的完整闭环。

# 本章学习目标

学习完本章，你应该能够回答：

- 什么是 Service Discovery（服务发现）？
- 为什么可以直接访问 `http://order-api`？
- CoreDNS 是什么？
- FQDN（完全限定域名）是什么？
- 不同 Namespace 为什么访问方式不同？
- DNS 查询完整流程是什么？
- 一次 HTTP 请求如何从浏览器最终到达 Pod？

------

# 第一节：什么是服务发现（Service Discovery）？

假设没有 DNS。

你的 Order API 想访问 Redis。

只能这样写：

```
10.96.0.25
```

如果：

Redis：

重新创建。

IP：

变了。

整个系统：

全部：

修改。

是不是非常痛苦？

于是。

Kubernetes 提供：

> **Service Discovery（服务发现）**

也就是：

**不用记 IP，只记名字。**

例如：

```
redis
```

以后：

真正访问：

哪个：

IP。

交给：

Kubernetes。

------

## 一个生活中的例子

你不会记：

```
朋友家的 GPS 坐标
```

而是：

记：

```
张三家
```

以后：

导航：

自动：

找到。

DNS：

就是：

导航。

------

# 第二节：DNS 在 Kubernetes 中负责什么？

假设：

有：

```
Service

order-api
```

对应：

```
ClusterIP

10.96.0.15
```

应用：

访问：

```
http://order-api
```

DNS：

返回：

```
10.96.0.15
```

然后：

继续：

访问：

Service。

整个过程：

```
order-api

↓

DNS

↓

10.96.0.15

↓

Service

↓

Pod
```

所以。

DNS：

只是：

负责：

名字：

解析。

真正：

处理：

请求：

的是：

Service。

------

# 第三节：CoreDNS 是什么？

很多新人认为：

DNS：

就是：

系统功能。

其实。

Kubernetes：

自己：

部署：

了一套：

DNS。

就是：

**CoreDNS**。

一句话：

> **CoreDNS 是 Kubernetes 集群中的 DNS 服务器。**

安装 Kubernetes 后。

一般都会看到：

```
kubectl get pods -n kube-system
```

例如：

```
coredns-xxxxx

Running
```

这就是：

整个：

集群：

负责：

解析：

Service 名称：

的：

组件。

------

# 第四节：CoreDNS 如何工作？

假设：

你的程序：

执行：

```
http://redis
```

发生：

什么？

流程：

如下：

```
ASP.NET Core

↓

查询：

redis

↓

CoreDNS

↓

返回：

10.96.0.25

↓

继续：

HTTP 请求
```

是不是：

其实：

跟：

Windows：

访问：

```
google.com
```

一样？

只是：

DNS：

服务器：

变成：

CoreDNS。

------

# 第五节：FQDN（完全限定域名）

很多人：

只知道：

```
http://redis
```

其实。

完整：

名字：

应该：

是：

```
redis.default.svc.cluster.local
```

它：

叫：

> **FQDN（Fully Qualified Domain Name，完全限定域名）**

拆开来看：

```
redis

↓

Service 名称

────────────

default

↓

Namespace

────────────

svc

↓

Service 类型

────────────

cluster.local

↓

集群域名
```

------

## 一个生活中的例子

想象：

邮寄：

信件。

只写：

```
张三
```

可能：

不知道：

是谁。

但是：

写：

```
张三

北京市

朝阳区

XX街道
```

是不是：

唯一？

FQDN：

就是：

完整地址。

------

# 第六节：为什么平时只写 redis？

因为：

DNS：

有：

搜索域（Search Domain）。

假设：

当前：

Pod：

就在：

```
default
```

Namespace。

访问：

```
redis
```

CoreDNS：

会：

自动：

尝试：

```
redis.default.svc.cluster.local
```

所以：

平时：

不用：

写：

完整：

名字。

------

# 第七节：跨 Namespace 如何访问？

假设：

Redis：

在：

```
database
```

Namespace。

你的 API：

在：

```
default
```

如果：

写：

```
http://redis
```

结果：

失败。

因为：

默认：

会：

查找：

```
redis.default
```

而不是：

```
redis.database
```

正确：

应该：

写：

```
redis.database
```

或者：

完整：

写：

```
redis.database.svc.cluster.local
```

------

# 第八节：DNS 查询完整流程

假设：

API：

访问：

```
http://order-api
```

整个：

流程：

如下：

```
ASP.NET Core

        │

        ▼

CoreDNS

        │

返回：

10.96.0.15

        │

        ▼

ClusterIP

        │

        ▼

kube-proxy

        │

        ▼

Pod
```

是不是：

整个：

网络：

终于：

串：

起来：

了？

------

# 第九节：浏览器访问 Kubernetes 的完整过程

现在。

来看：

最完整：

的一张图。

假设：

用户：

打开：

```
https://api.company.com/api/order
```

整个：

过程：

如下：

```
Browser

        │

DNS（公网）

        │

        ▼

Ingress Controller

        │

        ▼

Service

        │

ClusterIP

        │

        ▼

kube-proxy

        │

        ▼

EndpointSlice

        │

        ▼

Pod

        │

ASP.NET Core

        │

Redis

MySQL
```

以后。

排查：

任何：

网络：

问题。

基本：

都是：

这一条：

链路。

------

# 第十节：DNS 为什么这么重要？

假设：

CoreDNS：

挂了。

会：

发生：

什么？

例如：

应用：

访问：

```
redis
```

结果：

```
Name or service not known
```

或者：

```
Temporary failure in name resolution
```

是不是：

所有：

微服务：

几乎：

全部：

不能：

通信？

因此：

生产：

一般：

至少：

部署：

```
CoreDNS

2

副本
```

保证：

高可用。

------

# 第十一节：如何排查 DNS 问题？

生产环境常用方法：

首先：

进入：

Pod：

```
kubectl exec -it <pod-name> -- sh
```

查看：

DNS 配置：

```
cat /etc/resolv.conf
```

通常会看到：

```
search default.svc.cluster.local svc.cluster.local cluster.local

nameserver 10.x.x.x
```

这里的 `nameserver` 通常就是集群 DNS（CoreDNS）的 Service 地址。

然后：

测试：

解析：

```
nslookup redis
```

或者：

```
getent hosts redis
```

如果：

失败。

继续：

检查：

```
kubectl get svc -A
```

确认：

CoreDNS：

Service：

是否：

存在。

再检查：

```
kubectl get pods -n kube-system
```

确认：

CoreDNS：

Pod：

是否：

Running。

------

# 第十二节：为什么 Service 名称不能重复？

假设：

同一个 Namespace：

有：

```
redis

redis
```

DNS：

根本：

不知道：

返回：

谁。

因此：

同一个 Namespace 内，Service 名称必须唯一。

但是：

不同 Namespace：

可以：

都有：

```
redis
```

例如：

```
dev

↓

redis

────────────

prod

↓

redis
```

这也是 Namespace 的隔离能力之一。

------

# 第十三节：DNS 缓存

很多人会问：

每次请求都查 DNS，会不会很慢？

实际上：

很多客户端和运行时都会缓存 DNS 查询结果。

另外：

CoreDNS：

本身：

也：

支持：

缓存。

因此：

绝大多数情况下。

DNS：

查询：

开销：

非常：

小。

不过要注意，不同语言和运行时（例如 Java、Go、.NET）对 DNS 缓存策略有所不同，这也是生产环境排查问题时需要关注的细节。

------

# 第十四节：完整知识体系

现在。

把：

整个：

Kubernetes：

网络：

全部：

串起来。

```
             Browser
                 │
                 ▼
           公网 DNS
                 │
                 ▼
        Ingress Controller
                 │
                 ▼
             Service
                 │
         ClusterIP（VIP）
                 │
                 ▼
           kube-proxy
                 │
                 ▼
          EndpointSlice
                 │
                 ▼
               Pod
                 │
        ASP.NET Core API
                 │
        ┌────────┴────────┐
        ▼                 ▼
     Redis             MySQL
        │
        ▼
     CoreDNS
（负责解析服务名称）
```

> 严格来说，**CoreDNS 并不在每次请求的数据转发路径上**。它只负责把服务名称解析为 IP，真正的 HTTP/TCP 数据流不会经过 CoreDNS。为了便于理解，这里将它放在体系图中展示它在整个网络架构中的作用。

------

# 第十五节：Kubernetes 网络知识总结

经过三章学习。

你已经理解：

```
Pod

↓

独立 IP

────────────

Service

↓

稳定入口

────────────

ClusterIP

↓

虚拟 IP

────────────

kube-proxy

↓

转发规则

────────────

iptables / IPVS

↓

真正转发

────────────

CoreDNS

↓

名称解析

────────────

EndpointSlice

↓

记录后端 Pod
```

这就是 Kubernetes 网络最核心的组成部分。

------

# 第十六节：本章总结（建议牢记）

请记住 Kubernetes DNS 最重要的几点：

1. **CoreDNS 是 Kubernetes 集群内的 DNS 服务器。**
2. **服务之间推荐通过 Service 名称通信，而不是直接使用 Pod IP。**
3. **完整的 Service 域名格式为：`<service>.<namespace>.svc.cluster.local`。**
4. **同 Namespace 可以直接使用 Service 名称，跨 Namespace 建议带上 Namespace 或完整域名。**
5. **CoreDNS 负责名称解析，真正的数据转发由 Service、kube-proxy 和网络组件完成。**
6. **DNS 故障会导致大量服务之间无法互相发现，因此 CoreDNS 通常需要高可用部署。**

------

# 到这里，你已经完整掌握 Kubernetes 网络体系

从 Pod、Service、ClusterIP、kube-proxy，到 DNS、CoreDNS、EndpointSlice，你已经能够理解一次请求如何在 Kubernetes 集群中流转。

这是学习更高级云原生技术的重要基础。

------

# 下一章预告：存储原理——Volume、PV、PVC、StorageClass

接下来，我们将进入 Kubernetes 的另一大核心领域：**存储（Storage）**。

你将学习：

- 为什么 Pod 重启后数据会丢失？
- Volume 和 Docker Volume 有什么区别？
- PV（PersistentVolume）到底是什么？
- PVC（PersistentVolumeClaim）为什么叫"声明"？
- StorageClass 如何实现动态创建存储？
- NFS、云硬盘、Ceph 等存储如何接入 Kubernetes？
- ASP.NET Core 上传文件应该保存在哪里？

这一章结束后，你将真正理解 Kubernetes **有状态应用（Stateful Application）** 的部署方式，为后续学习 **StatefulSet、数据库部署、云存储** 打下坚实基础。

# 第四阶段 第一章：Kubernetes 存储（上）——Volume、PV、PVC 与 StorageClass

> **本章重点：理解 Kubernetes 的存储模型，而不是死记 YAML。**

------

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入 Kubernetes 的**第二大核心能力：存储（Storage）**。

很多初学者学习 Kubernetes 时，都会遇到一个经典问题：

> **为什么我的数据库重启之后，数据全没了？**

或者：

> **为什么我的 ASP.NET Core 上传的图片，一重建 Pod 就消失了？**

如果不能理解 Kubernetes 的存储，那么 **MySQL、PostgreSQL、Redis（持久化）、MinIO、Jenkins、GitLab、ELK、Prometheus** 等几乎所有有状态应用都无法正确部署。

这一章建议认真理解，因为它直接关系到生产环境的数据安全。

# 本章学习目标

学习完本章，你应该能够回答：

- 为什么 Pod 重建后数据会丢失？
- Volume 是什么？
- emptyDir、hostPath、PersistentVolume 有什么区别？
- PV（PersistentVolume）是什么？
- PVC（PersistentVolumeClaim）为什么叫"声明"？
- StorageClass 又解决了什么问题？
- Kubernetes 为什么要把存储拆成这么多对象？

------

# 第一节：为什么 Pod 重建后数据会丢失？

我们先来看一个最常见的场景。

假设你部署了一个 ASP.NET Core 应用：

```
Deployment
    │
    ▼
Pod
    │
    ▼
/app/uploads
```

用户上传了一张图片：

```
/app/uploads/a.jpg
```

第二天，你发布新版本：

```
Deployment
    │
删除旧 Pod
    │
创建新 Pod
```

结果：

```
a.jpg
```

消失了。

为什么？

因为：

**Pod 的文件系统默认是临时的（Ephemeral）。**

可以把 Pod 想象成：

> **酒店房间。**

入住时：

```
房间里有：

床

桌子

电视
```

退房以后：

酒店会把房间恢复原样。

下一位住客：

不会看到你的东西。

Pod 也是一样。

删除 Pod，本地文件系统通常就一起消失了。

------

# 第二节：Pod 的生命周期和数据生命周期不同

这里有一个非常重要的设计思想：

> **应用生命周期 ≠ 数据生命周期。**

例如：

应用：

```
今天升级

↓

删除 Pod

↓

重新创建
```

这是正常操作。

但是：

数据库的数据：

```
订单

用户

图片

日志
```

不能跟着删除。

所以 Kubernetes 把：

- **计算（Pod）**
- **存储（Volume）**

彻底分开。

------

# 第三节：什么是 Volume？

Volume 可以理解成：

> **挂载到 Pod 的一块存储空间。**

例如：

```
Pod
│
├── /app
├── /tmp
└── /data  ← Volume
```

应用程序：

访问：

```
/data
```

其实就是访问 Volume。

类似于 Windows：

```
C:
D:
E:
```

其中：

```
D:
```

可以理解成挂载进来的存储。

------

# 第四节：emptyDir —— 最简单的 Volume

最简单的 Volume：

就是：

```
volumes:
- name: cache
  emptyDir: {}
```

它的特点：

- Pod 创建时自动创建
- Pod 删除时一起删除
- 适合缓存
- 不适合保存重要数据

例如：

```
Pod
│
├── Application
└── Cache（emptyDir）
```

如果 Pod 删除：

```
Cache
↓

一起删除
```

因此：

**emptyDir 不能作为数据库存储。**

------

## 适用场景

例如：

- 临时缓存
- 下载中间文件
- 解压 ZIP
- 编译缓存

这些数据即使丢失，也可以重新生成。

------

# 第五节：hostPath —— 使用 Node 本地磁盘

另一种 Volume：

```
hostPath:
  path: /data
```

意思：

直接使用：

Node：

上的：

```
/data
```

目录。

例如：

```
Node

/data
    │
    ▼
Pod
```

Pod：

访问：

```
/app/data
```

其实就是：

Node：

的：

```
/data
```

------

## hostPath 的优点

简单。

不用额外配置存储系统。

学习 Kubernetes 时非常方便。

------

## hostPath 的缺点（生产环境必须理解）

假设：

Pod：

原来运行：

```
Node1
```

数据：

保存在：

```
Node1:/data
```

后来：

Node1：

宕机。

Kubernetes：

自动把 Pod 调度到：

```
Node2
```

问题来了：

```
Node2:/data
```

里面：

什么都没有。

数据：

"丢失"了（准确地说，是留在了 Node1 上，新的 Pod 无法访问）。

因此：

> **hostPath 强依赖单台机器，不适合大多数生产环境。**

它更多用于：

- 学习
- 单节点集群
- 特殊运维场景（例如挂载宿主机日志目录）

------

# 第六节：真正的持久化存储应该是什么样？

理想情况下：

无论 Pod 在：

```
Node1
```

还是：

```
Node2
```

都应该访问：

同一份数据。

例如：

```
          存储
      （共享磁盘）
          │
   ┌──────┴──────┐
   ▼             ▼
Node1         Node2
   │             │
   ▼             ▼
 Pod A         Pod B
```

这就是 Kubernetes 持久化存储设计的目标。

共享存储可以来自：

- 云盘（AWS EBS、Azure Disk、阿里云 ESSD 等）
- NFS
- Ceph
- Longhorn
- OpenEBS
- SAN/NAS 等

------

# 第七节：什么是 PersistentVolume（PV）？

PV 全称：

> **PersistentVolume（持久卷）**

一句话理解：

> **PV 表示集群中已经存在的一块存储资源。**

注意：

PV **不是 Pod 的一部分**。

它属于：

整个 Kubernetes 集群。

例如：

```
Kubernetes Cluster
│
├── Pod
├── Service
└── PV（100Gi）
```

PV 可以来自：

- 一块云硬盘
- 一个 NFS 目录
- 一个 Ceph 卷
- 一个本地磁盘（Local PV）

------

## 一个生活中的例子

把 Kubernetes 想象成一个大型仓库。

仓库里已经有：

```
100㎡ 仓库A

200㎡ 仓库B

500㎡ 仓库C
```

这些仓库：

就是：

PV。

它们已经存在。

等待别人使用。

------

# 第八节：什么是 PVC（PersistentVolumeClaim）？

很多新人第一次看到 PVC：

都会问：

> 为什么不直接使用 PV？

这是 Kubernetes 一个非常优秀的设计。

PVC：

全称：

> **PersistentVolumeClaim（持久卷声明）**

可以理解成：

> **"我要申请一块存储。"**

例如：

```
resources:
  requests:
    storage: 20Gi
```

意思：

```
我需要：

20Gi

存储
```

至于：

到底：

给：

哪一块。

由 Kubernetes 决定。

------

## 仓库的例子

继续用仓库比喻。

PV：

相当于：

仓库管理员手上的仓库：

```
A：100㎡

B：200㎡

C：500㎡
```

开发人员不会说：

> 我要 A 仓库。

而是说：

> 我需要一个至少 100㎡ 的仓库。

管理员再去分配。

PVC 就是这个"申请单"。

------

# 第九节：为什么要分成 PV 和 PVC？

这是 Kubernetes 最经典的设计之一。

如果没有 PVC：

开发人员需要知道：

- 哪块磁盘可用
- 哪块磁盘容量够
- 哪块磁盘已经被占用

这意味着：

开发与运维紧密耦合。

有了 PVC 后：

开发只需要声明需求：

```
我要：

50Gi

ReadWriteOnce
```

至于：

具体绑定哪个 PV。

交给 Kubernetes。

这就是：

**声明式（Declarative）** 思想。

------

# 第十节：PVC 与 PV 的绑定关系

关系如下：

```
Pod
 │
 ▼
PVC（申请）
 │
 ▼
PV（实际存储）
 │
 ▼
NFS / 云盘 / Ceph ...
```

Pod 永远不会直接使用 PV。

它只引用 PVC。

这样即使底层存储发生变化（例如从 NFS 换成云盘），应用配置也可以保持不变。

------

# 第十一节：什么是 StorageClass？

到这里你可能会发现一个问题：

如果每次都要：

运维先创建：

PV：

开发再创建：

PVC。

流程还是比较繁琐。

于是：

Kubernetes 引入了：

> **StorageClass（存储类）**

它的作用就是：

> **定义"如何创建存储"。**

例如：

可以定义：

```
fast-ssd
```

表示：

高速 SSD。

或者：

```
standard
```

表示：

普通磁盘。

开发人员只需要在 PVC 中指定：

```
storageClassName: fast-ssd
```

如果没有合适的 PV：

Kubernetes 会调用对应的存储驱动，**动态创建** 一块新的存储，然后自动绑定给 PVC。

这就是现代 Kubernetes 集群最常见的模式。

------

# 第十二节：本章知识关系图

请把下面这张图记住：

```
             Pod
              │
              ▼
            PVC（我要 20Gi）
              │
              ▼
     StorageClass（如何创建）
              │
              ▼
     动态创建或匹配已有 PV
              │
              ▼
   云盘 / NFS / Ceph / Longhorn ...
```

这张图体现了 Kubernetes 存储的核心思想：**应用只描述需求，不关心底层实现。**

------

# 第十三节：本章总结（建议牢记）

请记住 Kubernetes 存储最重要的几点：

1. **Pod 默认使用临时文件系统，Pod 删除后数据通常会丢失。**
2. **Volume 是挂载到 Pod 的存储空间。**
3. **`emptyDir` 适合临时数据，不适合重要业务数据。**
4. **`hostPath` 依赖单个 Node，不适合作为大多数生产环境的持久化方案。**
5. **PV 表示实际存在的存储资源，PVC 表示应用对存储的声明。**
6. **Pod 应该引用 PVC，而不是直接使用 PV。**
7. **StorageClass 让存储能够动态创建，是现代 Kubernetes 持久化存储的推荐方式。**

------

# 一个容易混淆但必须记住的区别

| 对象         | 作用             | 谁关心          |
| ------------ | ---------------- | --------------- |
| Volume       | Pod 内挂载点     | 应用            |
| PV           | 实际存储资源     | 运维 / 存储系统 |
| PVC          | 对存储的申请     | 开发、应用      |
| StorageClass | 定义如何创建存储 | 平台运维        |

理解这四者之间的关系，比背 YAML 更重要。

------

# 下一章预告：Kubernetes 存储（下）——StatefulSet、数据库部署与生产实践

下一章，我们将把这些概念真正应用起来，学习：

- 为什么数据库不能随便使用 Deployment？
- StatefulSet 与 Deployment 的本质区别是什么？
- Headless Service 是什么？
- MySQL、PostgreSQL、Redis 应该如何部署？
- PVC 删除后数据一定会删除吗？
- StorageClass 的回收策略（Retain、Delete）有什么区别？
- 企业生产环境如何设计 Kubernetes 存储架构？

这一章结束后，你将真正具备部署**有状态应用（Stateful Applications）**的能力，也会理解 Kubernetes 如何安全地管理数据库和持久化数据。

# 第四阶段 第二章：StatefulSet、数据库部署与生产实践

> **这一章是 Kubernetes 存储体系的核心，也是生产环境的重点。**

------

提示词：

```
开始下一章
```

正文：

非常好。

这一章，我们正式进入 Kubernetes 中**最容易出事故的领域**：

> **有状态应用（Stateful Application）**

如果前面学习 Deployment，是在学习：

> **如何部署 Web API。**

那么这一章，就是学习：

> **如何部署数据库、缓存、消息队列等基础设施。**

也是很多企业面试最喜欢问的一章。

# 本章学习目标

学习完本章，你应该能够回答：

- 为什么 MySQL 不建议使用 Deployment？
- StatefulSet 和 Deployment 到底有什么区别？
- Headless Service 是什么？
- StatefulSet 为什么必须配合 Headless Service？
- PVC 为什么不会随着 Pod 删除？
- StorageClass 的回收策略（Retain、Delete）是什么？
- 企业如何部署 MySQL、Redis、PostgreSQL 等有状态应用？

------

# 第一节：什么是有状态应用（Stateful Application）？

我们先区分两类应用。

## 无状态应用（Stateless）

例如：

- ASP.NET Core Web API
- Vue 前端
- Nginx
- Java Spring Boot API

特点：

```
请求来了
      │
      ▼
任何一个 Pod 都能处理
```

例如：

```
Pod1

Pod2

Pod3
```

用户访问：

任何一个都可以。

因为：

用户数据：

保存在：

数据库。

Pod 本身：

没有重要数据。

所以：

Pod 删除：

没关系。

重新创建即可。

------

## 有状态应用（Stateful）

例如：

- MySQL
- PostgreSQL
- MongoDB
- Redis（开启持久化）
- Kafka
- Elasticsearch
- RabbitMQ

特点：

**数据就在应用内部。**

例如：

MySQL：

```
订单

用户

库存

日志
```

都保存在：

磁盘。

如果：

Pod 删除：

数据库文件：

丢失。

整个系统：

就崩了。

所以：

数据库：

不能：

像 API 一样：

随便删。

------

# 第二节：为什么 MySQL 不建议使用 Deployment？

很多新人：

第一次部署：

MySQL：

直接：

```
kind: Deployment
```

表面：

可以运行。

实际上：

隐藏：

很多问题。

例如：

Deployment：

重新创建 Pod：

可能：

变成：

```
mysql-84fd6

↓

mysql-6a72d
```

Pod：

名称：

变了。

IP：

变了。

磁盘：

如果没正确配置：

也可能：

变了。

数据库：

非常讨厌：

这些：

变化。

因为：

数据库：

希望：

身份：

稳定。

------

# 第三节：Deployment 与 StatefulSet 的本质区别

Deployment：

认为：

Pod：

都是：

一样的。

例如：

```
Pod A

Pod B

Pod C
```

谁都可以。

删除：

哪个：

都一样。

------

StatefulSet：

则认为：

每个 Pod：

都有：

自己的身份。

例如：

```
mysql-0

mysql-1

mysql-2
```

注意：

名字：

永远：

不会：

变化。

即使：

重建：

仍然：

叫：

```
mysql-0
```

不是：

随机：

名字。

------

## 一个生活中的例子

Deployment：

像出租车。

任何一辆：

都能接客。

坏了一辆：

换另一辆。

没人关心编号。

------

StatefulSet：

像银行柜员。

```
1号柜台

2号柜台

3号柜台
```

客户：

可能：

长期：

对应：

某一个柜台。

编号：

不能：

乱变。

------

# 第四节：StatefulSet 的三个核心特点

请牢牢记住：

StatefulSet 主要提供：

## ① 稳定的 Pod 名称

例如：

```
redis-0

redis-1

redis-2
```

永远：

固定。

------

## ② 稳定的网络身份

例如：

```
redis-0.redis

redis-1.redis

redis-2.redis
```

这些 DNS 名称：

长期有效。

即使：

Pod 重建。

名字：

仍然：

一样。

------

## ③ 稳定的存储

例如：

```
redis-0

↓

PVC A

────────────

redis-1

↓

PVC B
```

每个 Pod：

都有：

自己的：

PVC。

不会：

混用。

------

# 第五节：为什么 StatefulSet 必须配合 Headless Service？

这里是很多新人的疑问。

先回忆一下：

普通 Service：

```
Service

↓

ClusterIP

↓

随机选择 Pod
```

是不是：

不知道：

访问：

哪个：

Pod？

------

但是：

数据库：

很多时候：

必须：

指定：

节点。

例如：

Redis Cluster：

```
redis-0

redis-1

redis-2
```

需要：

彼此：

知道：

对方。

不能：

随机。

所以：

StatefulSet：

通常：

配合：

Headless Service。

------

# 第六节：什么是 Headless Service？

普通 Service：

有：

ClusterIP：

例如：

```
10.96.0.25
```

Headless Service：

配置：

```
clusterIP: None
```

没有：

ClusterIP。

DNS：

直接：

返回：

Pod。

例如：

查询：

```
redis
```

返回：

```
redis-0

redis-1

redis-2
```

或者：

查询：

```
redis-0.redis
```

直接：

返回：

```
10.244.x.x
```

这样：

集群中的节点可以精确找到彼此。

------

# 第七节：为什么每个 Pod 都有自己的 PVC？

假设：

三个：

MySQL：

共用：

一个：

PVC。

会发生什么？

```
MySQL1

↓

写：

ibdata1

────────────

MySQL2

↓

同时写：

ibdata1
```

数据：

很可能：

损坏。

因此：

StatefulSet：

会：

自动：

创建：

```
mysql-data-mysql-0

mysql-data-mysql-1

mysql-data-mysql-2
```

每个：

Pod：

独享：

自己的：

PVC。

------

# 第八节：Pod 删除后，PVC 为什么还在？

这是 StatefulSet 非常重要的一点。

假设：

删除：

```
mysql-0
```

Pod：

没了。

但是：

```
PVC

mysql-data-mysql-0
```

仍然：

存在。

重新创建：

```
mysql-0
```

继续：

挂载：

原来的：

PVC。

于是：

数据库：

数据：

仍然：

存在。

------

# 第九节：StorageClass 回收策略（Reclaim Policy）

很多人误以为：

删除 PVC：

磁盘一定删除。

其实：

不是。

StorageClass / PV 有：

回收策略。

最常见：

两种：

------

## Delete

删除：

PVC：

↓

自动：

删除：

底层：

云盘。

适合：

测试环境。

------

## Retain

删除：

PVC：

↓

底层：

磁盘：

保留。

适合：

生产环境。

因为：

数据：

更加：

安全。

> 还有一种较少使用的历史策略 `Recycle`，已在 Kubernetes 中废弃，不建议了解或使用。

------

# 第十节：生产环境推荐部署方式

不同类型应用推荐如下：

| 应用             | 推荐控制器  |
| ---------------- | ----------- |
| ASP.NET Core API | Deployment  |
| Vue              | Deployment  |
| Nginx            | Deployment  |
| MySQL            | StatefulSet |
| PostgreSQL       | StatefulSet |
| Redis（持久化）  | StatefulSet |
| Kafka            | StatefulSet |
| Elasticsearch    | StatefulSet |

一个简单的判断方法：

> **是否需要长期保存本地数据、稳定身份或固定网络名称？**

如果需要，大概率应该考虑 StatefulSet。

------

# 第十一节：企业真的自己部署数据库吗？

这是一个非常现实的问题。

答案是：

**很多时候，不会。**

例如：

在云平台：

```
阿里云 RDS

AWS RDS

Azure Database
```

数据库：

已经：

托管。

Kubernetes：

只部署：

API。

这样：

运维：

压力：

更小。

------

但是：

下面这些：

仍然：

经常：

部署：

在：

Kubernetes：

里面：

```
Redis

Kafka

RabbitMQ

MinIO

Elasticsearch

Prometheus
```

因此：

StatefulSet：

仍然：

非常：

重要。

------

# 第十二节：完整关系图

请把下面这张图记住：

```
          StatefulSet
                │
      ┌─────────┴─────────┐
      ▼         ▼         ▼
   mysql-0   mysql-1   mysql-2
      │         │         │
      ▼         ▼         ▼
    PVC-0     PVC-1     PVC-2
      │         │         │
      └─────────┴─────────┘
                │
        StorageClass
                │
                ▼
     云盘 / NFS / Ceph / Longhorn
```

这里可以清楚地看到：

- 每个 Pod 都有独立的 PVC。
- PVC 再绑定到底层存储。
- StatefulSet 保证 Pod、PVC 和网络身份的一一对应关系。

------

# 第十三节：Deployment 与 StatefulSet 对比

| 对比项       | Deployment     | StatefulSet              |
| ------------ | -------------- | ------------------------ |
| Pod 名称     | 随机后缀       | 固定编号（如 `mysql-0`） |
| Pod 身份     | 无状态         | 有状态                   |
| Pod 网络身份 | 不固定         | 固定 DNS 名称            |
| 存储         | 可共享或临时   | 每个 Pod 独立 PVC        |
| 适用场景     | Web、API、前端 | 数据库、中间件、消息队列 |

------

# 第十四节：新手最容易犯的错误

下面这些问题在实际工作中非常常见：

❌ 用 Deployment 部署 MySQL，然后发现升级后数据丢失。

❌ 多个数据库实例共享同一个读写卷，导致数据损坏。

❌ 删除 PVC 时，没有确认回收策略，误删生产数据。

❌ 没有做数据库备份，误以为 PVC 就等于备份。

请记住：

> **PVC 是存储，不是备份。**

磁盘损坏、误删除、逻辑错误等问题，仍然需要定期备份。

------

# 第十五节：本章总结（建议牢记）

请记住 StatefulSet 最重要的几点：

1. **Deployment 适用于无状态应用，StatefulSet 适用于有状态应用。**
2. **StatefulSet 提供稳定的 Pod 名称、网络身份和存储。**
3. **StatefulSet 通常与 Headless Service 配合使用，实现固定的 DNS 名称。**
4. **每个 StatefulSet Pod 都应拥有独立的 PVC。**
5. **生产环境应重点关注 StorageClass 的回收策略，避免误删数据。**
6. **PVC 提供持久化存储，但不能替代备份。**

------

# 到这里，你已经掌握了 Kubernetes 持久化存储体系

你已经理解了：

```
StatefulSet
      │
      ▼
固定 Pod 身份
      │
      ▼
独立 PVC
      │
      ▼
StorageClass
      │
      ▼
底层存储（云盘 / NFS / Ceph 等）
```

这套模型是部署数据库和中间件的基础。

------

# 下一章预告：Kubernetes 调度原理（Scheduler 深度解析）

前面我们一直在"创建 Pod"，但还有一个关键问题没有解释：

- Pod 为什么会被调度到某台 Node？
- Scheduler 是如何选择 Node 的？
- `nodeSelector`、`Node Affinity`、`Pod Affinity`、`Pod Anti-Affinity` 有什么区别？
- Taint（污点）和 Toleration（容忍）为什么能隔离工作负载？
- 企业如何保证数据库、GPU、AI 训练任务运行在指定节点？

下一章开始，我们将深入学习 **Kubernetes 调度器（Scheduler）** 的工作原理。这也是设计高可用集群、资源隔离和性能优化的关键知识。

# 第五阶段 第一章：Kubernetes 调度原理（Scheduler 深度解析）

> **本章重点：理解 Scheduler 如何为 Pod 选择 Node。**

------

提示词：

```
开始下一章
```

正文：

非常好。

从这一章开始，我们进入 Kubernetes 的**大脑**。

前面我们一直在执行：

```
kubectl apply -f deployment.yaml
```

然后：

Pod 就自动运行起来了。

很多新人会觉得：

> **Kubernetes 真智能。**

但是，你有没有想过：

> **Pod 为什么偏偏运行在 Node2，而不是 Node1？**

> **是谁决定的？**

答案就是：

> **kube-scheduler（调度器）**

这一章可以说是 Kubernetes 最有"智慧"的一部分。

理解 Scheduler，你以后就能理解：

- 为什么 Pod 一直 Pending？
- 为什么 GPU Pod 没有调度成功？
- 为什么数据库总跑到错误的 Node？
- 为什么某些 Node 永远很空，而另一些 Node 很忙？

这些都是企业生产环境每天都会遇到的问题。

# 本章学习目标

学习完本章，你应该能够回答：

- Scheduler 到底负责什么？
- Pod 创建以后发生了什么？
- Scheduler 如何选择 Node？
- Filter（过滤）和 Score（打分）是什么？
- 为什么 Pod 会一直 Pending？
- Resource Request 和 Limit 为什么影响调度？
- 企业如何优化调度策略？

------

# 第一节：Pod 创建以后，到底发生了什么？

很多人认为：

```
kubectl apply
      │
      ▼
Pod 就运行了
```

实际上，这中间经历了很多步骤。

完整流程如下：

```
kubectl apply
      │
      ▼
API Server
      │
      ▼
etcd（保存 Pod 定义）
      │
      ▼
Scheduler
      │
选择 Node
      │
      ▼
API Server 更新 Pod
      │
      ▼
Node 上的 kubelet
      │
      ▼
Container Runtime
      │
      ▼
Container Running
```

注意：

**Scheduler 只负责选择 Node。**

它：

不会：

拉镜像。

不会：

启动容器。

不会：

执行程序。

这些事情都是 kubelet 完成的。

------

# 第二节：Scheduler 到底是什么？

一句话：

> **Scheduler 就是 Kubernetes 的"资源调度员"。**

例如：

现在：

有：

三台 Node：

```
Node1

CPU：80%

Memory：90%
Node2

CPU：20%

Memory：30%
Node3

CPU：50%

Memory：40%
```

来了：

一个：

新的：

Pod。

Scheduler：

要决定：

放哪台？

它不会：

随机。

而是：

经过一整套计算。

------

## 一个生活中的例子

想象一家酒店。

来了：

一个：

旅行团。

酒店经理不会：

随便安排。

而是先看：

```
哪个房间空？

房间够大吗？

VIP 房是否预留？

是否允许宠物？
```

Scheduler：

就是：

这个酒店经理。

------

# 第三节：Scheduler 的两步核心流程

Kubernetes Scheduler 的核心流程非常简单：

```
第一步：

Filter（过滤）

↓

第二步：

Score（打分）
```

请把这两个单词牢牢记住。

以后阅读 Scheduler 源码、排查调度问题都会遇到。

------

# 第四节：第一步 —— Filter（过滤）

Filter 的作用：

> **把所有不符合条件的 Node 排除掉。**

例如：

集群：

有：

```
Node1

Node2

Node3

Node4
```

Pod：

需要：

```
CPU：

4 Core
```

但是：

Node1：

只有：

```
2 Core
```

直接：

淘汰。

Node2：

内存不足。

淘汰。

Node3：

被设置了污点（后面会讲）。

淘汰。

最后：

只剩：

```
Node4
```

这就是：

Filter。

------

# 第五节：第二步 —— Score（打分）

如果：

过滤之后：

还有：

多个 Node。

例如：

```
Node2

Node3

Node4
```

Scheduler：

就开始：

打分。

例如：

```
Node2

95 分
Node3

80 分
Node4

70 分
```

最后：

选择：

```
Node2
```

这就是：

Score。

------

# 第六节：Filter 会检查哪些内容？

最重要的几项：

------

## ① CPU 是否够？

例如：

Pod：

声明：

```
requests:
  cpu: 2
```

Node：

剩余：

```
1 Core
```

过滤。

不能调度。

------

## ② Memory 是否够？

例如：

Pod：

需要：

```
memory: 4Gi
```

Node：

剩：

```
2Gi
```

过滤。

------

## ③ Node 是否 Ready？

如果：

Node：

状态：

```
NotReady
```

Scheduler：

不会：

选择。

------

## ④ 是否满足节点选择条件？

例如：

Pod：

要求：

```
nodeSelector:
  disk: ssd
```

Node：

没有：

这个标签。

过滤。

------

后面的章节，我们还会学习：

- Node Affinity
- Taints
- Topology
- Pod Affinity

它们也属于过滤条件。

------

# 第七节：为什么 Pod 一直 Pending？

很多新人都会遇到：

```
kubectl get pods
```

结果：

```
Pending
```

不是：

CrashLoopBackOff。

不是：

ImagePullBackOff。

而是：

Pending。

这通常意味着：

> **Scheduler 还没有找到合适的 Node。**

最常见原因：

- CPU 不足
- Memory 不足
- Node 不可用
- 节点标签不匹配
- 污点未容忍
- PVC 尚未绑定（某些存储场景下也会影响调度）

------

# 第八节：为什么 Request 会影响调度？

这是 Kubernetes 非常重要的一个概念。

例如：

Pod：

声明：

```
requests:
  cpu: 2
```

即使：

程序：

当前：

几乎：

不用 CPU。

Scheduler：

也会认为：

> **必须预留 2 Core。**

因为：

Request：

表示：

**最低保障资源。**

Scheduler：

就是：

根据：

Request：

来调度。

而不是根据：

当前：

CPU 使用率。

------

## 一个生活中的例子

你订酒店。

要求：

```
双人房
```

即使：

现在：

只有：

一个人入住。

酒店：

也必须：

给你：

保留：

双人房。

Scheduler：

也是：

一样。

------

# 第九节：Limit 会影响调度吗？

很多新人容易混淆：

Request：

和：

Limit。

请记住：

| 配置    | Scheduler 是否参考 |
| ------- | ------------------ |
| Request | ✅ 是               |
| Limit   | ❌ 一般不是         |

为什么？

因为：

Limit：

只是：

告诉：

Linux：

> **最多允许使用多少资源。**

真正：

决定：

能不能：

调度。

看的是：

Request。

> 补充说明：在默认调度流程中，Scheduler 依据 Pod 的 **Request** 进行资源可用性计算。Limit 会影响运行时的资源限制（例如 CPU 限速、内存 OOM），但不会直接作为调度依据。

------

# 第十节：Scheduler 如何知道 Node 资源？

很多人会问：

Scheduler：

为什么知道：

Node：

还剩：

多少 CPU？

答案：

因为：

每个：

Node：

都有：

```
kubelet
```

kubelet：

持续：

向：

API Server：

汇报：

```
CPU

Memory

Pod

状态
```

Scheduler：

读取：

这些：

信息。

然后：

进行：

计算。

------

# 第十一节：一次完整调度流程

现在：

把整个过程串起来：

```
kubectl apply
      │
      ▼
API Server
      │
      ▼
etcd 保存 Pod
      │
      ▼
Scheduler
      │
Filter
      │
Score
      │
选出 Node
      │
      ▼
API Server 更新绑定关系
      │
      ▼
Node kubelet
      │
拉镜像
      │
创建容器
      │
Pod Running
```

这里有一个容易忽略的细节：

Scheduler 并不是直接通知 kubelet，而是通过 API Server 更新 Pod 的 `spec.nodeName`（绑定关系），目标 Node 上的 kubelet 观察到这个变化后，才开始真正创建 Pod。

------

# 第十二节：企业为什么要学习调度？

假设：

你的集群：

有：

```
GPU Node

普通 Node

数据库 Node
```

不同：

业务：

必须：

运行：

不同：

机器。

例如：

AI：

只能：

GPU。

数据库：

只能：

SSD。

日志：

只能：

大容量磁盘。

这些：

全部：

依赖：

Scheduler。

------

# 第十三节：本章知识关系图

```
             新 Pod
               │
               ▼
          API Server
               │
               ▼
          kube-scheduler
               │
      ┌────────┴────────┐
      ▼                 ▼
   Filter            Score
      │                 │
      └────────┬────────┘
               ▼
         选择最佳 Node
               │
               ▼
       API Server 更新绑定
               │
               ▼
          kubelet
               │
               ▼
        创建并运行 Pod
```

建议把这张图画到笔记中，它是理解后续所有调度策略的基础。

------

# 第十四节：本章总结（建议牢记）

请记住 Scheduler 最重要的几点：

1. **Scheduler 的职责只有一个：为 Pod 选择最合适的 Node。**
2. **调度过程分为两个阶段：Filter（过滤）和 Score（打分）。**
3. **Filter 会淘汰不符合条件的 Node，例如资源不足、标签不匹配、污点等。**
4. **Score 会在候选 Node 中选择最优节点。**
5. **Scheduler 根据 Pod 的 Request，而不是实际资源使用率进行调度。**
6. **Pod 长时间处于 Pending，首先要检查 Scheduler 是否找到了可用 Node。**
7. **Scheduler 通过 API Server 完成 Pod 与 Node 的绑定，真正启动容器的是 kubelet。**

------

# 本章结束后，你已经理解了 Scheduler 的整体工作方式

你现在已经知道：

```
创建 Pod
    │
    ▼
Scheduler
    │
Filter
    │
Score
    │
选择 Node
    │
API Server 绑定
    │
kubelet 创建 Pod
```

这就是 Kubernetes 调度器的完整工作流程。

------

# 下一章预告：节点选择与调度策略——nodeSelector、Node Affinity、Pod Affinity、Pod Anti-Affinity

下一章，我们将深入学习**如何控制 Pod 被调度到哪里**，包括：

- `nodeSelector` 是如何工作的？
- `Node Affinity` 为什么比 `nodeSelector` 更强大？
- `Pod Affinity` 和 `Pod Anti-Affinity` 有什么区别？
- 如何让两个服务尽量部署在一起？
- 如何让多个副本尽量分散到不同 Node，提高高可用？
- 企业如何为 GPU、数据库、AI 训练、日志节点设计调度规则？

学完这一章，你将能够精确控制 Kubernetes 的调度行为，这也是生产环境资源规划和高可用设计的重要能力。

# 第五阶段 第二章：节点选择与调度策略——nodeSelector、Node Affinity、Pod Affinity、Pod Anti-Affinity

> **本章重点：掌握如何控制 Pod 被调度到哪里。**

------

提示词：

```
开始下一章
```

正文：

非常好。

从这一章开始，我们学习 **Kubernetes 调度策略**。

上一章，我们已经知道：

```
Pod
    │
    ▼
Scheduler
    │
Filter
    │
Score
    │
Node
```

但是还有一个问题：

> **如果我不想让 Scheduler 自己决定，而是想"指定"Pod 运行的位置，该怎么办？**

例如：

- MySQL 只能运行在 SSD 服务器
- AI 模型只能运行在 GPU 节点
- 日志系统只能运行在大容量磁盘服务器
- 同一个服务的多个副本不要放在同一台机器

这些需求，就是这一章要解决的问题。

这一章也是企业生产环境**使用频率最高**的 Kubernetes 调度知识之一。

# 本章学习目标

学习完本章，你应该能够回答：

- nodeSelector 是什么？
- Node Label 是什么？
- Node Affinity 为什么比 nodeSelector 更强？
- Pod Affinity 与 Pod Anti-Affinity 有什么区别？
- 企业如何保证高可用？
- 为什么 StatefulSet 经常使用 Anti-Affinity？

------

# 第一节：为什么需要指定调度位置？

假设你的集群如下：

```
Kubernetes Cluster

Node1（SSD）
Node2（普通磁盘）
Node3（GPU）
Node4（GPU）
```

现在：

有三个应用：

```
MySQL

AI Training

Web API
```

如果：

完全随机调度：

可能发生：

```
MySQL

↓

Node2（普通磁盘）
```

性能：

下降。

或者：

```
AI

↓

Node1（没有 GPU）
```

根本：

运行不了。

所以：

需要：

告诉：

Scheduler：

"哪些 Node 可以运行。"

------

# 第二节：Node Label 是什么？

Kubernetes：

每个 Node：

都可以拥有：

很多标签（Label）。

例如：

```
Node1

disk=ssd

zone=a

gpu=false
```

Node2：

```
disk=hdd

zone=a

gpu=false
```

Node3：

```
disk=ssd

gpu=true

zone=b
```

这些：

Label：

就是：

Node 的属性。

------

查看：

Node Label：

```
kubectl get nodes --show-labels
```

或者：

```
kubectl describe node node1
```

------

添加：

Label：

```
kubectl label node node1 disk=ssd
```

删除：

Label：

```
kubectl label node node1 disk-
```

------

## 一个生活中的例子

可以把 Node 想象成酒店房间。

房间门口贴着标签：

```
VIP

海景

双床

禁烟
```

Scheduler：

就是：

前台。

入住要求：

```
必须：

海景
```

前台：

只会：

选择：

符合标签：

的房间。

------

# 第三节：nodeSelector（最简单的节点选择）

这是 Kubernetes 最简单的调度方式。

例如：

```
spec:
  nodeSelector:
    disk: ssd
```

意思：

```
只能：

调度到：

disk=ssd

的 Node
```

例如：

集群：

```
Node1

disk=ssd

────────────

Node2

disk=hdd

────────────

Node3

disk=ssd
```

最终：

只能：

运行：

```
Node1

或者

Node3
```

Node2：

直接：

被：

Filter：

掉。

------

# 第四节：nodeSelector 的限制

很多新人觉得：

nodeSelector：

已经很好用了。

实际上：

它只有：

**等于（=）**。

例如：

```
disk=ssd
```

不能：

表达：

下面这些需求：

```
SSD

或者

NVMe
```

也不能：

表达：

```
CPU

大于

16 Core
```

更不能：

表达：

```
最好：

GPU

没有也可以
```

于是：

Kubernetes：

推出：

Node Affinity。

------

# 第五节：Node Affinity（节点亲和性）

一句话：

> **Node Affinity 是 nodeSelector 的增强版。**

支持：

更多：

表达能力。

例如：

```
requiredDuringSchedulingIgnoredDuringExecution:
```

意思：

```
必须满足
```

例如：

```
matchExpressions:
- key: disk
  operator: In
  values:
    - ssd
    - nvme
```

表示：

```
SSD

或者

NVMe
```

都可以。

------

# 第六节：Node Affinity 支持哪些操作？

最常见：

几个：

Operator：

| Operator     | 含义             |
| ------------ | ---------------- |
| In           | 在集合中         |
| NotIn        | 不在集合中       |
| Exists       | 标签存在         |
| DoesNotExist | 标签不存在       |
| Gt           | 大于（数值标签） |
| Lt           | 小于（数值标签） |

例如：

```
operator: In
values:
- ssd
- nvme
```

表示：

```
disk

属于

SSD

或者

NVMe
```

------

# 第七节：硬亲和 vs 软亲和

这是 Node Affinity 最重要的知识。

Kubernetes：

提供：

两种：

策略。

------

## ① requiredDuringSchedulingIgnoredDuringExecution

一句话：

**必须满足。**

例如：

```
必须：

GPU
```

没有：

GPU：

就：

不调度。

Pod：

一直：

Pending。

------

## ② preferredDuringSchedulingIgnoredDuringExecution

一句话：

**最好满足。**

例如：

```
最好：

SSD
```

如果：

没有：

SSD。

普通：

磁盘：

也：

可以。

Scheduler：

只是：

优先：

选择：

SSD。

------

## 一个生活中的例子

入住酒店：

第一种：

```
必须：

无烟房
```

没有：

宁可：

不住。

第二种：

```
最好：

高楼层
```

没有：

低楼层：

也：

接受。

------

# 第八节：Pod Affinity（Pod 亲和）

前面：

都是：

选择：

Node。

现在：

变成：

选择：

Pod。

什么意思？

例如：

```
Redis

希望：

和

API

放一起
```

为什么？

因为：

通信：

更快。

于是：

Pod Affinity：

可以：

告诉：

Scheduler：

```
API

尽量：

和 Redis

同一个 Node
```

这样：

减少：

网络：

延迟。

------

# 第九节：Pod Anti-Affinity（Pod 反亲和）

这是企业最常使用的策略之一。

例如：

Deployment：

有：

三个：

副本：

```
API-1

API-2

API-3
```

如果：

全部：

跑：

```
Node1
```

Node1：

宕机。

三个：

全部：

没了。

怎么办？

使用：

Pod Anti-Affinity。

意思：

```
同一个应用

不要

放一起
```

例如：

结果：

变成：

```
Node1

API-1

────────────

Node2

API-2

────────────

Node3

API-3
```

这样：

Node1：

坏了。

还有：

两个：

副本。

------

# 第十节：为什么 StatefulSet 经常使用 Anti-Affinity？

假设：

Redis：

三副本：

```
redis-0

redis-1

redis-2
```

如果：

全部：

运行：

```
Node2
```

Node2：

断电。

整个：

Redis：

集群：

一起：

挂。

所以：

生产环境：

通常：

配置：

Pod Anti-Affinity。

例如：

```
redis-0

↓

Node1

────────────

redis-1

↓

Node2

────────────

redis-2

↓

Node3
```

提高：

高可用。

------

# 第十一节：企业生产环境的典型调度策略

下面是一些真实的生产实践：

| 应用类型                 | 推荐策略                                | 原因                                           |
| ------------------------ | --------------------------------------- | ---------------------------------------------- |
| MySQL                    | Node Affinity（SSD）+ Pod Anti-Affinity | 保证磁盘性能，并避免主从或副本集中在一台 Node  |
| Redis Cluster            | Pod Anti-Affinity                       | 避免多个节点实例因单机故障同时不可用           |
| Kafka                    | Pod Anti-Affinity                       | 提高 Broker 的容灾能力                         |
| Elasticsearch            | Pod Anti-Affinity                       | 防止多个数据节点落在同一台机器                 |
| GPU AI 训练              | Node Affinity（GPU）                    | 确保调度到具备 GPU 的节点                      |
| 日志采集（如 DaemonSet） | 通常无需亲和策略                        | DaemonSet 本身就是每个符合条件的 Node 一个 Pod |

------

# 第十二节：nodeSelector 与 Node Affinity 对比

| 对比项       | nodeSelector      | Node Affinity |
| ------------ | ----------------- | ------------- |
| 简单标签匹配 | ✅                 | ✅             |
| 多条件组合   | ❌                 | ✅             |
| In / NotIn   | ❌                 | ✅             |
| Exists       | ❌                 | ✅             |
| Gt / Lt      | ❌                 | ✅             |
| 软亲和       | ❌                 | ✅             |
| 硬亲和       | ❌（只有必须匹配） | ✅             |

一般建议：

- 简单场景：`nodeSelector`
- 企业生产环境：`Node Affinity`

------

# 第十三节：一次完整的调度决策

假设：

Pod：

要求：

```
必须：

SSD

最好：

GPU

不要：

和同应用

放一起
```

Scheduler：

可能：

执行：

```
① Filter

↓

过滤：

不是 SSD

的 Node

↓

② Score

↓

GPU

加分

↓

③ Anti-Affinity

↓

已有同应用 Pod

减分

↓

④ 最终选择

Node3
```

你会发现：

Scheduler 并不是只看一个条件，而是综合多个规则共同决策。

------

# 第十四节：本章知识关系图

```
                 新 Pod
                    │
                    ▼
            Node Affinity
          （选择哪些 Node）
                    │
                    ▼
           Pod Affinity
      （靠近哪些 Pod）
                    │
                    ▼
        Pod Anti-Affinity
     （远离哪些 Pod）
                    │
                    ▼
             Scheduler
                    │
         Filter + Score
                    │
                    ▼
             最终选择 Node
```

建议把这张图整理到笔记中，它有助于理解 Kubernetes 的调度决策层次。

------

# 第十五节：本章总结（建议牢记）

请记住调度策略最重要的几点：

1. **Node Label 是节点的"标签"，Scheduler 根据标签做决策。**
2. **`nodeSelector` 适用于简单的标签匹配。**
3. **`Node Affinity` 是更强大的节点选择机制，支持复杂表达式和软/硬亲和。**
4. **`Pod Affinity` 用于让 Pod 尽量部署在一起，适合降低通信延迟。**
5. **`Pod Anti-Affinity` 用于让 Pod 尽量分散，提高高可用。**
6. **数据库、Redis、Kafka、Elasticsearch 等有状态应用通常会使用 `Pod Anti-Affinity`。**

------

# 🌟 企业经验：什么时候该用哪一种？

很多新手会纠结到底应该选哪种策略，其实可以用下面这张速查表：

| 需求                         | 推荐方案                                          |
| ---------------------------- | ------------------------------------------------- |
| 固定到 SSD 节点              | `nodeSelector` 或 `Node Affinity`                 |
| 固定到 GPU 节点              | `Node Affinity`                                   |
| 优先使用某类节点，但不是必须 | `preferredDuringSchedulingIgnoredDuringExecution` |
| API 与 Redis 尽量在一起      | `Pod Affinity`                                    |
| 多副本尽量分散               | `Pod Anti-Affinity`                               |

先想清楚你的目标是**选择 Node**还是**考虑其他 Pod 的位置**，再决定使用哪一种机制。

------

# 下一章预告：Taint（污点）与 Toleration（容忍）——企业资源隔离的核心

下一章，我们将学习 Kubernetes 中另一套极其重要的调度机制：

- 什么是 **Taint（污点）**？
- 什么是 **Toleration（容忍）**？
- 为什么 Node 会"拒绝"Pod？
- GPU 节点为什么要打污点？
- 如何防止普通业务占用数据库节点？
- `NoSchedule`、`PreferNoSchedule`、`NoExecute` 有什么区别？
- 企业如何利用污点实现节点资源隔离？

学完这一章，你将掌握 Kubernetes 在生产环境中进行**资源隔离、专用节点管理和高价值资源保护**的核心能力。

# 第五阶段 第三章：Taint（污点）与 Toleration（容忍）——企业资源隔离的核心

> **本章重点：理解 Node 如何主动"拒绝"Pod。**

------

提示词：

```
开始下一章
```

正文：

非常好。

如果说上一章学习的是：

> **"哪些 Pod 可以去哪些 Node。"**

那么这一章学习的是：

> **"哪些 Node 可以拒绝哪些 Pod。"**

很多初学者第一次看到 **Taint（污点）** 和 **Toleration（容忍）**，都会觉得非常抽象。

其实，它们可以理解成一句话：

> **Node 有权拒绝 Pod。**

这是 Kubernetes 企业生产环境中**资源隔离最重要的机制之一**。

很多 GPU 集群、数据库集群、AI 集群、CI/CD 集群，都会大量使用它。

这一章一定要理解思想，而不是死记 YAML。

# 本章学习目标

学习完本章，你应该能够回答：

- 为什么需要 Taint？
- Taint 和 Node Affinity 有什么区别？
- NoSchedule、PreferNoSchedule、NoExecute 分别是什么意思？
- Toleration 为什么不是"允许调度"？
- 为什么 GPU 节点几乎都会使用 Taint？
- 企业如何利用 Taint 做资源隔离？

------

# 第一节：为什么需要 Taint？

我们先来看一个集群。

```
Node1（普通服务器）

Node2（GPU）

Node3（数据库）
```

现在：

来了一个普通 Web API。

如果：

没有任何限制。

Scheduler：

可能：

选择：

```
Node2（GPU）
```

问题来了。

GPU：

价格：

非常贵。

如果：

普通 API：

占用了 GPU 节点。

是不是浪费？

或者：

MySQL：

需要：

SSD。

普通：

日志服务：

却跑到了：

数据库节点。

是不是：

影响性能？

所以：

Node：

需要一种能力：

> **我可以拒绝你。**

这就是：

Taint。

------

# 第二节：什么是 Taint？

一句话：

> **Taint（污点）就是 Node 身上的"禁止进入"标志。**

例如：

Node：

打上：

```
gpu=true
```

这只是：

Label。

不会：

拒绝：

任何 Pod。

但是：

如果：

打上：

```
gpu=true:NoSchedule
```

意思：

```
非 GPU Pod

禁止进入
```

这就是：

Taint。

------

## 一个生活中的例子

想象：

医院。

有：

```
普通病房

ICU
```

ICU 门口：

写着：

```
非工作人员

禁止进入
```

这个：

牌子。

就是：

Taint。

------

# 第三节：Label 与 Taint 的区别

很多新人：

都会混淆。

请一定：

牢记：

## Label

作用：

```
告诉：

Scheduler

我是：

什么机器
```

例如：

```
gpu=true
```

这是：

介绍自己。

------

## Taint

作用：

```
告诉：

Pod

不要来
```

例如：

```
gpu=true:NoSchedule
```

这是：

拒绝别人。

一句话总结：

> **Label 是"自我介绍"，Taint 是"谢绝入内"。**

------

# 第四节：如何添加 Taint？

例如：

GPU Node：

```
kubectl taint nodes node3 gpu=true:NoSchedule
```

查看：

```
kubectl describe node node3
```

会看到：

```
Taints:

gpu=true:NoSchedule
```

删除：

```
kubectl taint nodes node3 gpu=true:NoSchedule-
```

最后面的：

```
-
```

表示：

删除。

------

# 第五节：什么是 Toleration？

很多新人认为：

Toleration：

就是：

允许：

调度。

其实：

不是。

一句话：

> **Toleration 表示："我可以忍受这个污点。"**

注意：

不是：

一定：

调度。

只是：

允许：

进入：

候选列表。

例如：

Pod：

配置：

```
tolerations:
- key: gpu
  operator: Equal
  value: "true"
  effect: NoSchedule
```

意思：

```
我可以运行在

gpu=true

的节点
```

但是：

最终：

是否：

选择：

Node。

仍然：

由：

Scheduler：

决定。

------

## 一个生活中的例子

医院：

ICU：

门口：

写：

```
禁止进入
```

医生：

有：

特殊证件。

可以：

进入。

这个：

证件。

就是：

Toleration。

但是：

医生：

也：

不一定：

必须：

去 ICU。

只是：

有资格：

进去。

------

# 第六节：Taint 的三种 Effect

这是最重要的知识。

------

# ① NoSchedule

最常见。

意思：

> **没有对应 Toleration，就不能调度。**

例如：

```
Node

gpu=true:NoSchedule
```

普通：

Pod：

结果：

```
Pending
```

GPU：

Pod：

有：

Toleration。

可以：

调度。

------

## 适用场景

- GPU
- 数据库
- AI
- 高性能服务器

企业：

最常用。

------

# ② PreferNoSchedule

意思：

> **最好不要调度。**

注意：

不是：

绝对。

例如：

整个：

集群：

都没有：

空闲：

Node。

最后：

还是：

可能：

调度：

过去。

所以：

它属于：

"软限制"。

------

## 一个生活中的例子

酒店：

写：

```
最好：

不要吸烟
```

不是：

禁止。

只是：

建议。

------

# ③ NoExecute

这是：

最容易：

误解：

的：

一种。

它：

不仅：

阻止：

新的：

Pod。

还会：

处理：

已经运行：

的：

Pod。

例如：

Node：

突然：

打上：

```
NoExecute
```

没有：

Toleration：

的：

Pod：

会：

被：

驱逐（Evict）。

------

## 一个生活中的例子

商场：

突然：

发生：

火警。

广播：

```
立即：

撤离
```

所有：

没有：

特殊许可：

的人：

必须：

离开。

这就是：

NoExecute。

------

# 第七节：为什么 Kubernetes 自己也使用 Taint？

很多人不知道：

其实：

Kubernetes：

自己：

每天：

都在：

使用：

Taint。

例如：

Node：

异常：

```
NotReady
```

系统：

自动：

添加：

类似：

```
node.kubernetes.io/not-ready:NoExecute
```

如果：

Pod：

没有：

对应：

容忍。

一段时间后（取决于配置的 `tolerationSeconds`）：

会：

被：

迁移：

到：

其他：

Node。

是不是：

非常：

智能？

------

# 第八节：Node Affinity 与 Taint 到底有什么区别？

这是：

企业：

面试：

高频：

问题。

假设：

GPU：

服务器。

Node Affinity：

是：

```
Pod：

主动：

找 GPU
```

而：

Taint：

是：

```
GPU：

主动：

拒绝

普通 Pod
```

一个：

主动。

一个：

被动。

企业：

通常：

一起：

使用。

例如：

GPU：

训练：

Pod：

```
Node Affinity

+

Toleration
```

这样：

既：

主动：

寻找：

GPU。

又：

允许：

进入：

GPU：

节点。

------

# 第九节：企业真实案例

假设：

公司：

有：

```
Node1

Web

────────────

Node2

GPU

────────────

Node3

MySQL

────────────

Node4

Kafka
```

通常：

配置：

如下：

| Node  | Label      | Taint                 |
| ----- | ---------- | --------------------- |
| GPU   | gpu=true   | gpu=true:NoSchedule   |
| MySQL | db=true    | db=true:NoSchedule    |
| Kafka | kafka=true | kafka=true:NoSchedule |

GPU：

训练：

Pod：

配置：

```
Node Affinity

↓

gpu=true

+

Toleration

↓

gpu=true
```

这样：

只有：

GPU：

程序：

才能：

运行：

GPU：

服务器。

------

# 第十节：为什么不能只用 Node Affinity？

很多新人：

问：

我已经：

Node Affinity：

了。

为什么：

还要：

Taint？

例如：

GPU：

Node。

如果：

只有：

Affinity。

普通：

Pod：

没有：

Affinity。

仍然：

可能：

跑：

GPU。

因为：

Scheduler：

没有：

禁止。

只是：

GPU：

Pod：

更喜欢：

GPU。

所以：

真正：

企业：

都会：

同时：

配置：

```
Affinity

+

Taint
```

一个负责：

"找到正确的节点"。

一个负责：

"挡住错误的 Pod"。

------

# 第十一节：完整工作流程

现在：

把整个过程串起来。

```
             新 Pod
                 │
                 ▼
        Scheduler Filter
                 │
     检查 Node Label
                 │
                 ▼
      检查 Node Taint
                 │
      是否有 Toleration？
           │            │
          否           是
           │            │
           ▼            ▼
       排除 Node    进入候选列表
                         │
                         ▼
                     Score 打分
                         │
                         ▼
                     最终选择 Node
```

这里要注意一个细节：

**Toleration 并不会给节点加分。**

它只是：

告诉 Scheduler：

> **"这个污点我能接受，请不要因为这个污点把我淘汰。"**

------

# 第十二节：企业最佳实践

建议参考下面的思路：

| 场景           | 推荐方案                      |
| -------------- | ----------------------------- |
| GPU 节点       | Label + Taint + Node Affinity |
| 数据库节点     | Label + Taint + StatefulSet   |
| AI 训练节点    | Label + Taint                 |
| 普通 Web API   | 一般无需配置 Taint            |
| CI/CD 专用节点 | Label + Taint                 |

这样既能保证资源利用率，又能避免重要节点被普通业务占用。

------

# 第十三节：新手最容易犯的错误

❌ **误区一：有 Toleration 就一定会调度成功。**

错误。

Toleration 只是"允许"，如果 CPU 不够、Node 不符合 Affinity 等原因，仍然会调度失败。

------

❌ **误区二：Node Affinity 可以代替 Taint。**

错误。

Affinity 是 Pod 的意愿。

Taint 是 Node 的拒绝策略。

二者职责不同。

------

❌ **误区三：给所有节点都打 Taint。**

如果没有对应的 Toleration，最终可能导致：

```
所有 Pod

↓

Pending
```

因此，Taint 应用于需要隔离的节点，而不是默认给所有节点都加。

------

# 第十四节：本章知识关系图

```
              Node
        ┌──────────────┐
        │ Label        │
        │ gpu=true     │
        ├──────────────┤
        │ Taint        │
        │ NoSchedule   │
        └──────────────┘
                ▲
                │
        Node Affinity
      （我要去 GPU）
                │
                ▲
          Toleration
   （我能接受这个污点）
                │
                ▼
         Scheduler 调度
                │
                ▼
          Pod 成功运行
```

------

# 第十五节：本章总结（建议牢记）

请记住 Taint 与 Toleration 最重要的几点：

1. **Label 描述 Node，Taint 保护 Node。**
2. **Toleration 只是允许 Pod 忽略污点，不保证一定会调度成功。**
3. **`NoSchedule` 是生产环境最常用的污点效果。**
4. **`PreferNoSchedule` 是软限制，尽量避免调度。**
5. **`NoExecute` 不仅影响新 Pod，还可能驱逐已运行的 Pod。**
6. **企业通常将 `Node Affinity` 与 `Taint/Toleration` 组合使用，实现精准调度和资源隔离。**

------

# 🌟 企业经验：一套经典组合

在企业生产环境中，最常见的模式是：

```
Node
    │
    ├── Label（说明自己是什么）
    │
    ├── Taint（拒绝不该来的 Pod）
    │
Pod
    │
    ├── Node Affinity（主动寻找目标节点）
    │
    └── Toleration（允许进入有污点的节点）
```

可以把它理解成：

- **Label**：门牌。
- **Node Affinity**：导航，告诉 Pod 去哪里。
- **Taint**：门禁，告诉别人谁不能进。
- **Toleration**：门禁卡，告诉门卫"我是被允许进入的"。

这个模型几乎适用于所有生产环境。

------

# 下一章预告：资源管理与 QoS——Requests、Limits、ResourceQuota、LimitRange

下一章，我们将深入学习 Kubernetes 的资源管理机制，包括：

- 为什么一定要设置 `requests` 和 `limits`？
- CPU 为什么会被"限速"？
- 内存超限为什么会 OOMKilled？
- Kubernetes 的 QoS（服务质量）分为哪三类？
- `ResourceQuota` 如何限制一个 Namespace 的资源总量？
- `LimitRange` 如何为开发团队设置默认资源？
- 企业如何避免某个团队占满整个集群？

学完这一章，你将掌握 Kubernetes **资源管理与资源治理** 的核心能力，这是生产环境稳定运行不可或缺的一部分。

# 第五阶段 第四章：资源管理与 QoS——Requests、Limits、ResourceQuota、LimitRange

> **本章重点：理解 Kubernetes 如何管理 CPU、内存，以及如何防止资源被滥用。**

------

提示词：

```
开始下一章
```

正文：

非常好。

这一章，我们学习 Kubernetes **资源管理（Resource Management）**。

很多人第一次使用 Kubernetes，会觉得：

> **"我的服务器有 64 核 CPU、128GB 内存，为什么还要给 Pod 设置 CPU 和内存？直接让它自己用不就好了？"**

这其实是一个非常危险的想法。

如果 Kubernetes 没有资源管理，那么**一个有 Bug 的程序，就可能拖垮整台服务器，甚至影响整个集群**。

所以，在企业里有一句很经典的话：

> **不会配置资源限制的 Kubernetes，不是真正的生产环境 Kubernetes。**

这一章建议反复理解，因为它直接关系到**集群稳定性、成本控制和故障排查**。

# 本章学习目标

学习完本章，你应该能够回答：

- 为什么要设置 Requests 和 Limits？
- Request 与 Limit 到底有什么区别？
- CPU 超过 Limit 会怎么样？
- 内存超过 Limit 会怎么样？
- OOMKilled 是什么？
- QoS（服务质量）为什么会影响 Pod 被驱逐？
- ResourceQuota 与 LimitRange 分别解决什么问题？

------

# 第一节：为什么 Kubernetes 要管理资源？

先来看一个真实场景。

一台服务器：

```
CPU：16 Core

Memory：64GB
```

上面运行：

```
API A

API B

MySQL

Redis
```

突然：

API A：

因为程序 Bug：

进入死循环。

CPU：

变成：

```
100%
```

结果：

```
MySQL

Redis

API B
```

全部：

变慢。

甚至：

无法响应。

所以：

Kubernetes：

必须：

限制：

每个 Pod：

最多：

能使用：

多少资源。

------

## 一个生活中的例子

公司：

茶水间。

大家：

共用：

饮水机。

如果：

一个人：

拿：

20 个杯子。

别人：

怎么办？

所以：

公司：

规定：

```
每人：

最多：

2 杯
```

资源管理：

也是：

一样。

------

# 第二节：Requests（资源申请）

Request：

一句话：

> **Pod 向 Kubernetes 申请的"最低保障资源"。**

例如：

```
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
```

意思：

```
至少：

给我：

1 Core

2GB
```

Scheduler：

调度：

就是：

根据：

Request。

如果：

Node：

剩余：

只有：

```
CPU

0.5 Core
```

Scheduler：

直接：

过滤。

不会：

调度。

------

## Request 的作用

它主要负责两件事：

1. **调度（Scheduling）**

Scheduler 会检查 Node 是否有足够的可分配资源。

1. **资源预留（Reservation）**

即使程序当前没用到这些资源，Kubernetes 也会认为这些资源已经"预留"给了这个 Pod。

------

# 第三节：Limits（资源上限）

Limit：

一句话：

> **Pod 最多允许使用的资源。**

例如：

```
resources:
  limits:
    cpu: "2"
    memory: "4Gi"
```

意思：

```
CPU：

最多：

2 Core

Memory：

最多：

4GB
```

注意：

Request 和 Limit 可以不同。

例如：

```
requests:
  cpu: "1"

limits:
  cpu: "2"
```

表示：

```
保证：

1 Core

最高：

2 Core
```

------

# 第四节：CPU 超过 Limit 会怎么样？

很多新人以为：

超过：

Limit：

程序：

会：

退出。

其实：

不是。

CPU：

采用：

**限速（Throttling）**。

例如：

Pod：

Limit：

```
2 Core
```

程序：

突然：

想用：

```
6 Core
```

Linux：

Cgroup：

会：

限制：

CPU：

使用率。

程序：

继续：

运行。

只是：

变慢。

所以：

CPU 超限：

通常：

不会：

导致：

Pod：

退出。

------

## 一个生活中的例子

高速公路：

限速：

```
120km/h
```

你：

踩：

油门。

车：

不会：

爆炸。

只是：

不能：

跑：

200km/h。

------

# 第五节：内存超过 Limit 会怎么样？

这里：

和 CPU：

完全不同。

假设：

Limit：

```
Memory：

2Gi
```

程序：

突然：

申请：

```
3Gi
```

结果：

Linux：

OOM Killer：

直接：

杀掉：

进程。

Pod：

状态：

变成：

```
OOMKilled
```

这是 Kubernetes 中最常见的故障之一。

------

## 为什么内存不能像 CPU 一样限速？

因为：

CPU：

是：

"时间片"。

可以：

慢一点。

而：

内存：

不是。

如果：

程序：

真的：

需要：

3GB。

系统：

只有：

2GB：

不能：

"只给一半"。

否则：

程序：

数据：

会：

损坏。

所以：

只能：

终止。

------

# 第六节：OOMKilled 是什么？

很多人第一次看到：

```
OOMKilled
```

不知道：

什么意思。

OOM：

全称：

```
Out Of Memory
```

意思：

**内存不足。**

查看：

```
kubectl describe pod xxx
```

可能：

看到：

```
Last State:

Terminated

Reason:

OOMKilled
```

说明：

程序：

超过：

Memory Limit。

------

## 排查思路

遇到 OOMKilled，不要第一时间只想着把 Limit 调大，可以按这个顺序检查：

1. **应用是否存在内存泄漏？**
2. **Request / Limit 是否设置过小？**
3. **业务流量是否明显增加？**
4. **是否需要优化缓存策略或批量处理逻辑？**

生产环境中，**持续出现 OOMKilled 更值得先检查应用本身，而不是无限增加内存。**

------

# 第七节：Requests 与 Limits 的关系

下面这张图非常重要：

```
CPU

0 ─────── Request ───────── Limit
│              │                │
│        保证资源         最大资源
```

对于：

Memory：

也是：

一样。

Request：

保证。

Limit：

限制。

------

# 第八节：QoS（Quality of Service）

现在：

进入：

企业：

最重要：

的：

知识。

Kubernetes：

根据：

Request：

和：

Limit。

把：

Pod：

分成：

三类。

------

## 第一类：Guaranteed（最高等级）

要求：

```
Request

=

Limit
```

例如：

```
requests:
  cpu: "2"
  memory: "4Gi"

limits:
  cpu: "2"
  memory: "4Gi"
```

特点：

资源保障最强。

当节点资源不足时，**最不容易被驱逐（Evict）**。

适合：

- MySQL
- PostgreSQL
- Kafka
- Elasticsearch 等关键业务。

------

## 第二类：Burstable（可突发）

例如：

```
requests:
  cpu: "1"

limits:
  cpu: "4"
```

意思：

```
保证：

1 Core

最多：

4 Core
```

可以：

临时：

使用：

更多：

CPU。

这是企业最常见的配置。

------

## 第三类：BestEffort（最低等级）

没有：

Request。

没有：

Limit。

例如：

```
resources: {}
```

或者：

干脆：

不写。

这种：

Pod：

没有：

任何：

保障。

Node：

资源：

紧张。

最先：

被：

驱逐。

一般：

不建议：

生产：

使用。

------

# 第九节：Node 资源不足时，谁先被驱逐？

假设：

Node：

内存：

快满了。

Kubernetes：

需要：

驱逐：

Pod。

顺序：

通常：

如下：

```
BestEffort

↓

Burstable

↓

Guaranteed
```

这也是为什么：

数据库：

建议：

使用：

Guaranteed：

QoS。

------

# 第十节：ResourceQuota（资源配额）

前面：

限制：

一个：

Pod。

现在：

限制：

整个：

Namespace。

例如：

开发团队：

Namespace：

```
dev
```

限制：

```
CPU：

最多：

20 Core

Memory：

最多：

64GB
```

这样：

开发：

不能：

无限：

创建：

Pod。

------

## 一个生活中的例子

公司：

预算：

```
研发部：

100 万

市场部：

50 万
```

部门：

内部：

怎么：

花。

自己：

决定。

但是：

不能：

超过：

预算。

------

# 第十一节：LimitRange（默认资源）

很多新人：

忘记：

写：

Resources。

怎么办？

LimitRange：

可以：

自动：

补。

例如：

开发：

没有：

写：

```
resources:
```

系统：

自动：

添加：

```
Request：

500m

Memory：

512Mi
```

这样：

避免：

BestEffort：

Pod。

------

# 第十二节：企业资源治理模型

企业通常会这样设计：

```
Cluster
    │
    ▼
Namespace（团队）
    │
    ├── ResourceQuota
    │
    ▼
Pod
    │
    ├── LimitRange（默认值）
    ├── Request（保证）
    └── Limit（上限）
```

这样可以做到：

- 团队之间互不抢占全部资源。
- 开发人员即使忘记配置资源，也有合理默认值。
- 关键业务获得足够保障。

------

# 第十三节：新手最容易犯的错误

❌ **误区一：只设置 Limit，不设置 Request。**

结果：

Scheduler 无法正确预留资源，或者在某些集群策略下被自动补默认值，导致调度结果不符合预期。

------

❌ **误区二：Request 设置得特别大。**

例如：

程序：

实际：

需要：

```
500m
```

却：

写：

```
8 Core
```

Scheduler：

认为：

必须：

预留：

8 Core。

最终：

Pod：

一直：

Pending。

------

❌ **误区三：所有 Pod 都配置 Guaranteed。**

Guaranteed 并不等于"更好"。

如果所有应用都把 Request 设置得很高，集群会出现资源碎片，实际利用率下降。

很多 Web API 更适合 Burstable 配置。

------

# 第十四节：知识关系图

```
                 Pod
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
    Request             Limit
 （最低保障）          （最大限制）
        │                   │
        ▼                   ▼
   Scheduler           Linux Cgroup
        │                   │
        ▼                   ▼
      调度              CPU限速 / OOMKilled
                  │
                  ▼
                QoS
                  │
                  ▼
  Guaranteed / Burstable / BestEffort
```

------

# 第十五节：企业最佳实践

下面是一份非常常见的资源配置建议：

| 应用类型      | Request             | Limit                 | QoS 建议   |
| ------------- | ------------------- | --------------------- | ---------- |
| Web API       | 较低                | 较高                  | Burstable  |
| 前端（Nginx） | 较低                | 较低                  | Burstable  |
| MySQL         | 与 Limit 相同       | 与 Request 相同       | Guaranteed |
| Redis         | 与 Limit 相同或接近 | 与 Request 相同或接近 | Guaranteed |
| Kafka         | 与 Limit 相同       | 与 Request 相同       | Guaranteed |
| 临时 Job      | 根据实际需求        | 略高于 Request        | Burstable  |

------

# 第十六节：本章总结（建议牢记）

请记住资源管理最重要的几点：

1. **Request 决定调度，表示最低保障资源。**
2. **Limit 决定运行时资源上限。**
3. **CPU 超过 Limit 通常会被限速（Throttling），不会直接退出。**
4. **内存超过 Limit 会触发 OOMKilled。**
5. **QoS 分为 Guaranteed、Burstable、BestEffort，资源紧张时驱逐优先级依次为：BestEffort → Burstable → Guaranteed。**
6. **ResourceQuota 用于限制整个 Namespace 的资源总量。**
7. **LimitRange 用于提供默认资源配置，避免出现没有资源限制的 Pod。**

------

# 🌟 到这里，你已经掌握了 Kubernetes 的资源管理体系

现在可以把整个体系串起来：

```
             Namespace
                  │
         ResourceQuota（总预算）
                  │
                  ▼
              Pod 创建
                  │
         LimitRange（默认资源）
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   Request              Limit
（决定调度）        （决定运行上限）
        │                   │
        ▼                   ▼
   Scheduler          Linux Cgroup
                  │
                  ▼
                 QoS
```

这套机制是 Kubernetes 能够让多个团队、多个应用共享同一集群而保持稳定运行的关键。

------

# 下一章预告：滚动更新与应用发布——RollingUpdate、Recreate、金丝雀发布、蓝绿发布

下一章，我们将进入 Kubernetes 在企业中最常用的发布能力，学习：

- 为什么 Deployment 更新不会中断服务？
- RollingUpdate（滚动更新）到底是如何工作的？
- `maxSurge` 和 `maxUnavailable` 如何影响升级过程？
- Recreate 模式什么时候才适合使用？
- 什么是金丝雀发布（Canary）？
- 什么是蓝绿发布（Blue-Green）？
- Kubernetes 原生能做到哪些发布策略，哪些需要配合 Ingress Controller 或服务网格？

学完这一章，你将掌握 Kubernetes 从**开发发布到生产上线**的核心实践，也是 DevOps 流程中的重要一环。

# 第六阶段 第一章：滚动更新与应用发布——RollingUpdate、Recreate、金丝雀发布、蓝绿发布

> **本章重点：理解 Kubernetes 如何实现不停机发布。**

------

提示词：

```
开始下一章
```

正文：

非常好。

到目前为止，你已经学会了：

- 如何部署应用（Deployment）
- 如何暴露服务（Service）
- 如何配置流量入口（Ingress）
- 如何管理配置（ConfigMap、Secret）
- 如何持久化数据（PV、PVC、StatefulSet）
- 如何控制调度（Scheduler、Affinity、Taint）
- 如何管理资源（Request、Limit、QoS）

现在，我们进入 Kubernetes 最核心、也是企业每天都会使用的能力：

> **应用发布（Application Release）**

如果说 Docker 解决的是：

> **"如何运行应用。"**

那么 Kubernetes 解决的是：

> **"如何不停机地升级应用。"**

几乎所有互联网公司，都在使用 Kubernetes 做应用发布。

而其中最重要的，就是：

- Rolling Update（滚动更新）
- Rollback（回滚）
- Blue-Green（蓝绿发布）
- Canary（金丝雀发布）

这一章建议认真理解，因为它不仅是 Kubernetes 知识，也是 DevOps 和 CI/CD 的基础。

# 本章学习目标

学习完本章，你应该能够回答：

- Deployment 为什么可以不停机升级？
- RollingUpdate 到底是怎么工作的？
- maxSurge 与 maxUnavailable 有什么作用？
- Recreate 与 RollingUpdate 的区别是什么？
- Rollback 为什么能快速恢复？
- 蓝绿发布与金丝雀发布分别适用于哪些场景？
- Kubernetes 原生支持哪些发布策略？

------

# 第一节：为什么不能直接停止旧版本？

假设：

你的系统：

有：

```
API

3 个 Pod
```

用户：

正在：

访问。

现在：

发布：

新版本。

最简单：

的方法：

就是：

```
删除

↓

重新创建
```

例如：

```
Pod1

↓

删除

Pod2

↓

删除

Pod3

↓

删除

↓

创建

新版本
```

会发生什么？

整个过程：

没有：

任何：

Pod。

用户：

访问：

```
502

503
```

网站：

中断。

所以：

生产环境：

不能：

这样：

升级。

------

# 第二节：Rolling Update（滚动更新）

一句话：

> **一边删除旧 Pod，一边创建新 Pod。**

例如：

原来：

```
v1

Pod1

Pod2

Pod3
```

升级：

到：

v2。

不是：

全部：

删除。

而是：

这样：

```
Pod1(v1)

↓

Pod1(v2)

↓

Pod2(v1)

↓

Pod2(v2)

↓

Pod3(v1)

↓

Pod3(v2)
```

整个：

升级：

过程中：

一直：

都有：

Pod：

提供：

服务。

用户：

几乎：

感觉不到。

------

## 一个生活中的例子

高速公路：

维修。

不会：

全部：

封闭。

而是：

```
先修：

一条车道

另一条：

继续通车
```

修好：

以后：

再：

修：

另一条。

Rolling Update：

就是：

这个：

思想。

------

# 第三节：Deployment 为什么能做到不停机？

回忆：

Deployment：

其实：

管理：

的是：

ReplicaSet。

例如：

当前：

```
Deployment

↓

ReplicaSet(v1)

↓

3 Pods
```

升级：

以后：

变成：

```
Deployment

├── ReplicaSet(v1)

└── ReplicaSet(v2)
```

Deployment：

慢慢：

减少：

v1。

慢慢：

增加：

v2。

直到：

```
ReplicaSet(v1)

0 Pod

ReplicaSet(v2)

3 Pod
```

所以：

Deployment：

其实：

是在：

同时：

管理：

两个：

ReplicaSet。

------

# 第四节：maxSurge（最大额外 Pod）

这是 Rolling Update 最重要的参数。

例如：

原来：

```
3 Pods
```

配置：

```
maxSurge: 1
```

意思：

升级：

过程中：

允许：

多出来：

```
1 个 Pod
```

于是：

升级：

第一步：

```
旧：

3

新：

1

总共：

4
```

等：

确认：

新 Pod：

正常。

再：

删除：

一个：

旧 Pod。

这样：

服务：

始终：

有：

足够：

实例。

------

## 为什么需要多出来的 Pod？

假设：

没有：

额外：

Pod。

必须：

先：

删：

旧。

才能：

建：

新。

如果：

新版本：

启动：

很慢。

用户：

请求：

可能：

不够：

处理。

所以：

允许：

临时：

增加：

Pod。

------

# 第五节：maxUnavailable（最大不可用）

例如：

配置：

```
maxUnavailable: 1
```

意思：

升级：

过程中：

最多：

允许：

```
1 个 Pod

不可用
```

例如：

原来：

```
Pod1

Pod2

Pod3
```

允许：

先：

删除：

一个。

剩：

```
Pod2

Pod3
```

然后：

创建：

新的。

如果：

设置：

```
maxUnavailable: 0
```

表示：

升级：

期间：

**任何时刻都必须保持所有期望副本可用**。

这通常用于对可用性要求极高的业务，但需要确保集群有足够资源配合 `maxSurge` 创建新 Pod。

------

# 第六节：Rolling Update 整个流程

例如：

3 个：

副本。

```
Step1

旧：

3

新：

0
```

↓

```
Step2

旧：

3

新：

1
```

↓

```
Step3

旧：

2

新：

1
```

↓

```
Step4

旧：

2

新：

2
```

↓

```
Step5

旧：

1

新：

2
```

↓

```
Step6

旧：

1

新：

3
```

↓

```
Step7

旧：

0

新：

3
```

整个：

过程中：

一直：

有：

服务。

------

# 第七节：Recreate（重建）

另一种：

策略：

```
strategy:
  type: Recreate
```

意思：

```
全部：

删除

↓

全部：

创建
```

没有：

滚动。

没有：

平滑。

------

## 什么时候使用？

极少。

例如：

数据库：

升级。

或者：

应用：

必须：

保证：

同一时间：

只能：

运行：

一个：

实例。

否则：

Deployment：

默认：

都是：

RollingUpdate。

------

# 第八节：Rollout History（发布历史）

Deployment：

会：

保存：

历史：

版本。

查看：

```
kubectl rollout history deployment api
```

结果：

```
Revision 1

Revision 2

Revision 3
```

每次：

更新。

都会：

记录。

------

# 第九节：Rollback（回滚）

这是：

生产环境：

救命：

功能。

假设：

发布：

v2。

结果：

Bug。

怎么办？

执行：

```
kubectl rollout undo deployment api
```

Deployment：

立即：

恢复：

上一版。

因为：

旧的：

ReplicaSet：

还在。

所以：

回滚：

非常：

快。

------

## 一个生活中的例子

Word：

保存：

多个：

版本。

今天：

写坏：

文档。

点击：

```
恢复：

昨天
```

Deployment：

也是：

一样。

------

# 第十节：蓝绿发布（Blue-Green）

Blue：

旧版本。

Green：

新版本。

例如：

```
Blue

API-v1

────────────

Green

API-v2
```

两套：

同时：

运行。

用户：

流量：

全部：

进入：

Blue。

测试：

Green：

没问题。

最后：

切换：

Service：

或者：

Ingress：

全部：

进入：

Green。

Blue：

保留。

如果：

出问题。

立即：

切：

回去。

------

## 优点

回滚：

极快。

几乎：

秒级。

------

## 缺点

资源：

翻倍。

因为：

两套：

一起：

运行。

------

# 第十一节：金丝雀发布（Canary）

一句话：

> **先让一小部分用户使用新版本。**

例如：

```
90%

↓

v1
10%

↓

v2
```

观察：

一天。

没有：

问题。

改成：

```
50%

↓

v2
```

最后：

```
100%

↓

v2
```

------

## 为什么叫金丝雀？

以前：

煤矿：

工人：

下井。

先：

带：

金丝雀。

如果：

有毒气。

金丝雀：

先：

出问题。

工人：

立即：

撤离。

所以：

新版本：

先：

给：

少量：

用户。

------

# 第十二节：Kubernetes 原生支持金丝雀吗？

严格来说：

**Deployment 本身并不直接支持按百分比分流的金丝雀发布。**

Deployment 负责：

- 创建 Pod
- 滚动更新
- 回滚

真正的：

10%：

20%：

50%：

流量控制。

通常：

依赖：

- Ingress Controller（例如 NGINX Ingress 的 Canary 功能）
- 服务网格（如 Istio、Linkerd）

这些组件可以按请求比例、Header、Cookie、用户等维度进行精细流量控制。

------

# 第十三节：企业真实发布流程

一个典型流程如下：

```
开发提交代码
        │
        ▼
CI 构建镜像
        │
        ▼
推送镜像仓库
        │
        ▼
更新 Deployment
        │
        ▼
Rolling Update
        │
        ▼
监控
（日志、指标、告警）
        │
        ├──────────┐
        ▼          ▼
    正常       出现异常
        │          │
        ▼          ▼
   发布完成   Rollback
```

注意：

**Kubernetes 负责部署。**

CI/CD：

负责：

自动：

执行：

这些：

步骤。

------

# 第十四节：各种发布方式对比

| 发布方式       | 是否停机 | 回滚速度 | 资源消耗 | 适用场景                  |
| -------------- | -------- | -------- | -------- | ------------------------- |
| Recreate       | ❌ 会停机 | 快       | 低       | 极少使用                  |
| Rolling Update | ✅ 不停机 | 快       | 较低     | 默认方案，大多数 Web 服务 |
| Blue-Green     | ✅ 不停机 | 极快     | 高       | 核心业务、需要快速切换    |
| Canary         | ✅ 不停机 | 快       | 中等     | 新功能验证、降低发布风险  |

------

# 第十五节：新手最容易犯的错误

❌ **误区一：Rolling Update 一定不会影响用户。**

不是。

如果：

新版本：

启动：

失败。

或者：

Readiness Probe：

配置：

错误。

流量：

仍然：

可能：

受到：

影响。

因此：

**健康检查（Readiness Probe）是滚动更新成功的重要保障。**

------

❌ **误区二：Rollout Undo 可以恢复数据库。**

错误。

Rollback：

恢复：

Deployment：

不会：

恢复：

数据库：

数据。

数据库：

需要：

备份。

------

❌ **误区三：蓝绿发布就是 Rolling Update。**

不是。

Rolling Update：

逐步：

替换。

Blue-Green：

两套：

一起：

存在。

------

# 第十六节：知识关系图

```
                Deployment
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   ReplicaSet(v1)        ReplicaSet(v2)
          │                     │
     逐步减少 Pod          逐步增加 Pod
          └──────────┬──────────┘
                     ▼
               Rolling Update
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
     发布成功               发布失败
         │                       │
         ▼                       ▼
      完成升级             Rollback
```

------

# 第十七节：企业最佳实践

下面是比较成熟的建议：

| 场景           | 推荐方式                                                     |
| -------------- | ------------------------------------------------------------ |
| 普通 Web API   | Rolling Update                                               |
| 核心支付系统   | Blue-Green + 快速切换                                        |
| 新功能灰度验证 | Canary                                                       |
| 数据库升级     | 根据具体产品制定方案，很多情况下需要停机窗口或主从切换，而不是依赖 Deployment 策略 |
| 高频迭代业务   | Rolling Update + 自动回滚机制                                |

------

# 第十八节：本章总结（建议牢记）

请记住应用发布最重要的几点：

1. **Deployment 默认采用 RollingUpdate，实现不停机升级。**
2. **`maxSurge` 控制升级过程中允许额外创建多少 Pod。**
3. **`maxUnavailable` 控制升级过程中允许多少 Pod 不可用。**
4. **Recreate 会先删除旧版本，再创建新版本，会导致服务中断。**
5. **Deployment 保留历史 ReplicaSet，因此支持快速 Rollback。**
6. **Blue-Green 通过两套环境切换流量，回滚速度最快，但资源开销最大。**
7. **Canary 通过少量流量验证新版本，降低发布风险，通常需要 Ingress Controller 或服务网格配合实现。**

------

# 🌟 企业经验：发布策略如何选择？

可以参考下面这张速查表：

| 业务特点             | 推荐策略                                           |
| -------------------- | -------------------------------------------------- |
| 一般 Web 服务        | Rolling Update                                     |
| 对停机极其敏感       | Blue-Green                                         |
| 新功能逐步验证       | Canary                                             |
| 希望发布失败自动恢复 | Rolling Update + 健康检查 + 自动回滚（结合 CI/CD） |

------

# 下一章预告：Probe 深入原理与应用生命周期管理

# 第六阶段 第二章：Probe 深入原理与应用生命周期管理

> **关键词：kubelet、Probe 执行机制、优雅终止（Graceful Shutdown）、滚动发布、零停机、自愈**

------

提示词（当前章节后较多重复内容，执行过回退操作）：

```
开始下一章
```

正文：

很好，我们从这里正式**回退**。

从这一章开始，我会把前面的基础知识都视为**你已经掌握**，不会再解释 Pod、Deployment、Service、Probe 是什么，而是深入它们的工作机制、企业实践和生产环境经验。

我们的目标也正式调整为：

> **培养能够独立负责 Kubernetes 生产环境的平台工程师（Platform Engineer）或高级运维工程师，而不仅仅是会部署应用。**

# 本章学习目标

完成本章后，你应该能够回答：

- kubelet 是如何执行 Probe 的？
- 为什么 Readiness 失败不会重启 Pod，而 Liveness 会？
- Probe 与 Pod 生命周期有什么关系？
- 为什么很多公司的 Pod 会一直处于 Running，却无法提供服务？
- 如何让 ASP.NET Core 应用真正做到零停机发布？
- 企业为什么要把 `/health/live` 和 `/health/ready` 分开？
- 如何设计一个不会误判的健康检查？

------

# 第一节：重新认识 Probe —— 它不是 Deployment 的功能，而是 kubelet 的职责

很多初学者有一个误解：

> Deployment 在检查 Pod 是否健康。

实际上并不是。

Probe 的执行者不是 Deployment，而是：

```
Node
└── kubelet
      ├── 创建 Pod
      ├── 拉取镜像
      ├── 启动容器
      ├── 执行 Probe
      └── 上报状态给 API Server
```

这里要建立一个重要认知：

> **每个 Node 上运行的 kubelet，负责管理本节点所有 Pod 的生命周期。**

也就是说：

假设你的集群有 20 台 Node。

那么就有：

```
20 个 kubelet
```

每个 kubelet：

只管理自己节点上的 Pod。

因此：

Probe 完全发生在：

```
Node 本地
```

并不会：

每次都请求 API Server。

------

# 第二节：Probe 的执行流程（底层原理）

我们来看一次 HTTP Probe 到底发生了什么。

假设 Deployment 创建了一个 Pod。

流程如下：

```
Deployment
      │
      ▼
Scheduler
      │
      ▼
Node-1
      │
      ▼
kubelet
      │
      ▼
Container Runtime(containerd)
      │
      ▼
启动容器
      │
      ▼
kubelet 开始定时执行 Probe
      │
      ▼
HTTP GET /health/ready
```

注意：

HTTP 请求：

**不是来自集群其它 Pod。**

也不是来自：

Service。

而是：

> kubelet 自己发起。

这意味着：

Probe 即使失败，也不会经过 Ingress、Service 等网络路径。

------

# 第三节：为什么 Readiness 不会重启 Pod？

这是 Kubernetes 面试最经典的问题之一。

很多人会回答：

> 因为设计如此。

这不是答案。

真正原因要理解控制面的职责划分。

Readiness 的职责只有一个：

> **告诉 Endpoint Controller：这个 Pod 能不能加入 Service。**

整个过程如下：

```
Readiness Success
        │
        ▼
Pod Ready=True
        │
        ▼
EndpointSlice Controller
        │
        ▼
加入 Service Endpoint
```

如果失败：

```
Readiness Failed
        │
        ▼
Ready=False
        │
        ▼
从 EndpointSlice 中移除
```

注意：

整个过程中：

**没有任何一步涉及容器重启。**

因为：

Readiness：

影响的是：

**流量**

而不是：

**生命周期**

------

# 第四节：为什么 Liveness 会重启？

Liveness 完全不同。

它直接影响：

Container Runtime。

流程：

```
Liveness Failed
        │
        ▼
kubelet
        │
        ▼
Kill Container
        │
        ▼
Restart Container
```

这里很多新人不知道：

真正执行：

```
kill
restart
```

的不是 Kubernetes API。

而是：

containerd（或其他 CRI Runtime）。

kubelet：

只是：

发命令。

例如：

```
kubelet

↓

CRI

↓

containerd

↓

SIGTERM

↓

SIGKILL

↓

重新启动
```

所以：

Liveness：

属于：

**生命周期管理。**

Readiness：

属于：

**流量管理。**

一定不要混。

------

# 第五节：为什么 Startup Probe 能解决启动慢的问题？

很多教程会告诉你：

> Startup Probe 可以防止误杀。

但是：

为什么？

来看时间轴。

假设：

应用启动：

需要：

90 秒。

Liveness：

配置：

```
initialDelaySeconds: 10
periodSeconds: 5
failureThreshold: 3
```

意味着：

```
10s 开始检查

15s Fail

20s Fail

25s Fail

↓

Restart
```

应用：

永远：

启动不了。

加入 Startup Probe 后：

```
Startup Probe
        │
        ▼
启动期间：
Liveness 暂停
        │
Startup 成功
        │
        ▼
开始执行 Liveness
```

所以：

Startup 并不是第四种检查。

它实际上是：

> **Liveness 的"保护罩"。**

------

# 第六节：Readiness 与 Rolling Update 的真正关系

前面我们已经知道：

Rolling Update：

依赖：

Readiness。

现在来看完整流程。

假设：

Deployment：

更新：

v2。

流程：

```
创建新 Pod
      │
      ▼
Container Running
      │
      ▼
Readiness=False
      │
      ▼
Service 不发送流量
      │
      ▼
初始化完成
      │
      ▼
Readiness=True
      │
      ▼
加入 EndpointSlice
      │
      ▼
开始接收请求
      │
      ▼
Deployment 删除旧 Pod
```

这里有一个重要细节：

**Deployment 并不是等待容器 Running。**

它等待的是：

```
Ready=True
```

所以：

真正保证零停机的不是：

Running。

而是：

Ready。

------

# 第七节：Pod 删除时到底发生了什么？

很多人认为：

```
kubectl delete pod
```

就是：

立刻删除。

其实不是。

真实流程如下：

```
kubectl delete pod
        │
        ▼
API Server
        │
        ▼
Pod 状态：
Terminating
        │
        ▼
EndpointSlice 删除该 Pod
        │
        ▼
Service 停止转发流量
        │
        ▼
PreStop Hook（如果配置）
        │
        ▼
发送 SIGTERM
        │
        ▼
等待 terminationGracePeriodSeconds
        │
        ▼
SIGKILL
        │
        ▼
Pod 删除
```

这一流程非常重要，因为它决定了应用是否能**优雅退出**。

------

# 第八节：什么是优雅终止（Graceful Shutdown）？

假设你的 ASP.NET Core API 正在处理一个上传文件请求。

需要：

20 秒。

这时：

Deployment：

升级。

如果直接：

```
kill -9
```

用户上传到一半：

连接断开。

体验很差。

Kubernetes 的设计是：

1. **先把 Pod 从 Service 中移除。**
2. **不再接收新请求。**
3. **允许当前请求处理完成。**
4. **再关闭进程。**

这就是优雅终止。

------

# 第九节：ASP.NET Core 如何配合 Kubernetes？

这是与你最相关的部分。

ASP.NET Core 默认会响应 `SIGTERM`，进入 Host 的关闭流程。

建议结合：

- `IHostApplicationLifetime`（或新版 `IHostApplicationLifetime.ApplicationStopping`）
- 后台服务（`BackgroundService`）正确处理取消令牌（`CancellationToken`）
- 合理设置 `terminationGracePeriodSeconds`

推荐流程：

```
SIGTERM
      │
      ▼
ApplicationStopping
      │
      ▼
停止接收新任务
      │
      ▼
等待当前请求完成
      │
      ▼
释放数据库连接
      │
      ▼
关闭应用
```

这样 Rolling Update 才能真正做到平滑。

------

# 第十节：企业健康检查接口设计

很多团队把：

```
/health
```

写成：

```
{
  "status": "ok"
}
```

永远返回 200。

这种接口几乎没有价值。

推荐职责划分：

| 接口                      | 检查内容                        | 用途            |
| ------------------------- | ------------------------------- | --------------- |
| `/health/live`            | 进程是否正常、主循环是否运行    | Liveness Probe  |
| `/health/ready`           | 数据库、Redis、MQ、必要外部依赖 | Readiness Probe |
| `/health/startup`（可选） | 应用初始化是否完成              | Startup Probe   |

这样设计可以避免：

- 数据库短暂故障导致容器不断重启。
- 应用尚未初始化完成就开始接收流量。

------

# 第十一节：企业最常见的三个 Probe 配置错误

### 错误一：Liveness 检查数据库

数据库短暂抖动：

```
Liveness Fail
    │
    ▼
Pod 重启
```

数据库仍然没恢复。

于是：

不停重启。

正确做法：

数据库应该放到 Readiness。

------

### 错误二：Readiness 检查耗时太长

例如：

每次都调用：

第三方支付接口。

Probe：

变成：

3 秒。

整个节点：

可能有几百个 Pod。

kubelet：

检查压力巨大。

健康检查应该：

- 快
- 稳定
- 可预测

------

### 错误三：terminationGracePeriodSeconds 设置过小

例如：

```
terminationGracePeriodSeconds: 3
```

你的请求平均：

8 秒。

升级时：

请求全部中断。

通常需要根据业务请求耗时进行评估，而不是盲目使用很小的值。

------

# 第十二节：本章总结

这一章，请真正建立下面几个概念：

1. **Probe 的执行者是 kubelet，而不是 Deployment。**
2. **Readiness 管理的是流量，不管理容器生命周期。**
3. **Liveness 管理的是容器生命周期，失败会触发重启。**
4. **Startup Probe 本质上是 Liveness 的启动保护机制。**
5. **Rolling Update 等待的是 Pod Ready，而不是 Pod Running。**
6. **优雅终止（Graceful Shutdown）是零停机发布的重要组成部分。**
7. **健康检查接口应该按职责拆分，而不是所有检查都放进一个 `/health`。**

------

# 🌟 企业经验

很多团队把 Kubernetes 的健康检查理解为"探活"，其实真正的价值在于：

> **让 Kubernetes、应用程序和业务流量协同工作。**

一个设计良好的应用生命周期应该是：

```
启动
  │
  ▼
Startup Probe
  │
  ▼
Readiness Ready
  │
  ▼
接收流量
  │
  ▼
Liveness 持续监控
  │
  ▼
收到 SIGTERM
  │
  ▼
停止接收新流量
  │
  ▼
处理完已有请求
  │
  ▼
安全退出
```

理解这条完整链路，比单纯记住三个 Probe 的定义更重要。

------

## 下一章预告：Prometheus Operator 与 Kubernetes 监控体系

下一章我们不再介绍"Prometheus 是什么"，而是直接进入生产环境最常见的监控架构，重点学习：

- 为什么企业几乎都使用 **Prometheus Operator** 而不是手工部署 Prometheus？
- `ServiceMonitor`、`PodMonitor`、`PrometheusRule` 分别是什么？
- Exporter 如何接入监控体系？
- Prometheus 如何自动发现 Kubernetes 中的新服务？
- 如何设计一套可维护、可扩展的企业级监控平台？

从下一章开始，我们正式进入 Kubernetes 生态中最重要的 **Operator 模式**。

# 第六阶段 第三章：Prometheus Operator 与 Kubernetes 监控体系

> **关键词：Operator、CRD、ServiceMonitor、PodMonitor、PrometheusRule、Exporter**

------

提示词：

```
开始下一章
```

正文：

很好，从这一章开始，我们正式进入 **Kubernetes 生态**。

如果说前面的内容主要是在学习 **Kubernetes 本身**，那么从现在开始，你将学习的是：

> **企业是如何围绕 Kubernetes 构建整个运维平台的。**

这一章会涉及 **Operator**，这是 Kubernetes 最重要的扩展机制之一。

很多开发者会使用 Prometheus，却不知道为什么现在几乎没人再手动部署 Prometheus，而是都用 **Prometheus Operator**。

学完这一章，你不仅会知道怎么用，更会理解 Kubernetes 的扩展思想，这对后面学习 Helm、Argo CD、Cert-Manager、Istio 等都会有帮助。

# 本章学习目标

完成本章后，你应该能够回答：

- 为什么企业更倾向于使用 Prometheus Operator？
- Operator 与普通 Deployment 有什么区别？
- ServiceMonitor、PodMonitor 是如何工作的？
- Prometheus 如何自动发现新服务？
- Exporter 应该部署在哪里？
- 企业如何组织一套可维护的监控体系？

------

# 第一节：先回顾一个问题——为什么手动维护 Prometheus 很痛苦？

假设没有 Operator。

你部署了一个 Prometheus。

Prometheus 的配置文件（`prometheus.yml`）里可能写着：

```
scrape_configs:
  - job_name: order-service
    static_configs:
      - targets:
          - 10.10.1.12:8080

  - job_name: payment-service
    static_configs:
      - targets:
          - 10.10.1.18:8080
```

问题来了。

第二天：

Deployment 滚动更新。

新的 Pod IP：

```
10.10.1.36
```

旧的：

```
10.10.1.18
```

已经消失。

那么：

Prometheus 怎么知道？

如果还是静态配置。

它永远抓不到新的 Pod。

所以：

**静态配置完全不适合 Kubernetes。**

------

# 第二节：Kubernetes 的思想——不要写死 IP

Kubernetes 一直强调：

> **Everything is Dynamic（万物都是动态的）**

Pod：

会重建。

Node：

会增加。

IP：

会变化。

如果每变化一次，都要人工修改 Prometheus 配置：

那么 Kubernetes 就失去了自动化意义。

因此：

Prometheus 必须：

> **自动发现目标。**

------

# 第三节：Prometheus 的 Kubernetes Service Discovery

Prometheus 实际上支持：

```
Kubernetes API
```

它可以直接查询：

```
API Server
```

例如：

查询：

```
Namespace

↓

Service

↓

EndpointSlice

↓

Pod
```

于是：

Prometheus 可以知道：

```
现在有哪些 Pod？

它们的 IP 是多少？

哪些已经 Ready？

哪些已经删除？
```

这就是：

Service Discovery（服务发现）。

------

## 为什么企业还需要 Operator？

如果只有 Service Discovery。

你仍然需要：

维护：

```
scrape_configs
```

例如：

哪些 Service 要抓？

抓哪个端口？

抓哪个路径？

抓多久一次？

这些配置还是需要人工维护。

Operator 就是为了解决这个问题。

------

# 第四节：什么是 Operator？

这一节非常重要。

很多人第一次接触 Operator 时都会觉得抽象。

其实可以把它理解成：

> **一个懂业务的 Controller。**

还记得我们之前学过：

Deployment Controller。

它负责：

```
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

Deployment Controller：

懂 Deployment。

现在：

Prometheus Operator：

懂 Prometheus。

它负责：

```
Prometheus CR
       ↓
生成 Deployment

生成 StatefulSet

生成 Config

自动 Reload
```

Operator 本质上仍然是：

> **Controller。**

只是它管理的对象：

不是 Pod。

而是：

Prometheus。

------

# 第五节：Operator = CRD + Controller

这是 Kubernetes 最经典的公式。

```
Operator

=

CRD

+

Controller
```

什么意思？

CRD：

负责：

新增一种 Kubernetes 资源。

例如：

```
Prometheus
```

现在：

集群里：

可以执行：

```
kubectl get prometheus
```

再例如：

```
ServiceMonitor
```

于是：

可以：

```
kubectl get servicemonitors
```

这些资源：

原生 Kubernetes 是没有的。

都是：

Operator 安装时创建的。

------

Controller：

负责：

持续监听这些资源。

例如：

发现：

新增：

```
kind: ServiceMonitor
```

立即：

修改：

Prometheus 配置。

整个过程：

无需人工。

------

# 第六节：ServiceMonitor 是什么？

这是 Prometheus Operator 最重要的资源。

很多初学者第一次看到：

```
kind: ServiceMonitor
```

不知道它是什么。

实际上：

它不是 Service。

它只是告诉 Prometheus：

> **请监控符合条件的 Service。**

例如：

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

spec:
  selector:
    matchLabels:
      app: order-api
```

意思就是：

找到：

所有：

```
labels:
  app: order-api
```

的 Service。

然后：

自动：

抓：

它们。

------

## 为什么不是监控 Pod？

这是一个设计思想。

企业通常：

希望监控的是：

**服务**

而不是：

某一个：

Pod。

Pod：

可以随时销毁。

Service：

代表：

业务。

------

# 第七节：PodMonitor 又是什么？

有些程序：

没有：

Service。

例如：

DaemonSet：

```
Node Exporter
```

它：

每个 Node 一个 Pod。

没有必要：

建立：

Service。

怎么办？

使用：

```
kind: PodMonitor
```

直接：

匹配：

Pod Label。

例如：

```
selector:
  matchLabels:
    app: node-exporter
```

Prometheus：

直接抓：

Pod。

------

# 第八节：PrometheusRule

监控不仅要采集。

还要告警。

例如：

CPU：

超过：

90%。

PrometheusRule：

就是：

把规则写成 Kubernetes 资源。

例如：

```
kind: PrometheusRule
```

里面：

定义：

```
CPU > 90%

持续 5 分钟

发送告警
```

Operator：

自动：

同步：

到：

Prometheus。

无需：

修改：

配置文件。

------

# 第九节：Exporter 的作用

Prometheus：

只能：

采集：

Metrics。

但是：

很多程序：

不会：

自己：

暴露：

Metrics。

例如：

MySQL。

怎么办？

安装：

```
mysqld-exporter
```

Redis：

安装：

```
redis-exporter
```

Kafka：

安装：

```
kafka-exporter
```

Exporter：

负责：

把业务数据转换成：

Prometheus 能理解的格式。

例如：

```
mysql_connections 18

mysql_queries_total 108000
```

------

# 第十节：企业监控架构

企业一般会采用下面这种模式：

```
                    Prometheus Operator
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
     ServiceMonitor                 PrometheusRule
             │                           │
             ▼                           ▼
    自动发现 Service                自动同步规则
             │
             ▼
        Prometheus
             │
     抓取所有 Metrics
             │
             ▼
          Grafana
```

这里有一个重要特点：

**Prometheus 的配置几乎不再手工编辑。**

全部通过 Kubernetes 资源管理。

------

# 第十一节：为什么企业喜欢 Operator？

因为它符合 Kubernetes 的声明式思想。

不是：

```
修改配置

↓

重启 Prometheus
```

而是：

```
kubectl apply

↓

ServiceMonitor

↓

Operator 自动处理
```

整个过程：

没有人工维护配置文件。

------

# 第十二节：企业最佳实践

一般建议：

```
Application
      │
      ▼
暴露 /metrics
      │
      ▼
Service
      │
      ▼
ServiceMonitor
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

如果是：

Node：

则：

```
Node Exporter

↓

PodMonitor
```

------

# 第十三节：本章容易混淆的几个概念

| 名称                | 作用                             | 是否 Kubernetes 原生 |
| ------------------- | -------------------------------- | -------------------- |
| Prometheus          | 监控系统                         | ❌                    |
| Prometheus Operator | 管理 Prometheus                  | ❌                    |
| ServiceMonitor      | 告诉 Prometheus 监控哪些 Service | ❌（CRD）             |
| PodMonitor          | 告诉 Prometheus 监控哪些 Pod     | ❌（CRD）             |
| PrometheusRule      | 定义告警和记录规则               | ❌（CRD）             |
| Exporter            | 暴露指标                         | ❌                    |

这些都是 Prometheus Operator 生态提供的资源，而不是 Kubernetes 自带的。

------

# 第十四节：企业案例

假设你开发了一个 ASP.NET Core API。

你完成了以下几件事：

1. 应用暴露 `/metrics`。
2. 创建了一个 `Service`。
3. 创建了一个 `ServiceMonitor`。

之后发生了一次滚动更新：

```
Pod A 删除
Pod B 创建
```

你没有修改任何 Prometheus 配置。

因为：

- Service 自动指向新的 Pod。
- ServiceMonitor 仍然匹配这个 Service。
- Prometheus Operator 不需要更新规则。
- Prometheus 在下一次抓取时就会采集新 Pod 的指标。

这就是 Kubernetes 声明式管理带来的价值：**业务变化时，监控配置基本不需要人工干预。**

------

# 第十五节：本章总结

这一章最重要的不是记住 YAML，而是理解下面这套关系：

```
Exporter
    │
    ▼
/metrics
    │
    ▼
Service
    │
    ▼
ServiceMonitor（CRD）
    │
    ▼
Prometheus Operator
    │
    ▼
Prometheus
    │
    ▼
Grafana
```

请记住这几个关键点：

1. **Operator 本质上是"CRD + Controller"。**
2. **Prometheus Operator 用 Kubernetes 资源管理 Prometheus，而不是手工维护配置文件。**
3. **ServiceMonitor 面向 Service，PodMonitor 面向 Pod。**
4. **PrometheusRule 用于管理告警和记录规则。**
5. **Exporter 的职责是把各种系统和中间件的数据转换成 Prometheus 能采集的指标。**

------

# 🌟 企业经验

Prometheus Operator 不只是一个"部署工具"。

它体现的是 Kubernetes 最重要的扩展思想：

> **把所有配置都抽象成 Kubernetes API 对象，再由 Controller 持续协调到目标状态。**

理解了这一点，后面学习：

- Helm
- Cert-Manager
- Argo CD
- Crossplane
- ExternalDNS

你会发现：

它们几乎都遵循同样的模式：

```
CRD
   +
Controller
   =
Operator
```

这也是 Kubernetes 生态能够不断扩展，而无需修改 Kubernetes 核心代码的根本原因。

------

## 下一章预告：CRD（CustomResourceDefinition）深入解析

虽然这一章已经接触了 CRD，但我们只是把它作为 Prometheus Operator 的一部分。

下一章，我们将单独深入学习 **CRD**：

- Kubernetes 为什么允许你"发明一种新的资源"？
- CRD 与内置资源（Deployment、Service）有什么区别？
- 如何定义属于自己的 `Kind`？
- Version（v1、v1beta1）如何演进？
- OpenAPI Schema 如何做字段校验？
- Status、Spec、Finalizer 分别有什么作用？

这一章学完后，你将真正理解 Kubernetes 为什么不仅是一个容器编排平台，更是一个**可扩展的平台框架**。