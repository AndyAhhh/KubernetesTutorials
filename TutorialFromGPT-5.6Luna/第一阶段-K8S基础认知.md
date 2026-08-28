# 第 1 章：为什么需要 Kubernetes

> **本章目标：**
>
> 在真正学习 Kubernetes 的具体对象和命令之前，先回答一个最根本的问题：
>
> **为什么我们需要 Kubernetes？**
>
> 本章不深入 Pod、Deployment、Service 等具体对象，只建立完整的认知框架。后续章节再分别深入。

------

## 1.1 传统应用部署方式的问题

在理解 Kubernetes 之前，我们先看最传统的应用部署方式。

假设你开发了一个 .NET Web API：

```
用户
  │
  ▼
.NET Web API
  │
  ├── MySQL
  └── Redis
```

最简单的部署方式，就是准备一台 Linux 服务器：

```
┌──────────────────────────────┐
│         Linux Server         │
│                              │
│      .NET Web API            │
│                              │
│      MySQL                   │
│                              │
│      Redis                   │
│                              │
└──────────────────────────────┘
```

然后通过 SSH 登录服务器：

```
ssh user@server
```

安装运行环境：

```
dotnet --version
```

上传程序：

```
scp MyApp.dll user@server:/app/
```

然后运行：

```
dotnet MyApp.dll
```

看起来非常简单。

但是随着应用规模增长，问题会逐渐出现。

------

### 1.1.1 环境不一致

假设开发环境是：

```
Windows
.NET 8
Node.js 22
MySQL 8
Redis 7
```

生产服务器可能是：

```
Linux
.NET 8
Node.js 20
MySQL 8
Redis 6
```

于是经常出现：

> “我本地运行完全正常，为什么生产环境不行？”

原因可能包括：

- 操作系统不同
- Runtime 版本不同
- 系统库不同
- 环境变量不同
- 文件路径不同
- 网络配置不同
- 依赖软件版本不同

这就是传统部署非常典型的问题：

> **应用和运行环境高度耦合。**

------

### 1.1.2 部署过程依赖人工操作

传统部署可能是：

```
开发完成
 ↓
编译
 ↓
打包
 ↓
SSH 登录服务器
 ↓
停止旧程序
 ↓
上传新程序
 ↓
修改配置
 ↓
启动程序
 ↓
检查日志
```

当只有一台服务器时问题不大。

但是如果有：

```
10 台服务器
```

你可能需要重复 10 次。

如果有：

```
100 台服务器
```

人工操作就非常困难。

------

### 1.1.3 扩容困难

假设一开始：

```
1 台服务器
1 个 API 实例
```

随着用户增长：

```
1 个 API
    ↓
CPU 越来越高
    ↓
增加实例
```

变成：

```
Server 1 → API
Server 2 → API
Server 3 → API
```

此时你需要解决：

- 新服务器在哪里？
- 如何安装运行环境？
- 如何部署应用？
- 如何保证版本一致？
- 流量怎么分配？
- 哪台服务器出现故障怎么办？

应用数量和服务器数量增加之后，**人工管理成本会快速增长。**

------

### 1.1.4 应用故障需要人工处理

例如：

```
Linux Server
    │
    └── .NET API
            ↓
          崩溃
```

服务器本身可能完全正常：

```
CPU     20%
Memory  40%
Disk    30%
```

但是 API 进程已经退出。

传统方式可能需要：

```
发现故障
 ↓
登录服务器
 ↓
查看日志
 ↓
重新启动
```

如果晚上 3 点发生怎么办？

如果同时有 20 个应用怎么办？

------

### 1.1.5 发布存在停机风险

例如现在运行：

```
MyApp v1.0
```

需要升级：

```
MyApp v1.1
```

传统方式：

```
停止 v1.0
     ↓
部署 v1.1
     ↓
启动 v1.1
```

中间可能存在：

```
服务不可用
```

如果发布失败，还需要手工恢复旧版本。

------

### 1.1.6 应用越来越多

最开始：

```
user-api
```

后来：

```
user-api
order-api
product-api
payment-api
notification-api
file-api
```

再后来可能几十、几百个服务。

这时候问题就从：

> “如何运行一个程序？”

变成：

> **“如何统一管理大量应用？”**

这就是 Kubernetes 开始体现价值的地方。

------

## 1.2 Docker 解决了什么问题

在 Kubernetes 出现之前，我们先理解 Docker。

Docker 最重要的价值之一，是：

> **把应用以及应用运行所需要的环境打包到一个标准化的容器中。**

传统方式：

```
服务器
 │
 ├── 操作系统
 ├── Runtime
 ├── 系统依赖
 ├── 应用
 └── 配置
```

应用运行依赖服务器环境。

Docker 则变成：

```
Docker Image
 ├── 应用
 ├── Runtime
 ├── 依赖
 └── 文件系统
       ↓
    Container
```

例如：

```
docker run my-api:v1
```

Docker 会根据镜像创建 Container。

------

### 1.2.1 “一次构建，到处运行”

例如你的 .NET API：

```
源代码
  ↓
构建 Docker Image
  ↓
my-api:v1.0
```

然后这个镜像可以交给：

```
开发环境
测试环境
生产环境
```

使用。

核心思想是：

```
Build Once
   ↓
Run Anywhere
```

当然，“到处运行”不是绝对意义上的任何环境，而是指：

> **只要目标环境具备兼容的容器运行能力，就可以使用相同镜像运行应用。**

------

### 1.2.2 Docker 解决了环境一致性问题

例如：

```
my-api:v1.0
```

里面已经包含应用运行所需要的大部分环境。

于是：

```
开发
  ↓
my-api:v1.0

测试
  ↓
my-api:v1.0

生产
  ↓
my-api:v1.0
```

避免了：

```
开发环境一个版本
测试环境一个版本
生产环境另一个版本
```

------

### 1.2.3 Docker 解决了应用打包问题

以前：

```
源码
+
Runtime
+
依赖
+
配置
+
部署脚本
```

现在可以形成：

```
Docker Image
```

例如：

```
my-company/user-api:1.0.0
```

它成为应用交付的重要载体。

------

### 1.2.4 但是 Docker 没有解决所有问题

这是非常重要的一点。

Docker 可以解决：

```
应用打包
环境一致性
容器运行
资源隔离
应用交付
```

但是当你有：

```
100 台服务器
1000 个 Container
```

你还需要解决：

```
Container 应该运行在哪台机器？
Container 挂了怎么办？
如何自动创建新的 Container？
如何扩容？
如何缩容？
如何进行滚动发布？
如何发现服务？
如何管理网络？
如何管理存储？
如何管理配置？
如何进行权限控制？
```

这些已经超出了单纯 Docker 的核心职责。

于是：

> **Kubernetes 出现了。**

------

## 1.3 Docker 和 Kubernetes 的关系

这是初学者必须建立的第一个正确认知。

不要理解成：

```
Docker VS Kubernetes
```

更合理的是：

```
Docker / Container Runtime
          ↓
       运行容器

Kubernetes
          ↓
     管理容器化应用
```

------

### 1.3.1 Docker 主要解决什么？

可以简单理解：

> **Docker 解决“怎么把应用做成容器并运行起来”。**

例如：

```
Dockerfile
    ↓
docker build
    ↓
Image
    ↓
docker run
    ↓
Container
```

------

### 1.3.2 Kubernetes 主要解决什么？

Kubernetes 更关注：

> **“有大量容器之后，如何自动化地管理这些应用？”**

例如：

```
                    Kubernetes
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
      部署              调度              网络
       │                 │                 │
      扩容              故障恢复           服务发现
       │                 │                 │
      发布              存储              配置
       │                 │                 │
      回滚              权限              监控
```

------

### 1.3.3 一个非常重要的关系

现代 Kubernetes 并不是必须依赖 Docker Engine。

Kubernetes 使用的是**容器运行时**。

常见运行时包括：

```
containerd
CRI-O
```

因此学习 Kubernetes 时，你需要逐渐建立这样的分层概念：

```
                    Kubernetes
                         │
                   管理工作负载
                         │
                    Container
                         │
                  Container Runtime
                    │           │
               containerd     CRI-O
```

Docker 仍然是非常重要的容器生态工具，但：

> **Kubernetes ≠ Docker。**

------

## 1.4 Kubernetes 解决什么问题

现在我们可以正式回答：

> **Kubernetes 到底解决什么问题？**

一句话：

> **Kubernetes 解决的是大规模容器化应用的自动化部署、管理、调度、扩缩容、故障恢复和服务治理问题。**

我们逐项理解。

------

### 1.4.1 自动部署

你告诉 Kubernetes：

```
我需要运行这个应用
使用这个镜像
运行 3 个实例
```

Kubernetes 根据你的声明完成部署。

------

### 1.4.2 自动恢复

假设：

```
需要 3 个实例
```

实际：

```
Pod A
Pod B
Pod C
```

其中一个发生故障：

```
Pod A ❌
Pod B ✅
Pod C ✅
```

Kubernetes 可以发现实际状态与期望状态不一致，并进行恢复。

最终：

```
Pod B
Pod C
Pod D
```

仍然维持：

```
3 个实例
```

这就是 Kubernetes 的**声明式 + 自动调谐**思想。

------

### 1.4.3 自动扩缩容

例如：

```
正常流量
 ↓
3 个实例
```

流量增加：

```
高流量
 ↓
10 个实例
```

流量降低：

```
低流量
 ↓
3 个实例
```

这就是自动扩缩容。

------

### 1.4.4 服务发现

假设有：

```
user-api
order-api
payment-api
```

其中：

```
order-api
```

需要调用：

```
user-api
```

Kubernetes 可以提供稳定的服务发现机制，让应用不需要硬编码某个具体 Pod 的 IP。

因为：

> **Pod 是动态的。**

Pod 可能：

```
创建
 ↓
删除
 ↓
重新创建
 ↓
IP 改变
```

所以应用不能依赖某个具体 Pod IP。

这就是后面 Service/DNS 要解决的问题。

------

### 1.4.5 滚动发布

例如：

```
v1.0
 ↓
v1.1
```

Kubernetes 可以逐步替换旧实例，而不是：

```
全部停止
 ↓
全部启动
```

这样可以降低发布期间的服务中断风险。

------

### 1.4.6 回滚

如果：

```
v1.1
 ↓
发现严重 Bug
```

可以回退：

```
v1.1
 ↓
Rollback
 ↓
v1.0
```

------

### 1.4.7 自动调度

假设集群有：

```
Node 1
Node 2
Node 3
```

你只需要告诉 Kubernetes：

```
我要运行 10 个 Pod
```

Kubernetes Scheduler 会根据资源、约束、亲和性等条件决定：

```
Pod 1 → Node 1
Pod 2 → Node 2
Pod 3 → Node 3
...
```

你不需要手工决定每个 Pod 具体放在哪台机器。

------

### 1.4.8 资源管理

你可以告诉 Kubernetes：

```
这个应用需要：

CPU：500m
Memory：512Mi
```

Kubernetes 可以根据资源请求进行调度，并结合限制控制资源使用。

后面会详细学习：

- Requests
- Limits
- QoS
- OOMKilled
- CPU Throttling

------

## 1.5 Kubernetes 能做什么、不能做什么

这一部分非常重要。

不要把 Kubernetes 神化成：

> “只要用了 K8S，什么问题都解决了。”

这是错误的。

------

### 1.5.1 Kubernetes 能做什么

Kubernetes 非常擅长：

#### 应用部署

```
Deployment
StatefulSet
DaemonSet
Job
CronJob
```

#### 容器生命周期管理

```
启动
停止
重启
替换
扩容
缩容
```

#### 服务发现

```
Service
DNS
```

#### 网络流量管理

```
Service
Ingress
Gateway
```

#### 配置管理

```
ConfigMap
Secret
```

#### 存储管理

```
PV
PVC
StorageClass
CSI
```

#### 资源调度

```
CPU
Memory
Node
Affinity
Taints
Tolerations
```

#### 自动扩缩容

```
HPA
```

#### 权限控制

```
RBAC
ServiceAccount
```

#### 自愈

```
Container Restart
Pod Replacement
Replica Reconciliation
```

------

### 1.5.2 Kubernetes 不能替代你的应用设计

例如你的程序存在：

```
内存泄漏
```

Kubernetes 可以发现：

```
OOMKilled
```

然后重新启动。

但它不会自动修复：

> **你的代码为什么内存泄漏。**

------

### 1.5.3 Kubernetes 不能替代数据库设计

例如：

```
MySQL
```

出现：

```
慢查询
锁竞争
索引错误
事务设计问题
数据模型问题
```

Kubernetes 并不能解决这些问题。

------

### 1.5.4 Kubernetes 不能自动保证高可用

部署：

```
3 个 Pod
```

不代表一定高可用。

如果三个 Pod 全部运行在：

```
同一个 Node
```

Node 挂了：

```
3 个 Pod
     ↓
全部不可用
```

因此：

> **Kubernetes 提供高可用能力，但最终的高可用需要正确的架构设计。**

------

### 1.5.5 Kubernetes 不能自动保证安全

Kubernetes 有：

```
RBAC
NetworkPolicy
SecurityContext
Secret
Pod Security
```

但如果你配置错误：

```
权限过大
Secret 泄露
容器使用 privileged
网络策略错误
```

Kubernetes 不会自动替你修复。

------

### 1.5.6 Kubernetes 不能替代 CI/CD

Kubernetes 可以负责：

```
部署
运行
管理
```

但：

```
代码提交
 ↓
编译
 ↓
测试
 ↓
构建镜像
 ↓
推送 Registry
```

通常属于：

> **CI/CD 系统**

例如：

- GitHub Actions
- GitLab CI
- Jenkins

后面我们会把它们和 Kubernetes 串起来。

------

## 1.6 K8S 的典型应用场景

Kubernetes 最常见的应用场景主要有以下几类。

------

### 1.6.1 微服务

例如：

```
                 API Gateway
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   User Service   Order Service   Payment Service
       │              │              │
       └──────────────┼──────────────┘
                      ▼
                  Database
```

每个服务都可能拥有：

- 独立镜像
- 独立部署
- 独立扩缩容
- 独立发布

Kubernetes 非常适合管理这种架构。

------

### 1.6.2 Web/API 应用

例如：

```
Vue
 ↓
Nginx
 ↓
.NET API
 ↓
Redis
 ↓
PostgreSQL
```

这是我们后面实战最主要的场景。

------

### 1.6.3 高并发系统

例如：

```
电商
支付
社交
游戏
内容平台
```

这些系统通常需要：

```
多实例
自动扩容
负载均衡
高可用
滚动发布
监控
```

Kubernetes 可以提供基础设施层面的支持。

------

### 1.6.4 批处理任务

例如：

```
每天凌晨 2 点
 ↓
统计昨日订单
 ↓
生成报表
```

可以使用：

```
Job
CronJob
```

------

### 1.6.5 AI / GPU 工作负载

现在 Kubernetes 也广泛用于：

```
GPU
AI Training
Inference
Model Serving
```

例如：

```
GPU Node
    │
    ├── Model Server
    ├── Inference Pod
    └── Training Job
```

不过这属于更高级的 Kubernetes 使用场景。

------

## 1.7 K8S 与虚拟机、Docker Compose 的区别

这一部分非常容易混淆。

我们分别比较。

------

### 1.7.1 Kubernetes vs 虚拟机

传统虚拟机架构：

```
Physical Server
       │
   Hypervisor
   ┌───┼───┐
   ▼   ▼   ▼
  VM   VM   VM
```

每台 VM 都有完整的 Guest OS。

容器则更接近：

```
Physical / VM
      │
     OS
      │
Container Runtime
 ┌────┼────┐
 ▼    ▼    ▼
C1    C2    C3
```

容器通常比完整虚拟机更加轻量。

但这不是简单的：

> “Container 一定比 VM 好。”

两者解决的问题不同。

------

### 1.7.2 Kubernetes vs Docker Compose

Docker Compose：

```
适合：

单机
小规模
开发环境
测试环境
简单应用
```

Kubernetes：

```
适合：

多节点
大规模
高可用
自动扩缩容
复杂发布
统一运维
```

简单理解：

```
Docker Compose
      ↓
管理一台机器上的多个容器


Kubernetes
      ↓
管理整个集群中的工作负载
```

------

### 1.7.3 三者定位

可以先建立这个认知：

| 技术           | 主要解决的问题       |
| -------------- | -------------------- |
| 虚拟机         | 虚拟化计算资源       |
| Docker         | 容器化应用           |
| Docker Compose | 单机多容器编排       |
| Kubernetes     | 集群级容器编排与管理 |

这只是第一层理解，后面学习容器和 K8S 架构时会进一步细化。

------

## 1.8 从开发者视角理解 K8S

如果你是后端开发，刚开始接触 Kubernetes，很容易看到大量陌生名词：

```
Pod
Deployment
Service
Ingress
ConfigMap
Secret
PV
PVC
Namespace
ServiceAccount
RBAC
HPA
```

很容易产生一种感觉：

> “为什么这么复杂？”

其实可以换一个开发者视角。

你平时开发一个应用，需要考虑：

```
应用是什么？
 ↓
怎么运行？
 ↓
需要什么配置？
 ↓
需要多少资源？
 ↓
如何暴露 API？
 ↓
如何访问数据库？
 ↓
挂了怎么办？
 ↓
怎么发布？
```

Kubernetes 只是把这些问题进行了标准化。

例如：

```
我有什么应用？
       ↓
     Image

我要运行几个？
       ↓
    Deployment

应用怎么被访问？
       ↓
     Service

外部用户怎么访问？
       ↓
     Ingress

应用配置在哪里？
       ↓
   ConfigMap

密码放在哪里？
       ↓
     Secret

数据存哪里？
       ↓
     PVC

需要多少 CPU/内存？
       ↓
 Resources

什么时候算健康？
       ↓
    Probe

流量大了怎么办？
       ↓
      HPA
```

所以不要把 Kubernetes 看成：

> **“很多 YAML。”**

应该把它看成：

> **“把应用运行过程中需要解决的问题标准化。”**

------

## 1.9 一次完整的 K8S 应用部署流程

现在我们把前面所有内容串起来。

假设你有一个：

```
.NET Web API
```

------

### 第一步：开发应用

```
.NET Source Code
```

例如：

```
MyApi/
├── Controllers/
├── Services/
├── Models/
└── Program.cs
```

------

### 第二步：构建 Docker Image

编写：

```
Dockerfile
```

然后：

```
docker build -t my-api:1.0.0 .
```

得到：

```
my-api:1.0.0
```

------

### 第三步：推送到镜像仓库

例如：

```
Docker Registry
```

最终：

```
Registry
   │
   └── my-api:1.0.0
```

生产 Kubernetes 集群就可以从 Registry 获取镜像。

------

### 第四步：告诉 Kubernetes 如何运行

你声明：

```
我要运行 my-api:1.0.0
需要 3 个实例
```

Kubernetes 根据声明创建对应工作负载。

后面的核心链路会逐渐变成：

```
Deployment
     ↓
ReplicaSet
     ↓
Pod
     ↓
Container
```

------

### 第五步：Kubernetes 调度

集群假设有：

```
Node 1
Node 2
Node 3
```

Scheduler 根据资源和各种约束决定：

```
Pod 1 → Node 1
Pod 2 → Node 2
Pod 3 → Node 3
```

------

### 第六步：Pod 启动

Node 上的 kubelet 负责让对应工作负载运行起来。

最终：

```
Node 1
 └── API Pod

Node 2
 └── API Pod

Node 3
 └── API Pod
```

------

### 第七步：提供稳定访问入口

Pod IP 是动态的。

所以不能让其他应用直接依赖：

```
10.x.x.x
```

而是通过：

```
Service
```

提供稳定的服务访问入口。

------

### 第八步：提供外部访问

用户访问：

```
https://api.example.com
```

流量进入 Kubernetes：

```
Internet
   ↓
Ingress
   ↓
Service
   ↓
Pod
   ↓
.NET API
```

------

### 第九步：应用运行

现在用户就可以正常访问：

```
https://api.example.com
```

而 Kubernetes 在后台不断维护：

```
期望状态
    ↓
实际状态
    ↓
自动调谐
```

------

### 第十步：出现故障

例如：

```
Pod 2
  ↓
Crash
```

Kubernetes 发现：

```
期望：3
实际：2
```

然后采取相应的恢复动作。

------

### 第十一步：流量增加

例如：

```
100 req/s
    ↓
5000 req/s
```

如果配置了 HPA，可以：

```
3 Pod
 ↓
5 Pod
 ↓
10 Pod
```

------

### 第十二步：发布新版本

现在：

```
v1.0
```

升级：

```
v1.1
```

Kubernetes 可以通过滚动更新逐步替换旧实例。

如果新版本有严重问题：

```
v1.1
 ↓
Rollback
 ↓
v1.0
```

------

## 1.10 K8S 学习路线和核心知识地图

到这里，你应该已经知道 Kubernetes 为什么存在。

接下来真正学习 Kubernetes 时，我建议始终围绕下面这条主线：

```
                  Kubernetes
                       │
                       ▼
                Cluster 集群
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
    Control Plane                Node
          │                         │
          │                    Container
          │                         │
          └──────────┬──────────────┘
                     ▼
                    Pod
                     │
                     ▼
               Deployment
                     │
                     ▼
                  Service
                     │
                     ▼
                 Ingress
                     │
                     ▼
                    用户
```

然后在这条主线周围逐渐加入：

```
                    Kubernetes
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
     Workload          Network           Config
       │                 │                 │
   Deployment         Service         ConfigMap
   StatefulSet        Ingress           Secret
   DaemonSet          DNS
   Job
   CronJob
       │
       ├──────────── Storage
       │                │
       │              PV/PVC
       │            StorageClass
       │
       ├──────────── Resources
       │                │
       │          Request/Limit
       │          HPA/Scheduler
       │
       ├──────────── Security
       │                │
       │             RBAC
       │        ServiceAccount
       │        NetworkPolicy
       │
       ├──────────── Observability
       │                │
       │          Logging/Metrics
       │          Prometheus/Grafana
       │
       └──────────── Delivery
                        │
                     Helm
                     CI/CD
                    GitOps
```

------

## 本章总结

现在可以用一条完整的演进路线理解 Kubernetes：

```
传统部署
   │
   │ 环境不一致
   │ 手工部署
   │ 扩容困难
   │ 故障恢复困难
   │ 发布困难
   ▼
Docker
   │
   │ 解决应用容器化
   │ 解决环境一致性
   │ 解决应用交付
   ▼
大量 Container
   │
   │ 如何调度？
   │ 如何扩容？
   │ 如何恢复？
   │ 如何发布？
   │ 如何服务发现？
   │ 如何管理网络？
   │ 如何管理存储？
   ▼
Kubernetes
   │
   ├── 自动部署
   ├── 自动调度
   ├── 自动恢复
   ├── 服务发现
   ├── 滚动更新
   ├── 回滚
   ├── 自动扩缩容
   ├── 配置管理
   ├── 存储管理
   ├── 权限管理
   └── 统一运维
```

### 本章最核心的一句话

> **Docker 解决“如何标准化地运行容器化应用”，Kubernetes 解决“如何在集群中自动化地部署、调度、管理和维护大量容器化应用”。**

# 第 2 章：Kubernetes 核心架构

本章围绕 Kubernetes 的核心架构展开，重点建立以下整体认知：

```
kubectl
   ↓
API Server
   ↓
Kubernetes API
   ↓
etcd / Scheduler / Controller
   ↓
Worker Node
   ↓
kubelet
   ↓
Container Runtime
   ↓
Container
   ↓
Application
```

同时理解 Kubernetes 最核心的运行机制：

```
Desired State
      ↓
Controller
      ↓
Actual State
      ↓
Reconciliation
      ↓
不断趋近 Desired State
```

------

## 2.1 Kubernetes Cluster

### 2.1.1 什么是 Kubernetes Cluster

Kubernetes Cluster，中文通常称为 **Kubernetes 集群**。

它是一组共同组成 Kubernetes 系统的计算节点和控制组件。

一个最基本的 Kubernetes 集群可以理解为：

```
Kubernetes Cluster
│
├── Control Plane
│
├── Worker Node
│
├── Worker Node
│
└── Worker Node
```

其中：

- **Control Plane**：负责管理和控制整个集群
- **Worker Node**：负责实际运行应用

------

### 2.1.2 一个简单的 Kubernetes Cluster

```
┌─────────────────────────────────────────────┐
│              Kubernetes Cluster             │
│                                             │
│  ┌──────────────────┐                       │
│  │  Control Plane   │                       │
│  │                  │                       │
│  │  API Server      │                       │
│  │  Scheduler       │                       │
│  │  Controller      │                       │
│  │  etcd            │                       │
│  └──────────────────┘                       │
│                                             │
│  ┌──────────────────┐  ┌──────────────────┐ │
│  │  Worker Node 1   │  │  Worker Node 2   │ │
│  │                  │  │                  │ │
│  │  kubelet         │  │  kubelet         │ │
│  │  kube-proxy      │  │  kube-proxy      │ │
│  │  Runtime         │  │  Runtime         │ │
│  │                  │  │                  │ │
│  │  Pod             │  │  Pod             │ │
│  └──────────────────┘  └──────────────────┘ │
│                                             │
└─────────────────────────────────────────────┘
```

后续学习 Kubernetes 时，这张图是最基础的架构图。

------

### 2.1.3 Cluster 中的两类核心节点

可以先把 Kubernetes 集群划分成两部分：

```
Kubernetes Cluster
│
├── Control Plane
│   └── 管理集群
│
└── Worker Node
    └── 运行应用
```

需要注意，这只是逻辑上的职责划分。

实际生产环境中，Control Plane 节点也可能运行某些工作负载，但对于学习 Kubernetes 架构而言，应首先建立：

> **Control Plane 负责控制，Worker Node 负责运行。**

------

## 2.2 Control Plane

### 2.2.1 什么是 Control Plane

Control Plane 可以理解为：

> **Kubernetes 集群的大脑。**

它负责：

- 接收用户和其他组件的请求
- 提供 Kubernetes API
- 保存集群状态
- 决定 Pod 的调度
- 运行各种 Controller
- 维护集群的期望状态和实际状态之间的一致性

核心组件包括：

```
Control Plane
│
├── kube-apiserver
├── etcd
├── kube-scheduler
└── kube-controller-manager
```

在某些集群中还会存在：

```
cloud-controller-manager
```

------

### 2.2.2 Control Plane 与 Worker Node 的区别

可以简单理解：

```
Control Plane
      │
      │ 决策、管理、控制
      ▼
Worker Node
      │
      │ 实际执行
      ▼
Pod / Container
```

例如：

```
Control Plane：

“我需要 3 个 my-api Pod”
```

Worker Node：

```
“好的，我负责把这些 Pod 真正运行起来。”
```

------

### 2.2.3 Control Plane 不等于应用运行环境

初学者很容易认为：

> Kubernetes 的所有组件都在 Control Plane 上运行。

实际上更准确的理解是：

```
Control Plane
    ↓
负责控制集群

Worker Node
    ↓
负责运行工作负载
```

Control Plane 的核心职责是**控制和管理**，而不是直接运行你的业务 Container。

------

## 2.3 Worker Node

### 2.3.1 什么是 Worker Node

Worker Node 是：

> **Kubernetes 集群中实际运行应用工作负载的节点。**

它可以是一台：

- 物理服务器
- 虚拟机
- 云服务器

一个典型 Worker Node 可以理解为：

```
Worker Node
│
├── kubelet
├── kube-proxy
├── Container Runtime
│
├── Pod
│   └── Container
│
├── Pod
│   └── Container
│
└── Pod
    └── Container
```

------

### 2.3.2 Node 与服务器的关系

初学阶段可以简单理解：

```
Linux Server ≈ Kubernetes Node
```

但严格来说，Node 是 Kubernetes 对计算资源的抽象。

例如：

```
Node 1
CPU：8 Core
Memory：32 GB

Node 2
CPU：16 Core
Memory：64 GB

Node 3
CPU：16 Core
Memory：64 GB
```

Kubernetes Scheduler 会根据资源和各种调度条件决定 Pod 应该运行在哪个 Node。

------

### 2.3.3 Worker Node 上有哪些重要组件

本章需要掌握：

```
Worker Node
│
├── kubelet
│
├── kube-proxy
│
└── Container Runtime
```

它们的职责分别是：

| 组件              | 主要职责                    |
| ----------------- | --------------------------- |
| kubelet           | 管理 Node 上的 Pod 生命周期 |
| kube-proxy        | 参与 Service 网络转发       |
| Container Runtime | 实际创建和运行 Container    |

------

## 2.4 API Server

### 2.4.1 API Server 是什么

`kube-apiserver` 是 Kubernetes 最核心的组件之一。

可以把它理解成：

> **Kubernetes 集群统一的 API 入口。**

整体关系：

```
                  Kubernetes API
                        │
                        ▼
                  kube-apiserver
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
        etcd        Scheduler     Controller
```

用户、Controller、Scheduler、kubelet 等组件都会通过 Kubernetes API 与集群进行交互。

------

### 2.4.2 kubectl 与 API Server 的关系

你以后会经常执行：

```
kubectl get pods
```

或者：

```
kubectl apply -f deployment.yaml
```

这里的 `kubectl` 是：

> **Kubernetes API 的命令行客户端。**

它不是直接操作 Pod。

实际关系是：

```
kubectl
   ↓
Kubernetes API
   ↓
kube-apiserver
   ↓
Kubernetes 内部组件
```

------

### 2.4.3 API Server 负责什么

API Server 主要负责：

```
接收 API 请求
    ↓
认证
    ↓
授权
    ↓
Admission
    ↓
对象验证
    ↓
处理 Kubernetes Object
    ↓
持久化状态
```

例如：

```
kubectl apply -f pod.yaml
```

并不是：

```
kubectl
   ↓
直接登录 Node
   ↓
启动 Container
```

而是：

```
kubectl
   ↓
API Server
   ↓
创建 / 更新 Kubernetes Object
```

之后才由其他组件继续完成调度和运行。

------

## 2.5 etcd

### 2.5.1 etcd 是什么

`etcd` 是一个分布式键值存储系统。

在 Kubernetes 中可以理解为：

> **保存 Kubernetes 集群核心状态的数据库。**

例如 Kubernetes 需要知道：

```
有哪些 Node？
有哪些 Namespace？
有哪些 Pod？
有哪些 Deployment？
有哪些 Service？
ConfigMap 是什么？
Secret 是什么？
```

这些 Kubernetes 对象的状态会被持久化到 etcd。

------

### 2.5.2 etcd 与 API Server 的关系

简化理解：

```
kubectl
   ↓
API Server
   ↓
etcd
```

例如：

```
kubectl apply -f deployment.yaml
```

API Server 处理请求后，会把相关 Kubernetes Object 的状态保存到 etcd。

------

### 2.5.3 etcd 保存的是 Kubernetes 状态

例如：

```
Deployment
    │
    ├── name: my-api
    ├── replicas: 3
    └── image: my-api:v1
```

这些属于 Kubernetes 的状态。

可以理解为：

```
Kubernetes
    │
    └── etcd
         └── Cluster State
```

------

### 2.5.4 etcd 不等于业务数据库

这一点必须区分。

你的业务可能使用：

```
MySQL
PostgreSQL
MongoDB
Redis
```

保存：

```
用户
订单
商品
支付记录
```

而 etcd 保存的是：

```
Kubernetes Cluster State
```

两者职责完全不同。

------

### 2.5.5 为什么 etcd 很重要

因为 etcd 保存的是 Kubernetes 控制面的核心状态。

因此生产环境中必须重点考虑：

```
etcd 高可用
etcd Backup
etcd Restore
```

如果 etcd 中的核心状态丢失，Kubernetes 控制面会受到严重影响。

------

## 2.6 Scheduler

### 2.6.1 Scheduler 是什么

`kube-scheduler` 的核心职责是：

> **决定一个尚未绑定 Node 的 Pod 应该运行在哪个 Node。**

例如：

```
Pod
 │
 ├── CPU：2 Core
 ├── Memory：4 GB
 └── 需要运行
```

集群存在：

```
Node A
Node B
Node C
```

Scheduler 会评估：

```
Node A → CPU 不足 ❌
Node B → 满足条件 ✅
Node C → 不符合约束 ❌
```

最终：

```
Pod → Node B
```

------

### 2.6.2 Scheduler 不负责启动 Container

这是非常重要的区别。

Scheduler 做的是：

```
决定：

Pod → Node B
```

而不是：

```
启动 Container
```

后续真正的运行链路是：

```
Scheduler
   ↓
确定 Node
   ↓
kubelet
   ↓
Container Runtime
   ↓
Container
```

------

### 2.6.3 Scheduler 会考虑什么

Scheduler 的调度决策可能考虑：

```
CPU / Memory
Node 状态
Node Selector
Node Affinity
Pod Affinity
Pod Anti-Affinity
Taints
Tolerations
Topology
```

因此 Scheduler 并不是简单地：

> 找一台当前最空闲的服务器。

而是根据一系列调度规则选择合适的 Node。

------

## 2.7 Controller Manager

### 2.7.1 Controller 是什么

Controller 是 Kubernetes 最重要的设计思想之一。

它负责不断观察：

```
Actual State
```

并与：

```
Desired State
```

进行比较。

如果发现两者不一致：

```
Actual State ≠ Desired State
```

Controller 就采取相应行动。

------

### 2.7.2 Controller Manager 是什么

`kube-controller-manager` 负责运行 Kubernetes 中的多个 Controller。

例如：

```
Deployment Controller
ReplicaSet Controller
Node Controller
Job Controller
Namespace Controller
```

不同 Controller 负责不同类型的资源。

------

### 2.7.3 Controller 的基本工作方式

例如：

```
Desired State：

replicas = 3
```

实际：

```
Pod A
Pod B
```

Controller 发现：

```
Desired = 3
Actual = 2
```

于是：

```
创建一个 Pod
```

最终：

```
Pod A
Pod B
Pod C
```

达到：

```
Desired = Actual = 3
```

------

## 2.8 kubelet

### 2.8.1 kubelet 是什么

kubelet 是运行在每个 Worker Node 上的 Agent。

可以理解为：

> **kubelet 负责让 Node 上的 Pod 按 Kubernetes 的要求运行。**

例如 Scheduler 决定：

```
my-api Pod
    ↓
Node 2
```

Node 2 上的 kubelet 会负责后续的 Pod 生命周期管理。

------

### 2.8.2 kubelet 的基本工作流程

```
API Server
    ↓
Node 2 kubelet
    ↓
读取 Pod 配置
    ↓
Container Runtime
    ↓
创建 / 启动 Container
```

------

### 2.8.3 kubelet 还负责什么

kubelet 会持续关注 Node 上 Pod 的运行状态，例如：

```
Container 是否运行？
Pod 是否正常？
Container 是否退出？
Liveness Probe 是否失败？
Readiness Probe 是否正常？
```

并将相关状态反馈给 Kubernetes。

------

### 2.8.4 kubelet 不等于 Container Runtime

两者必须区分：

```
kubelet
    ↓
管理 / 协调 Pod 生命周期

Container Runtime
    ↓
真正创建和运行 Container
```

因此：

> kubelet 是 Kubernetes Node 上的管理者，而 Container Runtime 是实际执行 Container 操作的组件。

------

## 2.9 kube-proxy

### 2.9.1 kube-proxy 是什么

`kube-proxy` 是 Kubernetes Node 上与网络相关的重要组件。

初学阶段可以记住：

> **kube-proxy 主要参与 Kubernetes Service 的网络转发。**

例如：

```
Client
   ↓
Service
   ↓
Pod
```

Service 本身通常不是一个真正运行业务程序的进程。

Kubernetes 需要将访问 Service 的流量转发到对应的后端 Pod。

------

### 2.9.2 kube-proxy 的作用

例如：

```
Service
   │
   │ Service IP
   ▼
网络转发规则
   │
   ▼
Pod
```

传统的 kube-proxy 实现会涉及：

```
iptables
IPVS
```

具体机制取决于 Kubernetes 版本和集群网络方案。

因此不要简单理解为：

> “Service 就是 kube-proxy。”

更准确的是：

> **kube-proxy 是 Kubernetes Service 网络实现中的重要组件之一。**

------

## 2.10 Container Runtime

### 2.10.1 Container Runtime 是什么

Container Runtime 是：

> **真正负责创建和运行 Container 的组件。**

常见的 Container Runtime 包括：

```
containerd
CRI-O
```

------

### 2.10.2 Container Runtime 负责什么

例如：

```
拉取镜像
创建 Container
启动 Container
停止 Container
删除 Container
```

可以理解为：

```
kubelet
   ↓
Container Runtime
   ↓
Container
```

------

### 2.10.3 Kubernetes 与 Container Runtime 的关系

Kubernetes 并不会直接执行：

```
docker run
```

来运行你的应用。

而是：

```
Kubernetes
   ↓
Pod 定义
   ↓
kubelet
   ↓
Container Runtime
   ↓
Container
   ↓
Application
```

因此：

> **Kubernetes 负责容器工作负载的编排，Container Runtime 负责实际运行 Container。**

------

## 2.11 Kubernetes API

### 2.11.1 Kubernetes 是 API 驱动的系统

Kubernetes 的核心设计之一是：

> **Kubernetes API。**

Kubernetes 中的大量资源都可以通过 API 进行操作。

例如：

```
Pod
Deployment
Service
ConfigMap
Secret
Namespace
Node
```

这些都可以被视为 Kubernetes API 中的对象。

------

### 2.11.2 kubectl 只是 Kubernetes API 的客户端

例如：

```
kubectl get pods
```

本质上是在请求：

> “请告诉我当前 Kubernetes 集群中的 Pod 状态。”

所以：

```
Kubernetes API
       ↑
       │
   ┌───┴────┐
   │        │
kubectl   Controller
```

`kubectl` 并不是 Kubernetes 本身。

------

### 2.11.3 为什么 API 很重要

因为 Kubernetes 不只是给人操作。

还可以被：

```
CI/CD
Controller
Operator
Dashboard
自动化程序
```

调用。

因此 Kubernetes 的核心不是：

```
kubectl 命令
```

而是：

```
Kubernetes API
```

------

## 2.12 Controller / Reconciliation Loop

### 2.12.1 什么是 Reconciliation

Reconciliation 可以理解为：

> **不断比较期望状态和实际状态，并采取措施使实际状态趋近于期望状态。**

基本过程：

```
Desired State
      ↓
比较
      ↓
Actual State
      ↓
发现差异
      ↓
Controller
      ↓
执行操作
      ↓
Actual State 改变
      ↓
再次检查
      ↺
```

这就是 Kubernetes 最核心的运行思想之一。

------

### 2.12.2 为什么需要不断调谐

现实环境会不断变化。

例如：

```
Desired：
3 个 Pod
```

实际：

```
Pod A
Pod B
Pod C
```

此时：

```
Desired = Actual
```

然后：

```
Pod B
   ↓
崩溃
```

实际状态变成：

```
Pod A
Pod C
```

Controller 再次检查：

```
Desired = 3
Actual = 2
```

于是：

```
创建 Pod D
```

最终：

```
Pod A
Pod C
Pod D
```

再次达到：

```
Desired = Actual
```

------

### 2.12.3 Reconciliation 是持续发生的

它不是：

```
创建一次
   ↓
结束
```

而是：

```
观察
 ↓
比较
 ↓
调整
 ↓
再次观察
 ↓
再次比较
 ↓
再次调整
 ↓
……
```

因此 Kubernetes 可以持续维护应用状态。

------

## 2.13 Desired State 与 Actual State

### 2.13.1 Desired State

Desired State：

> **期望状态。**

例如：

```
spec:
  replicas: 3
```

表达的意思不是：

> “现在立刻启动三个 Pod。”

而是：

> **“我希望最终保持三个副本。”**

------

### 2.13.2 Actual State

Actual State：

> **当前真实状态。**

例如：

```
Pod A
Pod B
```

那么：

```
Desired = 3
Actual  = 2
```

------

### 2.13.3 Kubernetes 如何处理差异

```
Desired State
      │
      │ 3 Pods
      ▼
Controller
      │
      │ 检查
      ▼
Actual State
      │
      │ 2 Pods
      ▼
发现差异
      │
      ▼
创建 Pod
      │
      ▼
Actual State = 3 Pods
```

最终：

```
Desired = Actual
```

------

### 2.13.4 spec 与 status

在 Kubernetes Object 中经常会看到：

```
spec
status
```

可以先简单理解为：

```
spec
 ↓
期望状态

status
 ↓
当前状态
```

例如：

```
spec:
  replicas: 3
```

而实际状态可能类似：

```
status:
  readyReplicas: 2
```

那么：

```
Desired = 3
Actual Ready = 2
```

Controller 就会继续调谐。

------

## 2.14 K8S 对象（Object）的概念

### 2.14.1 什么是 Kubernetes Object

Kubernetes 中大量资源都被抽象为：

> **Object（对象）**

例如：

```
Pod
Deployment
Service
ConfigMap
Secret
Namespace
Node
```

可以先理解为：

> **Object 是 Kubernetes API 中描述某种资源及其状态的一种结构化数据。**

------

### 2.14.2 一个典型 Object

例如：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-api

spec:
  replicas: 3
```

这份 YAML 描述了一个：

```
Deployment Object
```

其中包含：

```
API Version
Object Type
Metadata
Desired State
```

------

### 2.14.3 apiVersion

```
apiVersion: apps/v1
```

表示：

> 使用哪个 Kubernetes API Group / Version 来解释这个对象。

例如：

```
v1
apps/v1
batch/v1
```

不同资源使用不同的 API 版本。

------

### 2.14.4 kind

```
kind: Deployment
```

表示：

> 这是一个什么类型的 Kubernetes Object。

例如：

```
Pod
Deployment
Service
ConfigMap
Secret
```

------

### 2.14.5 metadata

```
metadata:
  name: my-api
```

metadata 描述对象的元信息。

常见内容包括：

```
name
namespace
labels
annotations
```

例如：

```
metadata:
  name: my-api
  namespace: production
```

------

### 2.14.6 spec

```
spec:
  replicas: 3
```

`spec` 通常描述：

> **你希望这个对象最终是什么状态。**

例如 Deployment：

```
spec:
  replicas: 3
```

表示：

```
期望有 3 个副本
```

------

### 2.14.7 status

`status` 描述：

> **Kubernetes 当前观察到的实际状态。**

例如：

```
spec:
  replicas: 3

status:
  readyReplicas: 2
```

可以理解为：

```
期望：3
当前 Ready：2
```

Controller 会继续进行调谐。

------

## 2.15 从 `kubectl apply` 到 Pod 启动发生了什么

这一节把前面的所有组件串起来。

假设有一个 Pod：

```
apiVersion: v1
kind: Pod

metadata:
  name: my-api

spec:
  containers:
    - name: my-api
      image: nginx:latest
```

执行：

```
kubectl apply -f pod.yaml
```

------

### 2.15.1 第一步：kubectl 读取 YAML

kubectl 读取：

```
pod.yaml
```

解析：

```
apiVersion
kind
metadata
spec
```

然后准备向 Kubernetes API 发起请求。

------

### 2.15.2 第二步：kubectl 请求 API Server

通信关系：

```
kubectl
   ↓
Kubernetes API
   ↓
kube-apiserver
```

kubectl 不会直接连接 Worker Node 来启动 Container。

------

### 2.15.3 第三步：API Server 处理请求

API Server 会进行一系列处理，例如：

```
认证 Authentication
      ↓
授权 Authorization
      ↓
Admission
      ↓
对象验证
      ↓
持久化
```

最终将对象状态保存到 etcd。

```
kubectl
   ↓
API Server
   ↓
验证
   ↓
etcd
```

此时：

> **Pod Object 已经进入 Kubernetes 的状态体系。**

但是：

> **Container 此时不一定已经启动。**

------

### 2.15.4 第四步：Scheduler 发现待调度 Pod

刚创建的 Pod 可能还没有确定 Node。

可以理解为：

```
Pod
├── name: my-api
├── image: nginx
└── nodeName: 未确定
```

Scheduler 发现这个 Pod：

> “它需要被调度到一个合适的 Node。”

------

### 2.15.5 第五步：Scheduler 选择 Node

假设集群有：

```
Node 1
Node 2
Node 3
```

Scheduler 根据：

```
资源
调度约束
亲和性
反亲和性
Taints
Tolerations
Topology
```

等因素进行决策。

最终：

```
my-api Pod
     ↓
Node 2
```

------

### 2.15.6 第六步：Node 2 上的 kubelet 获取 Pod 信息

Node 2 上运行着 kubelet。

kubelet 发现：

```
Node 2 需要运行：

Pod my-api
image: nginx:latest
```

于是开始按照 Pod 定义执行工作。

------

### 2.15.7 第七步：kubelet 调用 Container Runtime

kubelet 不直接创建 Container。

它通过相应的容器运行接口请求 Container Runtime。

```
kubelet
   ↓
Container Runtime
```

Container Runtime 可能是：

```
containerd
CRI-O
```

------

### 2.15.8 第八步：Container Runtime 启动 Container

Container Runtime 负责：

```
检查镜像
   ↓
拉取镜像
   ↓
创建 Container
   ↓
启动 Container
```

最终：

```
Node 2
│
├── kubelet
├── kube-proxy
├── containerd
│
└── Pod
    └── nginx Container
```

此时 Container 才真正运行起来。

------

### 2.15.9 第九步：状态反馈给 Kubernetes

Container 启动后，kubelet 会持续获取和检查 Pod 的运行状态。

然后：

```
Container
    ↓
kubelet
    ↓
API Server
    ↓
Pod Status
```

最终你执行：

```
kubectl get pods
```

可能看到：

```
NAME     READY   STATUS    RESTARTS
my-api   1/1     Running   0
```

------

### 2.15.10 完整流程

把整个过程连起来：

```
                     kubectl apply
                           │
                           ▼
                    kube-apiserver
                           │
                           ▼
                         etcd
                           │
                           │
                           ▼
                      Scheduler
                           │
                           ▼
                         Node 2
                           │
                         kubelet
                           │
                           ▼
                  Container Runtime
                           │
                           ▼
                       Container
                           │
                           ▼
                      Application
```

而运行状态又不断反馈：

```
Application
    ↓
Container
    ↓
kubelet
    ↓
API Server
    ↓
Kubernetes State
    ↓
Controller / Scheduler
    ↓
持续调谐
```

因此 Kubernetes 并不是：

```
kubectl apply
    ↓
执行一次命令
    ↓
结束
```

而是：

```
提交期望状态
      ↓
Kubernetes 持续观察
      ↓
持续比较
      ↓
持续调整
      ↓
持续维护
```

------

## 本章总结

将本章所有内容放到一张图中：

```
                         Kubernetes Cluster
                                  │
                  ┌───────────────┴───────────────┐
                  │                               │
                  ▼                               ▼
           Control Plane                    Worker Node
                  │                               │
       ┌──────────┼──────────┐           ┌────────┼────────┐
       │          │          │           │        │        │
       ▼          ▼          ▼           ▼        ▼        ▼
 API Server      etcd    Scheduler    kubelet  kube-proxy Runtime
       │                     │           │                  │
       │                     │           │                  ▼
       │                     │           │              Container
       │                     │           │                  │
       │                     └───────────┤                  ▼
       │                                 │             Application
       │                                 │
       └─────────────────────────────────┘
```

其中：

```
API Server
    ↓
所有组件与 Kubernetes API 交互的核心入口
etcd
    ↓
保存 Kubernetes 集群状态
Scheduler
    ↓
决定 Pod → 哪个 Node
Controller
    ↓
Desired State → Actual State
kubelet
    ↓
负责 Node 上 Pod 生命周期
Container Runtime
    ↓
真正运行 Container
```

------

### 本章核心知识速查

| 知识点             | 核心理解                               |
| ------------------ | -------------------------------------- |
| Kubernetes Cluster | 整个 Kubernetes 集群                   |
| Control Plane      | 管理和控制集群                         |
| Worker Node        | 实际运行工作负载                       |
| API Server         | Kubernetes API 的统一入口              |
| etcd               | 保存 Kubernetes 集群状态               |
| Scheduler          | 决定 Pod 运行在哪个 Node               |
| Controller Manager | 运行各种 Controller                    |
| Controller         | 持续调谐 Desired State 与 Actual State |
| kubelet            | Node 上管理 Pod 生命周期               |
| kube-proxy         | 参与 Service 网络转发                  |
| Container Runtime  | 创建和运行 Container                   |
| Kubernetes API     | Kubernetes 的核心操作接口              |
| Object             | Kubernetes API 中的资源对象            |
| spec               | 期望状态                               |
| status             | 当前观察到的状态                       |
| Reconciliation     | 持续让实际状态趋近期望状态             |

------

### 本章必须掌握的三条核心链路

#### 1.请求链路

```
kubectl
   ↓
Kubernetes API
   ↓
API Server
```

------

#### 2.启动链路

```
API Server
   ↓
Scheduler
   ↓
Node
   ↓
kubelet
   ↓
Container Runtime
   ↓
Container
   ↓
Application
```

------

#### 3.Kubernetes 自我调谐链路

```
Desired State
      ↓
Controller
      ↓
Actual State
      ↓
发现差异
      ↓
采取行动
      ↓
Actual State 改变
      ↓
再次检查
      ↺
```

这三条链路是理解后续 Kubernetes 所有知识的基础。

# 第 3 章：Kubernetes 基础操作

本章开始真正接触 Kubernetes 的日常操作。

前面第 2 章主要解决了一个问题：

> **Kubernetes 内部是怎么工作的？**

本章解决另一个问题：

> **作为运维人员或开发者，我应该怎么操作 Kubernetes？**

日常使用 Kubernetes 时，最常用的工具就是：

```
kubectl
```

后续你部署应用、查看 Pod、排查故障、进入容器、查看日志、切换集群，都会大量使用 `kubectl`。

------

## 3.1 `kubectl` 是什么

### 3.1.1 `kubectl` 的定义

`kubectl` 是 Kubernetes 官方命令行客户端。

可以简单理解为：

> **kubectl 是我们操作 Kubernetes 集群最主要的命令行工具。**

例如：

```
kubectl get pods
```

查看 Pod。

```
kubectl get nodes
```

查看 Node。

```
kubectl get services
```

查看 Service。

```
kubectl delete pod my-api
```

删除 Pod。

------

### 3.1.2 `kubectl` 和 Kubernetes 的关系

需要明确：

> `kubectl` 不是 Kubernetes。

它只是 Kubernetes API 的一个客户端。

关系是：

```
┌──────────────┐
│   kubectl    │
└──────┬───────┘
       │
       │ HTTP / HTTPS
       ▼
┌──────────────┐
│ API Server   │
└──────┬───────┘
       │
       ▼
Kubernetes Cluster
```

因此：

```
kubectl get pods
```

并不是 kubectl 自己去找 Pod。

而是：

```
kubectl
   ↓
向 API Server 发请求
   ↓
API Server 查询 Kubernetes 状态
   ↓
返回结果
   ↓
kubectl 展示结果
```

------

### 3.1.3 `kubectl` 的基本语法

最常见的语法：

```
kubectl <command> <resource> [name] [flags]
```

例如：

```
kubectl get pods
```

可以拆成：

```
kubectl
  │
  ├── get
  │
  └── pods
```

其中：

- `kubectl`：客户端
- `get`：操作
- `pods`：资源类型

再例如：

```
kubectl delete pod my-api
```

可以理解为：

```
kubectl
  │
  ├── delete
  │
  ├── pod
  │
  └── my-api
```

------

### 3.1.4 查看 kubectl 版本

```
kubectl version
```

通常可以查看客户端以及服务端版本信息。

也可以：

```
kubectl version --client
```

只查看客户端版本。

------

### 3.1.5 查看 kubectl 帮助

```
kubectl --help
```

查看所有一级命令。

例如：

```
kubectl get --help
```

查看 `get` 命令帮助。

这是非常重要的习惯。

不要认为 Kubernetes 的所有命令都必须背下来。

实际工作中：

> **知道命令体系 + 会使用 `--help` + 会查官方文档，比死记命令更重要。**

------

## 3.2 kubeconfig

### 3.2.1 kubeconfig 是什么

当你执行：

```
kubectl get pods
```

kubectl 必须知道：

```
我要连接哪个 Kubernetes 集群？

API Server 地址是什么？

使用什么身份访问？

使用什么证书？

当前使用哪个 Namespace？
```

这些信息通常来自：

> **kubeconfig**

------

### 3.2.2 kubeconfig 的作用

可以把 kubeconfig 理解为：

> **kubectl 访问 Kubernetes 集群所需要的连接和认证配置。**

逻辑上可以理解为：

```
kubeconfig
│
├── Cluster
│   └── API Server 地址
│
├── User
│   └── 身份认证信息
│
└── Context
    └── Cluster + User + Namespace
```

------

### 3.2.3 kubeconfig 默认位置

Linux / macOS 通常：

```
~/.kube/config
```

Windows 通常：

```
%USERPROFILE%\.kube\config
```

例如：

```
C:\Users\Andy\.kube\config
```

------

### 3.2.4 `KUBECONFIG` 环境变量

kubectl 默认读取：

```
~/.kube/config
```

也可以通过：

```
KUBECONFIG
```

指定其他配置文件。

Linux：

```
export KUBECONFIG=/path/to/config
```

Windows PowerShell：

```
$env:KUBECONFIG="C:\path\to\config"
```

------

### 3.2.5 查看当前 kubeconfig

```
kubectl config view
```

可以查看当前 kubeconfig 内容。

但是生产环境中要注意：

> kubeconfig 可能包含敏感认证信息，不要随意把完整内容发到聊天工具、Git 仓库或者公开平台。

------

## 3.3 Context

### 3.3.1 什么是 Context

Context 是 kubeconfig 中非常重要的概念。

它决定：

> **kubectl 当前应该使用哪个 Cluster、哪个 User，以及默认 Namespace。**

可以理解为：

```
Context
   │
   ├── Cluster
   ├── User
   └── Namespace
```

------

### 3.3.2 为什么需要 Context

假设你同时管理：

```
开发环境
测试环境
生产环境
```

对应：

```
dev-cluster
test-cluster
prod-cluster
```

如果每次都手动指定连接信息，会非常麻烦。

Context 就解决了这个问题。

例如：

```
dev-context
    ↓
dev-cluster
    ↓
dev-user
    ↓
dev namespace
```

切换：

```
kubectl config use-context dev-context
```

然后：

```
kubectl get pods
```

默认就操作开发集群。

切换生产：

```
kubectl config use-context prod-context
```

之后：

```
kubectl get pods
```

操作的就是生产集群。

------

### 3.3.3 查看 Context

```
kubectl config get-contexts
```

例如：

```
CURRENT   NAME             CLUSTER        AUTHINFO       NAMESPACE
*         dev-context      dev-cluster    dev-user       dev
          test-context     test-cluster   test-user      test
          prod-context     prod-cluster   prod-user      production
```

`*` 表示：

> 当前正在使用的 Context。

------

### 3.3.4 查看当前 Context

```
kubectl config current-context
```

例如：

```
prod-context
```

这是一个非常重要的生产操作习惯。

在生产环境执行任何危险命令之前，建议先确认：

```
kubectl config current-context
```

然后再执行：

```
kubectl delete ...
```

因为：

```
操作错 Context
    ↓
操作错 Cluster
    ↓
可能造成生产事故
```

------

### 3.3.5 切换 Context

```
kubectl config use-context dev-context
```

切换到：

```
dev-context
```

然后确认：

```
kubectl config current-context
```

------

## 3.4 Cluster / User / Context

这是 kubeconfig 中最容易混淆的三个概念。

一定要把它们区分清楚。

------

### 3.4.1 Cluster

Cluster 表示：

> **我要连接哪个 Kubernetes 集群。**

例如：

```
dev-cluster
test-cluster
prod-cluster
```

Cluster 通常包含：

```
API Server 地址
CA Certificate
```

例如：

```
clusters:
- name: prod-cluster
  cluster:
    server: https://k8s-api.example.com:6443
```

------

### 3.4.2 User

User 表示：

> **我以什么身份访问 Kubernetes。**

例如：

```
admin
developer
readonly
ci-cd
```

认证方式可能包括：

```
Client Certificate
Bearer Token
OIDC
Cloud IAM
Exec Plugin
```

例如：

```
users:
- name: developer
  user:
    token: ...
```

实际生产环境中，User 往往对应不同权限。

例如：

```
readonly
    ↓
只能查看

developer
    ↓
可以部署开发环境

admin
    ↓
拥有更高权限
```

------

### 3.4.3 Context

Context 把：

```
Cluster
User
Namespace
```

组合起来。

例如：

```
Context: prod-readonly

Cluster:
    prod-cluster

User:
    readonly

Namespace:
    production
```

所以可以记成：

```
Cluster = 去哪里
User    = 以谁的身份
Context = 用谁的身份去哪里，并默认操作哪个 Namespace
```

这是非常重要的一句话。

------

### 3.4.4 三者关系

```
             Context
          ┌─────┼─────┐
          │     │     │
          ▼     ▼     ▼
       Cluster User Namespace
          │      │
          │      │
          ▼      ▼
      去哪个集群  谁在操作
```

例如：

```
prod-admin
    │
    ├── Cluster: prod-cluster
    ├── User: admin
    └── Namespace: production
```

------

## 3.5 `kubectl get`

### 3.5.1 `get` 是最常用的命令

`kubectl get` 用于：

> **查询 Kubernetes Object。**

例如：

```
kubectl get pods
kubectl get nodes
kubectl get services
kubectl get deployments
```

------

### 3.5.2 查看 Pod

```
kubectl get pods
```

例如：

```
NAME      READY   STATUS    RESTARTS   AGE
my-api    1/1     Running   0          10m
```

常见字段：

| 字段     | 含义                 |
| -------- | -------------------- |
| NAME     | Pod 名称             |
| READY    | Ready Container 数量 |
| STATUS   | Pod 当前状态         |
| RESTARTS | Container 重启次数   |
| AGE      | Pod 创建时间         |

------

### 3.5.3 查看 Node

```
kubectl get nodes
```

例如：

```
NAME       STATUS   ROLES           AGE
master-01  Ready    control-plane   10d
worker-01  Ready    <none>          10d
worker-02  Ready    <none>          10d
```

------

### 3.5.4 查看指定资源

```
kubectl get pod my-api
```

------

### 3.5.5 查看更多信息

```
kubectl get pods -o wide
```

可能看到：

```
NAME      READY   STATUS    IP           NODE
my-api    1/1     Running   10.244.1.10  worker-01
```

这样可以看到 Pod：

```
IP
Node
```

------

### 3.5.6 查看 YAML

```
kubectl get pod my-api -o yaml
```

这在排查问题时非常有用。

可以查看：

```
metadata
spec
status
labels
annotations
container
volume
probe
```

------

### 3.5.7 查看 JSON

```
kubectl get pod my-api -o json
```

------

### 3.5.8 查看所有 Namespace

默认情况下：

```
kubectl get pods
```

只查看当前 Namespace。

查看所有 Namespace：

```
kubectl get pods -A
```

等价于：

```
kubectl get pods --all-namespaces
```

------

### 3.5.9 指定 Namespace

```
kubectl get pods -n production
```

其中：

```
-n
```

就是：

```
--namespace
```

------

### 3.5.10 查看资源类型

```
kubectl api-resources
```

可以看到当前 Kubernetes 支持哪些资源。

例如：

```
pods
services
deployments
configmaps
secrets
statefulsets
daemonsets
jobs
cronjobs
```

------

## 3.6 `kubectl describe`

### 3.6.1 `describe` 是什么

`kubectl describe` 用于：

> **查看 Kubernetes Object 的详细信息。**

例如：

```
kubectl describe pod my-api
```

------

### 3.6.2 `get` 和 `describe` 的区别

这是日常排障非常重要的区别。

```
kubectl get pod my-api
```

主要看：

> **当前状态概览。**

而：

```
kubectl describe pod my-api
```

主要看：

> **详细信息以及 Kubernetes 事件。**

可以简单记：

```
get
 ↓
快速看状态

describe
 ↓
深入看细节和事件
```

------

### 3.6.3 describe Pod

```
kubectl describe pod my-api
```

输出通常包括：

```
Name
Namespace
Node
Labels
Annotations
Status
IP
Containers
Conditions
Volumes
Events
```

其中最重要的区域之一：

```
Events:
```

------

### 3.6.4 Events 的重要性

例如 Pod 启动失败：

```
Events:
  FailedScheduling
  FailedMount
  Pulling
  Failed
  BackOff
```

Events 经常能够直接告诉你：

> **Kubernetes 为什么没有按照预期运行。**

因此排查 Pod 问题时非常常见的操作是：

```
kubectl get pod my-api
kubectl describe pod my-api
kubectl logs my-api
```

三者配合使用。

------

## 3.7 `kubectl create`

### 3.7.1 `create` 是什么

`kubectl create` 用于：

> **创建 Kubernetes Object。**

例如：

```
kubectl create namespace test
```

创建 Namespace。

------

### 3.7.2 create 与 YAML

可以：

```
kubectl create -f pod.yaml
```

根据 YAML 创建资源。

------

### 3.7.3 create 的特点

`create` 强调：

> **创建一个不存在的资源。**

如果资源已经存在：

```
kubectl create -f pod.yaml
```

通常会报：

```
AlreadyExists
```

这也是它与 `apply` 的一个重要区别。

------

## 3.8 `kubectl apply`

### 3.8.1 apply 是什么

`kubectl apply` 是 Kubernetes 日常使用中最重要的命令之一。

例如：

```
kubectl apply -f deployment.yaml
```

它表达的是：

> **将 YAML 描述的期望状态应用到 Kubernetes 集群。**

------

### 3.8.2 apply 的核心思想

例如：

```
spec:
  replicas: 3
```

执行：

```
kubectl apply -f deployment.yaml
```

你是在告诉 Kubernetes：

> 我希望这个 Deployment 最终保持 3 个副本。

之后 Kubernetes 自己负责：

```
Controller
    ↓
不断调谐
    ↓
维持 3 个副本
```

------

### 3.8.3 apply 不等于“执行 YAML”

很多初学者容易产生误解：

```
kubectl apply
    ↓
执行 YAML
```

更准确的理解是：

```
YAML
 ↓
描述 Desired State
 ↓
API Server
 ↓
Kubernetes Object
 ↓
Controller
 ↓
逐渐达到 Desired State
```

------

### 3.8.4 create 和 apply 的区别

| 命令     | 核心思想           |
| -------- | ------------------ |
| `create` | 创建资源           |
| `apply`  | 声明并维护期望状态 |

例如第一次：

```
kubectl create -f app.yaml
```

资源不存在：

```
创建成功
```

再次执行：

```
kubectl create -f app.yaml
```

可能：

```
AlreadyExists
```

而：

```
kubectl apply -f app.yaml
```

可以反复执行。

```
第一次
 ↓
创建

第二次
 ↓
根据配置更新

第三次
 ↓
没有变化则基本无需修改
```

因此实际项目中：

> **声明式部署通常以 `kubectl apply` 为核心。**

------

## 3.9 `kubectl delete`

### 3.9.1 删除资源

例如：

```
kubectl delete pod my-api
```

删除 Pod。

------

### 3.9.2 删除 YAML 中定义的资源

```
kubectl delete -f deployment.yaml
```

删除 YAML 中对应的资源。

------

### 3.9.3 删除 Namespace

```
kubectl delete namespace test
```

需要特别注意：

> 删除 Namespace 会导致其中的很多资源一起被删除。

所以生产环境执行：

```
kubectl delete namespace production
```

属于非常危险的操作。

------

### 3.9.4 delete Pod 后为什么可能又出现

例如：

```
kubectl delete pod my-api-xxx
```

Pod 被删除。

过了一会儿：

```
kubectl get pods
```

又出现了一个新的 Pod。

这并不一定是删除失败。

可能是：

```
Deployment
    ↓
ReplicaSet
    ↓
发现副本数量不足
    ↓
创建新 Pod
```

这正是第 2 章学习的：

> **Controller / Reconciliation Loop**

在发挥作用。

------

## 3.10 `kubectl logs`

### 3.10.1 logs 是什么

用于：

> **查看 Container 的标准输出和标准错误日志。**

例如：

```
kubectl logs my-api
```

------

### 3.10.2 实际应用非常多

例如应用启动失败：

```
Pod
 ↓
Container
 ↓
Application
 ↓
启动异常
```

可以：

```
kubectl logs my-api
```

查看：

```
Exception
Connection refused
Database connection failed
Configuration missing
Port already in use
```

------

### 3.10.3 查看实时日志

```
kubectl logs -f my-api
```

其中：

```
-f
```

表示持续跟踪日志。

------

### 3.10.4 查看最近日志

例如：

```
kubectl logs --tail=100 my-api
```

表示查看最后 100 行。

------

### 3.10.5 查看最近一段时间

例如：

```
kubectl logs --since=1h my-api
```

查看最近一小时。

------

### 3.10.6 Pod 有多个 Container

如果一个 Pod 有多个 Container：

```
Pod
├── app
└── sidecar
```

执行：

```
kubectl logs my-api
```

可能无法确定查看哪个 Container。

需要：

```
kubectl logs my-api -c app
```

------

### 3.10.7 查看上一次 Container 日志

如果 Container 已经重启：

```
kubectl logs my-api --previous
```

这个命令在排查：

```
CrashLoopBackOff
```

等问题时非常重要。

------

## 3.11 `kubectl exec`

### 3.11.1 exec 是什么

`kubectl exec` 用于：

> **在 Pod 中的 Container 内执行命令。**

例如：

```
kubectl exec -it my-api -- /bin/sh
```

进入 Container。

------

### 3.11.2 `-i` 和 `-t`

```
-i
```

保持标准输入。

```
-t
```

分配终端。

因此：

```
kubectl exec -it my-api -- /bin/sh
```

通常用于交互式进入容器。

------

### 3.11.3 为什么使用 `--`

例如：

```
kubectl exec -it my-api -- /bin/sh
```

`--` 后面的内容：

```
/bin/sh
```

属于：

> **要在 Container 中执行的命令。**

可以理解为：

```
kubectl exec -it my-api
        │
        └── Kubernetes 参数

--
        │
        └── 分隔线

/bin/sh
        │
        └── Container 内命令
```

------

### 3.11.4 执行单条命令

不进入 Shell，也可以：

```
kubectl exec my-api -- ls /app
```

例如：

```
kubectl exec my-api -- env
```

查看 Container 环境变量。

------

### 3.11.5 多 Container Pod

```
kubectl exec -it my-api -c app -- /bin/sh
```

指定进入：

```
app
```

Container。

------

### 3.11.6 exec 的注意事项

生产环境中不要形成这样的习惯：

```
应用出问题
 ↓
kubectl exec
 ↓
进入 Container
 ↓
手工修改文件
 ↓
问题解决
```

因为 Container 通常应该被视为：

> **可随时销毁和重新创建的实例。**

如果你在 Container 内手工修改配置：

```
Container
    ↓
手工修改
```

之后：

```
Pod 删除
    ↓
新 Pod
    ↓
修改全部消失
```

正确方式通常是修改：

```
Deployment
ConfigMap
Secret
Image
环境变量
Volume
```

然后重新部署。

------

## 3.12 `kubectl port-forward`

### 3.12.1 port-forward 是什么

`kubectl port-forward` 用于：

> **将本地端口转发到 Kubernetes Pod 或 Service。**

例如：

```
kubectl port-forward pod/my-api 8080:80
```

表示：

```
本机 8080
   ↓
Kubernetes
   ↓
Pod 80
```

然后你可以访问：

```
http://localhost:8080
```

------

### 3.12.2 一个典型场景

假设 Pod：

```
my-api
IP: 10.244.1.10
Port: 8080
```

你暂时不想配置：

```
Ingress
LoadBalancer
NodePort
```

可以直接：

```
kubectl port-forward pod/my-api 8080:8080
```

然后：

```
Browser
   ↓
localhost:8080
   ↓
Pod:8080
```

------

### 3.12.3 port-forward 的特点

它非常适合：

```
本地调试
临时访问
开发测试
排查问题
```

但不应该把它当成生产环境的正式流量入口。

因为：

> `port-forward` 主要是临时的本地端口转发机制。

------

## 3.13 `kubectl explain`

### 3.13.1 explain 是什么

`kubectl explain` 用于：

> **查看 Kubernetes API Object 的字段说明。**

例如：

```
kubectl explain pod
```

查看 Pod 的 API 定义。

------

### 3.13.2 查看字段

例如：

```
kubectl explain pod.spec
```

查看：

```
PodSpec
```

再例如：

```
kubectl explain deployment.spec
```

------

### 3.13.3 查看具体字段

例如：

```
kubectl explain deployment.spec.replicas
```

可以查看：

```
replicas
```

字段的含义、类型以及说明。

------

### 3.13.4 为什么 explain 很重要

Kubernetes Object 字段很多。

例如 Deployment：

```
spec:
  replicas:
  selector:
  template:
  strategy:
```

没必要全部死记。

可以：

```
kubectl explain deployment.spec
```

逐层查询。

所以：

> `kubectl explain` 是学习 Kubernetes YAML 非常有价值的工具。

------

## 3.14 Label 和 Selector

### 3.14.1 Label 是什么

Label 是 Kubernetes Object 的：

> **标签。**

例如：

```
metadata:
  labels:
    app: my-api
    env: production
```

可以理解为给对象贴标签：

```
Pod
 ├── app=my-api
 └── env=production
```

------

### 3.14.2 为什么需要 Label

假设有 100 个 Pod：

```
my-api-1
my-api-2
my-api-3
...
my-api-100
```

如果我们想找：

```
所有 my-api
```

不能每次手动指定名字。

可以使用 Label：

```
app=my-api
```

然后：

```
kubectl get pods -l app=my-api
```

------

### 3.14.3 查看 Label

```
kubectl get pods --show-labels
```

例如：

```
NAME      LABELS
my-api-1  app=my-api,env=prod
my-api-2  app=my-api,env=prod
```

------

### 3.14.4 Selector 是什么

Selector 是：

> **根据 Label 选择对象的规则。**

例如：

```
Selector:

app=my-api
```

匹配：

```
Pod A
app=my-api

Pod B
app=my-api
```

不匹配：

```
Pod C
app=frontend
```

------

### 3.14.5 Label 与 Selector 的关系

可以记住：

```
Label
 ↓
给对象贴标签

Selector
 ↓
按照标签找对象
```

例如：

```
Pod A
app=my-api

Pod B
app=my-api

Pod C
app=frontend
```

Selector：

```
app=my-api
```

得到：

```
Pod A
Pod B
```

------

### 3.14.6 Deployment 与 Label

Deployment 中经常看到：

```
spec:
  selector:
    matchLabels:
      app: my-api

  template:
    metadata:
      labels:
        app: my-api
```

这里非常重要：

```
Deployment Selector
        ↓
app=my-api
        ↓
Pod Label
app=my-api
```

Deployment 通过 Selector 找到属于自己的 Pod。

------

### 3.14.7 Service 与 Label

Service 也大量使用 Selector：

```
spec:
  selector:
    app: my-api
```

例如：

```
Service
   │
   │ selector: app=my-api
   ▼
┌───────────────┐
│ Pod A         │
│ app=my-api    │
└───────────────┘

┌───────────────┐
│ Pod B         │
│ app=my-api    │
└───────────────┘
```

因此：

> **Label + Selector 是 Kubernetes 中非常重要的对象关联机制。**

------

## 3.15 Namespace

### 3.15.1 Namespace 是什么

Namespace 是 Kubernetes 中用于：

> **对资源进行逻辑隔离和组织的机制。**

例如：

```
Cluster
│
├── default
├── development
├── testing
└── production
```

------

### 3.15.2 为什么需要 Namespace

假设一个 Kubernetes Cluster 同时运行：

```
开发环境
测试环境
生产环境
```

可以使用：

```
development
testing
production
```

进行逻辑隔离。

例如：

```
kubectl get pods -n development
kubectl get pods -n production
```

------

### 3.15.3 Namespace 不是 Node

这是初学者经常混淆的概念。

Node 是：

```
物理 / 虚拟计算资源
```

Namespace 是：

```
逻辑资源隔离
```

例如：

```
Cluster
│
├── Namespace: development
│   ├── Pod
│   └── Service
│
└── Namespace: production
    ├── Pod
    └── Service
```

这些 Pod 最终仍然可能运行在同一个 Worker Node 上。

------

### 3.15.4 创建 Namespace

可以：

```
kubectl create namespace development
```

也可以使用 YAML：

```
apiVersion: v1
kind: Namespace

metadata:
  name: development
```

然后：

```
kubectl apply -f namespace.yaml
```

------

### 3.15.5 查看 Namespace

```
kubectl get namespaces
```

简写：

```
kubectl get ns
```

------

### 3.15.6 指定 Namespace

```
kubectl get pods -n production
```

如果没有指定：

```
kubectl get pods
```

默认使用当前 Context 配置的 Namespace。

通常默认是：

```
default
```

------

### 3.15.7 Namespace 不等于安全隔离

这一点非常重要。

Namespace 可以提供：

```
资源组织
逻辑隔离
权限边界
资源配额边界
```

但它不是：

> 一个完全独立的 Kubernetes 集群。

不同 Namespace 中的资源仍然属于同一个 Cluster。

------

## 3.16 常用 `kubectl` 命令体系

这一部分建立以后实际工作中的命令地图。

------

### 3.16.1 查询类

```
kubectl get
kubectl describe
kubectl explain
```

例如：

```
kubectl get pods
kubectl describe pod my-api
kubectl explain deployment.spec
```

可以记成：

```
get
 ↓
快速查看

describe
 ↓
详细查看

explain
 ↓
查看 API 字段定义
```

------

### 3.16.2 创建 / 修改类

```
kubectl create
kubectl apply
```

核心区别：

```
create
 ↓
创建资源

apply
 ↓
声明 / 更新期望状态
```

日常 YAML 部署中，最常见的是：

```
kubectl apply -f xxx.yaml
```

------

### 3.16.3 删除类

```
kubectl delete
```

例如：

```
kubectl delete pod my-api
```

或者：

```
kubectl delete -f deployment.yaml
```

------

### 3.16.4 调试类

```
kubectl logs
kubectl exec
kubectl port-forward
```

可以理解为：

```
logs
 ↓
看应用输出

exec
 ↓
进入 / 执行命令

port-forward
 ↓
临时访问应用
```

------

### 3.16.5 配置类

```
kubectl config
```

常见：

```
kubectl config get-contexts
kubectl config current-context
kubectl config use-context dev-context
kubectl config view
```

------

### 3.16.6 API / 资源发现类

```
kubectl api-resources
```

查看资源类型。

```
kubectl api-versions
```

查看 API Version。

```
kubectl explain
```

查看资源字段。

------

### 3.16.7 一个实用的命令分类

以后可以按照下面的体系记忆：

```
kubectl
│
├── 配置
│   └── config
│
├── 查询
│   ├── get
│   ├── describe
│   └── explain
│
├── 创建 / 修改
│   ├── create
│   └── apply
│
├── 删除
│   └── delete
│
├── 调试
│   ├── logs
│   ├── exec
│   └── port-forward
│
└── API / 资源
    ├── api-resources
    └── api-versions
```

------

## 实际工作中的基础操作流程

假设你接手一个 Kubernetes 应用：

```
my-api
```

通常不会一上来就执行复杂命令。

可以按照下面的思路操作。

------

### 第一步：确认当前集群

```
kubectl config current-context
```

------

### 第二步：确认 Node

```
kubectl get nodes
```

确认：

```
Node 是否 Ready
```

------

### 第三步：查看 Namespace

```
kubectl get ns
```

------

### 第四步：查看 Pod

```
kubectl get pods -n production
```

------

### 第五步：发现异常时 describe

```
kubectl describe pod my-api -n production
```

重点查看：

```
Events
```

------

### 第六步：查看日志

```
kubectl logs my-api -n production
```

如果发生过重启：

```
kubectl logs my-api -n production --previous
```

------

### 第七步：必要时进入 Container

```
kubectl exec -it my-api -n production -- /bin/sh
```

进行进一步排查。

------

### 第八步：临时访问应用

```
kubectl port-forward pod/my-api 8080:8080 -n production
```

然后本机访问：

```
http://localhost:8080
```

------

## 本章命令速查表

| 操作                      | 命令                                      |
| ------------------------- | ----------------------------------------- |
| 查看 kubectl 版本         | `kubectl version`                         |
| 查看帮助                  | `kubectl --help`                          |
| 查看 Pod                  | `kubectl get pods`                        |
| 查看 Node                 | `kubectl get nodes`                       |
| 查看所有 Namespace 的 Pod | `kubectl get pods -A`                     |
| 指定 Namespace            | `kubectl get pods -n production`          |
| 查看更多 Pod 信息         | `kubectl get pods -o wide`                |
| 查看 YAML                 | `kubectl get pod my-api -o yaml`          |
| 查看详细信息              | `kubectl describe pod my-api`             |
| 创建资源                  | `kubectl create -f app.yaml`              |
| 应用 YAML                 | `kubectl apply -f app.yaml`               |
| 删除资源                  | `kubectl delete pod my-api`               |
| 查看日志                  | `kubectl logs my-api`                     |
| 实时日志                  | `kubectl logs -f my-api`                  |
| 查看上一次日志            | `kubectl logs my-api --previous`          |
| 进入 Container            | `kubectl exec -it my-api -- /bin/sh`      |
| 执行命令                  | `kubectl exec my-api -- ls /app`          |
| 端口转发                  | `kubectl port-forward pod/my-api 8080:80` |
| 查看 API 字段             | `kubectl explain pod.spec`                |
| 查看资源类型              | `kubectl api-resources`                   |
| 查看 Namespace            | `kubectl get ns`                          |
| 查看 Context              | `kubectl config get-contexts`             |
| 查看当前 Context          | `kubectl config current-context`          |
| 切换 Context              | `kubectl config use-context xxx`          |
| 查看 kubeconfig           | `kubectl config view`                     |

------

## 本章核心知识总结

本章最重要的不是背几十条命令，而是建立以下操作模型：

```
                    kubectl
                       │
          ┌────────────┼────────────┐
          │            │            │
        查询          修改          调试
          │            │            │
       get          apply         logs
       describe     create        exec
       explain      delete        port-forward
          │
          ▼
      API Server
          │
          ▼
   Kubernetes Cluster
```

同时掌握 kubeconfig：

```
kubeconfig
    │
    ├── Cluster
    │     └── 去哪个集群
    │
    ├── User
    │     └── 以谁的身份
    │
    └── Context
          └── Cluster + User + Namespace
```

以及 Kubernetes 中非常重要的对象关联机制：

```
Label
  ↓
给 Object 打标签

Selector
  ↓
按照标签选择 Object
```

最后建立一个非常实用的故障排查思维：

```
1. 我现在连接的是哪个集群？
        ↓
   kubectl config current-context

2. Node 是否正常？
        ↓
   kubectl get nodes

3. Pod 当前是什么状态？
        ↓
   kubectl get pods

4. 为什么是这个状态？
        ↓
   kubectl describe pod

5. 应用自己报了什么错误？
        ↓
   kubectl logs

6. 还需要进入容器检查什么？
        ↓
   kubectl exec

7. 是否需要从本地临时访问？
        ↓
   kubectl port-forward
```

这套流程以后会成为你进行 Kubernetes **日常运维和故障排查的基础操作习惯**。
