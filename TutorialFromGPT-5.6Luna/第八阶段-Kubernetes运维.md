# 第 21 章：日志系统

日志是 Kubernetes 生产运维中最重要的基础设施之一。

应用出现问题时，我们通常首先需要回答：

```
什么时候发生？
哪个 Pod 发生？
哪个 Node 发生？
哪个 Container 发生？
发生了什么？
请求经过了哪里？
```

如果没有完善的日志系统，Kubernetes 集群即使能够正常运行，也很难进行生产故障排查。

本章重点建立一个完整认识：

```
Application
    ↓
Container
    ↓
Pod
    ↓
Node
    ↓
Log Collector
    ↓
Log Storage
    ↓
Log Search / Visualization
```

------

## 21.1 Kubernetes 日志模型

Kubernetes 本身**不是一个完整的日志平台**。

它主要负责：

```
运行 Container
管理 Pod
管理 Node
```

而日志通常由：

```
Container Runtime
+
Node
+
日志采集组件
+
日志存储系统
+
查询系统
```

共同组成。

最基本的结构：

```
┌───────────────────────┐
│        Pod            │
│                       │
│  Application          │
│      │                │
│      ▼                │
│  stdout / stderr      │
└──────────┬────────────┘
           │
           ▼
   Container Runtime
           │
           ▼
      Node 日志
           │
           ▼
     Log Collector
           │
           ▼
      Log Storage
           │
           ▼
       查询平台
```

Kubernetes 推荐应用将日志输出到：

```
stdout
stderr
```

而不是直接依赖某个固定的文件路径。

例如：

```
Console.WriteLine("Order created");
```

或者 ASP.NET Core：

```
logger.LogInformation(
    "Order {OrderId} created",
    orderId);
```

最终日志写到标准输出。

------

## 21.2 `kubectl logs`

最常用的 Kubernetes 日志命令：

```
kubectl logs POD_NAME
```

例如：

```
kubectl logs shop-api-7d9f8d6f8b-x2abc
```

查看指定 Namespace：

```
kubectl logs \
  -n shop \
  shop-api-7d9f8d6f8b-x2abc
```

------

### 21.2.1 查看最近日志

```
kubectl logs \
  -n shop \
  shop-api-7d9f8d6f8b-x2abc \
  --tail=100
```

只查看最后 100 行。

生产排查时非常有用。

否则：

```
kubectl logs POD
```

如果应用运行了很长时间，可能输出大量内容。

------

### 21.2.2 持续查看日志

```
kubectl logs \
  -n shop \
  shop-api-7d9f8d6f8b-x2abc \
  -f
```

`-f` 相当于：

```
follow
```

类似 Linux：

```
tail -f
```

可以实时观察：

```
Application
    ↓
stdout
    ↓
kubectl logs -f
```

------

### 21.2.3 查看指定时间的日志

例如最近 1 小时：

```
kubectl logs \
  -n shop \
  shop-api-7d9f8d6f8b-x2abc \
  --since=1h
```

最近 10 分钟：

```
kubectl logs \
  -n shop \
  shop-api-7d9f8d6f8b-x2abc \
  --since=10m
```

这比直接打印全部历史日志更适合故障排查。

------

### 21.2.4 Pod 有多个 Container

如果一个 Pod：

```
Pod
├── api
└── sidecar
```

直接执行：

```
kubectl logs pod-name
```

可能无法确定应该查看哪个 Container。

需要：

```
kubectl logs pod-name -c api
```

或者：

```
kubectl logs pod-name -c sidecar
```

------

### 21.2.5 查看上一次 Container 的日志

这是生产排障非常重要的命令。

假设 Container：

```
Running
    ↓
Crash
    ↓
Kubernetes Restart
```

当前 Container 已经重新启动。

可以查看上一个 Container：

```
kubectl logs \
  -n shop \
  pod-name \
  -c api \
  --previous
```

这个命令经常用于排查：

```
CrashLoopBackOff
OOMKilled
Application Crash
```

例如：

```
kubectl get pod

NAME        READY   STATUS
shop-api    1/1     Running
```

虽然现在是 `Running`，但它可能刚刚发生过：

```
Container Crash
```

这时候：

```
kubectl logs --previous
```

可能直接找到真正原因。

------

## 21.3 Container Logs

Kubernetes 日志的基础是 Container 日志。

例如应用：

```
.NET API
```

执行：

```
Console.WriteLine("Hello");
```

Container：

```
┌───────────────────────┐
│ Container             │
│                       │
│ Application           │
│        │              │
│        ▼              │
│ stdout / stderr       │
└───────────┬───────────┘
            │
            ▼
     Container Runtime
```

Container Runtime 负责处理 Container 的标准输出。

现代 Kubernetes 环境常见的 Runtime 是：

```
containerd
```

需要注意：

> Kubernetes 并不是把日志永久保存在 Pod 对象里。

`kubectl logs` 实际上是在请求 Kubernetes 节点上的容器运行时日志。

------

## 21.4 Pod Logs

一个 Pod 可以包含：

```
Pod
├── Container A
├── Container B
└── Container C
```

所以严格来说：

> Pod 没有一个独立于 Container 的“日志文件”。

我们看到的：

```
kubectl logs pod-name
```

实际上是查看 Pod 中 Container 的日志。

例如：

```
shop-api
└── api
```

执行：

```
kubectl logs shop-api
```

相当于查看：

```
shop-api
   ↓
api Container
   ↓
stdout / stderr
```

如果是多 Container：

```
shop-api
├── api
└── envoy
```

需要分别查看。

```
kubectl logs shop-api -c api
kubectl logs shop-api -c envoy
```

------

### 21.4.1 查看 Pod 所有 Container

```
kubectl get pod shop-api \
  -n shop \
  -o jsonpath='{.spec.containers[*].name}'
```

可能得到：

```
api envoy
```

然后分别查询。

------

### 21.4.2 查看 Deployment 的日志

`kubectl logs` 主要针对 Pod。

如果希望通过 Deployment 快速查看：

```
kubectl logs \
  -n shop \
  deployment/shop-api
```

如果 Deployment 有多个 Pod，kubectl 会根据选择的 Pod 获取日志。

生产环境不要把它当成完整日志查询方案，因为：

```
Pod 会变化
Pod 会被删除
Pod 会被重新创建
```

因此不能依赖单个 Pod 的日志作为长期历史记录。

------

## 21.5 Node Logs

Kubernetes 日志最终与 Node 有很强的关系。

例如：

```
Node-01
├── Pod A
├── Pod B
└── Pod C
```

Container 日志通常存放在 Node 本地。

在常见 Linux Kubernetes 环境中，可以看到：

```
/var/log/containers/
```

以及：

```
/var/log/pods/
```

例如：

```
ls /var/log/containers/
```

可能看到：

```
shop-api-xxx_shop_api-xxx.log
```

具体日志路径和格式与 Container Runtime、Kubernetes 版本以及发行版有关，因此生产环境不要把某一个具体文件路径当成 Kubernetes 的绝对规范。

------

### 21.5.1 为什么 Node 日志很重要

假设：

```
Pod
    ↓
Crash
    ↓
Pod 被删除
```

如果日志只存在于 Pod 生命周期对应的 Node 本地文件中，那么 Pod 消失之后，本地日志也可能无法再方便地访问。

因此生产环境需要：

```
Node
 ↓
Log Collector
 ↓
Central Log Storage
```

而不是：

```
Node
 ↓
本地日志
 ↓
永远保留
```

------

## 21.6 日志轮转

如果应用不断输出：

```
1 MB/s
```

那么一天：

```
1 MB × 60 × 60 × 24
≈ 86 GB
```

如果没有日志轮转：

```
Node Disk
   ↓
不断增长
   ↓
Disk Full
```

最终可能导致严重问题。

例如：

```
Node
├── kubelet
├── containerd
├── Pod
└── logs
          ↓
      Disk 100%
```

磁盘耗尽后可能影响：

```
Pod 创建
Image Pull
Container Runtime
Kubelet
```

甚至触发：

```
DiskPressure
```

------

### 21.6.1 日志轮转是什么

日志轮转就是：

```
当前日志
    ↓
达到大小
    ↓
切割
    ↓
生成新日志文件
```

例如：

```
container.log
container.log.1
container.log.2
container.log.3
```

这样避免：

```
一个日志文件无限增长
```

------

### 21.6.2 生产环境需要控制什么

通常需要考虑：

```
单文件大小
保留文件数量
保留时间
总磁盘占用
日志采集速度
```

例如：

```
单文件：100 MB
保留：5 个
```

实际参数取值需要结合：

```
业务日志量
Node 磁盘大小
日志采集频率
日志平台容量
```

进行设计。

不要简单地认为：

```
日志越多越好
```

真正生产环境更重要的是：

> **保留有价值的信息，同时控制成本和磁盘风险。**

------

## 21.7 Fluent Bit

Fluent Bit 是 Kubernetes 中非常常见的日志采集 Agent。

它通常以：

```
DaemonSet
```

部署。

为什么？

因为我们希望：

```
每个 Node
   ↓
运行一个 Fluent Bit
```

架构：

```
Node 1
├── Pod
├── Pod
└── Fluent Bit
       ↓

Node 2
├── Pod
├── Pod
└── Fluent Bit
       ↓

Node 3
├── Pod
├── Pod
└── Fluent Bit
```

然后：

```
Fluent Bit
     ↓
Log Backend
```

------

### 21.7.1 Fluent Bit 做什么

基本职责：

```
读取
 ↓
解析
 ↓
过滤
 ↓
丰富
 ↓
转发
```

例如：

```
/var/log/containers/*.log
```

读取后：

```
Fluent Bit
    ↓
解析 Kubernetes Metadata
    ↓
Namespace
Pod
Container
Node
    ↓
发送到日志平台
```

于是原本：

```
ERROR database connection failed
```

可以附带：

```
namespace=shop
pod=shop-api-xxx
container=api
node=node-01
```

查询时就非常方便。

------

### 21.7.2 Fluent Bit 为什么通常使用 DaemonSet

因为日志产生在：

```
Node
```

所以最自然的方式：

```
Node
 ↓
Fluent Bit
 ↓
读取本机日志
```

使用 DaemonSet 就可以确保：

```
每个符合条件的 Node
```

运行一个 Agent。

这正是第 7 章 DaemonSet 知识在生产中的典型应用。

------

## 21.8 Fluentd

Fluentd 也是非常成熟的日志收集系统。

与 Fluent Bit 相比，可以粗略理解为：

```
Fluent Bit
    ↓
轻量级
资源占用较低
适合 Agent

Fluentd
    ↓
功能丰富
插件生态成熟
更适合复杂日志处理
```

一种常见架构：

```
Node
 ↓
Fluent Bit
 ↓
Fluentd
 ↓
Elasticsearch
```

也就是：

```
Fluent Bit
```

负责：

```
Node-level Collection
```

而：

```
Fluentd
```

负责：

```
Aggregation
Transformation
Routing
```

不过现在并不是所有环境都需要两层。

如果：

```
Fluent Bit
    ↓
Loki / Elasticsearch
```

已经能够满足需求，就没有必要为了架构复杂而增加 Fluentd。

生产设计应该遵循：

> **满足需求的最简单架构通常更容易维护。**

------

## 21.9 Loki

Loki 是 Grafana 生态中的日志系统。

它的一个重要设计思想是：

> **日志正文不需要像传统全文搜索系统那样对所有内容建立高成本索引。**

Loki 更强调：

```
Labels
```

例如：

```
namespace=shop
pod=shop-api-xxx
container=api
```

然后通过标签定位日志，再查询日志内容。

典型架构：

```
Kubernetes
     │
     ▼
Fluent Bit / Promtail
     │
     ▼
    Loki
     │
     ▼
   Grafana
```

------

### 21.9.1 Loki 的优势

对于 Kubernetes 环境，它非常适合：

```
Kubernetes Metadata
+
Grafana
```

例如在 Grafana 中：

```
namespace="shop"
```

进一步：

```
app="shop-api"
```

再查询：

```
"database connection failed"
```

非常适合：

```
应用日志查询
Pod 日志查询
Namespace 日志查询
```

------

### 21.9.2 Loki 的一个重要认知

不要把 Loki 理解成：

```
“另一个 Elasticsearch”
```

它们的索引和存储设计理念不同。

选择日志平台时应该考虑：

```
日志量
查询方式
全文搜索需求
成本
运维复杂度
团队熟悉程度
Grafana 生态
```

而不是简单比较：

```
哪个更高级
```

------

## 21.10 Elasticsearch

Elasticsearch 是非常成熟的分布式搜索和分析引擎。

经典 Kubernetes 日志架构：

```
Pod
 ↓
Container stdout
 ↓
Node
 ↓
Fluent Bit / Fluentd
 ↓
Elasticsearch
 ↓
Kibana
```

Elasticsearch 适合：

```
全文搜索
复杂查询
结构化日志
聚合分析
大规模日志检索
```

例如查询：

```
status >= 500
```

或者：

```
service = shop-api
AND
response_time > 1000
```

或者：

```
trace_id = xxx
```

------

### 21.10.1 Elasticsearch 的成本

Elasticsearch 功能强，但运维成本也比较高。

需要关注：

```
CPU
Memory
Disk
Shard
Replica
Index
Retention
Snapshot
Cluster Health
```

尤其是日志系统。

如果每天：

```
500 GB
```

日志进入 Elasticsearch：

```
500 GB/day
```

一个月理论上就是：

```
约 15 TB
```

还没计算：

```
副本
索引开销
Segment
Metadata
```

因此生产环境必须设计：

```
Retention
```

例如：

```
热数据
7 天

温数据
30 天

归档
90 天
```

具体周期取决于业务和合规要求。

------

## 21.11 Kibana

Kibana 是 Elasticsearch 生态中的可视化和查询平台。

架构：

```
Fluent Bit
     ↓
Elasticsearch
     ↓
   Kibana
```

Kibana 可以：

```
查询日志
过滤日志
创建 Dashboard
创建图表
分析错误
```

例如：

```
namespace = shop
```

再：

```
container = api
```

再：

```
level = ERROR
```

最终定位：

```
2026-08-31 09:20:01
OrderService
Database timeout
```

------

### 21.11.1 Kibana 与 Elasticsearch 的关系

可以理解成：

```
Elasticsearch
    ↓
负责存储 / 搜索

Kibana
    ↓
负责查询 / 可视化
```

两者职责不同。

类似：

```
Database
    +
Management UI
```

但 Kibana 本身不是日志存储系统。

------

## 21.12 日志采集架构

现在把前面的内容完整组合起来。

最简单的生产架构：

```
                 Kubernetes Cluster

┌──────────────────────────────────────────────┐
│                                              │
│ Node 1                                       │
│                                              │
│ ┌────────┐  ┌────────┐                      │
│ │ Pod A  │  │ Pod B  │                      │
│ └───┬────┘  └───┬────┘                      │
│     │            │                           │
│     └──────┬─────┘                           │
│            ▼                                 │
│       Container Logs                         │
│            │                                 │
│            ▼                                 │
│       Fluent Bit                            │
│            │                                 │
└────────────┼─────────────────────────────────┘
             │
             ▼
        Log Backend
             │
       ┌─────┴─────┐
       ▼           ▼
     Loki      Elasticsearch
       │           │
       ▼           ▼
   Grafana      Kibana
```

------

### 21.12.1 DaemonSet 采集模式

如果集群有：

```
Node 1
Node 2
Node 3
Node 4
```

Fluent Bit：

```
Node 1 → Fluent Bit
Node 2 → Fluent Bit
Node 3 → Fluent Bit
Node 4 → Fluent Bit
```

因此：

```
4 Nodes
=
4 Fluent Bit Pods
```

通常通过：

```
kind: DaemonSet
```

实现。

------

### 21.12.2 日志元数据

生产日志最好不是单纯：

```
2026-08-31 ERROR failed
```

而是包含：

```
timestamp
namespace
pod
container
node
application
level
message
```

例如：

```
{
  "timestamp": "2026-08-31T09:20:01Z",
  "level": "ERROR",
  "service": "shop-api",
  "namespace": "shop",
  "pod": "shop-api-7d9f8d6f8b-x2abc",
  "container": "api",
  "message": "Database connection failed"
}
```

这样日志系统才能真正发挥作用。

------

### 21.12.3 JSON 日志

生产环境推荐应用输出结构化 JSON 日志。

例如：

```
{
  "timestamp": "2026-08-31T09:20:01.123Z",
  "level": "Information",
  "service": "shop-api",
  "traceId": "abc123",
  "message": "Order created",
  "orderId": 10001
}
```

相比：

```
2026-08-31 09:20:01 INFO Order 10001 created
```

JSON 更适合机器处理：

```
Collector
 ↓
Parser
 ↓
Storage
 ↓
Query
```

例如可以直接查询：

```
level = ERROR
```

而不是从一大段字符串中通过正则表达式猜测日志级别。

------

## 21.13 生产环境日志方案

生产环境首先要明确：

> **日志不是越多越好，而是必须“可查询、可关联、可控制成本”。**

推荐基本架构：

```
Application
    ↓
stdout / stderr
    ↓
Container Runtime
    ↓
Node
    ↓
DaemonSet Log Agent
    ↓
Central Log System
    ↓
Visualization
```

例如：

```
.NET / Node.js / Java
        │
        ▼
   JSON stdout
        │
        ▼
    containerd
        │
        ▼
 /var/log/containers
        │
        ▼
   Fluent Bit
        │
        ▼
      Loki
        │
        ▼
    Grafana
```

这是一个相对简单的方案。

另一种：

```
Application
     ↓
stdout
     ↓
Fluent Bit
     ↓
Elasticsearch
     ↓
Kibana
```

适合对搜索和分析能力要求较高的环境。

------

### 21.13.1 生产环境不要让应用直接写 Node 路径

不推荐：

```
Application
    ↓
/var/log/myapp/app.log
```

然后让业务应用自己管理：

```
权限
日志目录
轮转
文件大小
磁盘
```

更推荐：

```
Application
    ↓
stdout / stderr
    ↓
Kubernetes Logging Pipeline
```

让基础设施统一处理。

------

### 21.13.2 日志必须考虑生命周期

假设：

```
Pod A
```

运行在：

```
Node 01
```

然后发生：

```
Pod 删除
```

或者：

```
Node 故障
```

如果日志只在：

```
Node 01
```

那么历史日志可能无法继续访问。

因此：

```
Pod Log
   ↓
Node
   ↓
Collector
   ↓
Central Storage
```

中央日志系统才是生产环境的长期查询入口。

------

### 21.13.3 日志保留策略

生产环境必须定义：

```
Retention Policy
```

例如：

```
实时日志
     ↓
Loki / Elasticsearch
     ↓
保留 7~30 天
     ↓
删除 / 归档
```

而不是：

```
永久保存所有日志
```

因为日志成本会随着时间持续增长。

------

### 21.13.4 日志级别

应用一般需要：

```
TRACE
DEBUG
INFO
WARN
ERROR
```

生产环境通常不会长期打开大量：

```
DEBUG
TRACE
```

因为可能产生巨大日志量。

通常：

```
Production
    ↓
INFO
WARN
ERROR
```

出现问题时，根据实际情况临时提高日志级别。

------

### 21.13.5 不要把敏感信息写入日志

这一点非常重要。

禁止：

```
password
access token
refresh token
credit card
secret key
```

直接写入日志。

错误：

```
{
  "username": "andy",
  "password": "123456"
}
```

正确：

```
{
  "username": "andy",
  "login": "success"
}
```

即使 Kubernetes Secret 做得很好：

```
Secret
```

如果应用把 Secret 写进：

```
stdout
```

最终它仍然可能进入：

```
Loki
Elasticsearch
Kibana
备份
```

所以：

> **Secret 的安全不仅是 Kubernetes 配置问题，也是应用日志设计问题。**

------

### 21.13.6 日志与 Trace ID

生产排障时，仅有：

```
Pod
Container
时间
```

还不够。

更好的方式是：

```
Request
    ↓
Trace ID
    ↓
API
    ↓
Redis
    ↓
PostgreSQL
```

例如：

```
traceId=abc123
```

API 日志：

```
{
  "traceId": "abc123",
  "service": "shop-api",
  "message": "Create order"
}
```

数据库相关日志：

```
{
  "traceId": "abc123",
  "service": "shop-api",
  "message": "Database timeout"
}
```

这样查询：

```
traceId = abc123
```

就可以把一次请求相关的日志串起来。

这也是生产环境日志系统从：

```
“看日志”
```

走向：

```
“追踪一次请求”
```

的重要一步。

------

## 常见日志故障排查

### Pod 显示 CrashLoopBackOff

先看：

```
kubectl get pod -n shop
```

然后：

```
kubectl logs \
  -n shop \
  POD_NAME
```

如果当前 Container 已经重启：

```
kubectl logs \
  -n shop \
  POD_NAME \
  --previous
```

再看：

```
kubectl describe pod \
  -n shop \
  POD_NAME
```

重点关注：

```
Last State
Reason
Exit Code
Events
```

------

### Pod 没有日志

先检查：

```
kubectl logs POD_NAME -n shop
```

如果为空，可能是：

```
应用没有输出 stdout/stderr
```

或者：

```
应用把日志写到了文件
```

或者：

```
Container 根本没有启动
```

可以检查：

```
kubectl describe pod POD_NAME -n shop
```

------

### 日志突然消失

重点检查：

```
Pod 是否被删除
Node 是否故障
Container 是否重启
日志轮转
Collector 是否正常
Central Storage 是否正常
```

不要只检查：

```
kubectl logs
```

因为：

```
kubectl logs
```

主要适合当前 Kubernetes 资源的实时/近期排查，不是生产历史日志平台。

------

### Node 磁盘满了

检查：

```
df -h
```

查看：

```
du -sh /var/log/*
```

重点关注：

```
Container Logs
System Logs
Journal
```

Kubernetes：

```
kubectl describe node NODE_NAME
```

查看是否出现：

```
DiskPressure
```

------

### Fluent Bit 不采集日志

先查看：

```
kubectl get daemonset -A
```

然后：

```
kubectl get pod -A | grep -i fluent
```

查看 Fluent Bit 日志：

```
kubectl logs \
  -n logging \
  POD_NAME
```

重点排查：

```
日志路径
权限
Parser
Kubernetes API 权限
网络
Backend 地址
认证
```

------

### Elasticsearch 磁盘越来越大

首先不要简单地：

```
加磁盘
```

应该先分析：

```
每天产生多少日志？
哪些 Namespace 产生最多？
哪些应用日志量异常？
Retention 是否配置？
Index 是否正确 rollover？
是否保存了大量 DEBUG？
```

生产环境应该建立：

```
日志量监控
+
存储容量监控
+
Retention
+
告警
```

------

## 本章核心知识总结

Kubernetes 日志最核心的认识是：

```
Application
     │
     │ stdout / stderr
     ▼
Container
     │
     ▼
Container Runtime
     │
     ▼
Node
     │
     ▼
Log Collector
     │
     ▼
Central Log Storage
     │
     ▼
Search / Visualization
```

常见组件对应关系：

| 组件              | 主要职责                       |
| ----------------- | ------------------------------ |
| `kubectl logs`    | 临时查看 Pod/Container 日志    |
| Container Runtime | 管理 Container 日志            |
| Fluent Bit        | 轻量日志采集 Agent             |
| Fluentd           | 日志处理、聚合、转发           |
| Loki              | 日志存储与查询                 |
| Elasticsearch     | 搜索、分析、存储日志           |
| Kibana            | Elasticsearch 日志查询与可视化 |
| Grafana           | Loki 等数据源的可视化          |

生产环境最重要的原则可以归纳成：

```
① 应用输出 stdout / stderr

② 使用 DaemonSet 在 Node 上采集

③ 日志发送到集中式存储

④ 日志使用结构化 JSON

⑤ 添加 Kubernetes Metadata

⑥ 控制日志级别

⑦ 配置日志轮转

⑧ 设置日志 Retention

⑨ 禁止记录密码、Token 等敏感信息

⑩ 使用 Trace ID / Request ID 关联请求
```

最终不要把生产日志体系理解成：

```
kubectl logs
```

而应该理解成：

```
                  Kubernetes
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
     Node 1         Node 2         Node 3
       │              │              │
   Fluent Bit     Fluent Bit     Fluent Bit
       │              │              │
       └──────────────┼──────────────┘
                      ▼
              Central Log System
                 │           │
                 ▼           ▼
               Loki    Elasticsearch
                 │           │
                 ▼           ▼
             Grafana       Kibana
```

这才是 Kubernetes 生产环境真正可运维的日志体系。

# 第 22 章：监控与指标

上一章我们学习了**日志系统**。

日志主要回答：

> **“刚才发生了什么？”**

而监控指标主要回答：

> **“系统现在怎么样？是不是正在变差？什么时候开始异常？”**

在生产环境中，日志和 Metrics 通常需要配合：

```
Metrics
  ↓
发现异常
  ↓
Alert
  ↓
定位问题
  ↓
Logs
  ↓
找到具体错误
```

例如：

```
API 延迟突然升高
        ↓
Prometheus 发现 P99 延迟 > 1s
        ↓
Alertmanager 发送告警
        ↓
运维人员查看 Grafana
        ↓
发现某个 Pod CPU 很高
        ↓
kubectl / 日志进一步排查
```

本章重点建立 Kubernetes 生产环境监控体系的完整认知。

------

## 22.1 Metrics

### 22.1.1 Metrics 是什么

Metrics 就是**指标数据**。

例如：

```
CPU 使用率
Memory 使用量
Pod 数量
HTTP 请求数量
HTTP 请求延迟
错误数量
磁盘空间
网络流量
```

与日志相比，Metrics 通常是结构化的数值。

例如：

```
CPU = 72%
Memory = 1.2 GB
Request Rate = 850 req/s
Error Rate = 0.5%
P99 Latency = 320 ms
```

------

### 22.1.2 为什么需要 Metrics

假设一个 API 服务：

```
shop-api
```

突然变慢。

如果只有日志：

```
ERROR timeout
ERROR timeout
ERROR timeout
```

你知道出现了错误，但不一定知道：

```
什么时候开始？
影响多少请求？
影响多少 Pod？
错误率是多少？
趋势是什么？
```

Metrics 可以告诉你：

```
10:00  100 req/s
10:10  120 req/s
10:20  500 req/s
10:30  1000 req/s
```

同时：

```
P99 Latency
100ms
120ms
300ms
1500ms
```

于是可以明显看出：

```
流量 ↑
延迟 ↑
```

这就是 Metrics 的价值。

------

### 22.1.3 Metrics 的基本形式

Prometheus 体系中的指标通常类似：

```
http_requests_total{method="GET",status="200"} 12345
```

或者：

```
node_memory_MemAvailable_bytes 8589934592
```

它通常包含：

```
Metric Name
+
Labels
+
Value
+
Timestamp
```

例如：

```
http_requests_total
```

是指标名称。

```
method="GET"
status="200"
```

是标签。

```
12345
```

是当前值。

------

### 22.1.4 Counter、Gauge 等基本概念

生产环境使用 Prometheus 时，经常会看到不同类型的指标。

**Counter**

只增不减，例如：

```
HTTP 请求总数
错误总数
订单创建总数
```

例如：

```
http_requests_total = 10000
```

下一次：

```
http_requests_total = 10001
```

如果应用重启，Counter 通常会重新开始计数。

------

**Gauge**

可以上升，也可以下降：

```
CPU 使用率
Memory 使用量
当前连接数
队列长度
```

例如：

```
memory_usage = 2GB
```

之后：

```
memory_usage = 1.5GB
```

这是正常的。

------

## 22.2 Metrics Server

Metrics Server 是 Kubernetes 中非常重要的组件，但初学者很容易误解：

> **Metrics Server 不是完整的生产监控系统。**

它主要提供 Kubernetes 资源的实时资源使用指标，例如：

```
Pod CPU
Pod Memory
Node CPU
Node Memory
```

这些数据主要用于：

```
kubectl top
```

以及：

```
HPA
```

------

### 22.2.1 查看 Node 资源

安装并正常运行 Metrics Server 后：

```
kubectl top nodes
```

可能看到：

```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-01    350m         17%    2048Mi          26%
node-02    820m         41%    4096Mi          52%
```

------

### 22.2.2 查看 Pod 资源

```
kubectl top pods -A
```

例如：

```
NAMESPACE   NAME                         CPU   MEMORY
shop        shop-api-xxx                120m  350Mi
shop        shop-worker-xxx             500m  1Gi
redis       redis-0                     80m   200Mi
```

指定 Namespace：

```
kubectl top pods -n shop
```

------

### 22.2.3 Metrics Server 的工作原理

可以简化理解为：

```
Node
  │
  ├── kubelet
  │
  ▼
Metrics Server
  │
  ▼
Kubernetes Metrics API
  │
  ├── kubectl top
  │
  └── HPA
```

Metrics Server 从 Kubernetes Node 上的 kubelet 等来源获取资源使用数据，然后通过 Metrics API 提供给 Kubernetes。

------

### 22.2.4 Metrics Server 不负责什么

它不是：

```
❌ 长期指标存储
❌ 完整 Dashboard
❌ 强大的历史数据查询
❌ 完整告警系统
```

例如：

> “过去 30 天 shop-api 的 CPU 趋势是什么？”

Metrics Server 并不是为这种需求设计的。

这时候通常需要：

```
Prometheus
```

------

## 22.3 Prometheus

Prometheus 是 Kubernetes 生产环境最核心的 Metrics 系统之一。

可以把它理解成：

> **负责收集、存储和查询 Metrics 的监控系统。**

典型架构：

```
Application
     │
     ▼
  Metrics
     │
     ▼
 Prometheus
     │
     ├── Storage
     │
     └── PromQL
            │
            ▼
         Grafana
```

------

### 22.3.1 Prometheus 为什么需要

Metrics Server：

```
kubectl top
HPA
```

主要解决：

```
当前资源使用情况
```

Prometheus 则可以进一步解决：

```
历史数据
趋势分析
复杂查询
监控规则
告警
应用指标
```

例如：

```
过去 7 天 CPU 使用趋势
过去 1 小时 HTTP 5xx 比例
过去 30 分钟 P99 延迟
```

------

### 22.3.2 Prometheus 的核心工作方式

Prometheus 最经典的模型是：

```
Prometheus
    │
    │ HTTP
    ▼
/metrics
```

也就是说 Prometheus 会主动访问目标的 Metrics Endpoint。

例如：

```
http://shop-api:8080/metrics
```

返回：

```
http_requests_total 12345
http_request_duration_seconds 0.32
```

Prometheus 定期抓取：

```
Scrape
```

于是：

```
Application
     │
     │ /metrics
     ▼
Prometheus
     │
     ▼
TSDB
```

------

### 22.3.3 Pull 模型

Prometheus 的经典模型是：

```
Prometheus
     │
     ├── scrape → API
     ├── scrape → Node
     ├── scrape → Kubernetes
     └── scrape → Exporter
```

而不是：

```
Application
     │
     └── push → Prometheus
```

理解这一点非常重要。

------

### 22.3.4 PromQL

Prometheus 使用：

```
PromQL
```

查询 Metrics。

例如：

```
up
```

表示查询目标是否正常。

CPU 使用率示例：

```
rate(container_cpu_usage_seconds_total[5m])
```

HTTP 请求速率：

```
rate(http_requests_total[5m])
```

错误比例：

```
rate(http_requests_total{status=~"5.."}[5m])
```

PromQL 是 Prometheus 非常核心的能力。

------

## 22.4 Grafana

Prometheus 负责：

```
采集
存储
查询
```

但直接使用 PromQL 对初学者并不直观。

Grafana 主要负责：

```
Dashboard
Visualization
```

架构：

```
Prometheus
     │
     ▼
  Grafana
     │
     ├── CPU Dashboard
     ├── Memory Dashboard
     ├── Kubernetes Dashboard
     └── Application Dashboard
```

------

### 22.4.1 Grafana 能做什么

例如一个生产 API Dashboard：

```
┌────────────────────────────────────┐
│ Shop API                           │
├──────────────┬─────────────────────┤
│ QPS          │ 850 req/s           │
│ Error Rate   │ 0.3%                │
│ P99          │ 280ms               │
│ Pods         │ 8                   │
├──────────────┴─────────────────────┤
│ CPU Usage                           │
│ ███████████                         │
│                                     │
├─────────────────────────────────────┤
│ Memory Usage                        │
│ ███████████████                     │
└─────────────────────────────────────┘
```

这样运维人员可以快速了解系统状态。

------

### 22.4.2 Grafana 不是 Metrics 存储

这是一个重要概念：

```
Prometheus
    ↓
存储 / 查询 Metrics

Grafana
    ↓
展示 Metrics
```

所以：

> Grafana 本身不是 Prometheus 的替代品。

------

## 22.5 kube-state-metrics

`kube-state-metrics` 是 Kubernetes 监控中非常重要的组件。

它与 Metrics Server 的定位完全不同。

它主要关注：

> **Kubernetes 对象的状态。**

例如：

```
Deployment
Pod
ReplicaSet
StatefulSet
DaemonSet
Job
Node
Service
```

------

### 22.5.1 为什么需要 kube-state-metrics

Metrics Server 告诉你：

```
Pod CPU = 200m
Pod Memory = 500Mi
```

但生产运维还需要知道：

```
Deployment 想要几个 Pod？
实际运行几个？
几个 Ready？
几个不可用？
Pod 当前是什么状态？
Node 是否 Ready？
```

这些属于：

```
Kubernetes Object State
```

`kube-state-metrics` 就是解决这个问题的。

------

### 22.5.2 举例

Deployment：

```
spec:
  replicas: 5
```

理论上：

```
Desired = 5
```

实际：

```
Available = 3
```

那么：

```
5 desired
3 available
```

这就是一个非常值得关注的生产指标。

------

### 22.5.3 它与 Metrics Server 的区别

| 项目                | Metrics Server     | kube-state-metrics  |
| ------------------- | ------------------ | ------------------- |
| CPU                 | ✅                  | ❌                   |
| Memory              | ✅                  | ❌                   |
| Pod 当前资源使用    | ✅                  | ❌                   |
| Deployment 副本状态 | ❌                  | ✅                   |
| Pod Phase           | ❌                  | ✅                   |
| Node 状态           | 部分资源指标       | Kubernetes 对象状态 |
| HPA                 | ✅                  | ❌                   |
| Prometheus 采集     | 可通过 Metrics API | 直接暴露 Metrics    |

简单记：

```
Metrics Server
    ↓
“用了多少资源？”

kube-state-metrics
    ↓
“Kubernetes 对象现在是什么状态？”
```

------

## 22.6 Node Exporter

Node Exporter 主要用于：

> **暴露 Linux Node 的系统级指标。**

例如：

```
CPU
Memory
Disk
Filesystem
Network
Load
```

典型架构：

```
Node
 │
 ├── Linux Kernel
 ├── CPU
 ├── Memory
 ├── Disk
 └── Network
       │
       ▼
Node Exporter
       │
       ▼
Prometheus
```

------

### 22.6.1 为什么使用 Node Exporter

Metrics Server 能告诉你：

```
Node CPU 使用情况
Node Memory 使用情况
```

但生产环境往往需要更详细的 OS 指标。

例如：

```
filesystem usage
disk IOPS
network packets
load average
inode usage
```

Node Exporter 可以提供大量这类指标。

------

### 22.6.2 Node Exporter 通常怎么部署

与 Fluent Bit 类似：

```
DaemonSet
```

例如：

```
Node 1 → Node Exporter
Node 2 → Node Exporter
Node 3 → Node Exporter
```

原因很简单：

> 每个 Node 都需要监控。

------

## 22.7 Pod 指标

生产环境需要关注 Pod 的资源和状态。

典型指标包括：

```
CPU
Memory
Network Receive
Network Transmit
Restart Count
Container Status
```

例如：

```
shop-api
├── CPU: 500m
├── Memory: 800Mi
├── Network RX: 10MB/s
├── Network TX: 5MB/s
└── Restarts: 3
```

------

### 22.7.1 Pod CPU

例如 Prometheus 中常见的 Container CPU 指标：

```
container_cpu_usage_seconds_total
```

这是 Counter 类型。

通常需要使用：

```
rate(...)
```

转换成某段时间内的使用速率。

例如：

```
rate(container_cpu_usage_seconds_total[5m])
```

------

### 22.7.2 Pod Memory

常见指标：

```
container_memory_working_set_bytes
```

可以用于观察 Container 的内存工作集。

例如：

```
container_memory_working_set_bytes
```

生产环境可以进一步结合：

```
namespace
pod
container
```

进行过滤。

------

### 22.7.3 Pod Restart

Pod 重启次数是非常重要的健康指标。

例如：

```
restart_count = 10
```

可能意味着：

```
应用崩溃
OOMKilled
Probe 失败
配置错误
依赖服务异常
```

因此生产 Dashboard 通常需要关注 Restart。

------

## 22.8 Node 指标

Node 是 Kubernetes 集群的基础资源。

如果 Node 出问题：

```
Node
 ↓
多个 Pod
 ↓
多个业务
```

都可能受到影响。

生产环境至少需要监控：

```
CPU
Memory
Disk
Network
Filesystem
Node Ready
```

------

### 22.8.1 Node CPU

例如：

```
CPU Usage = 85%
```

短时间 85% 不一定有问题。

真正应该关注：

```
持续时间
业务负载
CPU Request
CPU Limit
Pod 调度情况
```

例如：

```
CPU 90%
持续 30 分钟
```

就值得关注。

------

### 22.8.2 Node Memory

Memory 比 CPU 更敏感。

例如：

```
Memory = 95%
```

可能导致：

```
Memory Pressure
Pod Eviction
OOM
```

需要及时处理。

------

### 22.8.3 Node Disk

生产环境非常容易忽略 Disk。

需要监控：

```
Disk Usage
Inode Usage
Disk I/O
```

尤其是：

```
/var/lib/containerd
/var/log
```

等目录所在的文件系统。

磁盘满了可能直接影响：

```
Image Pull
Container Runtime
Pod 创建
日志写入
Kubelet
```

------

## 22.9 Application Metrics

这是生产监控最重要的一部分之一。

Kubernetes 自己的监控只能告诉你：

```
Pod CPU
Pod Memory
Node CPU
Node Memory
```

但它无法直接回答：

```
API QPS 是多少？
订单创建成功率是多少？
支付失败率是多少？
数据库连接池是否耗尽？
Redis 命中率是多少？
```

这些是：

```
Application Metrics
```

------

### 22.9.1 .NET Application Metrics

例如 ASP.NET Core 可以使用：

```
OpenTelemetry
```

或其他 Metrics instrumentation。

应用最终提供类似：

```
/metrics
```

的数据。

例如：

```
http_server_request_duration
http_server_request_count
```

然后：

```
Prometheus
     ↓
scrape
     ↓
.NET API
```

------

### 22.9.2 Node.js

Node.js 应用也可以通过：

```
OpenTelemetry
```

等方式暴露 Metrics。

例如：

```
HTTP Request Count
HTTP Duration
Event Loop
Process Memory
```

------

### 22.9.3 业务指标

真正有价值的生产监控往往不仅是：

```
CPU
Memory
```

还包括业务指标。

例如电商：

```
orders_created_total
orders_failed_total
payment_success_total
payment_failed_total
```

可以计算：

```
支付成功率
```

这比单纯看到：

```
CPU = 30%
```

更接近业务真正的健康状态。

------

## 22.10 CPU / Memory / Network / Disk

生产监控至少应该覆盖四大类资源。

------

### 22.10.1 CPU

关注：

```
Node CPU
Pod CPU
Container CPU
CPU Throttling
Load
```

特别需要注意：

> CPU 使用率高不一定代表故障。

例如：

```
CPU = 90%
```

但：

```
Latency 正常
Error Rate 正常
QPS 正常
```

可能只是正常高负载。

------

### 22.10.2 Memory

关注：

```
Node Memory
Pod Memory
Container Memory
OOMKilled
Memory Pressure
```

例如：

```
Memory
 ↓
95%
 ↓
OOM
 ↓
Container Restart
```

这就是典型生产故障链。

------

### 22.10.3 Network

关注：

```
Network RX
Network TX
Packet Errors
Packet Drops
Bandwidth
```

例如某个 API：

```
Network RX
突然从 100MB/s
变成 1GB/s
```

可能意味着：

```
流量暴增
异常请求
数据传输异常
网络配置问题
```

------

### 22.10.4 Disk

关注：

```
Disk Usage
Disk I/O
IOPS
Latency
Inode
```

例如：

```
Disk = 98%
```

这已经是非常危险的状态。

生产环境一般需要提前告警，而不是等：

```
100%
```

才处理。

------

## 22.11 告警

监控系统真正有价值的地方之一，就是：

> **发现问题后主动通知人。**

否则：

```
Prometheus
    ↓
发现 CPU 99%
```

但没人看 Dashboard。

结果：

```
生产已经故障
运维却不知道
```

------

### 22.11.1 什么是 Alert

例如：

```
CPU > 90%
持续 10 分钟
```

定义成：

```
Alert
```

或者：

```
Pod Restart > 5
```

或者：

```
Error Rate > 5%
```

------

### 22.11.2 告警应该关注什么

生产环境不应该只监控：

```
CPU > 80%
```

更应该关注：

```
服务不可用
错误率
延迟
容量
资源耗尽
副本不足
```

例如 API：

```
Availability < 99.9%
Error Rate > 2%
P99 > 1s
```

这些通常比：

```
CPU > 80%
```

更加接近用户体验。

------

### 22.11.3 告警级别

可以设计：

```
Info
Warning
Critical
```

例如：

```
Warning
Node Disk > 80%

Critical
Node Disk > 95%
```

或者：

```
Warning
Error Rate > 1%

Critical
Error Rate > 5%
```

具体阈值需要根据业务实际情况确定。

------

### 22.11.4 避免告警风暴

这是生产监控非常重要的问题。

例如一个 Node 挂掉：

```
Node Down
   ↓
50 Pods Down
   ↓
50 Alerts
```

如果继续：

```
Service Down
Deployment Replica Low
Pod Not Ready
HTTP Error
```

最终可能收到：

```
500 条告警
```

但真正的根因只有：

```
Node Down
```

所以生产环境需要：

```
Grouping
Deduplication
Inhibition
Routing
```

将大量相关告警聚合起来。

------

## 22.12 Alertmanager

Prometheus 负责：

```
发现指标异常
```

Alertmanager 负责：

```
告警处理
```

架构：

```
Prometheus
    │
    │ Alert
    ▼
Alertmanager
    │
    ├── Grouping
    ├── Deduplication
    ├── Inhibition
    └── Routing
           │
           ├── Email
           ├── Slack
           ├── Webhook
           └── PagerDuty 等
```

------

### 22.12.1 Prometheus 与 Alertmanager 的职责

可以简单理解：

```
Prometheus
    ↓
“出问题了！”

Alertmanager
    ↓
“应该通知谁？怎么通知？哪些告警合并？”
```

------

### 22.12.2 一个简单的告警规则

例如：

```
groups:
  - name: node
    rules:
      - alert: NodeHighCPU
        expr: |
          100 - (
            avg by(instance) (
              rate(node_cpu_seconds_total{
                mode="idle"
              }[5m])
            ) * 100
          ) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node CPU usage is high"
          description: "Node CPU usage has been above 90% for 10 minutes."
```

这里重点理解：

```
expr
```

判断条件。

```
for: 10m
```

不是瞬间超过 90% 就报警，而是持续 10 分钟。

```
labels
```

用于分类。

```
annotations
```

用于描述告警信息。

------

### 22.12.3 为什么需要 `for`

假设：

```
CPU
80%
85%
92%
88%
```

如果：

```
CPU > 90%
```

就立刻告警：

```
Alert
Alert
Alert
```

容易产生大量误报。

设置：

```
for: 10m
```

意味着：

```
CPU > 90%
持续 10 分钟
```

才触发。

------

## 22.13 生产监控体系

现在把整个 Kubernetes 生产监控体系组合起来。

一个比较典型的架构：

```
                         Kubernetes Cluster
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
             ▼                    ▼                    ▼
        Kubernetes             Node               Application
          Metrics             Metrics               Metrics
             │                    │                    │
             │              Node Exporter              │
             │                    │                    │
             └──────────────┬─────┴────────────────────┘
                            ▼
                       Prometheus
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
             PromQL                 Alert Rules
                 │                     │
                 ▼                     ▼
              Grafana             Alertmanager
                 │                     │
                 │              ┌──────┴──────┐
                 │              ▼             ▼
                 │           Email          Webhook
                 │
                 ▼
             Dashboard
```

------

### 22.13.1 Kubernetes 层

监控：

```
Pod
Deployment
StatefulSet
DaemonSet
Job
Node
```

核心问题：

```
Pod 是否正常？
副本是否足够？
Node 是否 Ready？
Deployment 是否 Available？
```

这里：

```
kube-state-metrics
```

非常重要。

------

### 22.13.2 Infrastructure 层

监控：

```
CPU
Memory
Disk
Network
Filesystem
```

这里：

```
Node Exporter
```

非常重要。

------

### 22.13.3 Application 层

监控：

```
QPS
Latency
Error Rate
Active Connections
Thread Pool
Connection Pool
Business Metrics
```

例如：

```
.NET API
Node.js
Java
```

都应该尽量提供自己的 Application Metrics。

------

### 22.13.4 Business 层

生产环境真正成熟的监控，还需要业务指标：

```
订单量
支付成功率
注册成功率
库存不足
支付失败
任务积压
```

例如：

```
系统 CPU 正常
Memory 正常
Pod 正常
```

但：

```
支付成功率 = 20%
```

这仍然是严重生产事故。

所以：

> **基础设施健康 ≠ 业务健康。**

------

## 生产环境推荐监控层次

可以把生产监控分成四层：

```
┌──────────────────────────────┐
│       Business Metrics       │
│ 订单 / 支付 / 转化 / 成功率   │
└───────────────▲──────────────┘
                │
┌───────────────┴──────────────┐
│      Application Metrics     │
│ QPS / Error / Latency / Pool  │
└───────────────▲──────────────┘
                │
┌───────────────┴──────────────┐
│      Kubernetes Metrics       │
│ Pod / Deployment / Node State │
└───────────────▲──────────────┘
                │
┌───────────────┴──────────────┐
│     Infrastructure Metrics    │
│ CPU / Memory / Disk / Network │
└──────────────────────────────┘
```

这样排查问题时可以从上到下，也可以从下到上。

------

## 常见监控故障排查

### `kubectl top` 没有数据

执行：

```
kubectl top nodes
```

如果：

```
error: Metrics API not available
```

首先检查：

```
kubectl get pods -n kube-system
```

寻找：

```
metrics-server
```

然后：

```
kubectl get apiservice | grep metrics
```

进一步：

```
kubectl describe apiservice v1beta1.metrics.k8s.io
```

再查看 Metrics Server 日志：

```
kubectl logs \
  -n kube-system \
  deployment/metrics-server
```

常见原因包括：

```
Metrics Server 没安装
Pod 没启动
TLS / kubelet 连接问题
网络问题
权限问题
```

------

### Prometheus 没有采集到指标

首先检查 Prometheus Target。

重点关注：

```
UP
DOWN
```

如果 Target 是：

```
DOWN
```

需要检查：

```
Service
Endpoint
ServiceMonitor / PodMonitor
Network
Port
Path
TLS
Authentication
```

例如应用声明：

```
/metrics
```

但 Prometheus 实际请求：

```
/metric
```

自然无法采集。

------

### Grafana 没有数据

不要第一时间认为：

```
Grafana 坏了
```

应该逐层排查：

```
Application
 ↓
Exporter
 ↓
Prometheus
 ↓
PromQL
 ↓
Grafana
```

首先直接进入 Prometheus 查询：

```
up
```

如果 Prometheus 已经有数据：

```
Prometheus ✅
```

再检查：

```
Grafana Data Source
```

以及 Dashboard 查询。

------

### Pod CPU 很高

不要马上：

```
重启 Pod
```

应该依次确认：

```
CPU 是否持续高？
QPS 是否增加？
CPU Limit 是多少？
是否 CPU Throttling？
Pod 是否 HPA 扩容？
应用是否存在死循环？
某个请求是否异常？
```

例如：

```
QPS ↑
CPU ↑
Latency ↑
```

可能是：

```
正常流量增长
```

而：

```
QPS →
CPU ↑↑
Latency ↑↑
```

则更值得怀疑：

```
应用性能问题
```

------

### Node Memory 很高

检查：

```
kubectl top nodes
```

再：

```
kubectl top pods -A
```

找到高内存 Pod。

然后：

```
kubectl describe node NODE_NAME
```

检查：

```
MemoryPressure
```

进一步检查：

```
OOMKilled
Pod Restart
Memory Limit
```

------

### 告警太多

如果每天收到：

```
几百条
甚至几千条
```

通常不是“监控太完善”，而是告警设计存在问题。

需要检查：

```
阈值
for
Grouping
Deduplication
Inhibition
告警级别
告警是否真正可行动
```

一个好的告警应该满足：

> **收到告警后，值班人员知道需要做什么。**

而不是：

```
CPU 81%
```

这种没有明确行动价值的告警。

------

## 本章核心知识总结

这一章最重要的是区分几个组件。

```
Metrics Server
    ↓
Kubernetes 实时资源指标
    ↓
kubectl top / HPA
Prometheus
    ↓
Metrics 采集、存储、查询
Grafana
    ↓
Metrics 可视化
kube-state-metrics
    ↓
Kubernetes Object 状态
Node Exporter
    ↓
Linux Node 系统指标
Alertmanager
    ↓
告警处理、聚合、路由、通知
```

完整关系：

```
                     ┌──────────────────┐
                     │    Kubernetes    │
                     └────────┬─────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   Metrics Server     kube-state-metrics    Node Exporter
          │                   │                   │
          │                   └────────┬──────────┘
          │                            │
          │                            ▼
          │                       Prometheus
          │                            │
          │              ┌─────────────┴─────────────┐
          │              ▼                           ▼
          │           Grafana                    Alert Rules
          │                                          │
          │                                          ▼
          │                                     Alertmanager
          │                                          │
          │                                  Email / Webhook
          │
          ▼
     kubectl top / HPA
```

最终要形成一个生产运维思维：

```
                 业务
                  │
          ┌───────▼───────┐
          │ Application    │
          │ QPS/Error/RT   │
          └───────┬───────┘
                  │
          ┌───────▼───────┐
          │ Kubernetes     │
          │ Pod/Node/Workload
          └───────┬───────┘
                  │
          ┌───────▼───────┐
          │ Infrastructure │
          │ CPU/Mem/Disk   │
          └───────────────┘
```

而完整生产监控体系最终应该做到：

```
发现异常
   ↓
Metrics
   ↓
告警
   ↓
Grafana 判断影响范围
   ↓
Logs 定位具体错误
   ↓
必要时进入 Trace
   ↓
定位根因
```

因此，**Metrics 的核心价值不是“画几个 CPU 图表”，而是建立一套能够及时发现、量化和定位生产系统异常的可观测性基础。**

# 第 23 章：Kubernetes 故障排查

Kubernetes 生产环境运维中，**故障排查能力比记住多少 `kubectl` 命令更重要**。

因为实际生产事故很少直接告诉你：

```
Pod 因为镜像拉取失败导致启动失败
```

更多时候你看到的只是：

```
用户访问失败
HTTP 503
```

然后需要自己一步一步找到：

```
Ingress
  ↓
Service
  ↓
Pod
  ↓
Container
  ↓
Application
  ↓
Database / Redis / External Service
```

所以这一章不只是学习各种错误状态，而是建立一套**固定、可重复的排障方法**。

------

## 23.1 Pod 起不来怎么办

这是 Kubernetes 最常见的问题之一。

看到：

```
kubectl get pods
```

例如：

```
NAME                        READY   STATUS             RESTARTS   AGE
shop-api-7d8f9c7d8-x2abc   0/1     ImagePullBackOff   0          2m
```

不要直接执行：

```
kubectl delete pod
```

应该先判断：

> **Pod 到底卡在哪一个阶段？**

------

### 23.1.1 第一层：查看 Pod 状态

```
kubectl get pods -A
```

指定 Namespace：

```
kubectl get pods -n shop
```

观察：

```
STATUS
```

常见状态：

```
Pending
ContainerCreating
Running
CrashLoopBackOff
ImagePullBackOff
ErrImagePull
Terminating
```

------

### 23.1.2 第二层：describe Pod

```
kubectl describe pod POD_NAME -n NAMESPACE
```

重点看最后：

```
Events:
```

例如：

```
Events:
  Type     Reason     Message
  ----     ------     -------
  Warning  Failed     Failed to pull image
```

通常：

> **Event 是排查 Kubernetes 故障最重要的入口之一。**

------

### 23.1.3 第三层：查看日志

如果 Container 已经启动过：

```
kubectl logs POD_NAME -n NAMESPACE
```

如果有多个 Container：

```
kubectl logs POD_NAME -n NAMESPACE -c CONTAINER_NAME
```

如果 Container 已经崩溃并重启：

```
kubectl logs POD_NAME -n NAMESPACE --previous
```

这一条非常重要。

例如：

```
Current container:
没有日志

Previous container:
数据库连接失败
```

那么真正的问题可能在 `--previous` 中。

------

### 23.1.4 第四层：确认 Pod 属于谁

```
kubectl get pod POD_NAME -n NAMESPACE \
  -o jsonpath='{.metadata.ownerReferences}'
```

更常见的做法是：

```
kubectl get deployment -n NAMESPACE
```

然后：

```
kubectl describe deployment DEPLOYMENT_NAME -n NAMESPACE
```

因为生产环境一般不会直接手工管理 Pod，而是：

```
Deployment
    ↓
ReplicaSet
    ↓
Pod
```

所以：

> Pod 出问题，最终往往需要回到 Deployment 检查。

------

## 23.2 Pod 一直 Pending

例如：

```
kubectl get pods
```

显示：

```
shop-api-xxx   0/1   Pending   0   5m
```

`Pending` 表示：

> Pod 还没有进入正常运行状态。

最常见原因：

```
调度失败
资源不足
节点选择条件不满足
Taint / Toleration
Affinity
PVC
```

------

### 23.2.1 第一件事：describe

```
kubectl describe pod POD_NAME -n NAMESPACE
```

看：

```
Events:
```

例如：

```
0/3 nodes are available:
2 Insufficient cpu,
1 node(s) had untolerated taint
```

这已经直接告诉你原因。

------

### 23.2.2 CPU / Memory 不足

例如：

```
0/3 nodes are available:
3 Insufficient cpu
```

检查：

```
kubectl top nodes
```

再：

```
kubectl describe nodes
```

同时检查 Pod 的 Requests：

```
kubectl get pod POD_NAME -n NAMESPACE \
  -o yaml
```

例如：

```
resources:
  requests:
    cpu: "4"
    memory: "8Gi"
```

如果集群每个 Node 都没有至少 4 CPU 的可调度资源：

```
Pod
  ↓
无法调度
  ↓
Pending
```

------

### 23.2.3 Node Selector / Affinity

例如：

```
nodeSelector:
  disktype: ssd
```

但集群没有：

```
disktype=ssd
```

那么：

```
Pod
 ↓
找不到符合条件的 Node
 ↓
Pending
```

检查 Node Label：

```
kubectl get nodes --show-labels
```

------

### 23.2.4 Taint

如果 Node：

```
NoSchedule
```

而 Pod 没有对应 Toleration：

```
Pod
 ↓
不能调度到该 Node
```

查看：

```
kubectl describe node NODE_NAME
```

寻找：

```
Taints:
```

------

### 23.2.5 PVC 导致 Pending

如果 Pod 依赖 PVC：

```
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
```

检查：

```
kubectl get pvc -n NAMESPACE
```

如果：

```
STATUS
Pending
```

Pod 也可能无法正常启动。

进一步：

```
kubectl describe pvc PVC_NAME -n NAMESPACE
```

------

## 23.3 ImagePullBackOff

这是非常典型的镜像问题。

例如：

```
STATUS
ImagePullBackOff
```

或者：

```
ErrImagePull
```

表示：

> Kubernetes 无法成功拉取 Container Image。

------

### 23.3.1 查看具体错误

```
kubectl describe pod POD_NAME -n NAMESPACE
```

例如：

```
Failed to pull image "registry.example.com/shop/api:v1.2.3":
pull access denied
```

------

### 23.3.2 常见原因

#### 镜像不存在

```
image: nginx:999999
```

检查 Registry 中是否存在对应 Tag。

------

#### 镜像名称错误

例如：

```
registry.example.com/shop/api
```

实际应该：

```
registry.example.com/shop-api
```

------

#### 私有 Registry 没有认证

需要：

```
imagePullSecrets:
  - name: registry-secret
```

检查：

```
kubectl get secret -n NAMESPACE
```

------

#### Registry 网络不可达

Node 需要能够访问：

```
Registry
```

注意：

> **拉镜像的是 Node 上的 Container Runtime，不是你的本地电脑。**

因此：

```
你的电脑 → Registry
```

能访问：

并不代表：

```
Kubernetes Node → Registry
```

能访问。

这是生产环境很容易踩的坑。

------

### 23.3.3 ImagePullBackOff 中的 BackOff

Kubernetes 发现拉取失败后不会无限快速重试。

它会逐渐增加重试间隔：

```
失败
 ↓
等待
 ↓
重试
 ↓
失败
 ↓
等待更久
```

因此看到：

```
ImagePullBackOff
```

不是一个新的根因，而是：

> **镜像拉取失败后 Kubernetes 正在退避重试。**

------

## 23.4 CrashLoopBackOff

例如：

```
shop-api-xxx   0/1   CrashLoopBackOff
```

表示：

> Container 启动后很快退出，Kubernetes 不断尝试重新启动它。

典型过程：

```
Container Start
     ↓
Application 启动
     ↓
Application Crash
     ↓
Container Exit
     ↓
Kubernetes Restart
     ↓
Application Crash
     ↓
CrashLoopBackOff
```

------

### 23.4.1 第一件事

```
kubectl logs POD_NAME -n NAMESPACE
```

如果当前日志不完整：

```
kubectl logs POD_NAME -n NAMESPACE --previous
```

------

### 23.4.2 常见原因

#### 应用启动异常

例如：

```
Connection refused
```

可能是：

```
Database
Redis
External API
```

不可访问。

------

#### 环境变量缺失

例如应用需要：

```
DATABASE_URL
```

但是 Kubernetes 没有注入。

------

#### Secret / ConfigMap 配置错误

例如：

```
ConnectionString
```

错误导致应用启动失败。

------

#### Port 配置错误

应用实际：

```
8080
```

但配置认为：

```
5000
```

可能进一步导致 Probe 失败。

------

#### Startup Probe / Liveness Probe 配置错误

应用实际上启动需要：

```
60 秒
```

但是：

```
initialDelaySeconds: 5
```

然后 Liveness Probe 很快失败：

```
Probe Failed
 ↓
Container Restart
 ↓
CrashLoopBackOff
```

所以看到 CrashLoopBackOff 时：

> 不要只看应用日志，也要检查 Probe。

------

## 23.5 OOMKilled

看到：

```
Reason: OOMKilled
```

意思是：

> Container 因为内存不足被杀掉。

OOM = Out Of Memory。

------

### 23.5.1 查看

```
kubectl describe pod POD_NAME -n NAMESPACE
```

寻找：

```
Last State:
  Terminated:
    Reason: OOMKilled
```

也可以：

```
kubectl get pod POD_NAME -n NAMESPACE -o yaml
```

------

### 23.5.2 常见原因

例如：

```
resources:
  limits:
    memory: 512Mi
```

应用实际需要：

```
700Mi
```

那么：

```
Application
    ↓
Memory ↑
    ↓
512Mi
    ↓
超过 Limit
    ↓
OOMKilled
```

------

### 23.5.3 排查方法

先看：

```
kubectl top pod POD_NAME -n NAMESPACE
```

再看：

```
kubectl describe pod POD_NAME -n NAMESPACE
```

然后检查：

```
memory requests
memory limits
restart count
previous logs
```

如果长期增长：

```
Memory
 │
 │        /
 │      /
 │    /
 │  /
 └────────────
```

要进一步检查应用是否存在：

```
Memory Leak
```

而不是简单地把 Limit 无限调大。

------

## 23.6 Service 无法访问

假设：

```
Pod
正常运行
```

但是：

```
curl http://shop-api
```

失败。

不要立即认为 Service 有问题。

需要逐层检查：

```
Client
 ↓
Service
 ↓
EndpointSlice
 ↓
Pod
 ↓
Container Port
 ↓
Application
```

------

### 23.6.1 Service 是否存在

```
kubectl get svc -n shop
```

查看：

```
kubectl describe svc shop-api -n shop
```

------

### 23.6.2 Selector 是否正确

例如 Service：

```
selector:
  app: shop-api
```

Pod：

```
labels:
  app: shop-api
```

必须匹配。

检查：

```
kubectl get pods -n shop --show-labels
```

------

### 23.6.3 EndpointSlice

这是非常重要的一层。

```
kubectl get endpointslice -n shop
```

也可以：

```
kubectl get endpoints -n shop
```

如果 Service 存在：

```
Service
```

但是没有后端：

```
Endpoints = <none>
```

那么流量自然没有地方发送。

------

### 23.6.4 检查 Pod 是否 Ready

```
kubectl get pods -n shop
```

例如：

```
READY
1/1
```

才表示 Ready。

如果：

```
0/1
```

Service 通常不会把正常业务流量发送给这个 Pod。

进一步：

```
kubectl describe pod POD_NAME -n shop
```

检查 Readiness Probe。

------

## 23.7 Ingress 无法访问

假设：

```
https://api.example.com
```

返回：

```
502
503
404
```

应该按：

```
DNS
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

逐层检查。

------

### 23.7.1 Ingress 是否存在

```
kubectl get ingress -A
```

例如：

```
NAME       HOST              ADDRESS
shop       api.example.com   1.2.3.4
```

------

### 23.7.2 Ingress Controller

例如 Nginx：

```
kubectl get pods -A | grep ingress
```

查看 Controller：

```
kubectl logs \
  -n ingress-nginx \
  deployment/ingress-nginx-controller
```

具体 Namespace 和 Deployment 名称以你的集群实际安装方式为准。

------

### 23.7.3 Service 是否有 Endpoint

```
kubectl get endpointslice -n shop
```

如果：

```
没有后端
```

那么 Ingress Controller 即使正常，也无法访问应用。

------

### 23.7.4 502 与 503

虽然不同 Ingress Controller 的具体行为可能不同，但排查时可以形成经验：

```
502
```

经常需要重点检查：

```
Ingress Controller → Service → Pod
```

例如：

```
连接后端失败
端口错误
协议错误
```

而：

```
503
```

常见方向包括：

```
Service 没有可用 Endpoint
Pod 全部 NotReady
Ingress Controller 找不到可用后端
```

但**不要仅凭 502/503 就直接下结论**，必须结合 Controller 日志和 Endpoint 实际状态。

------

## 23.8 DNS 解析失败

Kubernetes 内部服务通常通过 DNS 访问。

例如：

```
shop-api.shop.svc.cluster.local
```

如果：

```
curl http://shop-api.shop.svc.cluster.local
```

失败，需要检查 DNS。

------

### 23.8.1 先进入测试 Pod

例如：

```
kubectl run dns-test \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- sh
```

然后：

```
nslookup shop-api.shop.svc.cluster.local
```

或者：

```
nslookup kubernetes.default.svc.cluster.local
```

------

### 23.8.2 检查 CoreDNS

```
kubectl get pods -n kube-system
```

找到：

```
coredns-xxx
```

查看：

```
kubectl logs -n kube-system -l k8s-app=kube-dns
```

检查：

```
kubectl get svc -n kube-system kube-dns
```

------

### 23.8.3 DNS 排查路径

```
Pod
 ↓
/etc/resolv.conf
 ↓
CoreDNS Service
 ↓
CoreDNS Pod
 ↓
Service / DNS Record
```

进入 Pod：

```
cat /etc/resolv.conf
```

通常会看到类似：

```
nameserver 10.x.x.x
search shop.svc.cluster.local svc.cluster.local cluster.local
```

如果 DNS Server 本身不可访问，需要继续检查：

```
Pod Network
CoreDNS
Service
NetworkPolicy
CNI
```

------

## 23.9 Pod 网络异常

Pod 网络问题通常比普通应用问题复杂。

典型现象：

```
Pod A
  ↓
无法访问
Pod B
```

或者：

```
Pod
  ↓
无法访问 Service
```

或者：

```
Pod
  ↓
无法访问 Internet
```

------

### 23.9.1 第一层：确认 Pod IP

```
kubectl get pods -o wide
```

例如：

```
NAME        IP            NODE
api-xxx     10.244.1.20   node-01
redis-xxx   10.244.2.30   node-02
```

------

### 23.9.2 Pod → Pod

进入 Pod：

```
kubectl exec -it POD_NAME -n NAMESPACE -- sh
```

测试：

```
curl http://10.244.2.30:6379
```

或者：

```
nc -vz 10.244.2.30 6379
```

具体工具是否存在取决于镜像。

------

### 23.9.3 Pod → Service

测试：

```
curl http://redis:6379
```

然后：

```
nslookup redis
```

如果：

```
DNS 正常
Service 存在
```

但连接失败：

```
检查 EndpointSlice
检查 NetworkPolicy
检查目标 Pod
检查 Service Port
```

------

### 23.9.4 NetworkPolicy

如果集群启用了 NetworkPolicy：

```
Pod A
 ↓
NetworkPolicy
 ↓
Pod B
```

可能被明确禁止。

检查：

```
kubectl get networkpolicy -A
```

以及：

```
kubectl describe networkpolicy NETWORK_POLICY_NAME -n NAMESPACE
```

------

## 23.10 PVC 挂载失败

典型现象：

```
Pod
ContainerCreating
```

然后 Event：

```
FailedMount
```

------

### 23.10.1 查看 PVC

```
kubectl get pvc -n NAMESPACE
```

检查：

```
STATUS
Bound
Pending
```

------

### 23.10.2 查看 PVC

```
kubectl describe pvc PVC_NAME -n NAMESPACE
```

------

### 23.10.3 查看 PV

```
kubectl get pv
```

------

### 23.10.4 查看 StorageClass

```
kubectl get storageclass
```

如果使用动态供应：

```
PVC
 ↓
StorageClass
 ↓
CSI
 ↓
Storage Backend
 ↓
PV
 ↓
Pod
```

任何一层出问题都可能导致挂载失败。

------

### 23.10.5 常见原因

```
PVC Pending
PV 不存在
StorageClass 错误
CSI Driver 异常
云盘挂载失败
Node 无法访问存储
权限问题
Volume 已经被其他 Node 占用
```

第一原则仍然是：

```
kubectl describe pod POD_NAME -n NAMESPACE
```

看 Event。

------

## 23.11 Node NotReady

执行：

```
kubectl get nodes
```

例如：

```
NAME      STATUS
node-01   Ready
node-02   NotReady
```

这意味着：

> Kubernetes Control Plane 当前认为 node-02 不正常。

------

### 23.11.1 查看 Node

```
kubectl describe node node-02
```

重点看：

```
Conditions
```

例如：

```
Ready = False
```

以及：

```
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

------

### 23.11.2 查看 Node 上 kubelet

如果可以登录 Node：

```
systemctl status kubelet
```

查看日志：

```
journalctl -u kubelet -n 200
```

------

### 23.11.3 常见原因

```
kubelet 挂掉
Node 关机
网络断开
磁盘满
Memory Pressure
Container Runtime 异常
证书问题
```

------

### 23.11.4 Node 故障后 Pod 怎么办

如果 Node：

```
NotReady
```

Kubernetes 会根据 Pod 类型和控制器策略进行处理。

例如 Deployment：

```
Node-01
  ↓
Pod A
  ↓
Node 故障
  ↓
Deployment
  ↓
在其他可用 Node 创建替代 Pod
```

但需要注意：

> Pod 是否能够重新调度，还取决于资源、Affinity、Taint、PVC 等条件。

------

## 23.12 Deployment 更新失败

例如执行：

```
kubectl set image deployment/shop-api \
  api=registry.example.com/shop-api:v2 \
  -n shop
```

然后：

```
kubectl rollout status deployment/shop-api -n shop
```

一直不完成。

------

### 23.12.1 查看 Rollout

```
kubectl rollout status deployment/shop-api -n shop
```

查看：

```
kubectl get rs -n shop
```

再：

```
kubectl get pods -n shop
```

------

### 23.12.2 常见情况

例如：

```
旧 Pod
5/5

新 Pod
0/5
```

可能是：

```
ImagePullBackOff
CrashLoopBackOff
Readiness Probe 失败
资源不足
配置错误
```

所以：

> Deployment 更新失败，通常最终还是需要回到 Pod 排查。

------

### 23.12.3 查看 Deployment

```
kubectl describe deployment shop-api -n shop
```

------

### 23.12.4 回滚

如果确定新版本有问题：

```
kubectl rollout history deployment/shop-api -n shop
```

回滚：

```
kubectl rollout undo deployment/shop-api -n shop
```

查看：

```
kubectl rollout status deployment/shop-api -n shop
```

生产环境中：

> 回滚应该是标准操作能力，而不是临时救火手段。

------

## 23.13 应用偶发 502 / 503

这是生产环境非常典型的问题。

关键是：

> **“偶发”比“完全失败”更难排查。**

因为：

```
Pod A → 正常
Pod B → 正常
Pod C → 异常
```

用户请求可能随机进入：

```
A → 200
B → 200
C → 503
```

最终表现就是：

```
偶发 503
```

------

### 23.13.1 第一层：查看 Pod

```
kubectl get pods -n shop -o wide
```

检查：

```
READY
RESTARTS
STATUS
NODE
```

------

### 23.13.2 第二层：查看 EndpointSlice

```
kubectl get endpointslice -n shop
```

确认后端 Pod 是否正确。

------

### 23.13.3 第三层：检查 Readiness

如果 Pod：

```
Running
```

不代表：

```
Ready
```

检查：

```
kubectl describe pod POD_NAME -n shop
```

尤其是：

```
Readiness Probe
```

------

### 23.13.4 第四层：Ingress Controller 日志

例如：

```
kubectl logs \
  -n ingress-nginx \
  deployment/ingress-nginx-controller
```

检查：

```
upstream timeout
connection refused
no live upstreams
```

等信息。

------

### 23.13.5 第五层：应用日志

```
kubectl logs POD_NAME -n shop
```

如果有重启：

```
kubectl logs POD_NAME -n shop --previous
```

------

### 23.13.6 常见根因

```
Pod 部分异常
Readiness Probe 不准确
应用启动/退出过程处理不当
Service Endpoint 变化
Ingress 超时
网络抖动
Node 异常
应用自身偶发错误
下游 Redis / Database 慢
```

------

## 23.14 CPU 飙高

例如：

```
kubectl top pods -A
```

发现：

```
shop-api-xxx   1900m
```

首先不要急着扩容或重启。

------

### 23.14.1 确认是不是持续

短时间：

```
CPU 95%
```

不一定是问题。

应该观察：

```
过去 5 分钟
过去 30 分钟
过去 1 小时
```

Prometheus/Grafana 更适合观察长期趋势。

------

### 23.14.2 对比 QPS

如果：

```
QPS ↑
CPU ↑
```

可能是正常流量增长。

如果：

```
QPS →
CPU ↑↑
```

可能存在：

```
代码性能问题
死循环
CPU 密集型任务
异常请求
锁竞争
```

------

### 23.14.3 检查 CPU Throttling

如果 Container 配置了很低的 CPU Limit：

```
resources:
  limits:
    cpu: "500m"
```

应用可能因为 CPU Limit 受到 throttling。

所以需要同时关注：

```
CPU Usage
CPU Limit
CPU Throttling
Latency
```

------

## 23.15 Memory 飙高

典型：

```
Memory
500Mi
700Mi
900Mi
1.1Gi
1.3Gi
```

持续上涨。

------

### 23.15.1 第一层

```
kubectl top pods -A
```

------

### 23.15.2 第二层

检查 Limit：

```
kubectl get pod POD_NAME -n NAMESPACE -o yaml
```

例如：

```
resources:
  limits:
    memory: 1Gi
```

如果：

```
Memory Usage → 1Gi
```

很可能：

```
OOMKilled
```

------

### 23.15.3 判断是不是内存泄漏

如果：

```
流量不变
Memory 持续上涨
```

需要怀疑：

```
Memory Leak
```

如果：

```
流量上涨
Memory 跟着上涨
```

可能只是业务负载增加。

所以不能简单根据：

```
Memory > 80%
```

判断应用有问题。

------

## 23.16 网络延迟

网络问题最容易出现：

```
偶发
```

例如：

```
API
 ↓
Redis
```

平时：

```
1ms
```

偶尔：

```
500ms
```

------

### 23.16.1 排查思路

首先确认：

```
客户端 → API
```

还是：

```
API → Redis
```

还是：

```
API → Database
```

发生延迟。

------

### 23.16.2 Pod → Pod

测试：

```
kubectl exec -it POD_A -n NAMESPACE -- sh
```

然后：

```
ping POD_IP
```

如果镜像没有 `ping`，可以使用：

```
nc -vz POD_IP PORT
```

------

### 23.16.3 Pod → Service

```
nslookup SERVICE_NAME
```

然后：

```
nc -vz SERVICE_NAME PORT
```

检查：

```
DNS
Service
EndpointSlice
NetworkPolicy
CNI
```

------

### 23.16.4 生产环境不要只看网络

如果：

```
API → Database
```

响应慢，不一定是 Kubernetes 网络。

也可能是：

```
Database CPU 高
Database Lock
Connection Pool 耗尽
慢 SQL
磁盘 I/O
```

因此：

> 网络延迟必须结合 Application Metrics 一起判断。

------

## 23.17 磁盘满

这是生产环境非常危险的问题。

Node 上执行：

```
df -h
```

例如：

```
Filesystem      Size  Used Avail Use%
/dev/sda1       100G   98G    2G  98%
```

------

### 23.17.1 为什么磁盘满很危险

可能影响：

```
Container Runtime
Image Pull
Container Logs
Kubelet
Pod 创建
临时文件
Database
```

甚至导致：

```
Node DiskPressure
```

------

### 23.17.2 Kubernetes 层面检查

```
kubectl describe node NODE_NAME
```

寻找：

```
DiskPressure
```

------

### 23.17.3 常见原因

```
Container Logs
旧 Image
Container Layer
临时文件
应用日志
Core Dump
数据库文件
```

如果使用 containerd，可以检查：

```
crictl images
```

以及：

```
crictl ps -a
```

具体命令是否可用取决于 Node 的运行环境。

------

### 23.17.4 Inode 满了

非常容易被忽略。

即使：

```
Disk Usage = 70%
```

也可能：

```
Inode = 100%
```

检查：

```
df -i
```

因此生产环境需要同时监控：

```
Disk Capacity
+
Inode
```

------

## 23.18 从 Event → Pod → Node → Application 逐层排查

这是本章最重要的内容。

以后遇到 Kubernetes 故障，不要随机执行命令。

建立一个固定流程。

------

### 第一层：Event

首先：

```
kubectl get events -A --sort-by='.lastTimestamp'
```

指定 Namespace：

```
kubectl get events -n shop --sort-by='.lastTimestamp'
```

也可以：

```
kubectl describe pod POD_NAME -n shop
```

看：

```
Events
```

目标：

> **先找 Kubernetes 自己告诉你的异常。**

------

### 第二层：Pod

检查：

```
kubectl get pods -n shop -o wide
```

关注：

```
STATUS
READY
RESTARTS
NODE
```

然后：

```
kubectl describe pod POD_NAME -n shop
```

最后：

```
kubectl logs POD_NAME -n shop
```

如果发生过重启：

```
kubectl logs POD_NAME -n shop --previous
```

------

### 第三层：Workload

Pod 通常不是独立存在的。

检查：

```
Deployment
StatefulSet
DaemonSet
Job
CronJob
```

例如：

```
kubectl get deployment -n shop
kubectl describe deployment shop-api -n shop
```

确认：

```
Desired
Current
Ready
Available
Updated
```

------

### 第四层：Service

如果 Pod 本身正常，但访问失败：

```
kubectl get svc -n shop
kubectl describe svc shop-api -n shop
```

然后：

```
kubectl get endpointslice -n shop
```

确认：

```
Service
 ↓
EndpointSlice
 ↓
Pod
```

是否完整。

------

### 第五层：Ingress

如果内部访问正常，外部访问失败：

```
Ingress
 ↓
Ingress Controller
 ↓
Service
```

检查：

```
kubectl get ingress -A
```

然后查看 Controller 日志。

------

### 第六层：Node

如果多个 Pod 同时出现问题，而且集中在同一个 Node：

```
kubectl get pods -A -o wide
```

例如：

```
Pod A → node-03 → Crash
Pod B → node-03 → Crash
Pod C → node-03 → Crash
```

那么就应该重点调查：

```
node-03
```

执行：

```
kubectl describe node node-03
```

必要时登录：

```
systemctl status kubelet
```

以及：

```
journalctl -u kubelet -n 200
```

检查：

```
CPU
Memory
Disk
Network
Container Runtime
kubelet
```

------

### 第七层：Application

如果 Kubernetes 层全部正常：

```
Pod Ready
Service Endpoint 正常
Node Ready
```

但用户仍然报错：

```
HTTP 500
HTTP 502
HTTP 503
Latency 高
```

就应该进入：

```
Application
```

检查：

```
Application Logs
Application Metrics
Database
Redis
External API
Connection Pool
Thread Pool
```

------

## 一个生产环境通用排障决策树

以后遇到：

> **“Kubernetes 里的应用访问不了”**

可以按照下面的顺序：

```
用户访问失败
      │
      ▼
   DNS 正常？
      │
   ┌──┴──┐
   否    是
   │      │
   ▼      ▼
 DNS    Ingress
问题      │
          ▼
    Controller 正常？
          │
          ▼
       Service
          │
          ▼
   EndpointSlice 有后端？
          │
     ┌────┴────┐
     否        是
     │          │
     ▼          ▼
 Selector     Pod Ready？
 /Pod状态       │
               ▼
          Container 正常？
               │
               ▼
         Application 正常？
               │
               ▼
        Redis / DB / 外部依赖
```

这比：

```
kubectl delete pod
kubectl restart
kubectl rollout restart
```

更加重要。

------

## 生产环境故障排查命令清单

下面这些命令建议熟练掌握。

### 集群状态

```
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by='.lastTimestamp'
```

### Pod

```
kubectl get pods -o wide -n NAMESPACE
kubectl describe pod POD -n NAMESPACE
kubectl logs POD -n NAMESPACE
kubectl logs POD -n NAMESPACE --previous
kubectl exec -it POD -n NAMESPACE -- sh
```

### Workload

```
kubectl get deployment -n NAMESPACE
kubectl describe deployment DEPLOYMENT -n NAMESPACE
kubectl get rs -n NAMESPACE
kubectl rollout status deployment/DEPLOYMENT -n NAMESPACE
kubectl rollout history deployment/DEPLOYMENT -n NAMESPACE
```

### Service

```
kubectl get svc -n NAMESPACE
kubectl describe svc SERVICE -n NAMESPACE
kubectl get endpointslice -n NAMESPACE
```

### Ingress

```
kubectl get ingress -A
kubectl describe ingress INGRESS -n NAMESPACE
```

### 存储

```
kubectl get pvc -A
kubectl get pv
kubectl get storageclass
kubectl describe pvc PVC -n NAMESPACE
```

### 资源

```
kubectl top nodes
kubectl top pods -A
```

### Node

```
kubectl describe node NODE
```

Linux Node：

```
systemctl status kubelet
journalctl -u kubelet -n 200
df -h
df -i
```

------

## 一个非常重要的生产排障原则

遇到问题时，尽量遵循：

```
现象
 ↓
范围
 ↓
证据
 ↓
假设
 ↓
验证
 ↓
根因
 ↓
修复
 ↓
验证
```

例如：

```
现象：
API 偶发 503

↓

范围：
只有 shop-api

↓

证据：
3 个 Pod 中 1 个 Readiness 不正常

↓

假设：
其中一个 Pod 本身异常

↓

验证：
kubectl describe pod
kubectl logs

↓

根因：
数据库连接池耗尽

↓

修复：
调整连接池 + 应用配置

↓

验证：
503 消失
Latency 恢复
Error Rate 恢复
```

这才是真正的运维排障。

------

## 本章核心知识总结

本章最重要的不是记住：

```
ImagePullBackOff
CrashLoopBackOff
OOMKilled
```

而是建立：

> **看到现象 → 找证据 → 缩小范围 → 定位根因**

的思维方式。

生产环境最常用的排查链路可以浓缩成：

```
                  故障
                   │
                   ▼
                 Event
                   │
                   ▼
                  Pod
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
      Status      Logs      Describe
        │
        ▼
      Workload
        │
        ▼
      Service
        │
        ▼
   EndpointSlice
        │
        ▼
       Node
        │
        ▼
   Application
        │
        ▼
 Database / Redis / External
```

不同故障对应的第一关注点：

| 故障                | 第一检查方向                         |
| ------------------- | ------------------------------------ |
| Pod Pending         | Event / Scheduler / Resource         |
| ImagePullBackOff    | Image / Registry / Secret            |
| CrashLoopBackOff    | Logs / Previous Logs / Probe         |
| OOMKilled           | Memory / Limit / Application         |
| Service 无法访问    | Selector / EndpointSlice / Pod Ready |
| Ingress 无法访问    | DNS / Controller / Service           |
| DNS 失败            | CoreDNS / Service / Pod Network      |
| Pod 网络异常        | CNI / NetworkPolicy / DNS            |
| PVC 挂载失败        | PVC / PV / StorageClass / CSI        |
| Node NotReady       | Node Condition / kubelet / Runtime   |
| Deployment 更新失败 | ReplicaSet / New Pod / Probe         |
| 502 / 503           | Ingress → Service → Endpoint → Pod   |
| CPU 飙高            | QPS / CPU / Throttling / Application |
| Memory 飙高         | Usage / Limit / OOM / Memory Leak    |
| 网络延迟            | DNS / Network / Application / DB     |
| 磁盘满              | Disk / Inode / Logs / Runtime        |

最后记住一个非常实用的原则：

> **不要看到 Kubernetes 报错就重启 Pod。**

重启可能暂时让：

```
红色 → 绿色
```

但如果根因仍然存在：

```
问题
 ↓
Pod 重启
 ↓
暂时正常
 ↓
问题再次出现
```

这不是解决故障，只是掩盖故障。

真正的生产运维，是能够回答：

> **“为什么会发生、影响范围多大、根因在哪里、修复后如何证明已经恢复。”**