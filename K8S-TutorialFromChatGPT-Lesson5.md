## 第五阶段 第一章：Kubernetes 调度原理（Scheduler 深度解析）

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

### 本章学习目标

学习完本章，你应该能够回答：

- Scheduler 到底负责什么？
- Pod 创建以后发生了什么？
- Scheduler 如何选择 Node？
- Filter（过滤）和 Score（打分）是什么？
- 为什么 Pod 会一直 Pending？
- Resource Request 和 Limit 为什么影响调度？
- 企业如何优化调度策略？

------

### 第一节：Pod 创建以后，到底发生了什么？

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

### 第二节：Scheduler 到底是什么？

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

#### 一个生活中的例子

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

### 第三节：Scheduler 的两步核心流程

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

### 第四节：第一步 —— Filter（过滤）

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

### 第五节：第二步 —— Score（打分）

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

### 第六节：Filter 会检查哪些内容？

最重要的几项：

------

#### ① CPU 是否够？

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

#### ② Memory 是否够？

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

#### ③ Node 是否 Ready？

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

#### ④ 是否满足节点选择条件？

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

### 第七节：为什么 Pod 一直 Pending？

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

### 第八节：为什么 Request 会影响调度？

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

#### 一个生活中的例子

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

### 第九节：Limit 会影响调度吗？

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

### 第十节：Scheduler 如何知道 Node 资源？

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

### 第十一节：一次完整调度流程

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

### 第十二节：企业为什么要学习调度？

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

### 第十三节：本章知识关系图

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

### 第十四节：本章总结（建议牢记）

请记住 Scheduler 最重要的几点：

1. **Scheduler 的职责只有一个：为 Pod 选择最合适的 Node。**
2. **调度过程分为两个阶段：Filter（过滤）和 Score（打分）。**
3. **Filter 会淘汰不符合条件的 Node，例如资源不足、标签不匹配、污点等。**
4. **Score 会在候选 Node 中选择最优节点。**
5. **Scheduler 根据 Pod 的 Request，而不是实际资源使用率进行调度。**
6. **Pod 长时间处于 Pending，首先要检查 Scheduler 是否找到了可用 Node。**
7. **Scheduler 通过 API Server 完成 Pod 与 Node 的绑定，真正启动容器的是 kubelet。**

------

### 本章结束后，你已经理解了 Scheduler 的整体工作方式

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

### 下一章预告：节点选择与调度策略——nodeSelector、Node Affinity、Pod Affinity、Pod Anti-Affinity

下一章，我们将深入学习**如何控制 Pod 被调度到哪里**，包括：

- `nodeSelector` 是如何工作的？
- `Node Affinity` 为什么比 `nodeSelector` 更强大？
- `Pod Affinity` 和 `Pod Anti-Affinity` 有什么区别？
- 如何让两个服务尽量部署在一起？
- 如何让多个副本尽量分散到不同 Node，提高高可用？
- 企业如何为 GPU、数据库、AI 训练、日志节点设计调度规则？

学完这一章，你将能够精确控制 Kubernetes 的调度行为，这也是生产环境资源规划和高可用设计的重要能力。

## 第五阶段 第二章：节点选择与调度策略——nodeSelector、Node Affinity、Pod Affinity、Pod Anti-Affinity

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

### 本章学习目标

学习完本章，你应该能够回答：

- nodeSelector 是什么？
- Node Label 是什么？
- Node Affinity 为什么比 nodeSelector 更强？
- Pod Affinity 与 Pod Anti-Affinity 有什么区别？
- 企业如何保证高可用？
- 为什么 StatefulSet 经常使用 Anti-Affinity？

------

### 第一节：为什么需要指定调度位置？

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

### 第二节：Node Label 是什么？

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

#### 一个生活中的例子

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

### 第三节：nodeSelector（最简单的节点选择）

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

### 第四节：nodeSelector 的限制

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

### 第五节：Node Affinity（节点亲和性）

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

### 第六节：Node Affinity 支持哪些操作？

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

### 第七节：硬亲和 vs 软亲和

这是 Node Affinity 最重要的知识。

Kubernetes：

提供：

两种：

策略。

------

#### ① requiredDuringSchedulingIgnoredDuringExecution

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

#### ② preferredDuringSchedulingIgnoredDuringExecution

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

#### 一个生活中的例子

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

### 第八节：Pod Affinity（Pod 亲和）

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

### 第九节：Pod Anti-Affinity（Pod 反亲和）

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

### 第十节：为什么 StatefulSet 经常使用 Anti-Affinity？

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

### 第十一节：企业生产环境的典型调度策略

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

### 第十二节：nodeSelector 与 Node Affinity 对比

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

### 第十三节：一次完整的调度决策

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

### 第十四节：本章知识关系图

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

### 第十五节：本章总结（建议牢记）

请记住调度策略最重要的几点：

1. **Node Label 是节点的"标签"，Scheduler 根据标签做决策。**
2. **`nodeSelector` 适用于简单的标签匹配。**
3. **`Node Affinity` 是更强大的节点选择机制，支持复杂表达式和软/硬亲和。**
4. **`Pod Affinity` 用于让 Pod 尽量部署在一起，适合降低通信延迟。**
5. **`Pod Anti-Affinity` 用于让 Pod 尽量分散，提高高可用。**
6. **数据库、Redis、Kafka、Elasticsearch 等有状态应用通常会使用 `Pod Anti-Affinity`。**

------

### 🌟 企业经验：什么时候该用哪一种？

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

### 下一章预告：Taint（污点）与 Toleration（容忍）——企业资源隔离的核心

下一章，我们将学习 Kubernetes 中另一套极其重要的调度机制：

- 什么是 **Taint（污点）**？
- 什么是 **Toleration（容忍）**？
- 为什么 Node 会"拒绝"Pod？
- GPU 节点为什么要打污点？
- 如何防止普通业务占用数据库节点？
- `NoSchedule`、`PreferNoSchedule`、`NoExecute` 有什么区别？
- 企业如何利用污点实现节点资源隔离？

学完这一章，你将掌握 Kubernetes 在生产环境中进行**资源隔离、专用节点管理和高价值资源保护**的核心能力。

## 第五阶段 第三章：Taint（污点）与 Toleration（容忍）——企业资源隔离的核心

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

### 本章学习目标

学习完本章，你应该能够回答：

- 为什么需要 Taint？
- Taint 和 Node Affinity 有什么区别？
- NoSchedule、PreferNoSchedule、NoExecute 分别是什么意思？
- Toleration 为什么不是"允许调度"？
- 为什么 GPU 节点几乎都会使用 Taint？
- 企业如何利用 Taint 做资源隔离？

------

### 第一节：为什么需要 Taint？

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

### 第二节：什么是 Taint？

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

#### 一个生活中的例子

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

### 第三节：Label 与 Taint 的区别

很多新人：

都会混淆。

请一定：

牢记：

#### Label

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

#### Taint

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

### 第四节：如何添加 Taint？

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

### 第五节：什么是 Toleration？

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

#### 一个生活中的例子

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

### 第六节：Taint 的三种 Effect

这是最重要的知识。

------

### ① NoSchedule

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

#### 适用场景

- GPU
- 数据库
- AI
- 高性能服务器

企业：

最常用。

------

### ② PreferNoSchedule

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

#### 一个生活中的例子

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

### ③ NoExecute

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

#### 一个生活中的例子

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

### 第七节：为什么 Kubernetes 自己也使用 Taint？

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

### 第八节：Node Affinity 与 Taint 到底有什么区别？

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

### 第九节：企业真实案例

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

### 第十节：为什么不能只用 Node Affinity？

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

### 第十一节：完整工作流程

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

### 第十二节：企业最佳实践

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

### 第十三节：新手最容易犯的错误

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

### 第十四节：本章知识关系图

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

### 第十五节：本章总结（建议牢记）

请记住 Taint 与 Toleration 最重要的几点：

1. **Label 描述 Node，Taint 保护 Node。**
2. **Toleration 只是允许 Pod 忽略污点，不保证一定会调度成功。**
3. **`NoSchedule` 是生产环境最常用的污点效果。**
4. **`PreferNoSchedule` 是软限制，尽量避免调度。**
5. **`NoExecute` 不仅影响新 Pod，还可能驱逐已运行的 Pod。**
6. **企业通常将 `Node Affinity` 与 `Taint/Toleration` 组合使用，实现精准调度和资源隔离。**

------

### 🌟 企业经验：一套经典组合

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

### 下一章预告：资源管理与 QoS——Requests、Limits、ResourceQuota、LimitRange

下一章，我们将深入学习 Kubernetes 的资源管理机制，包括：

- 为什么一定要设置 `requests` 和 `limits`？
- CPU 为什么会被"限速"？
- 内存超限为什么会 OOMKilled？
- Kubernetes 的 QoS（服务质量）分为哪三类？
- `ResourceQuota` 如何限制一个 Namespace 的资源总量？
- `LimitRange` 如何为开发团队设置默认资源？
- 企业如何避免某个团队占满整个集群？

学完这一章，你将掌握 Kubernetes **资源管理与资源治理** 的核心能力，这是生产环境稳定运行不可或缺的一部分。

## 第五阶段 第四章：资源管理与 QoS——Requests、Limits、ResourceQuota、LimitRange

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

### 本章学习目标

学习完本章，你应该能够回答：

- 为什么要设置 Requests 和 Limits？
- Request 与 Limit 到底有什么区别？
- CPU 超过 Limit 会怎么样？
- 内存超过 Limit 会怎么样？
- OOMKilled 是什么？
- QoS（服务质量）为什么会影响 Pod 被驱逐？
- ResourceQuota 与 LimitRange 分别解决什么问题？

------

### 第一节：为什么 Kubernetes 要管理资源？

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

#### 一个生活中的例子

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

### 第二节：Requests（资源申请）

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

#### Request 的作用

它主要负责两件事：

1. **调度（Scheduling）**

Scheduler 会检查 Node 是否有足够的可分配资源。

1. **资源预留（Reservation）**

即使程序当前没用到这些资源，Kubernetes 也会认为这些资源已经"预留"给了这个 Pod。

------

### 第三节：Limits（资源上限）

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

### 第四节：CPU 超过 Limit 会怎么样？

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

#### 一个生活中的例子

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

### 第五节：内存超过 Limit 会怎么样？

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

#### 为什么内存不能像 CPU 一样限速？

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

### 第六节：OOMKilled 是什么？

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

#### 排查思路

遇到 OOMKilled，不要第一时间只想着把 Limit 调大，可以按这个顺序检查：

1. **应用是否存在内存泄漏？**
2. **Request / Limit 是否设置过小？**
3. **业务流量是否明显增加？**
4. **是否需要优化缓存策略或批量处理逻辑？**

生产环境中，**持续出现 OOMKilled 更值得先检查应用本身，而不是无限增加内存。**

------

### 第七节：Requests 与 Limits 的关系

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

### 第八节：QoS（Quality of Service）

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

#### 第一类：Guaranteed（最高等级）

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

#### 第二类：Burstable（可突发）

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

#### 第三类：BestEffort（最低等级）

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

### 第九节：Node 资源不足时，谁先被驱逐？

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

### 第十节：ResourceQuota（资源配额）

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

#### 一个生活中的例子

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

### 第十一节：LimitRange（默认资源）

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

### 第十二节：企业资源治理模型

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

### 第十三节：新手最容易犯的错误

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

### 第十四节：知识关系图

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

### 第十五节：企业最佳实践

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

### 第十六节：本章总结（建议牢记）

请记住资源管理最重要的几点：

1. **Request 决定调度，表示最低保障资源。**
2. **Limit 决定运行时资源上限。**
3. **CPU 超过 Limit 通常会被限速（Throttling），不会直接退出。**
4. **内存超过 Limit 会触发 OOMKilled。**
5. **QoS 分为 Guaranteed、Burstable、BestEffort，资源紧张时驱逐优先级依次为：BestEffort → Burstable → Guaranteed。**
6. **ResourceQuota 用于限制整个 Namespace 的资源总量。**
7. **LimitRange 用于提供默认资源配置，避免出现没有资源限制的 Pod。**

------

### 🌟 到这里，你已经掌握了 Kubernetes 的资源管理体系

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

### 下一章预告：滚动更新与应用发布——RollingUpdate、Recreate、金丝雀发布、蓝绿发布

下一章，我们将进入 Kubernetes 在企业中最常用的发布能力，学习：

- 为什么 Deployment 更新不会中断服务？
- RollingUpdate（滚动更新）到底是如何工作的？
- `maxSurge` 和 `maxUnavailable` 如何影响升级过程？
- Recreate 模式什么时候才适合使用？
- 什么是金丝雀发布（Canary）？
- 什么是蓝绿发布（Blue-Green）？
- Kubernetes 原生能做到哪些发布策略，哪些需要配合 Ingress Controller 或服务网格？

学完这一章，你将掌握 Kubernetes 从**开发发布到生产上线**的核心实践，也是 DevOps 流程中的重要一环。