## 第三阶段 第一章：Kubernetes 生产故障排查（Troubleshooting）

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

### 本章学习目标

学习完本章，你应该能够回答：

- Kubernetes 故障排查的正确顺序是什么？
- 为什么不能一上来就看日志？
- 如何分析 Pod 一直 Pending？
- CrashLoopBackOff 是什么？
- ImagePullBackOff 怎么解决？
- 如何系统排查 Service、Ingress、DNS 和网络问题？
- 如何利用 kubectl 工具快速定位问题？

------

### 第一节：建立正确的排查思维

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

### 第二节：第一步——先看集群

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

### 第三节：第二步——查看 Pod 状态

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

### 第四节：Pending 怎么排查？

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

### 第五节：CrashLoopBackOff 是什么？

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

#### 常见原因

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

### 第六节：ImagePullBackOff

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

### 第七节：Pod Running，但访问失败怎么办？

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

#### 第一步：检查 Service

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

#### 第二步：检查 Endpoint

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

### 第八节：Ingress 无法访问怎么办？

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

### 第九节：DNS 问题如何排查？

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

### 第十节：探针导致不断重启

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

### 第十一节：生产环境最常用的排查命令

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

### 第十二节：建立自己的排查流程

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

### 第十三节：一个完整案例

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

### 本章总结（建议牢记）

请记住 Kubernetes 故障排查最重要的几点：

1. **先看资源状态，再看日志，不要一开始就盲目查看日志。**
2. **`kubectl describe pod` 和 `Events` 是定位 Pending、调度失败、探针失败的重要工具。**
3. **`CrashLoopBackOff` 表示应用不断崩溃重启，`ImagePullBackOff` 表示镜像拉取失败。**
4. **Service 无法访问时，重点检查 Selector 和 Endpoints。**
5. **Ingress 问题需要同时检查 Ingress 资源、Ingress Controller 和后端 Service。**
6. **建立固定的排查流程，比记忆大量命令更重要。**

------

### 到这里，你已经具备了基础生产故障排查能力

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

### 下一章预告：Kubernetes 生产最佳实践（Best Practices）

下一章，我们将总结企业中最常见的 Kubernetes 使用规范，包括：

- 为什么不能使用 `latest` 标签？
- 为什么一定要设置 `resources.requests` 和 `resources.limits`？
- 为什么必须配置 Readiness/Liveness Probe？
- 为什么 Deployment 副本数建议至少为 2？
- 为什么生产环境尽量不要直接使用 `NodePort`？
- 为什么需要 PodDisruptionBudget（PDB）？
- 如何设计适合生产环境的 Helm Chart？

这一章会把前面所有知识串联成一套**企业级 Kubernetes 使用规范**，帮助你从"会用 Kubernetes"迈向"正确地用 Kubernetes"。

## 第三阶段 第二章：Kubernetes 生产最佳实践（Best Practices）

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

### 本章学习目标

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

### 第一节：不要使用 latest 标签

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

#### 正确做法

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

### 第二节：一定要配置 resources

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

#### 正确做法

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

##### requests

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

##### limits

表示：

> **最多只能使用这么多。**

超过：

限制。

可能会被限速（CPU）或因内存不足而被终止（Memory）。

------

#### 一个生活例子

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

### 第三节：Readiness Probe 比 Liveness Probe 更重要

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

### 第四节：Deployment 至少两个副本

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

### 第五节：使用 Rolling Update，而不是 Recreate

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

### 第六节：配置 HPA，而不是固定副本数

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

### 第七节：Pod 不要全部放在同一个 Node

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

### 第八节：不要把所有配置写死

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

### 第九节：不要把密码放进 Git

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

### 第十节：健康检查一定要有

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

### 第十一节：建立统一的标签（Labels）

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

### 第十二节：不要忽略 Namespace

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

### 第十三节：生产环境上线前检查清单（Checklist）

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

### 第十四节：企业部署一个 ASP.NET Core 服务的推荐架构

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

### 本章总结（建议牢记）

请记住 Kubernetes 生产最佳实践中最重要的几点：

1. **永远不要使用 `latest` 作为生产镜像标签。**
2. **始终配置 `resources.requests` 和 `resources.limits`。**
3. **Readiness Probe 决定是否接收流量，Liveness Probe 决定是否需要重启。**
4. **生产环境 Deployment 建议至少 2 个副本，并采用 Rolling Update。**
5. **敏感信息不要提交到 Git，应使用 Secret 或专业密钥管理方案。**
6. **统一使用 Namespace、Labels 和 Helm 管理应用。**
7. **上线前使用固定的 Checklist，可以显著降低生产事故风险。**

------

### 到这里，你已经具备了企业级 Kubernetes 部署思维

现在，你已经不仅知道**如何部署**，更知道**如何安全、稳定地部署**。

这是从"能跑起来"到"能长期稳定运行"的关键一步。

------

### 下一章预告：Kubernetes 网络原理（深度篇）

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

## 第三阶段 第三章：Kubernetes 网络原理（上）

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

### 本章学习目标

学习完本章，你应该能够回答：

- Kubernetes 网络为什么这么复杂？
- 为什么每个 Pod 都有自己的 IP？
- Pod 为什么可以直接互相访问？
- Pod 跨 Node 为什么还能通信？
- CNI 到底是什么？
- Flannel、Calico、Cilium 分别负责什么？
- Kubernetes 网络模型到底是什么？

------

### 第一节：为什么 Kubernetes 网络这么复杂？

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

### 第二节：Kubernetes 网络模型

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

### 第三节：为什么每个 Pod 都有自己的 IP？

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

#### 一个生活中的例子

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

### 第四节：Pod IP 是谁分配的？

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

### 第五节：什么是 CNI？

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

### 第六节：最常见的 CNI 插件

目前企业最常见的有三个。

------

#### Flannel

一句话：

> **最简单，最容易上手。**

特点：

- 安装简单
- 学习成本低
- 功能相对较少
- 适合学习、小型集群

很多教学环境都会选择 Flannel。

------

#### Calico

一句话：

> **企业使用最广泛的网络方案之一。**

特点：

- 性能优秀
- 支持 NetworkPolicy
- 路由能力强
- 大规模集群表现稳定

很多生产环境都会选择 Calico。

------

#### Cilium

一句话：

> **基于 eBPF 的新一代 Kubernetes 网络方案。**

特点：

- 性能非常高
- 支持高级网络策略
- 可观测性强
- 与 Service Mesh、安全能力结合紧密

近年来越来越多新项目开始采用 Cilium。

------

### 第七节：Pod 为什么能跨 Node 通信？

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

### 第八节：Node 为什么也能访问 Pod？

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

### 第九节：Pod IP 是固定的吗？

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

### 第十节：为什么不能直接依赖 Pod IP？

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

### 第十一节：本章知识关系图

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

### 第十二节：本章总结（建议牢记）

请记住 Kubernetes 网络最重要的几点：

1. **Kubernetes 的核心设计目标是：任何 Pod 都可以直接访问任何其他 Pod。**
2. **每个 Pod 都拥有独立 IP，把 Pod 看成一台"小服务器"最容易理解。**
3. **Pod IP 由 CNI 插件分配，不是 Kubernetes 自己分配。**
4. **CNI 是标准接口，Flannel、Calico、Cilium 都是它的实现。**
5. **Pod IP 是临时的，不应该写死在配置或代码中。**
6. **服务之间应通过 Service 名称访问，而不是直接访问 Pod IP。**

------

### 到这里，你已经理解了 Kubernetes 网络设计思想

这是学习 Kubernetes 网络的第一步，也是最重要的一步。

后面的内容，我们将开始真正进入底层实现。

------

### 下一章预告：Kubernetes 网络原理（下）——Service、kube-proxy、iptables 与 IPVS

下一章，我们将回答 Kubernetes 网络中最经典的几个问题：

- 为什么 **Service 明明没有 Pod，却可以访问？**
- ClusterIP 到底是不是一个真实存在的 IP？
- kube-proxy 究竟做了什么？
- iptables 是如何把请求转发到 Pod 的？
- IPVS 为什么性能更高？
- 一个请求从浏览器进入集群，到达 Pod，中间究竟经历了哪些步骤？

学完下一章，你将真正理解 **Service 的底层实现原理**，也会明白为什么 Kubernetes 能实现稳定的服务发现与负载均衡。这也是排查网络故障和理解 Ingress、Service Mesh 的重要基础。

## 第三阶段 第四章：Kubernetes 网络原理（下）——Service、kube-proxy、iptables 与 IPVS

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

### 本章学习目标

学习完本章，你应该能够回答：

- Service 为什么可以访问？
- ClusterIP 是什么？
- 为什么 ClusterIP 不是一个真实的 IP？
- kube-proxy 到底负责什么？
- iptables 和 IPVS 的区别是什么？
- Service 是如何实现负载均衡的？
- 一个请求从客户端到 Pod 的完整路径是什么？

------

### 第一节：先回顾一个问题

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

### 第二节：Service 到底是什么？

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

#### 一个生活中的例子

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

### 第三节：ClusterIP 是什么？

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

### 第四节：ClusterIP 是真实存在的吗？

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

#### 一个生活中的例子

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

### 第五节：既然不存在，那为什么还能访问？

答案：

因为：

有：

```
kube-proxy
```

------

### 第六节：kube-proxy 是什么？

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

### 第七节：请求到底发生了什么？

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

### 第八节：iptables 是什么？

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

#### 一个生活中的例子

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

### 第九节：Service 如何实现负载均衡？

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

### 第十节：为什么后来出现了 IPVS？

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

### 第十一节：iptables 与 IPVS 的区别

| 对比项         | iptables       | IPVS               |
| -------------- | -------------- | ------------------ |
| 实现方式       | Netfilter 规则 | Linux IPVS 模块    |
| 数据结构       | 规则链         | 哈希表等高效结构   |
| 大规模集群性能 | 一般           | 更好               |
| 配置复杂度     | 较低           | 略高               |
| 历史使用       | 较早默认方案   | 长期作为高性能方案 |

> **补充说明：** 从较新的 Kubernetes 版本开始，社区越来越多地推荐使用 **eBPF**（例如 Cilium）作为现代数据平面。在一些新建集群中，甚至可以完全绕过 kube-proxy（称为 *kube-proxy replacement*），获得更好的性能和可观测性。我们会在后续讲解 Cilium 和 eBPF 时详细介绍。

------

### 第十二节：Service 为什么不用修改？

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

### 第十三节：一次请求完整流程

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

### 第十四节：Endpoint 与 EndpointSlice

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

### 第十五节：本章知识关系图

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

### 第十六节：本章总结（建议牢记）

请记住 Service 网络最重要的几点：

1. **Service 是稳定访问入口，不是真正提供服务的程序。**
2. **ClusterIP 是虚拟 IP（VIP），不是绑定在某块网卡上的真实地址。**
3. **真正接收请求的是后端 Pod。**
4. **kube-proxy 负责维护 Service 到 Pod 的转发规则。**
5. **iptables 和 IPVS 都可以实现 Service 转发，IPVS 更适合大规模集群。**
6. **现代 Kubernetes 默认使用 EndpointSlice 管理 Service 后端。**
7. **Pod 可以随时变化，而 Service 地址保持稳定，这正是 Kubernetes 服务发现的核心思想。**

------

### 到这里，你已经掌握了 Kubernetes Service 的底层原理

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

### 下一章预告：Kubernetes 网络原理（终章）——DNS、CoreDNS、Service Discovery 与完整请求链路

下一章，我们将把网络体系的最后一块拼图补上，包括：

- 为什么 `http://order-api` 就能访问？
- Kubernetes DNS 是如何工作的？
- CoreDNS 的职责是什么？
- FQDN（完全限定域名）是什么？
- 不同 Namespace 为什么会影响服务发现？
- 一个请求从浏览器到 Ingress，再到 Service，最终到 Pod 的完整网络链路是什么？

学完下一章，你将完整掌握 **Kubernetes 网络体系**，真正理解一次请求在集群中的完整流转过程，为后续学习 **Ingress Controller、Gateway API、Service Mesh（Istio）** 打下坚实基础。

## 第三阶段 第五章：Kubernetes 网络原理（终章）——DNS、CoreDNS、Service Discovery

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

### 本章学习目标

学习完本章，你应该能够回答：

- 什么是 Service Discovery（服务发现）？
- 为什么可以直接访问 `http://order-api`？
- CoreDNS 是什么？
- FQDN（完全限定域名）是什么？
- 不同 Namespace 为什么访问方式不同？
- DNS 查询完整流程是什么？
- 一次 HTTP 请求如何从浏览器最终到达 Pod？

------

### 第一节：什么是服务发现（Service Discovery）？

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

#### 一个生活中的例子

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

### 第二节：DNS 在 Kubernetes 中负责什么？

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

### 第三节：CoreDNS 是什么？

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

### 第四节：CoreDNS 如何工作？

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

### 第五节：FQDN（完全限定域名）

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

#### 一个生活中的例子

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

### 第六节：为什么平时只写 redis？

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

### 第七节：跨 Namespace 如何访问？

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

### 第八节：DNS 查询完整流程

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

### 第九节：浏览器访问 Kubernetes 的完整过程

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

### 第十节：DNS 为什么这么重要？

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

### 第十一节：如何排查 DNS 问题？

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

### 第十二节：为什么 Service 名称不能重复？

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

### 第十三节：DNS 缓存

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

### 第十四节：完整知识体系

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

### 第十五节：Kubernetes 网络知识总结

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

### 第十六节：本章总结（建议牢记）

请记住 Kubernetes DNS 最重要的几点：

1. **CoreDNS 是 Kubernetes 集群内的 DNS 服务器。**
2. **服务之间推荐通过 Service 名称通信，而不是直接使用 Pod IP。**
3. **完整的 Service 域名格式为：`<service>.<namespace>.svc.cluster.local`。**
4. **同 Namespace 可以直接使用 Service 名称，跨 Namespace 建议带上 Namespace 或完整域名。**
5. **CoreDNS 负责名称解析，真正的数据转发由 Service、kube-proxy 和网络组件完成。**
6. **DNS 故障会导致大量服务之间无法互相发现，因此 CoreDNS 通常需要高可用部署。**

------

### 到这里，你已经完整掌握 Kubernetes 网络体系

从 Pod、Service、ClusterIP、kube-proxy，到 DNS、CoreDNS、EndpointSlice，你已经能够理解一次请求如何在 Kubernetes 集群中流转。

这是学习更高级云原生技术的重要基础。

------

### 下一章预告：存储原理——Volume、PV、PVC、StorageClass

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