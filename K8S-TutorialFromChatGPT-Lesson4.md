## 第四阶段 第一章：Kubernetes 存储（上）——Volume、PV、PVC 与 StorageClass

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

### 本章学习目标

学习完本章，你应该能够回答：

- 为什么 Pod 重建后数据会丢失？
- Volume 是什么？
- emptyDir、hostPath、PersistentVolume 有什么区别？
- PV（PersistentVolume）是什么？
- PVC（PersistentVolumeClaim）为什么叫"声明"？
- StorageClass 又解决了什么问题？
- Kubernetes 为什么要把存储拆成这么多对象？

------

### 第一节：为什么 Pod 重建后数据会丢失？

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

### 第二节：Pod 的生命周期和数据生命周期不同

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

### 第三节：什么是 Volume？

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

### 第四节：emptyDir —— 最简单的 Volume

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

#### 适用场景

例如：

- 临时缓存
- 下载中间文件
- 解压 ZIP
- 编译缓存

这些数据即使丢失，也可以重新生成。

------

### 第五节：hostPath —— 使用 Node 本地磁盘

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

#### hostPath 的优点

简单。

不用额外配置存储系统。

学习 Kubernetes 时非常方便。

------

#### hostPath 的缺点（生产环境必须理解）

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

### 第六节：真正的持久化存储应该是什么样？

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

### 第七节：什么是 PersistentVolume（PV）？

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

#### 一个生活中的例子

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

### 第八节：什么是 PVC（PersistentVolumeClaim）？

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

#### 仓库的例子

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

### 第九节：为什么要分成 PV 和 PVC？

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

### 第十节：PVC 与 PV 的绑定关系

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

### 第十一节：什么是 StorageClass？

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

### 第十二节：本章知识关系图

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

### 第十三节：本章总结（建议牢记）

请记住 Kubernetes 存储最重要的几点：

1. **Pod 默认使用临时文件系统，Pod 删除后数据通常会丢失。**
2. **Volume 是挂载到 Pod 的存储空间。**
3. **`emptyDir` 适合临时数据，不适合重要业务数据。**
4. **`hostPath` 依赖单个 Node，不适合作为大多数生产环境的持久化方案。**
5. **PV 表示实际存在的存储资源，PVC 表示应用对存储的声明。**
6. **Pod 应该引用 PVC，而不是直接使用 PV。**
7. **StorageClass 让存储能够动态创建，是现代 Kubernetes 持久化存储的推荐方式。**

------

### 一个容易混淆但必须记住的区别

| 对象         | 作用             | 谁关心          |
| ------------ | ---------------- | --------------- |
| Volume       | Pod 内挂载点     | 应用            |
| PV           | 实际存储资源     | 运维 / 存储系统 |
| PVC          | 对存储的申请     | 开发、应用      |
| StorageClass | 定义如何创建存储 | 平台运维        |

理解这四者之间的关系，比背 YAML 更重要。

------

### 下一章预告：Kubernetes 存储（下）——StatefulSet、数据库部署与生产实践

下一章，我们将把这些概念真正应用起来，学习：

- 为什么数据库不能随便使用 Deployment？
- StatefulSet 与 Deployment 的本质区别是什么？
- Headless Service 是什么？
- MySQL、PostgreSQL、Redis 应该如何部署？
- PVC 删除后数据一定会删除吗？
- StorageClass 的回收策略（Retain、Delete）有什么区别？
- 企业生产环境如何设计 Kubernetes 存储架构？

这一章结束后，你将真正具备部署**有状态应用（Stateful Applications）**的能力，也会理解 Kubernetes 如何安全地管理数据库和持久化数据。

## 第四阶段 第二章：StatefulSet、数据库部署与生产实践

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

### 本章学习目标

学习完本章，你应该能够回答：

- 为什么 MySQL 不建议使用 Deployment？
- StatefulSet 和 Deployment 到底有什么区别？
- Headless Service 是什么？
- StatefulSet 为什么必须配合 Headless Service？
- PVC 为什么不会随着 Pod 删除？
- StorageClass 的回收策略（Retain、Delete）是什么？
- 企业如何部署 MySQL、Redis、PostgreSQL 等有状态应用？

------

### 第一节：什么是有状态应用（Stateful Application）？

我们先区分两类应用。

#### 无状态应用（Stateless）

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

#### 有状态应用（Stateful）

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

### 第二节：为什么 MySQL 不建议使用 Deployment？

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

### 第三节：Deployment 与 StatefulSet 的本质区别

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

#### 一个生活中的例子

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

### 第四节：StatefulSet 的三个核心特点

请牢牢记住：

StatefulSet 主要提供：

#### ① 稳定的 Pod 名称

例如：

```
redis-0

redis-1

redis-2
```

永远：

固定。

------

#### ② 稳定的网络身份

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

#### ③ 稳定的存储

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

### 第五节：为什么 StatefulSet 必须配合 Headless Service？

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

### 第六节：什么是 Headless Service？

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

### 第七节：为什么每个 Pod 都有自己的 PVC？

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

### 第八节：Pod 删除后，PVC 为什么还在？

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

### 第九节：StorageClass 回收策略（Reclaim Policy）

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

#### Delete

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

#### Retain

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

### 第十节：生产环境推荐部署方式

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

### 第十一节：企业真的自己部署数据库吗？

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

### 第十二节：完整关系图

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

### 第十三节：Deployment 与 StatefulSet 对比

| 对比项       | Deployment     | StatefulSet              |
| ------------ | -------------- | ------------------------ |
| Pod 名称     | 随机后缀       | 固定编号（如 `mysql-0`） |
| Pod 身份     | 无状态         | 有状态                   |
| Pod 网络身份 | 不固定         | 固定 DNS 名称            |
| 存储         | 可共享或临时   | 每个 Pod 独立 PVC        |
| 适用场景     | Web、API、前端 | 数据库、中间件、消息队列 |

------

### 第十四节：新手最容易犯的错误

下面这些问题在实际工作中非常常见：

❌ 用 Deployment 部署 MySQL，然后发现升级后数据丢失。

❌ 多个数据库实例共享同一个读写卷，导致数据损坏。

❌ 删除 PVC 时，没有确认回收策略，误删生产数据。

❌ 没有做数据库备份，误以为 PVC 就等于备份。

请记住：

> **PVC 是存储，不是备份。**

磁盘损坏、误删除、逻辑错误等问题，仍然需要定期备份。

------

### 第十五节：本章总结（建议牢记）

请记住 StatefulSet 最重要的几点：

1. **Deployment 适用于无状态应用，StatefulSet 适用于有状态应用。**
2. **StatefulSet 提供稳定的 Pod 名称、网络身份和存储。**
3. **StatefulSet 通常与 Headless Service 配合使用，实现固定的 DNS 名称。**
4. **每个 StatefulSet Pod 都应拥有独立的 PVC。**
5. **生产环境应重点关注 StorageClass 的回收策略，避免误删数据。**
6. **PVC 提供持久化存储，但不能替代备份。**

------

### 到这里，你已经掌握了 Kubernetes 持久化存储体系

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

### 下一章预告：Kubernetes 调度原理（Scheduler 深度解析）

前面我们一直在"创建 Pod"，但还有一个关键问题没有解释：

- Pod 为什么会被调度到某台 Node？
- Scheduler 是如何选择 Node 的？
- `nodeSelector`、`Node Affinity`、`Pod Affinity`、`Pod Anti-Affinity` 有什么区别？
- Taint（污点）和 Toleration（容忍）为什么能隔离工作负载？
- 企业如何保证数据库、GPU、AI 训练任务运行在指定节点？

下一章开始，我们将深入学习 **Kubernetes 调度器（Scheduler）** 的工作原理。这也是设计高可用集群、资源隔离和性能优化的关键知识。