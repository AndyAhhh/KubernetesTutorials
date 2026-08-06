## 第六阶段 第一章：滚动更新与应用发布——RollingUpdate、Recreate、金丝雀发布、蓝绿发布

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

### 本章学习目标

学习完本章，你应该能够回答：

- Deployment 为什么可以不停机升级？
- RollingUpdate 到底是怎么工作的？
- maxSurge 与 maxUnavailable 有什么作用？
- Recreate 与 RollingUpdate 的区别是什么？
- Rollback 为什么能快速恢复？
- 蓝绿发布与金丝雀发布分别适用于哪些场景？
- Kubernetes 原生支持哪些发布策略？

------

### 第一节：为什么不能直接停止旧版本？

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

### 第二节：Rolling Update（滚动更新）

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

#### 一个生活中的例子

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

### 第三节：Deployment 为什么能做到不停机？

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

### 第四节：maxSurge（最大额外 Pod）

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

#### 为什么需要多出来的 Pod？

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

### 第五节：maxUnavailable（最大不可用）

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

### 第六节：Rolling Update 整个流程

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

### 第七节：Recreate（重建）

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

#### 什么时候使用？

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

### 第八节：Rollout History（发布历史）

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

### 第九节：Rollback（回滚）

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

#### 一个生活中的例子

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

### 第十节：蓝绿发布（Blue-Green）

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

#### 优点

回滚：

极快。

几乎：

秒级。

------

#### 缺点

资源：

翻倍。

因为：

两套：

一起：

运行。

------

### 第十一节：金丝雀发布（Canary）

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

#### 为什么叫金丝雀？

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

### 第十二节：Kubernetes 原生支持金丝雀吗？

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

### 第十三节：企业真实发布流程

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

### 第十四节：各种发布方式对比

| 发布方式       | 是否停机 | 回滚速度 | 资源消耗 | 适用场景                  |
| -------------- | -------- | -------- | -------- | ------------------------- |
| Recreate       | ❌ 会停机 | 快       | 低       | 极少使用                  |
| Rolling Update | ✅ 不停机 | 快       | 较低     | 默认方案，大多数 Web 服务 |
| Blue-Green     | ✅ 不停机 | 极快     | 高       | 核心业务、需要快速切换    |
| Canary         | ✅ 不停机 | 快       | 中等     | 新功能验证、降低发布风险  |

------

### 第十五节：新手最容易犯的错误

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

### 第十六节：知识关系图

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

### 第十七节：企业最佳实践

下面是比较成熟的建议：

| 场景           | 推荐方式                                                     |
| -------------- | ------------------------------------------------------------ |
| 普通 Web API   | Rolling Update                                               |
| 核心支付系统   | Blue-Green + 快速切换                                        |
| 新功能灰度验证 | Canary                                                       |
| 数据库升级     | 根据具体产品制定方案，很多情况下需要停机窗口或主从切换，而不是依赖 Deployment 策略 |
| 高频迭代业务   | Rolling Update + 自动回滚机制                                |

------

### 第十八节：本章总结（建议牢记）

请记住应用发布最重要的几点：

1. **Deployment 默认采用 RollingUpdate，实现不停机升级。**
2. **`maxSurge` 控制升级过程中允许额外创建多少 Pod。**
3. **`maxUnavailable` 控制升级过程中允许多少 Pod 不可用。**
4. **Recreate 会先删除旧版本，再创建新版本，会导致服务中断。**
5. **Deployment 保留历史 ReplicaSet，因此支持快速 Rollback。**
6. **Blue-Green 通过两套环境切换流量，回滚速度最快，但资源开销最大。**
7. **Canary 通过少量流量验证新版本，降低发布风险，通常需要 Ingress Controller 或服务网格配合实现。**

------

### 🌟 企业经验：发布策略如何选择？

可以参考下面这张速查表：

| 业务特点             | 推荐策略                                           |
| -------------------- | -------------------------------------------------- |
| 一般 Web 服务        | Rolling Update                                     |
| 对停机极其敏感       | Blue-Green                                         |
| 新功能逐步验证       | Canary                                             |
| 希望发布失败自动恢复 | Rolling Update + 健康检查 + 自动回滚（结合 CI/CD） |

------

### 下一章预告：Probe 深入原理与应用生命周期管理

## 第六阶段 第二章：Probe 深入原理与应用生命周期管理

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

### 本章学习目标

完成本章后，你应该能够回答：

- kubelet 是如何执行 Probe 的？
- 为什么 Readiness 失败不会重启 Pod，而 Liveness 会？
- Probe 与 Pod 生命周期有什么关系？
- 为什么很多公司的 Pod 会一直处于 Running，却无法提供服务？
- 如何让 ASP.NET Core 应用真正做到零停机发布？
- 企业为什么要把 `/health/live` 和 `/health/ready` 分开？
- 如何设计一个不会误判的健康检查？

------

### 第一节：重新认识 Probe —— 它不是 Deployment 的功能，而是 kubelet 的职责

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

### 第二节：Probe 的执行流程（底层原理）

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

### 第三节：为什么 Readiness 不会重启 Pod？

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

### 第四节：为什么 Liveness 会重启？

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

### 第五节：为什么 Startup Probe 能解决启动慢的问题？

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

### 第六节：Readiness 与 Rolling Update 的真正关系

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

### 第七节：Pod 删除时到底发生了什么？

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

### 第八节：什么是优雅终止（Graceful Shutdown）？

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

### 第九节：ASP.NET Core 如何配合 Kubernetes？

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

### 第十节：企业健康检查接口设计

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

### 第十一节：企业最常见的三个 Probe 配置错误

##### 错误一：Liveness 检查数据库

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

##### 错误二：Readiness 检查耗时太长

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

##### 错误三：terminationGracePeriodSeconds 设置过小

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

### 第十二节：本章总结

这一章，请真正建立下面几个概念：

1. **Probe 的执行者是 kubelet，而不是 Deployment。**
2. **Readiness 管理的是流量，不管理容器生命周期。**
3. **Liveness 管理的是容器生命周期，失败会触发重启。**
4. **Startup Probe 本质上是 Liveness 的启动保护机制。**
5. **Rolling Update 等待的是 Pod Ready，而不是 Pod Running。**
6. **优雅终止（Graceful Shutdown）是零停机发布的重要组成部分。**
7. **健康检查接口应该按职责拆分，而不是所有检查都放进一个 `/health`。**

------

### 🌟 企业经验

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

#### 下一章预告：Prometheus Operator 与 Kubernetes 监控体系

下一章我们不再介绍"Prometheus 是什么"，而是直接进入生产环境最常见的监控架构，重点学习：

- 为什么企业几乎都使用 **Prometheus Operator** 而不是手工部署 Prometheus？
- `ServiceMonitor`、`PodMonitor`、`PrometheusRule` 分别是什么？
- Exporter 如何接入监控体系？
- Prometheus 如何自动发现 Kubernetes 中的新服务？
- 如何设计一套可维护、可扩展的企业级监控平台？

从下一章开始，我们正式进入 Kubernetes 生态中最重要的 **Operator 模式**。

## 第六阶段 第三章：Prometheus Operator 与 Kubernetes 监控体系

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

### 本章学习目标

完成本章后，你应该能够回答：

- 为什么企业更倾向于使用 Prometheus Operator？
- Operator 与普通 Deployment 有什么区别？
- ServiceMonitor、PodMonitor 是如何工作的？
- Prometheus 如何自动发现新服务？
- Exporter 应该部署在哪里？
- 企业如何组织一套可维护的监控体系？

------

### 第一节：先回顾一个问题——为什么手动维护 Prometheus 很痛苦？

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

### 第二节：Kubernetes 的思想——不要写死 IP

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

### 第三节：Prometheus 的 Kubernetes Service Discovery

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

#### 为什么企业还需要 Operator？

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

### 第四节：什么是 Operator？

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

### 第五节：Operator = CRD + Controller

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

### 第六节：ServiceMonitor 是什么？

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

#### 为什么不是监控 Pod？

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

### 第七节：PodMonitor 又是什么？

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

### 第八节：PrometheusRule

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

### 第九节：Exporter 的作用

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

### 第十节：企业监控架构

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

### 第十一节：为什么企业喜欢 Operator？

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

### 第十二节：企业最佳实践

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

### 第十三节：本章容易混淆的几个概念

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

### 第十四节：企业案例

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

### 第十五节：本章总结

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

### 🌟 企业经验

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

#### 下一章预告：CRD（CustomResourceDefinition）深入解析

虽然这一章已经接触了 CRD，但我们只是把它作为 Prometheus Operator 的一部分。

下一章，我们将单独深入学习 **CRD**：

- Kubernetes 为什么允许你"发明一种新的资源"？
- CRD 与内置资源（Deployment、Service）有什么区别？
- 如何定义属于自己的 `Kind`？
- Version（v1、v1beta1）如何演进？
- OpenAPI Schema 如何做字段校验？
- Status、Spec、Finalizer 分别有什么作用？

这一章学完后，你将真正理解 Kubernetes 为什么不仅是一个容器编排平台，更是一个**可扩展的平台框架**。

## 第六阶段 第四章：CRD（CustomResourceDefinition）深入解析

> **关键词：API Extension、Spec、Status、OpenAPI Schema、Version、Finalizer、Controller**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章开始，我们进入 **Kubernetes 最核心的设计思想**。

如果说：

前面的内容是在学习：

> **如何使用 Kubernetes。**

那么从这一章开始，我们学习的是：

> **Kubernetes 为什么能够不断扩展。**

很多开发者学习 Kubernetes 三四年，都没有真正理解 CRD。

但是，一旦理解了 CRD，你会发现：

- Helm
- Prometheus Operator
- Argo CD
- Cert-Manager
- Istio
- KEDA
- Crossplane

这些项目，本质上都在做同一件事情。

> **向 Kubernetes 添加新的资源类型，然后让 Controller 去管理它们。**

所以，本章不仅仅是学习一个功能，而是学习 Kubernetes 的**扩展模型（Extension Model）**。

### 本章学习目标

学完本章，你应该能够回答：

- 为什么 Kubernetes 可以拥有无限种资源？
- CRD 和 Deployment 有什么本质区别？
- Spec 和 Status 为什么要分开？
- Controller 为什么不能修改 Spec？
- 为什么 Kubernetes 要引入 Finalizer？
- 企业如何设计自己的 CRD？

------

### 第一节：Kubernetes 并不是"固定"的系统

很多初学者认为：

Kubernetes 只有这些资源：

```
Pod
Deployment
Service
Ingress
ConfigMap
Secret
PVC
```

实际上，这只是 Kubernetes **自带（Built-in）** 的资源。

Kubernetes 从一开始就被设计成：

> **允许任何人扩展 API。**

也就是说：

今天：

你可以新增一种资源：

```
Database
```

明天：

别人可以新增：

```
RedisCluster
```

后天：

又可以新增：

```
Certificate
```

Kubernetes 本身并不需要修改源码。

------

#### 一个生活中的例子

把 Kubernetes 想象成一部智能手机。

手机出厂时：

只有：

```
电话

短信

相机
```

后来：

安装 App：

```
微信

支付宝

地图
```

手机：

没有：

重装系统。

只是：

增加：

功能。

CRD：

就像：

给 Kubernetes 安装一个新的 App。

------

### 第二节：API Server 为什么支持 CRD？

前面我们学过：

所有 Kubernetes 资源：

都会经过：

```
kubectl
     │
     ▼
API Server
```

例如：

```
kubectl apply -f deployment.yaml
```

API Server：

知道：

Deployment：

是什么。

因为：

它内置了：

Deployment 的 API。

那么：

如果：

执行：

```
kubectl apply -f database.yaml
```

里面：

```
kind: Database
```

API Server：

原本：

根本：

不知道：

Database。

怎么办？

答案就是：

先安装：

CRD。

------

### 第三节：CRD 到底做了什么？

很多教程说：

> CRD 定义了一种新的资源。

这句话不够准确。

更准确地说：

CRD 告诉 API Server：

> **以后请接受这种 Kind，并按照我定义的规则校验它。**

例如：

安装：

```
Database CRD
```

以后：

API Server：

就认识：

```
kind: Database
```

此时：

你可以：

```
kubectl get databases
```

甚至：

```
kubectl describe database mysql-prod
```

注意：

这里只是：

API Server：

认识：

Database。

**它还不会真正创建数据库。**

这一点非常重要。

------

### 第四节：CRD 只是"注册"，真正干活的是 Controller

这是很多新人最容易混淆的地方。

来看流程：

```
Database CRD
        │
        ▼
API Server
认识 Database
        │
        ▼
kubectl apply
        │
        ▼
Database 对象被保存到 etcd
```

到这里为止：

什么都没有发生。

为什么？

因为：

没有人：

处理它。

只有：

Controller：

持续监听：

```
Database
```

例如：

```
Database(mysql-prod)

↓

Controller

↓

创建 StatefulSet

↓

创建 Service

↓

创建 PVC

↓

初始化数据库
```

所以：

牢记一句话：

> **CRD 负责"定义资源"，Controller 负责"实现资源"。**

两者缺一不可。

------

### 第五节：一个完整的 CRD 生命周期

假设：

我们自己设计一种资源：

```
kind: RedisCluster
```

用户执行：

```
kubectl apply -f redis.yaml
```

整个过程：

```
kubectl
      │
      ▼
API Server
      │
校验 Schema
      │
      ▼
保存到 etcd
      │
      ▼
Redis Controller Watch
      │
      ▼
创建 StatefulSet
      │
创建 Service
      │
创建 PVC
      │
      ▼
更新 Status
```

有没有发现？

这和 Deployment Controller 的工作方式几乎一样。

这就是 Kubernetes 一直强调的：

> **控制器模式（Controller Pattern）**

------

### 第六节：为什么要区分 Spec 和 Status？

这是 Kubernetes API 设计中最经典的一部分。

几乎所有资源都有：

```
spec:
```

以及：

```
status:
```

例如：

Deployment：

```
spec:
  replicas: 5
```

表示：

**我希望有 5 个 Pod。**

然后：

Controller：

不断协调。

最后：

```
status:
  readyReplicas: 5
```

表示：

**现在真的已经有 5 个 Ready Pod。**

所以：

可以理解成：

```
Spec
    │
    ▼
用户期望

Status
    │
    ▼
系统现实
```

------

#### 一个生活中的例子

网约车：

你：

下单：

```
目的地：

机场
```

这是：

Spec。

司机：

真正：

开到：

哪里。

这是：

Status。

如果：

司机：

堵车。

Status：

不断：

变化。

但是：

Spec：

始终：

没有：

变。

------

### 第七节：为什么 Controller 不应该修改 Spec？

很多新人写 Controller 时会犯这个错误。

例如：

Controller：

发现：

Pod 太多。

于是：

修改：

```
spec:
  replicas: 3
```

这是错误的。

为什么？

因为：

Spec：

代表：

用户意图。

Controller：

只能：

努力：

让现实：

接近：

Spec。

而不是：

偷偷：

改变：

用户要求。

正确流程：

```
用户修改 Spec
       │
       ▼
Controller 感知变化
       │
       ▼
调整真实资源
       │
       ▼
更新 Status
```

**Controller 最应该修改的是 Status，而不是 Spec。**

------

### 第八节：OpenAPI Schema 的作用

如果没有 Schema。

下面这些内容：

API Server：

都会接受：

```
replicas: hello
```

或者：

```
cpu: abc
```

显然：

这是非法数据。

CRD 可以定义：

```
type: integer
minimum: 1
maximum: 10
```

这样：

API Server：

在保存前：

就会：

拒绝：

非法资源。

这就是：

OpenAPI Schema。

它的价值是：

**把错误挡在进入集群之前。**

------

### 第九节：Version（版本管理）

CRD 和 Kubernetes 内置资源一样，也支持版本。

例如：

```
v1alpha1
```

代表：

实验阶段。

然后：

```
v1beta1
```

代表：

功能基本稳定。

最后：

```
v1
```

代表：

正式版本。

企业通常会经历这样的演进：

```
v1alpha1
      │
功能验证
      │
      ▼
v1beta1
      │
稳定测试
      │
      ▼
v1
```

这样可以在不破坏旧资源的前提下持续演进 API。

------

### 第十节：Finalizer——为什么资源删不掉？

假设：

删除：

```
Database
```

如果：

立即：

删除对象。

那么：

Controller：

还没来得及：

删除：

云数据库。

最终：

产生：

资源泄漏。

因此：

Kubernetes 引入了：

```
Finalizer
```

流程：

```
kubectl delete Database
          │
          ▼
metadata.deletionTimestamp
          │
对象进入删除中状态
          │
Controller 收到通知
          │
删除真实数据库
          │
删除备份
          │
删除网络
          │
移除 Finalizer
          │
API Server 真正删除对象
```

所以：

Finalizer 的作用不是：

阻止删除。

而是：

> **保证删除前，清理工作一定完成。**

------

### 第十一节：企业如何设计一个好的 CRD？

通常遵循三个原则。

#### ① Spec 尽量声明式

例如：

```
spec:
  version: "8.0"
  storage: 100Gi
  replicas: 3
```

不要：

写：

```
spec:
  step1: create
  step2: init
```

Kubernetes：

强调：

**描述目标状态，而不是描述执行步骤。**

------

#### ② Status 记录真实状态

例如：

```
status:
  phase: Running
  readyReplicas: 3
  endpoint: mysql.default.svc
```

让用户能够快速了解当前情况。

------

#### ③ Controller 保持幂等（Idempotent）

幂等是 Controller 最重要的特性之一。

什么意思？

Controller：

可能会：

反复收到同一个事件。

因此：

每次执行：

都应该得到一致的结果。

例如：

发现：

StatefulSet 已经存在。

就不要再次创建。

而是：

检查是否符合 Spec。

如果不符合：

再修正。

------

### 第十二节：本章知识关系图

```
              用户
                │
        kubectl apply
                │
                ▼
        Custom Resource
                │
                ▼
           API Server
         （Schema 校验）
                │
                ▼
              etcd
                │
                ▼
        Custom Controller
                │
      协调真实资源状态
                │
                ▼
      更新 Status 字段
```

------

### 第十三节：本章总结（建议牢记）

请牢牢记住下面几个结论：

1. **CRD 用来扩展 Kubernetes API，让 API Server 认识新的资源类型。**
2. **CRD 只负责定义资源，不负责实现业务逻辑。**
3. **真正执行业务逻辑的是 Controller。**
4. **Spec 表示用户期望，Status 表示系统当前状态。**
5. **Controller 应尽量只修改 Status，而不是 Spec。**
6. **OpenAPI Schema 用于校验 CRD 字段是否合法。**
7. **Finalizer 用于保证删除资源前完成必要的清理工作。**

------

### 🌟 企业经验：为什么说 Kubernetes 更像一个"平台"，而不是一个"容器工具"？

很多人认为 Kubernetes 是：

> "运行容器的软件。"

其实，这是一个很大的误解。

Kubernetes 真正强大的地方在于：

> **它提供了一套通用的 API、存储和控制器机制。**

任何复杂系统，只要能描述为：

> **"用户声明目标状态 → Controller 持续协调现实状态"**

都可以构建在 Kubernetes 之上。

这也是为什么：

- 数据库（数据库 Operator）
- 消息队列（Kafka Operator）
- 证书管理（Cert-Manager）
- GitOps（Argo CD）
- 云资源管理（Crossplane）

都可以以 Kubernetes 资源的形式存在。

从这个角度看，Kubernetes 已经不仅是一个容器编排平台，而是一个**通用的声明式控制平台**。

------

#### 下一章预告：Operator 深入原理——Controller Runtime、Reconcile、Watch 与 Event

这一章我们理解了 **CRD 是什么**。

下一章，我们将继续深入另一半：

**Controller（Operator）**。

重点学习：

- Controller Runtime 是什么？
- Reconcile 循环为什么是 Operator 的核心？
- Watch 机制如何工作？
- 为什么 Controller 不需要"定时扫描"整个集群？
- Operator 如何保证最终一致性（Eventually Consistent）？
- 如何自己开发一个简单的 Kubernetes Operator？

学完下一章，你将真正理解 Kubernetes 控制器的内部工作方式，也具备阅读和开发 Operator 的基础。

## 第六阶段 第五章：Operator 深入原理——Reconcile、Watch 与 Controller Runtime

> **关键词：Informer、Watch、WorkQueue、Reconcile、Level-based、Eventually Consistent**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，是我认为整个 Kubernetes 课程里**最重要的章节之一**。

很多人工作了几年 Kubernetes：

会部署 Helm。

会写 YAML。

会用 ArgoCD。

甚至会写 CRD。

但是：

只要问一句：

> **Operator 为什么能自动工作？**

很多人就答不上来。

原因就是没有理解：

> **Reconcile（协调循环）**

实际上：

**整个 Kubernetes，几乎所有组件都建立在 Reconcile 思想上。**

你之前学习的：

- Deployment Controller
- ReplicaSet Controller
- Job Controller
- StatefulSet Controller
- HPA Controller
- EndpointSlice Controller

全部都是：

> **Reconcile Loop（协调循环）**

理解这一章之后，你会发现 Kubernetes 所有 Controller 的代码结构几乎一模一样。

### 本章学习目标

完成本章后，你应该能够回答：

- Operator 为什么不用定时扫描整个集群？
- Reconcile 为什么是 Kubernetes 的核心思想？
- Watch 和 List 有什么区别？
- Informer 为什么能大幅降低 API Server 压力？
- 为什么 Kubernetes 追求的是"最终一致性（Eventually Consistent）"，而不是"实时一致性"？
- 一个 Controller 为什么要设计成幂等（Idempotent）？

------

### 第一节：先理解一个问题——Controller 为什么不是定时任务？

很多新人第一次写 Controller 时，会想到这种思路：

```
每隔 5 秒：

查询所有 Deployment

↓

检查状态

↓

需要的话进行修复
```

看起来没有问题。

但如果一个集群有：

- 5000 个 Deployment
- 30000 个 Pod
- 1000 个 Node

每 5 秒扫描一次：

API Server 很快就会被压垮。

所以 Kubernetes 从来不是这样工作的。

------

### 第二节：Kubernetes 使用的是事件驱动（Event Driven）

真正的流程是：

```
用户修改 Deployment
        │
        ▼
API Server
        │
        ▼
发送 Watch Event
        │
        ▼
Deployment Controller
        │
        ▼
Reconcile()
```

注意：

Controller：

**不是主动查询。**

而是：

**被事件唤醒。**

这就是：

> Event Driven（事件驱动）

------

#### 一个生活中的例子

假设：

你是一家快递公司的客服。

方案 A：

每一分钟：

打电话问客户：

```
包裹到了吗？
```

效率很低。

方案 B：

物流系统：

包裹状态变化时：

自动：

推送消息。

客服：

收到后：

处理。

Kubernetes：

就是第二种。

------

### 第三节：Watch 到底是什么？

API Server 提供两种读取资源的方法：

#### List

例如：

```
kubectl get pods
```

API Server：

返回：

当前：

所有：

Pod。

然后：

结束。

这就是：

List。

------

#### Watch

Watch：

不同。

例如：

```
Pod A 创建
```

立即：

收到：

```
ADDED
```

Pod：

删除：

收到：

```
DELETED
```

Pod：

修改：

收到：

```
MODIFIED
```

所以：

Watch：

是一条：

**持续存在的事件流。**

而不是：

一次查询。

------

### 第四节：为什么还需要 Informer？

很多人会想：

既然有 Watch。

为什么还要：

Informer？

答案：

因为：

Watch 不能直接解决两个问题。

------

#### 问题一：

每个 Controller 都去 Watch？

假设：

集群里：

```
Deployment Controller

ReplicaSet Controller

HPA Controller

Operator A

Operator B

Operator C
```

大家：

都去：

Watch API Server。

API Server：

压力：

巨大。

------

Informer：

解决办法：

```
API Server
      │
      ▼
 Shared Informer
      │
 ┌────┼────┐
 ▼    ▼    ▼
Controller A
Controller B
Controller C
```

只建立少量 Watch 连接，再把事件分发给多个 Controller。

这就是为什么很多 Controller 会共享 Informer。

------

#### 问题二：

频繁查询对象

例如：

Reconcile：

里面：

需要：

读取：

Deployment。

如果：

每次：

都：

请求 API Server。

性能：

很差。

Informer：

内部维护：

本地缓存（Cache）。

流程：

```
API Server

↓

Informer Cache

↓

Controller
```

Controller：

读取：

Cache。

而不是：

API Server。

因此：

速度：

非常快。

------

### 第五节：WorkQueue——为什么事件不会丢？

Informer：

收到：

事件。

是不是：

马上：

执行：

Reconcile？

不是。

中间：

还有：

```
WorkQueue
```

流程：

```
Watch Event

↓

Informer

↓

WorkQueue

↓

Controller

↓

Reconcile()
```

为什么？

因为：

Controller：

可能：

很忙。

如果：

事件：

直接：

执行。

容易：

丢失。

WorkQueue：

负责：

排队。

保证：

最终：

都会：

处理。

------

### 第六节：Reconcile 是什么？

终于来到最核心的部分。

Reconcile：

翻译：

协调。

什么意思？

例如：

用户：

希望：

```
replicas: 5
```

实际：

只有：

```
3 Pods
```

Controller：

发现：

不一致。

于是：

创建：

两个：

Pod。

最后：

变成：

```
5 Pods
```

整个过程：

就是：

Reconcile。

所以：

可以理解成：

```
Spec

↓

Reality

↓

如果不同

↓

修正 Reality
```

而不是：

修改：

Spec。

------

### 第七节：Reconcile 为什么不是"执行步骤"？

这是 Kubernetes 最容易理解错的地方。

很多新人写 Controller：

```
第一步：

创建 Service

第二步：

创建 Deployment

第三步：

创建 Secret
```

这种写法：

叫：

流程控制。

而 Kubernetes：

不是。

Kubernetes：

每次：

Reconcile：

都会重新检查：

```
Deployment 在吗？

↓

不在？

↓

创建。
```

然后：

```
Service 在吗？

↓

没有？

↓

创建。
```

再：

```
Secret 存在吗？

↓

创建。
```

下一次：

Reconcile：

再执行。

发现：

都有了。

什么：

都不做。

这种设计：

就是：

幂等（Idempotent）。

------

### 第八节：为什么要幂等？

因为：

Reconcile：

可能：

执行：

很多次。

例如：

```
Pod 更新

↓

Deployment 更新

↓

Node 更新

↓

Informer 重连

↓

Controller 重启
```

都会：

触发：

Reconcile。

如果：

Controller：

每次：

都：

创建：

Deployment。

最终：

一定：

报错。

正确：

逻辑：

应该：

```
不存在？

↓

创建

存在？

↓

检查

↓

需要修改？

↓

Update
```

这就是：

幂等。

------

### 第九节：为什么 Kubernetes 追求最终一致性？

很多人会问：

为什么：

不是：

立即：

一致？

例如：

用户：

修改：

```
replicas: 100
```

是不是：

立刻：

100 个：

Pod？

不是。

流程：

```
API Server

↓

Deployment

↓

ReplicaSet

↓

Scheduler

↓

kubelet

↓

Container Runtime
```

需要：

时间。

因此：

Kubernetes：

追求：

```
Eventually Consistent
```

而不是：

Strong Consistency。

也就是说：

允许：

短时间：

不一致。

但是：

最终：

一定：

一致。

------

#### 一个生活中的例子

网上购物：

你：

下单。

并不是：

立刻：

收到：

商品。

而是：

经过：

```
订单

↓

仓库

↓

物流

↓

派送
```

最终：

送到。

Kubernetes：

也是：

类似：

不断协调。

------

### 第十节：Controller Runtime

前面我们一直说：

Controller。

实际上：

企业：

开发 Operator。

几乎：

都会：

使用：

controller-runtime。

它帮你封装了：

```
Watch

Informer

Cache

WorkQueue

Leader Election

Manager
```

开发者：

只需要：

实现：

一个：

```
Reconcile()
```

即可。

所以：

现在：

写 Operator：

重点：

不是：

写框架。

而是：

写：

Reconcile。

------

### 第十一节：一个典型的 Reconcile 流程

以一个 `RedisCluster` Operator 为例：

```
收到事件
    │
    ▼
读取 RedisCluster
    │
    ▼
是否存在 StatefulSet？
    │
 ┌──┴──┐
 │     │
否     是
 │     │
 ▼     ▼
创建   检查配置
 │       │
 ▼       ▼
创建 Service
 │
 ▼
更新 Status
```

如果下一次再次触发：

Controller：

仍然：

从第一步开始。

因为：

每一步：

都是：

检查：

目标状态。

而不是：

依赖：

上一次：

执行到了哪里。

------

### 第十二节：企业开发 Operator 的最佳实践

一个成熟的 Operator 通常遵循以下原则：

1. **只根据当前状态决策，不依赖历史执行过程。**
2. **每次 Reconcile 都可以重复执行。**
3. **所有外部资源都应具备唯一标识，避免重复创建。**
4. **通过 `Status` 反馈当前状态和错误信息。**
5. **使用 Finalizer 清理外部资源。**
6. **不要在 Reconcile 中执行长时间阻塞操作，应设计为可中断、可重试。**

------

### 第十三节：知识关系图

```
          kubectl apply
                 │
                 ▼
            API Server
                 │
            Watch Event
                 │
                 ▼
          Shared Informer
                 │
                 ▼
             WorkQueue
                 │
                 ▼
          Reconcile()
                 │
     ┌───────────┴───────────┐
     ▼                       ▼
读取当前状态             对比 Spec
     │                       │
     └───────────┬───────────┘
                 ▼
          修正真实资源
                 │
                 ▼
          更新 Status
```

------

### 第十四节：本章总结（建议牢记）

请重点记住以下几点：

1. **Controller 是事件驱动的，而不是靠定时扫描整个集群。**
2. **Watch 提供持续的资源变更事件，Informer 在此基础上提供共享 Watch 和本地缓存。**
3. **WorkQueue 保证事件按队列处理，并支持失败重试。**
4. **Reconcile 的职责是让真实状态逐步接近期望状态，而不是记录执行流程。**
5. **Controller 必须保持幂等，因为同一个对象可能会被多次 Reconcile。**
6. **Kubernetes 追求的是最终一致性，而不是瞬时一致性。**
7. **现代 Operator 开发通常基于 `controller-runtime`，核心工作就是实现 `Reconcile()`。**

------

### 🌟 企业经验：为什么说 Kubernetes 的一切几乎都是 Reconcile？

你会发现，我们学过的大多数控制器，本质都遵循同一个模式：

| 控制器                   | 期望状态（Spec） | 协调对象                     |
| ------------------------ | ---------------- | ---------------------------- |
| Deployment Controller    | 副本数、更新策略 | ReplicaSet、Pod              |
| StatefulSet Controller   | 副本数、稳定身份 | Pod、PVC                     |
| Job Controller           | 完成次数         | Pod                          |
| HPA Controller           | 目标副本数       | Deployment                   |
| EndpointSlice Controller | Service 的后端   | EndpointSlice                |
| Prometheus Operator      | Prometheus CR    | StatefulSet、Config、Service |
| Cert-Manager             | Certificate CR   | Secret、证书申请             |
| Argo CD                  | Git 仓库中的声明 | Kubernetes 集群资源          |

虽然它们管理的对象不同，但核心逻辑都可以归纳为一句话：

> **监听事件 → 对比期望状态与实际状态 → Reconcile 到一致。**

这就是 Kubernetes 的核心控制模型。

------

#### 下一章预告：Helm 深入原理——Chart、Template、Release 与包管理思想

接下来，我们将进入企业使用率极高的 **Helm**。

这一章不会停留在"安装 Helm、执行 `helm install`"的层面，而是深入理解：

- 为什么 Helm 会成为 Kubernetes 的事实标准包管理器？
- Chart 为什么不仅仅是一堆 YAML？
- Go Template 是如何生成 Kubernetes 资源的？
- `values.yaml`、`_helpers.tpl`、`templates/` 各自承担什么职责？
- Release 是什么？为什么 Helm 能够回滚？
- 企业如何设计一个可维护、可复用的 Helm Chart？

学完 Helm 后，你会发现企业中部署一个复杂系统（如 Prometheus、Grafana、GitLab、Harbor）往往只需要一条命令，而真正的关键在于理解 Helm 背后的设计思想。

## 第六阶段 第六章：Helm 深入原理——Chart、Template、Release 与 Kubernetes 包管理

> **关键词：Chart、Release、Repository、Template、Values、Go Template、Package Manager**

------

提示词：

```
开始下一章
```

正文：

很好。

如果让我选一个 **Kubernetes 生态中使用率最高的工具**。

答案几乎一定是：

> **Helm。**

你以后部署：

- Prometheus
- Grafana
- Loki
- Harbor
- GitLab
- Jenkins
- ArgoCD
- cert-manager
- ingress-nginx
- Redis
- PostgreSQL

几乎都会看到一句命令：

```
helm install ...
```

很多人会用 Helm。

但是不知道：

> **Helm 为什么会出现？**

这一章，我们不学习命令，而是学习 Helm 的设计思想。

因为只有理解了 Helm 的思想，你以后写自己的 Chart、维护企业 Helm 仓库、设计 CI/CD 才不会迷茫。

### 本章学习目标

学完本章，你应该能够回答：

- 为什么 Kubernetes 需要 Helm？
- Chart 到底是什么？
- 为什么 Helm 不是 YAML 的压缩包？
- values.yaml 为什么存在？
- Release 为什么能够回滚？
- Helm 如何管理不同环境的配置？

------

### 第一节：Helm 为什么会出现？

先看一个真实场景。

假设你准备部署一个完整的 Web 系统：

```
订单系统
```

需要：

```
Deployment
Service
Ingress
ConfigMap
Secret
HPA
ServiceAccount
Role
RoleBinding
NetworkPolicy
PVC
```

总共：

可能：

二十多个 YAML。

如果以后：

再部署：

测试环境。

你需要：

复制：

全部 YAML。

然后修改：

```
namespace: test
```

修改：

```
replicas: 2
```

修改：

```
host:
```

修改：

数据库地址。

修改：

Redis 地址。

修改：

镜像 Tag。

……

很快。

你会得到：

```
dev/

test/

prod/
```

三套：

几乎一样的 YAML。

后果就是：

维护成本越来越高。

------

#### 企业真正的问题

很多新人认为：

Helm：

解决：

安装问题。

实际上：

真正解决的是：

> **配置复用（Configuration Reuse）**

例如：

下面这些：

只有值不同。

```
镜像版本

Namespace

Ingress Host

CPU

Memory

Replica

数据库连接
```

资源类型：

却完全一样。

所以：

没有必要：

复制三份 YAML。

------

### 第二节：Helm 的设计思想

Helm 借鉴了很多成熟的软件包管理器。

例如：

Linux：

```
apt

yum
```

Node：

```
npm
```

Java：

```
Maven
```

Python：

```
pip
```

它们都有共同特点：

> **安装的是"软件包"，而不是一堆零散文件。**

Helm：

也是一样。

安装的不是：

Deployment.yaml。

而是：

```
Chart
```

------

### 第三节：Chart 到底是什么？

很多教程说：

> Chart 就是一堆 YAML。

这是不准确的。

Chart 更像：

> **一个可以生成 Kubernetes 资源的模板工程。**

例如：

```
mychart/

├── Chart.yaml
├── values.yaml
├── templates/
│      deployment.yaml
│      service.yaml
│      ingress.yaml
└── charts/
```

注意：

这里：

templates：

不是：

最终 YAML。

而是：

模板。

例如：

Deployment：

里面：

不会写：

```
replicas: 3
```

而是：

```
replicas: {{ .Values.replicaCount }}
```

真正的数字：

来自：

values.yaml。

------

### 第四节：为什么需要 Template？

假设：

不用模板。

开发环境：

```
replicas: 1
```

生产：

```
replicas: 8
```

你就要维护：

两份 Deployment。

用了 Template：

Deployment：

只有：

一份。

例如：

```
replicas: {{ .Values.replicaCount }}
```

开发：

```
replicaCount: 1
```

生产：

```
replicaCount: 8
```

Deployment：

永远：

不用修改。

------

#### 一个生活中的例子

想象一下：

酒店打印入住单。

模板：

```
欢迎 {{姓名}}

入住：

{{房号}}

入住时间：

{{日期}}
```

真正打印时：

只替换：

变量。

不会：

重新设计：

整张表。

Helm Template：

就是：

同样思想。

------

### 第五节：values.yaml 为什么重要？

很多人认为：

values.yaml：

只是：

配置文件。

实际上：

它承担了：

**Chart 的输入参数**。

例如：

```
image:
  repository: company/order-api
  tag: v1.3.5

replicaCount: 3

resources:
  requests:
    cpu: 500m
```

模板：

读取：

```
{{ .Values.image.tag }}
```

于是：

生成：

```
image:
  company/order-api:v1.3.5
```

所以：

Chart：

可以理解成：

```
Template

+

Values

↓

最终 YAML
```

------

### 第六节：Helm 渲染（Render）过程

很多人不知道：

Helm：

实际上：

不会：

直接创建资源。

它先：

渲染。

整个流程：

```
Chart

+

values.yaml

↓

Go Template

↓

生成最终 YAML

↓

kubectl apply（由 Helm 调用 API）
```

所以：

Helm：

本质：

就是：

一个：

模板引擎。

------

### 第七节：为什么 Helm 使用 Go Template？

因为：

Helm：

本身：

就是：

Go 编写。

因此：

直接使用：

Go Template。

例如：

判断：

```
{{ if .Values.ingress.enabled }}
```

循环：

```
{{ range .Values.ports }}
```

默认值：

```
{{ default 80 .Values.port }}
```

这些：

以后：

写 Chart：

都会用到。

------

### 第八节：Release 是什么？

这是 Helm 最重要的概念。

很多人：

一直：

没有理解。

例如：

安装：

```
helm install order-api .
```

Helm：

不会：

只创建：

Deployment。

还会：

记录：

一次：

安装历史。

例如：

```
Release:

order-api
```

里面：

保存：

```
Chart Version

Values

Rendered YAML

Revision
```

所以：

以后：

可以：

查看：

```
helm history order-api
```

例如：

```
Revision 1

Revision 2

Revision 3
```

这就是：

Helm：

能够：

回滚：

的原因。

------

### 第九节：为什么 Helm 能回滚？

假设：

今天：

升级：

```
v1.0

↓

v1.1
```

Helm：

不会：

覆盖：

历史。

而是：

保存：

新的：

Revision。

例如：

```
Release

↓

Revision 1

↓

Revision 2

↓

Revision 3
```

如果：

发现：

Bug。

执行：

```
helm rollback order-api 2
```

Helm：

重新：

应用：

Revision 2：

对应的 YAML。

这就是：

Rollback。

------

### 第十节：Repository（仓库）

Chart：

写好了。

如何：

分享？

Helm：

提供：

Repository。

例如：

```
https://charts.example.com
```

里面：

可能：

有：

```
nginx

redis

mysql

prometheus

grafana
```

安装：

只需要：

```
helm install prometheus prometheus-community/prometheus
```

和 Linux 安装软件包非常相似。

------

### 第十一节：企业如何管理多环境？

这是 Helm 最经典的用法。

例如：

```
values-dev.yaml

values-test.yaml

values-prod.yaml
```

模板：

只有：

一套。

部署：

开发：

```
helm install \
  -f values-dev.yaml
```

生产：

```
helm install \
  -f values-prod.yaml
```

最终：

生成：

不同：

Deployment。

这也是 Helm 在企业中广泛使用的重要原因。

------

### 第十二节：为什么 Helm Chart 不是"代码生成器"？

很多初学者认为：

Helm：

生成：

YAML。

结束。

其实：

不是。

Chart：

描述的是：

> **一套应用的安装方式。**

例如：

安装：

Redis。

除了：

Deployment。

还包括：

- Secret
- ConfigMap
- PVC
- Service
- NetworkPolicy

这些：

共同组成：

一个：

可安装的软件包。

所以：

Chart：

更像：

安装包。

而不是：

代码生成器。

------

### 第十三节：Chart 的生命周期

一个 Chart 从编写到运行，大致经历以下过程：

```
开发者编写 Chart
        │
        ▼
配置 values.yaml
        │
        ▼
Helm Render
        │
        ▼
生成 Kubernetes YAML
        │
        ▼
提交给 API Server
        │
        ▼
Deployment、Service 等资源创建
        │
        ▼
Release 保存安装历史
```

注意：

**Helm 的工作在资源提交给 API Server 后基本结束。**

之后：

真正维护资源状态的：

仍然是：

Kubernetes Controller。

这一点很重要。

------

### 第十四节：企业最佳实践

在企业中，一个优秀的 Helm Chart 通常遵循这些原则：

##### ① 模板保持通用

不要把环境差异写进模板。

应该通过：

```
values
```

控制。

------

##### ② 所有可配置项集中在 values.yaml

例如：

- 镜像
- 副本数
- CPU
- Memory
- Host
- TLS
- NodeSelector
- Tolerations

都应该：

参数化。

------

##### ③ Chart 不应该包含业务逻辑

Helm：

负责：

生成资源。

Controller：

负责：

维护资源。

应用：

负责：

业务。

职责：

不要混淆。

------

### 第十五节：知识关系图

```
              values.yaml
                    │
                    ▼
             Go Template
                    │
                    ▼
                Chart
                    │
              Helm Render
                    │
                    ▼
         Kubernetes YAML
                    │
                    ▼
              API Server
                    │
                    ▼
      Deployment / Service / ...
                    │
                    ▼
          Release（历史记录）
```

------

### 第十六节：本章总结（建议牢记）

请重点记住以下几个结论：

1. **Helm 是 Kubernetes 的包管理器，而不仅仅是模板工具。**
2. **Chart 是一个可安装的软件包，由模板、配置和元数据组成。**
3. **`values.yaml` 提供输入参数，模板负责生成最终资源。**
4. **Helm 先渲染（Render），再把生成的资源提交给 Kubernetes。**
5. **Release 保存安装历史，因此 Helm 支持升级与回滚。**
6. **多环境配置应通过不同的 `values` 文件实现，而不是复制多套 YAML。**

------

### 🌟 企业经验：Helm 的边界在哪里？

很多团队刚接触 Helm 时，会把所有自动化都塞进 Chart。

实际上，Helm 应该只负责：

> **"声明这套应用应该安装哪些 Kubernetes 资源，以及它们如何参数化。"**

而以下事情通常不属于 Helm 的职责：

- 应用发布审批（交给 CI/CD 或 GitOps）
- 持续监控资源是否漂移（交给 Argo CD、Flux 等）
- 资源运行后的协调（交给 Kubernetes Controller 或 Operator）

理解 Helm 的边界，你才能在后续学习 GitOps 时明白：

**Helm 负责生成资源，GitOps 负责持续同步资源，Kubernetes 负责持续维护资源。**

这是现代云原生平台最经典的职责划分。

------

#### 下一章预告：Helm Chart 实战——Chart 结构、模板语法与企业级 Chart 设计

这一章我们理解了 Helm 的设计思想。

下一章，我们将进入真正的实战：

- Chart 目录结构逐个解析
- `Chart.yaml`、`values.yaml`、`templates/`、`_helpers.tpl` 的职责
- Go Template 常用语法（变量、条件、循环、函数）
- 如何设计一个企业可复用的 Helm Chart
- Chart 依赖（Dependencies）如何管理
- 如何让一个 Chart 同时支持开发、测试、生产环境

这一章结束后，你将具备阅读、修改和编写企业 Helm Chart 的能力，而不仅仅是会执行 `helm install`。

## 第六阶段 第七章：Helm Chart 实战——目录结构、模板语法与企业级设计

> **关键词：Chart.yaml、values.yaml、templates、_helpers.tpl、include、define、pipeline、dependency**

------

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入 **Helm 的实战部分**。

前一章我们理解了 Helm 的思想。

这一章，我们开始真正学习：

> **一个 Helm Chart 到底是怎么写出来的。**

很多人会 `helm install`，却不会修改 Chart。

更不会自己写 Chart。

实际上，在企业里：

**90% 的工作不是从零写 Chart，而是修改别人写好的 Chart。**

所以这一章，我们不仅学习语法，更重要的是学习：

> **如何阅读一个企业级 Helm Chart。**

### 本章学习目标

完成本章后，你应该能够回答：

- 一个 Chart 的每个目录分别有什么作用？
- `_helpers.tpl` 为什么是企业 Chart 中最重要的文件之一？
- Go Template 的变量、条件、循环如何使用？
- `include` 和 `template` 有什么区别？
- 企业如何设计一个可维护、可扩展的 Chart？

------

### 第一节：一个 Helm Chart 的完整结构

假设执行：

```
helm create order-api
```

会生成下面的目录：

```
order-api/

├── Chart.yaml
├── values.yaml
├── .helmignore
├── charts/
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    ├── NOTES.txt
    ├── _helpers.tpl
    └── tests/
```

很多新手第一次看到会觉得文件很多。

实际上：

真正每天都会接触的只有几个。

------

#### 每个目录的职责

| 文件         | 作用                 | 是否经常修改 |
| ------------ | -------------------- | ------------ |
| Chart.yaml   | Chart 元数据         | ⭐⭐           |
| values.yaml  | 默认配置             | ⭐⭐⭐⭐⭐        |
| templates/   | 所有 Kubernetes 模板 | ⭐⭐⭐⭐⭐        |
| _helpers.tpl | 公共模板函数         | ⭐⭐⭐⭐⭐        |
| charts/      | 子 Chart 依赖        | ⭐⭐⭐          |
| NOTES.txt    | 安装后的提示信息     | ⭐            |
| tests/       | Helm 测试            | ⭐            |

企业里最常修改的是：

```
values.yaml

templates/

_helpers.tpl
```

------

### 第二节：Chart.yaml —— Chart 的身份证

这是 Helm Chart 的元数据。

例如：

```
apiVersion: v2

name: order-api

description: Order Service

type: application

version: 1.2.0

appVersion: "2.3.5"
```

很多人会混淆：

```
version
```

和：

```
appVersion
```

它们不是一回事。

------

#### version

表示：

Chart 自己的版本。

例如：

```
Chart

1.0

↓

1.1

↓

1.2
```

可能只是：

修改：

模板。

并没有：

升级：

程序。

------

#### appVersion

表示：

应用版本。

例如：

```
Order API

v2.3.5
```

所以：

以后看到：

```
version: 3.0.0

appVersion: 1.8.2
```

不要惊讶。

Chart 和应用：

分别：

维护版本。

------

### 第三节：values.yaml —— 整个 Chart 的配置中心

企业几乎所有配置：

都会放这里。

例如：

```
replicaCount: 3

image:
  repository: company/order-api
  tag: "v2.1.0"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: api.example.com
```

注意：

这里：

只是：

数据。

没有：

任何：

逻辑。

真正：

使用它的是：

模板。

------

### 第四节：模板如何读取 Values？

Deployment：

可能写：

```
spec:
  replicas: {{ .Values.replicaCount }}
```

Helm：

渲染：

得到：

```
replicas: 3
```

这里：

有一个重要概念：

```
.Values
```

就是：

整个：

values.yaml。

例如：

```
image:
  tag: v1
```

读取：

```
{{ .Values.image.tag }}
```

------

### 第五节：Go Template 基础语法

这是 Helm 最重要的部分。

先看变量。

例如：

```
{{ .Release.Name }}
```

表示：

Release 名称。

例如：

安装：

```
helm install prod-api
```

得到：

```
prod-api
```

------

再例如：

```
{{ .Chart.Name }}
```

得到：

```
order-api
```

------

常见对象：

| 对象            | 含义                   |
| --------------- | ---------------------- |
| `.Values`       | values.yaml            |
| `.Chart`        | Chart.yaml             |
| `.Release`      | 当前 Release           |
| `.Capabilities` | 集群能力（API 版本等） |
| `.Files`        | Chart 中的文件         |

这些对象以后会经常用到。

------

### 第六节：条件判断

例如：

Ingress：

可以：

开启：

或者：

关闭。

模板：

```
{{ if .Values.ingress.enabled }}

apiVersion: networking.k8s.io/v1

...

{{ end }}
```

如果：

```
enabled: false
```

整个：

Ingress：

不会：

生成。

这就是：

为什么：

很多 Chart：

一个 values：

可以：

控制：

几十个：

资源。

------

### 第七节：循环（range）

例如：

一个 Service：

需要：

多个端口。

values：

```
ports:

- 80

- 443

- 8080
```

模板：

```
{{ range .Values.ports }}

- port: {{ . }}

{{ end }}
```

渲染：

```
- port: 80

- port: 443

- port: 8080
```

这里：

```
.
```

表示：

当前：

循环对象。

------

### 第八节：Pipeline —— Go Template 最常见写法

例如：

名字：

最多：

63 个字符。

模板：

```
{{ .Release.Name | trunc 63 }}
```

Pipeline：

类似：

Linux：

```
command1 | command2
```

前一个结果：

作为：

后一个：

输入。

例如：

```
{{ .Values.name | lower | quote }}
```

表示：

```
转小写

↓

加引号
```

企业 Chart：

大量：

使用：

Pipeline。

------

### 第九节：_helpers.tpl —— 企业 Chart 的核心

这一节非常重要。

很多新手：

完全不知道：

为什么有：

```
_helpers.tpl
```

其实：

它相当于：

Go：

里的：

公共函数库。

例如：

很多资源：

名字：

都一样。

不要：

写：

```
name: {{ .Release.Name }}
```

几十次。

可以：

写：

```
{{ include "order.fullname" . }}
```

真正实现：

放：

```
_helpers.tpl
```

里面。

例如：

```
define "order.fullname"

↓

生成统一名称

↓

返回字符串
```

以后：

Deployment：

Service：

Ingress：

全部：

调用：

它。

------

#### 为什么企业都喜欢 _helpers.tpl？

因为：

修改：

命名规则。

只需要：

改：

一个：

地方。

例如：

以前：

```
order-api
```

以后：

改：

```
company-order-api
```

不用：

修改：

所有：

YAML。

------

### 第十节：include 与 template

很多教程：

没有讲清楚。

实际上：

企业：

几乎：

都：

使用：

```
include
```

原因：

它：

可以：

继续：

Pipeline。

例如：

```
{{ include "fullname" . | quote }}
```

template：

不能：

这样：

灵活。

所以：

现在：

推荐：

统一：

使用：

```
include
```

------

### 第十一节：Chart Dependency（依赖）

很多系统：

不仅：

有：

API。

还有：

Redis。

MySQL。

RabbitMQ。

怎么办？

Helm：

支持：

依赖。

例如：

```
dependencies:

- name: redis

- name: mysql
```

安装：

父 Chart。

子 Chart：

一起：

安装。

例如：

```
商城

↓

API

↓

Redis

↓

MySQL
```

一次：

完成。

------

### 第十二节：企业 Chart 设计原则

企业通常遵循以下规则：

#### ① 模板尽量少写逻辑

例如：

不要：

```
if

↓

if

↓

if

↓

if
```

模板：

太复杂。

维护：

困难。

复杂逻辑：

应尽量放到应用配置生成流程中，而不是模板中。

------

#### ② values.yaml 保持稳定

新增：

参数。

尽量：

兼容：

旧版本。

不要：

随意：

删除：

字段。

否则：

升级：

容易：

失败。

------

#### ③ 所有资源统一命名

统一：

调用：

```
_helpers.tpl
```

不要：

每个：

YAML：

自己：

拼接。

------

#### ④ 所有 Label 保持一致

例如：

统一：

生成：

```
app.kubernetes.io/name

app.kubernetes.io/version

helm.sh/chart
```

方便：

Prometheus、

Argo CD、

运维工具：

识别资源。

------

### 第十三节：一个优秀 Chart 的结构

企业推荐：

```
Chart

↓

values.yaml

↓

_helpers.tpl

↓

Deployment

↓

Service

↓

Ingress

↓

HPA

↓

NetworkPolicy

↓

PDB

↓

ServiceAccount
```

所有：

资源：

通过：

Values：

统一：

管理。

------

### 第十四节：本章知识关系图

```
           values.yaml
                 │
                 ▼
          Go Template
                 │
        ┌────────┴────────┐
        ▼                 ▼
   deployment.yaml   service.yaml
        │                 │
        └────────┬────────┘
                 ▼
          _helpers.tpl
                 │
                 ▼
          Render YAML
                 │
                 ▼
             API Server
```

------

### 第十五节：本章总结（建议牢记）

请记住以下几个核心观点：

1. **Chart.yaml 描述 Chart 本身，而不是应用程序。**
2. **values.yaml 是整个 Chart 的配置入口，应尽量保持稳定。**
3. **templates/ 保存 Kubernetes 资源模板，而不是最终 YAML。**
4. **`_helpers.tpl` 用于封装公共模板，是企业 Chart 的核心组成部分。**
5. **推荐使用 `include` 复用模板，并配合 Pipeline 提高灵活性。**
6. **Chart Dependency 可以组合多个应用，实现一键部署完整系统。**

------

### 🌟 企业经验：为什么很多公司规定"所有 Chart 都必须有 `_helpers.tpl`"？

因为企业维护的往往不是一个 Chart，而是几十甚至上百个。

如果每个 Chart：

- 命名方式不同
- Label 不一致
- Selector 拼接规则不同

后续接入：

- Prometheus
- Argo CD
- Service Mesh
- 自动化运维平台

都会变得困难。

因此，成熟团队通常会建立一套统一的 Helm Chart 规范，把资源命名、标签、注解等公共逻辑全部封装到 `_helpers.tpl` 中，实现整个组织的一致性。

------

#### 下一章预告：Helm 高级实践——Upgrade、Rollback、Hooks 与企业发布策略

下一章，我们将进入 Helm 在生产环境中的核心能力：

- `helm upgrade` 的底层流程
- 为什么 Helm 能实现版本回滚？
- Hooks（PreInstall、PostUpgrade、PreDelete）如何工作？
- 如何在升级前自动执行数据库迁移？
- `helm diff`、`helm lint`、`helm template` 在 CI/CD 中如何使用？
- 企业如何设计安全、可回滚、可审计的 Helm 发布流程？

这一章结束后，你将真正掌握 Helm 在生产环境中的发布与运维实践，而不仅仅是编写 Chart。

## 第六阶段 第八章：Helm 高级实践——Upgrade、Rollback、Hooks 与企业发布策略

> **关键词：Upgrade、Rollback、Revision、Hook、Atomic、Diff、Lint、Template**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，是 **Helm 真正进入生产环境** 的开始。

前面两章，我们已经理解了：

- Helm 为什么存在
- Chart 是什么
- values.yaml 如何参数化
- Go Template 如何生成 YAML

但是在企业里，真正每天都会执行的命令其实不是：

```
helm install
```

而是：

```
helm upgrade
```

因为：

> **生产环境几乎每天都在升级。**

所以这一章，我们学习 Helm 最重要的能力：

> **如何安全地升级、回滚和发布应用。**

这一章结束后，你会理解为什么很多公司把 Helm 当作 Kubernetes 应用发布的基础设施。

### 本章学习目标

完成本章后，你应该能够回答：

- Helm Upgrade 到底做了什么？
- 为什么 Helm 能回滚？
- Hook 在什么时候执行？
- 如何在升级前执行数据库迁移？
- 为什么企业发布前都会执行 helm template？
- 什么是 Atomic Upgrade？

------

### 第一节：生产环境为什么几乎不用 helm install？

很多新手练习时都是：

```
helm install order-api .
```

但在真实企业里，一个应用通常只会安装一次。

之后几乎所有发布都是：

```
v1.0

↓

v1.1

↓

v1.2

↓

v1.3
```

对应的命令就是：

```
helm upgrade order-api .
```

所以：

**Upgrade 才是 Helm 的核心能力。**

------

### 第二节：helm upgrade 到底发生了什么？

很多人以为：

Upgrade：

就是重新安装。

其实不是。

整个流程如下：

```
读取新的 Chart
        │
读取新的 Values
        │
Helm Render
        │
生成新的 YAML
        │
与当前 Release 比较
        │
调用 Kubernetes API
        │
更新 Deployment / Service ...
        │
保存新的 Revision
```

注意：

Helm 并不会删除整个应用再重新创建。

它会：

**计算资源差异，然后更新对应资源。**

------

#### 一个例子

旧版本：

```
replicas: 3
```

新版本：

```
replicas: 5
```

Helm 不会：

```
删除 Deployment

↓

重新创建
```

而是：

```
Update Deployment

↓

Deployment Controller

↓

滚动创建新 Pod
```

真正负责滚动升级的仍然是 Kubernetes，而不是 Helm。

------

### 第三节：Revision（修订版本）

每一次 Upgrade。

都会产生一个新的 Revision。

例如：

```
Revision 1
v1.0
```

升级：

```
Revision 2
v1.1
```

继续：

```
Revision 3
v1.2
```

查看：

```
helm history order-api
```

可能看到：

| Revision | Status     | Chart         |
| -------- | ---------- | ------------- |
| 1        | Superseded | order-api-1.0 |
| 2        | Superseded | order-api-1.1 |
| 3        | Deployed   | order-api-1.2 |

这里的 **Superseded** 表示：

> 曾经是当前版本，但后来被更新版本替代了。

------

### 第四节：为什么 Helm 能回滚？

因为：

每一次 Revision。

Helm 都保存了：

- Chart
- Values
- Render 后的 YAML
- Release Metadata

所以：

执行：

```
helm rollback order-api 2
```

实际上就是：

重新应用：

Revision 2 保存的那套资源定义。

然后：

产生新的 Revision。

例如：

```
Revision 1

↓

Revision 2

↓

Revision 3

↓

Rollback

↓

Revision 4（内容等同 Revision 2）
```

注意：

**Rollback 不会删除历史，而是生成新的历史。**

------

### 第五节：Hook 是什么？

很多业务：

升级应用前：

需要执行一些额外操作。

例如：

升级数据库。

或者：

清理缓存。

这些事情：

不是 Deployment 能完成的。

Helm 提供：

**Hook（钩子）**。

它允许：

在安装、升级、删除等生命周期节点执行额外资源。

------

#### 常见 Hook

| Hook         | 执行时机       |
| ------------ | -------------- |
| pre-install  | 安装前         |
| post-install | 安装后         |
| pre-upgrade  | 升级前         |
| post-upgrade | 升级后         |
| pre-delete   | 删除前         |
| post-delete  | 删除后         |
| test         | `helm test` 时 |

------

### 第六节：数据库迁移案例

假设：

你准备发布：

```
v2.0
```

数据库：

需要先执行：

```
ALTER TABLE orders
ADD COLUMN remark;
```

如果：

Deployment：

先升级。

应用启动：

发现：

数据库没有新字段。

直接：

报错。

怎么办？

企业一般这样设计：

```
pre-upgrade Hook

↓

执行 Migration Job

↓

成功？

↓

Upgrade Deployment
```

如果：

Migration：

失败。

整个升级：

停止。

Deployment：

不会更新。

这就是 Hook 的价值。

------

### 第七节：Hook 本质是什么？

很多人以为 Hook 是特殊脚本。

其实：

不是。

Helm Hook：

通常仍然是：

Kubernetes Resource。

例如：

一个：

Job。

只不过：

多了：

Annotation。

例如：

```
metadata:
  annotations:
    "helm.sh/hook": pre-upgrade
```

Helm：

识别：

这个注解。

在升级前：

先创建这个 Job。

等它完成。

再继续升级。

------

### 第八节：helm template —— 企业 CI 最常用命令

很多新手不知道：

CI/CD：

通常不会直接执行：

```
helm install
```

第一步通常是：

```
helm template
```

作用：

```
Chart

↓

Render

↓

输出最终 YAML
```

例如：

```
helm template order-api .
```

得到：

```
apiVersion: apps/v1
kind: Deployment
...
```

为什么？

因为：

企业希望：

先检查：

最终生成的 YAML。

确认：

没有问题。

再部署。

------

### 第九节：helm lint

企业：

提交 Chart 前。

通常都会执行：

```
helm lint .
```

检查：

例如：

```
Chart.yaml 是否正确

values 是否缺失

模板语法是否错误

变量是否不存在
```

可以理解为：

Chart 的：

静态检查。

类似：

Java：

```
CheckStyle
```

Go：

```
golangci-lint
```

------

### 第十节：helm diff

很多企业：

升级前：

都会：

先看：

到底改了什么。

例如：

执行：

```
helm diff upgrade
```

输出：

```
-replicas: 3

+replicas: 5
```

或者：

```
-image: v1.0

+image: v1.1
```

这样：

运维人员：

一眼：

就知道：

本次：

升级：

真正修改了哪些资源。

> **说明：** `helm diff` 来自一个常用的 Helm 插件，并不是 Helm 核心命令。因此在使用前需要安装对应插件。

------

### 第十一节：Atomic Upgrade

这是企业发布非常重要的参数。

例如：

```
helm upgrade \
  --atomic
```

什么意思？

假设：

升级过程中：

Deployment：

失败。

没有：

Ready。

如果：

不用：

Atomic。

结果：

```
升级一半

↓

失败

↓

系统停留在半更新状态
```

可能：

一半：

Pod：

新版本。

一半：

旧版本。

或者：

发布中断。

用了：

Atomic。

流程：

```
Upgrade

↓

失败

↓

自动 Rollback

↓

恢复上一版本
```

所以：

生产环境：

很多团队：

都会：

默认：

开启：

```
--atomic
```

------

### 第十二节：企业发布流程

成熟企业：

通常：

采用：

下面流程：

```
开发提交 Chart
        │
        ▼
helm lint
        │
        ▼
helm template
        │
        ▼
helm diff
        │
        ▼
审批
        │
        ▼
helm upgrade --atomic
        │
        ▼
Deployment Rolling Update
        │
        ▼
监控发布结果
```

注意：

Helm：

负责：

发布。

Deployment：

负责：

滚动更新。

Prometheus：

负责：

监控。

它们：

职责：

不同。

------

### 第十三节：Helm 与 Kubernetes 的职责划分

这是很多新人容易混淆的地方。

| Helm 做什么？ | Kubernetes 做什么？ |
| ------------- | ------------------- |
| 渲染模板      | 调度 Pod            |
| 管理 Release  | 滚动更新            |
| 保存 Revision | 自愈                |
| Upgrade       | Reconcile           |
| Rollback      | 维护最终状态        |

一句话总结：

> **Helm 决定"发布什么"，Kubernetes 决定"如何持续运行它"。**

------

### 第十四节：本章知识关系图

```
              Chart
                │
          values.yaml
                │
                ▼
         helm template
                │
                ▼
        Render YAML
                │
        helm diff 比较
                │
                ▼
      helm upgrade --atomic
                │
                ▼
         API Server
                │
                ▼
 Deployment Controller
                │
                ▼
 Rolling Update
                │
                ▼
 Revision 保存历史
```

------

### 第十五节：本章总结（建议牢记）

请重点记住以下几点：

1. **生产环境最常用的是 `helm upgrade`，而不是 `helm install`。**
2. **每次升级都会生成新的 Revision，因此 Helm 能够回滚。**
3. **Rollback 会创建新的 Revision，而不是恢复旧 Revision 的编号。**
4. **Hook 用于在安装、升级、删除等生命周期执行额外资源（通常是 Job）。**
5. **企业通常会在 CI 中先执行 `helm lint`、`helm template`，必要时再结合 `helm diff` 检查变更。**
6. **`--atomic` 可以在升级失败时自动回滚，提高发布安全性。**
7. **Helm 管理发布流程，Kubernetes 管理资源生命周期，两者职责不同。**

------

### 🌟 企业经验：为什么越来越多的团队把 Helm 和 GitOps 一起使用？

很多公司早期的发布流程是：

```
开发人员

↓

SSH 登录服务器

↓

helm upgrade
```

这种方式存在几个问题：

- 谁发布的？
- 发布了哪个版本？
- 为什么发布？
- 能否自动恢复？
- 如何保证所有集群一致？

于是，越来越多的团队演进到：

```
Git（Chart + Values）
        │
        ▼
Argo CD / Flux
        │
        ▼
自动检测 Git 变更
        │
        ▼
执行 Helm Render
        │
        ▼
同步到 Kubernetes
```

也就是说：

**Helm 不再由人手动执行，而是成为 GitOps 系统中的"渲染引擎"。**

这也是当前云原生社区最主流的部署模式。

------

#### 下一章预告：GitOps 与 Argo CD——声明式持续交付的核心思想

从下一章开始，我们将进入 Kubernetes 部署体系的最后一块拼图：

**GitOps**。

我们将深入理解：

- GitOps 为什么会取代传统 CI/CD？
- 为什么 Git 仓库会成为"唯一可信来源（Single Source of Truth）"？
- Argo CD 是如何持续监听 Git 并同步集群的？
- Sync、Diff、Health、Self-Heal 分别是什么意思？
- Helm 与 Argo CD 如何协同工作？
- 企业如何设计一套可审计、可回滚、自动化的 Kubernetes 发布体系？

学完 GitOps，你就能理解目前绝大多数云原生企业采用的现代 Kubernetes 发布架构。

## 第六阶段 第九章：GitOps 与 Argo CD——声明式持续交付的核心思想

> **关键词：GitOps、Desired State、Actual State、Sync、Drift、Self-Heal、Reconciliation**

------

提示词：

```
开始下一章
```

正文：

很好。

到目前为止，我们已经学习了：

> **Kubernetes 如何运行应用。**

也学习了：

> **Helm 如何发布应用。**

但是，企业真正的发布流程，还缺少最后一块拼图。

因为现实中，一个企业不可能每天都让运维工程师执行：

```
helm upgrade
```

如果几十个项目、几百个服务都靠人工执行：

很快就会出现：

- 谁发布的？
- 为什么发布？
- 发布了哪个版本？
- 有没有审批？
- 有没有回滚？
- 集群里的配置是不是被别人偷偷改了？

于是，云原生社区提出了一种新的理念：

> **GitOps。**

很多人把 GitOps 理解成：

> "Git + CI/CD"

这是不准确的。

GitOps 真正改变的是：

> **Git 成为了整个集群唯一可信的数据源（Single Source of Truth）。**

这一章，我们正式学习现代 Kubernetes 企业部署体系。

### 本章学习目标

完成本章后，你应该能够回答：

- GitOps 到底是什么？
- 为什么 Git 会成为 Kubernetes 的"数据库"？
- Argo CD 为什么不用 SSH 登录服务器？
- 什么叫 Drift（配置漂移）？
- Self-Heal 为什么是 GitOps 最大优势？
- Helm 与 Argo CD 是什么关系？

------

### 第一节：传统发布模式有什么问题？

先看看传统 CI/CD。

例如：

开发完成代码。

流程：

```
Git Push
      │
      ▼
CI 构建镜像
      │
      ▼
SSH 登录服务器
      │
      ▼
kubectl apply
```

看起来没有问题。

但是企业会遇到很多麻烦。

例如：

运维：

晚上：

登录集群。

执行：

```
kubectl edit deployment order-api
```

修改：

```
replicas: 20
```

第二天。

没人知道。

为什么？

因为：

Git：

没有记录。

------

#### 更严重的问题

例如：

有人：

直接：

删除：

Deployment。

或者：

修改：

Service。

Git：

完全：

不知道。

CI：

也不知道。

最终：

Git：

和：

集群：

越来越：

不一致。

这就是：

企业最大的痛点。

------

### 第二节：GitOps 的核心思想

GitOps：

提出一个非常大胆的原则：

> **不要直接修改 Kubernetes。**

所有修改。

必须：

先修改：

Git。

然后：

系统：

自动：

同步。

流程：

```
Git Repository
       │
       ▼
Argo CD
       │
       ▼
Kubernetes
```

所以：

以后：

真正的发布：

不是：

```
kubectl apply
```

而是：

```
Git Commit
```

------

#### 一个生活中的例子

假设：

Git：

就是：

公司合同。

员工：

不能：

直接：

改工资。

必须：

先：

修改：

合同。

系统：

自动：

更新：

工资。

GitOps：

也是：

一样。

Git：

就是：

唯一：

合法来源。

------

### 第三节：Single Source of Truth（唯一可信来源）

这是 GitOps 最重要的一句话。

以前：

真正状态：

可能：

来自：

```
Git

+

服务器

+

运维修改

+

kubectl edit
```

到底：

哪个：

是真的？

没人知道。

GitOps：

规定：

只有：

Git。

例如：

Git：

写：

```
replicas: 3
```

那么：

整个集群：

最终：

一定：

恢复：

3。

Git：

就是：

真相。

------

### 第四节：Argo CD 是什么？

很多新人认为：

Argo CD：

就是：

自动执行：

kubectl。

实际上：

不是。

Argo CD：

本质：

仍然是：

一个：

Controller。

是不是：

很熟悉？

没错。

它和：

Deployment Controller：

Prometheus Operator：

工作方式：

完全一样。

只是：

它监听的不是：

Pod。

而是：

Git。

------

#### Argo CD 的工作流程

```
Git Repository
       │
       ▼
Argo CD Watch
       │
       ▼
发现 Commit
       │
       ▼
Render Helm/Kustomize/YAML
       │
       ▼
Diff
       │
       ▼
Apply 到 Kubernetes
```

有没有发现？

还是：

Reconcile。

只是：

Desired State：

来自：

Git。

------

### 第五节：Desired State 与 Actual State

这是 GitOps 最核心的模型。

Git：

里面：

写：

```
replicas: 5
```

这是：

Desired State。

而：

集群：

现在：

```
Replicas = 3
```

这是：

Actual State。

Argo CD：

不断：

比较：

```
Git

↓

Desired

↓

Cluster

↓

Actual
```

如果：

不同。

立即：

同步。

------

### 第六节：Sync（同步）

例如：

Git：

修改：

```
image:
  tag: v2
```

Argo CD：

发现：

Commit。

执行：

```
Render

↓

Diff

↓

Apply
```

最后：

Deployment：

开始：

Rolling Update。

整个过程：

无需：

人工。

------

### 第七节：什么是 Drift（配置漂移）？

这是企业最关心的问题。

例如：

Git：

写：

```
replicas: 3
```

但是：

某位工程师：

执行：

```
kubectl scale deployment order-api --replicas=10
```

此时：

集群：

变成：

```
10 Pods
```

Git：

仍然：

```
3 Pods
```

两者：

不一致。

这就叫：

> **Configuration Drift（配置漂移）**

------

### 第八节：Self-Heal（自动修复）

Argo CD：

发现：

Drift。

怎么办？

如果：

开启：

Self-Heal。

它会：

自动：

恢复：

```
10

↓

3
```

整个过程：

无需：

人工。

这就是：

GitOps：

最大的价值。

------

#### 一个生活中的例子

Git：

像：

建筑蓝图。

现场：

工人：

偷偷：

把：

窗户：

改了。

监理：

发现：

和：

蓝图：

不一致。

立即：

要求：

恢复。

Argo CD：

就是：

这个：

监理。

------

### 第九节：为什么 Argo CD 不推荐手工修改集群？

因为：

手工：

修改：

马上：

会：

Drift。

随后：

Argo CD：

又：

改回来。

于是：

你会发现：

刚执行：

```
kubectl edit
```

一分钟后：

修改：

消失。

不是：

Bug。

而是：

GitOps：

故意：

这样设计。

因为：

Git：

才是：

唯一：

可信来源。

------

### 第十节：Helm 与 Argo CD 的关系

很多新人：

认为：

二选一。

其实：

不是。

企业：

几乎：

都是：

一起：

使用。

流程：

```
Git

↓

Helm Chart

↓

Argo CD

↓

Helm Render

↓

YAML

↓

API Server
```

注意：

Argo CD：

并不会：

替代：

Helm。

它只是：

调用：

Helm：

Render。

真正：

生成：

YAML：

仍然：

是：

Helm。

------

### 第十一节：Argo CD 的健康状态（Health）

Argo CD 不仅判断：

是否同步。

还判断：

资源：

是否健康。

例如：

Deployment：

```
Desired = 3

Ready = 3
```

状态：

Healthy。

如果：

```
Ready = 1
```

状态：

Degraded。

这意味着：

即使同步完成，也不代表应用真正可用。

------

### 第十二节：Sync 状态

Argo CD 常见状态有：

| 状态      | 含义             |
| --------- | ---------------- |
| Synced    | Git 与集群一致   |
| OutOfSync | Git 与集群不一致 |
| Unknown   | 无法判断         |
| Syncing   | 正在同步         |

这里要注意：

**Synced ≠ Healthy。**

例如：

Deployment 已经更新到最新 YAML（Synced）。

但新 Pod 一直 CrashLoopBackOff。

那么：

资源可能是：

```
Synced

+

Degraded
```

这是企业排查问题时非常重要的区别。

------

### 第十三节：企业 GitOps 架构

现代企业通常采用这样的部署流程：

```
开发提交代码
        │
        ▼
CI 构建镜像
        │
        ▼
更新 Helm Chart 或 values.yaml（Git）
        │
        ▼
Git Repository
        │
        ▼
Argo CD
        │
        ▼
Render Helm
        │
        ▼
Diff
        │
        ▼
Apply 到 Kubernetes
        │
        ▼
Deployment Rolling Update
        │
        ▼
Prometheus 监控
```

请注意一个关键点：

**CI 和 CD 的职责已经分离。**

- **CI（Continuous Integration）**：负责构建、测试、生成镜像。
- **CD（Continuous Delivery/Deployment）**：由 Argo CD 根据 Git 中的声明持续同步到 Kubernetes。

------

### 第十四节：GitOps 与传统发布对比

| 对比项   | 传统 CI/CD              | GitOps                 |
| -------- | ----------------------- | ---------------------- |
| 发布入口 | 人工执行 kubectl / helm | Git Commit             |
| 配置来源 | 集群 + Git              | Git 唯一来源           |
| 回滚     | 手工操作                | Git 回退 Commit 或版本 |
| 审计     | 较弱                    | Git 历史天然审计       |
| 漂移修复 | 人工                    | 自动 Self-Heal         |
| 状态同步 | 一次性                  | 持续 Reconcile         |

------

### 第十五节：本章知识关系图

```
          Git Repository
                │
      Desired State（目标状态）
                │
                ▼
            Argo CD
                │
      持续 Reconcile
                │
        ┌───────┴────────┐
        ▼                ▼
   Diff（发现差异）   Sync（同步）
        │                │
        └───────┬────────┘
                ▼
          Kubernetes 集群
                │
      Actual State（实际状态）
                │
         Drift？→ Self-Heal
```

------

### 第十六节：本章总结（建议牢记）

请重点记住以下几点：

1. **GitOps 的核心思想是：Git 是整个系统唯一可信来源（Single Source of Truth）。**
2. **Argo CD 本质上仍然是一个 Controller，它持续比较 Git 与集群状态并进行 Reconcile。**
3. **Desired State 来自 Git，Actual State 来自 Kubernetes 集群。**
4. **Configuration Drift 指 Git 与集群状态不一致。**
5. **Self-Heal 可以自动修复配置漂移，使集群重新回到 Git 中定义的状态。**
6. **Helm 与 Argo CD 不是竞争关系，企业通常使用 Argo CD 调用 Helm 渲染 Chart。**
7. **GitOps 将"发布"从人工执行命令，转变为管理 Git 仓库中的声明。**

------

### 🌟 企业经验：为什么 GitOps 正在成为 Kubernetes 的主流？

随着企业规模越来越大，手工发布的风险也越来越高。

GitOps 带来的最大价值，并不是"自动化"本身，而是：

- **所有变更都有 Git 历史可追溯。**
- **所有环境都以 Git 为标准，减少"这台机器怎么和那台机器不一样"的问题。**
- **任何人都不能绕过流程直接修改生产环境，而是通过代码评审、审批和 Git 提交完成变更。**
- **集群发生意外漂移时，可以自动恢复到预期状态。**

因此，GitOps 不只是一个部署工具，更是一种**运维治理模式**。

------

#### 下一章预告：Kustomize 深入解析——原生配置管理与 Helm 的区别

很多人会问：

> Kubernetes 已经有 Helm 了，为什么还需要 Kustomize？

下一章我们将深入分析：

- Kustomize 的设计思想
- Base / Overlay 是什么？
- Patch（Strategic Merge、JSON6902）如何工作？
- ConfigMapGenerator、SecretGenerator 的作用
- Helm 与 Kustomize 的优缺点对比
- 企业什么时候应该选择 Helm？什么时候应该选择 Kustomize？什么时候两者结合使用？

学完这一章，你将理解 Kubernetes 两大配置管理方案的定位与最佳实践，而不是简单地认为它们是互相替代的工具。

## 第六阶段 第十章：Kustomize 深入解析——原生配置管理与 Helm 的区别

> **关键词：Base、Overlay、Patch、Strategic Merge、JSON6902、Generator、Transformer**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，我们学习 Kubernetes 官方生态中另一个非常重要的工具：

> **Kustomize。**

很多新人都会有一个疑问：

> **既然 Helm 已经这么强大了，为什么 Kubernetes 官方还要推出 Kustomize？**

甚至很多公司同时使用：

- Helm
- Kustomize
- Argo CD

为什么不统一？

答案就在这一章。

这一章结束后，你会真正理解：

> **Helm 和 Kustomize 并不是竞争关系，而是解决不同的问题。**

### 本章学习目标

完成本章后，你应该能够回答：

- Kustomize 为什么会出现？
- Base 和 Overlay 是什么意思？
- 为什么 Kustomize 不使用模板？
- Patch 是如何修改 YAML 的？
- Kustomize 和 Helm 的区别是什么？
- 企业什么时候使用 Helm？什么时候使用 Kustomize？

------

### 第一节：Helm 已经很好了，为什么还要 Kustomize？

先来看 Helm 的工作方式。

例如：

Deployment：

```
replicas: {{ .Values.replicaCount }}
```

真正部署时：

```
replicaCount: 5
```

Helm：

Render：

得到：

```
replicas: 5
```

这里：

最核心的是：

> **Template（模板）**

------

但是：

很多 Kubernetes 社区开发者认为：

YAML：

已经：

够复杂了。

为什么：

还要：

学习：

Go Template？

例如：

下面：

这种：

```
{{ if .Values.xxx }}
```

对于很多运维来说：

阅读：

比较困难。

于是：

Kustomize：

提出：

一种：

完全不同的思路。

------

### 第二节：Kustomize 的思想——不生成 YAML，而是修改 YAML

这是本章最重要的一句话。

Helm：

思想：

```
Template

+

Values

↓

生成 YAML
```

Kustomize：

思想：

```
已有 YAML

↓

Patch（修改）

↓

得到最终 YAML
```

有没有发现？

它：

不是：

模板。

而是：

**补丁（Patch）。**

------

#### 一个生活中的例子

假设：

有一份合同。

Helm：

做法：

重新打印：

整份合同。

Kustomize：

做法：

贴：

修改页。

例如：

第一页：

第十行：

工资：

```
5000
```

改：

```
6000
```

其它：

全部：

保持：

不变。

这就是：

Patch。

------

### 第三节：Base（基础配置）

Kustomize：

首先：

定义：

基础资源。

例如：

```
base/

deployment.yaml

service.yaml

kustomization.yaml
```

Deployment：

例如：

```
replicas: 3
```

Service：

正常：

ClusterIP。

这里：

表示：

所有环境：

共同：

使用：

这一套资源。

------

### 第四节：Overlay（环境覆盖）

不同环境：

需要：

不同：

配置。

例如：

开发：

```
overlays/dev
```

生产：

```
overlays/prod
```

开发：

只修改：

```
replicas: 1
```

生产：

修改：

```
replicas: 10
```

Base：

完全：

不用：

复制。

------

#### 一个典型目录

```
project/

base/

deployment.yaml

service.yaml

kustomization.yaml

overlays/

dev/

prod/

uat/
```

这是企业：

最经典：

目录结构。

------

### 第五节：kustomization.yaml 是什么？

它相当于：

整个：

Kustomize：

入口。

例如：

```
resources:

- ../../base
```

表示：

先：

加载：

Base。

然后：

继续：

执行：

Patch。

所以：

Kustomization：

不是：

Deployment。

而是：

"如何组装资源"。

------

### 第六节：Patch 是什么？

假设：

Base：

Deployment：

```
replicas: 3
```

生产：

需要：

```
replicas: 10
```

Patch：

只写：

```
spec:
  replicas: 10
```

最终：

得到：

```
replicas: 10
```

Deployment：

其它：

全部：

保持：

不变。

这就是：

Patch。

------

### 第七节：Strategic Merge Patch（战略合并）

这是 Kustomize 最常见的 Patch。

例如：

Base：

```
containers:

- name: api

  image: v1
```

Patch：

```
containers:

- name: api

  image: v2
```

结果：

只有：

Image：

变：

```
v1

↓

v2
```

其它：

字段：

全部：

保留。

为什么？

因为：

它根据：

```
name
```

进行：

智能合并。

所以：

叫：

Strategic Merge。

------

### 第八节：JSON6902 Patch

有些时候：

Strategic Merge：

不够。

例如：

删除：

一个：

字段。

或者：

修改：

数组：

某一项。

这时候：

使用：

JSON6902。

例如：

```
replace

↓

/spec/replicas

↓

5
```

或者：

```
remove

↓

/metadata/annotations
```

它：

更灵活。

但是：

可读性：

没有：

Strategic Merge：

好。

企业：

一般：

只有：

复杂修改：

才会：

使用。

------

### 第九节：Generator（自动生成）

Kustomize：

还能：

自动：

生成：

ConfigMap。

例如：

配置文件：

```
app.properties
```

生成：

ConfigMap。

无需：

手写：

YAML。

Secret：

也一样。

例如：

```
password.txt
```

生成：

Secret。

这就是：

Generator。

------

### 第十节：Transformer（统一修改）

假设：

所有资源：

都需要：

增加：

Label：

```
team: payment
```

如果：

几十个：

Deployment。

几十个：

Service。

全部：

修改。

很麻烦。

Transformer：

一次：

完成。

例如：

```
所有资源

↓

增加：

team=payment
```

这也是：

Kustomize：

非常强大的能力。

------

### 第十一节：Helm 与 Kustomize 的根本区别

这是本章最重要的内容。

很多教程：

只是：

列：

优缺点。

实际上：

真正区别：

是：

**设计思想不同。**

| Helm            | Kustomize    |
| --------------- | ------------ |
| Template        | Patch        |
| Values          | Overlay      |
| Go Template     | 原生 YAML    |
| 包管理          | 配置变更     |
| Release 管理    | 无 Release   |
| 支持 Chart 仓库 | 不提供包仓库 |

一句话总结：

> **Helm 更像"安装软件"，Kustomize 更像"修改配置"。**

------

### 第十二节：企业什么时候使用 Helm？

例如：

安装：

- Prometheus
- Grafana
- Harbor
- Jenkins
- GitLab
- Loki

几乎：

一定：

Helm。

因为：

这些：

都是：

完整：

软件。

已经：

有人：

维护：

Chart。

直接：

安装：

即可。

------

### 第十三节：企业什么时候使用 Kustomize？

例如：

公司：

自己：

开发：

Order API。

Payment API。

User API。

Deployment：

已经：

写好了。

不同环境：

只有：

少量：

配置：

不同。

例如：

```
CPU

Memory

Replica

Host
```

这时候：

Kustomize：

非常：

合适。

因为：

不用：

模板。

只需要：

Patch。

------

### 第十四节：Helm + Kustomize 一起使用

很多企业：

最终：

都是：

这种：

模式。

例如：

```
Helm

↓

安装：

Prometheus
```

然后：

```
Kustomize

↓

增加：

Company Label

修改：

Ingress

增加：

Annotation
```

或者：

Argo CD：

先：

Render：

Helm。

然后：

Kustomize：

继续：

Patch。

这是很多大型企业常见的实践。

------

### 第十五节：企业案例

假设：

你们公司：

维护：

一个支付平台。

第三方：

提供：

Redis Helm Chart。

你们：

不希望：

Fork（复制维护）整个 Chart。

只是：

想：

增加：

```
nodeSelector:
  node-role: storage
```

以及：

```
team: payment
```

如果：

修改：

Chart。

以后：

升级：

第三方：

Chart：

很困难。

这时候：

Kustomize：

可以：

在 Helm 渲染后的资源上追加这些修改，避免长期维护自己的 Chart 分支。

这也是很多企业把 Helm 与 Kustomize 组合使用的原因。

------

### 第十六节：知识关系图

```
                Base
                  │
        deployment.yaml
        service.yaml
                  │
                  ▼
           kustomization.yaml
                  │
                  ▼
              Overlay
                  │
       Patch / Generator
                  │
                  ▼
          最终 Kubernetes YAML
                  │
                  ▼
             API Server
```

------

### 第十七节：本章总结（建议牢记）

请重点记住以下几点：

1. **Kustomize 不使用模板，而是基于已有 YAML 进行 Patch。**
2. **Base 保存公共资源，Overlay 保存环境差异。**
3. **Strategic Merge Patch 适合大多数修改场景，JSON6902 更灵活但更复杂。**
4. **Generator 可以自动生成 ConfigMap 和 Secret。**
5. **Transformer 可以批量修改资源（如统一添加 Label、Annotation）。**
6. **Helm 更偏向软件包管理，Kustomize 更偏向配置管理。**
7. **企业中 Helm 与 Kustomize 经常配合使用，而不是互相替代。**

------

### 🌟 企业经验：Helm 与 Kustomize，到底该怎么选？

这是很多团队都会讨论的问题。

可以用下面这个思路判断：

| 场景                                                         | 更推荐                     |
| ------------------------------------------------------------ | -------------------------- |
| 部署第三方成熟软件（Prometheus、Grafana、Harbor、GitLab 等） | Helm                       |
| 管理自己团队开发的应用，多个环境配置差异较小                 | Kustomize                  |
| 希望使用社区维护的 Chart，但又需要少量企业定制               | Helm + Kustomize           |
| 配合 GitOps（如 Argo CD）                                    | 两者都可以，按团队规范选择 |

真正成熟的团队，很少坚持"只允许 Helm"或"只允许 Kustomize"。

更常见的做法是：

- **Helm 负责软件打包与参数化。**
- **Kustomize 负责环境差异和企业级定制。**
- **GitOps（Argo CD 或 Flux）负责持续同步与治理。**

三者各司其职，共同组成现代云原生交付体系。

------

#### 下一章预告：Service Mesh 入门——为什么 Kubernetes 还需要 Istio？

到这里，你已经掌握了 Kubernetes 的部署体系。

下一章，我们将进入 Kubernetes 网络生态中最重要的话题之一：

**Service Mesh（服务网格）**。

我们将学习：

- 微服务为什么会催生 Service Mesh？
- Sidecar 是什么？为什么每个 Pod 都会多一个代理？
- Istio、Envoy 分别扮演什么角色？
- 服务发现、负载均衡、熔断、限流、灰度发布如何实现？
- Service Mesh 与 Ingress、Kubernetes Service 有什么区别？

这一章开始，我们将进入云原生架构的高级阶段，也是现代大型微服务平台的核心组成部分。