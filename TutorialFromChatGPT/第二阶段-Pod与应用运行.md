# 第 4 章：Pod——Kubernetes 最核心的概念

Pod 是 Kubernetes 中最核心、最基础的运行对象之一。

如果把 Kubernetes 的应用运行过程简化，可以先建立下面这张关系：

```
Kubernetes Cluster
        │
        ▼
      Worker Node
        │
        ▼
        Pod
        │
        ▼
    Container
        │
        ▼
   Application
```

其中最重要的一点是：

> **Kubernetes 调度和管理的基本单位是 Pod，而不是单独的 Container。**

------

## 4.1 Pod 是什么

Pod（豆荚）是 Kubernetes 中**最小的部署单元和调度单元**。

它是 Kubernetes 对应用运行实例进行抽象的基本对象。

最简单的 Pod 可以只包含一个 Container：

```
Pod
└── nginx Container
```

一个 Pod 也可以包含多个 Container：

```
Pod
├── Application Container
└── Sidecar Container
```

因此可以简单理解为：

> **Pod 是一个或多个紧密关联的 Container 的运行载体。**

Pod 本身并不是用来直接执行应用代码的。真正执行应用代码的是 Container。

可以理解成：

```
Pod
 │
 ├── 提供运行边界
 │
 ├── 提供网络环境
 │
 ├── 提供存储共享环境
 │
 └── 管理其中的 Container
```

而 Container：

```
Container
    │
    ▼
Application Process
```

------

### 4.1.1 Pod 是 Kubernetes 最小调度单位

这是 Pod 最重要的概念之一。

假设 Kubernetes 集群有三个 Worker Node：

```
Cluster
│
├── Node-01
├── Node-02
└── Node-03
```

现在创建一个 Pod：

```
Pod-A
```

Scheduler 会决定 Pod-A 运行在哪一个 Node：

```
Pod-A
   │
   ▼
Node-02
```

如果 Pod-A 中有两个 Container：

```
Pod-A
├── Container-A
└── Container-B
```

那么它们都会运行在 Node-02：

```
Node-02
└── Pod-A
    ├── Container-A
    └── Container-B
```

不会出现：

```
Container-A → Node-01
Container-B → Node-02
```

因为：

> **Pod 是调度的最小边界。**

Scheduler 选择的是：

```
Pod
```

而不是：

```
Container
```

------

### 4.1.2 Pod 是 Container 的运行边界

Pod 不只是把几个 Container 放在一起。

它还为这些 Container 提供一个共同的运行环境。

例如：

```
Pod
│
├── Container A
├── Container B
│
├── Network
│
└── Volumes
```

同一个 Pod 中的 Container 可以共享：

- 网络命名空间
- Pod IP
- Volume
- 本地通信环境

因此 Pod 实际上建立了一个重要的边界：

```
Pod Boundary
┌───────────────────────────┐
│                           │
│ Container A               │
│ Container B               │
│                           │
│ Shared Network            │
│ Shared Volumes            │
│                           │
└───────────────────────────┘
```

------

### 4.1.3 Pod 通常对应一个应用实例

在 Kubernetes 中，一个 Pod 通常可以理解为：

> **一个应用实例。**

例如：

```
Deployment
    │
    ├── Pod-A
    ├── Pod-B
    └── Pod-C
```

如果每个 Pod 都运行一个 API Container：

```
Pod-A → API Instance 1
Pod-B → API Instance 2
Pod-C → API Instance 3
```

那么实际上就是：

```
3 个 Pod
=
3 个 API 应用实例
```

这也是后面学习 Deployment、ReplicaSet、Service 和 HPA 的基础。

------

## 4.2 为什么不是直接管理 Container

如果 Container 已经能够运行应用，为什么 Kubernetes 不直接管理 Container？

因为 Kubernetes 要解决的问题远远不只是：

```
启动 Container
```

它还需要解决：

```
应用应该运行在哪个 Node？
Container 是否应该和其他 Container 放在一起？
哪些 Container 需要共享网络？
哪些 Container 需要共享存储？
应用实例如何进行故障恢复？
应用如何扩容？
应用如何滚动更新？
```

如果 Kubernetes 直接管理 Container：

```
Container-A
Container-B
Container-C
Container-D
```

Kubernetes 就需要额外维护它们之间的关系：

```
A 和 B 属于同一个应用
A 和 B 必须在同一个 Node
A 和 B 共享网络
A 和 B 共享 Volume
```

这会导致 Container 管理层面变得非常复杂。

因此 Kubernetes 引入了 Pod：

```
Pod-A
├── Container-A
└── Container-B

Pod-B
├── Container-C
└── Container-D
```

于是 Kubernetes 只需要对 Pod 进行调度：

```
Pod-A → Node-01
Pod-B → Node-02
```

Pod 内部的 Container 关系则由 Pod 来表达。

------

### 4.2.1 Pod 解决了 Container 编排问题

例如一个应用由：

```
Application
+
Log Collector
```

组成。

如果直接管理 Container：

```
Application Container
Log Collector Container
```

Kubernetes 需要理解：

```
这两个 Container 应该在一起
```

使用 Pod 后：

```
Pod
├── Application
└── Log Collector
```

这个关系就非常明确。

Pod 表示：

> 这两个 Container 是一个整体，应该共同运行。

------

### 4.2.2 Pod 解决了共同运行环境的问题

同一个 Pod 中的 Container 可以共享：

```
Network
Volume
Lifecycle
```

例如：

```
Pod
├── App
└── Sidecar
```

App：

```
localhost:8080
```

Sidecar：

```
localhost:9090
```

因为共享网络命名空间，所以：

```
App
  │
  └── localhost:9090
       ↓
    Sidecar
```

这就是 Pod 作为运行边界的重要价值。

------

## 4.3 Pod 与 Container 的关系

Pod 和 Container 是两个不同层次的概念。

正确关系：

```
Kubernetes
     │
     ▼
    Pod
     │
     ▼
 Container
     │
     ▼
Application Process
```

不要理解成：

```
Pod = Container
```

正确理解应该是：

```
Pod
├── Container A
└── Container B
```

Pod 可以包含：

```
1 个 Container
```

也可以包含：

```
多个 Container
```

------

### 4.3.1 Pod 中的 Container 必须运行在同一个 Node

假设：

```
Pod-A
├── Container-A
└── Container-B
```

Scheduler 会把整个 Pod 调度到一个 Node：

```
Node-01
└── Pod-A
    ├── Container-A
    └── Container-B
```

因此不能：

```
Node-01
└── Container-A

Node-02
└── Container-B
```

这也是为什么 Pod 是调度单位，而 Container 不是。

------

### 4.3.2 Pod 中的 Container 共享网络

同一个 Pod 中的 Container 共享网络命名空间。

例如：

```
Pod IP
10.244.1.10

Pod
├── App       :8080
└── Sidecar   :9090
```

App 可以访问：

```
localhost:9090
```

Sidecar 可以访问：

```
localhost:8080
```

因此可以把它理解为：

```
Pod
┌─────────────────────────┐
│                         │
│  App       :8080        │
│      ↕ localhost        │
│  Sidecar   :9090        │
│                         │
│  Pod IP: 10.244.1.10    │
│                         │
└─────────────────────────┘
```

------

### 4.3.3 Pod、Container 与 Image 的关系

这三个概念非常容易混淆。

它们之间的关系是：

```
Image
  │
  │ 创建
  ▼
Container
  │
  │ 运行在
  ▼
Pod
```

例如：

```
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

这里：

```
nginx:1.27
```

是 Image。

Container Runtime 根据 Image 创建 Container：

```
nginx:1.27
     ↓
nginx Container
```

然后这个 Container 运行在 Pod 中：

```
Pod
└── nginx Container
```

因此：

```
Image
    ≠
Container
    ≠
Pod
```

三者职责完全不同：

| 对象      | 作用                                       |
| --------- | ------------------------------------------ |
| Image     | 应用运行所需的软件包和文件系统模板         |
| Container | Image 的一个运行实例                       |
| Pod       | Kubernetes 管理和调度 Container 的基本单位 |

------

## 4.4 Pod 的生命周期

Pod 从创建到结束，大致经历：

```
Pod 创建
   ↓
Pending
   ↓
调度到 Node
   ↓
初始化
   ↓
Container 启动
   ↓
Running
   ↓
Container 退出
   ↓
Succeeded / Failed
```

Pod 的生命周期可以从两个角度理解。

第一种是运行过程：

```
创建
 ↓
调度
 ↓
启动
 ↓
运行
 ↓
终止
```

第二种是 Kubernetes 定义的 Pod Phase：

```
Pending
Running
Succeeded
Failed
Unknown
```

这里需要特别注意：

> **Pod Phase 与 `kubectl get pods` 显示的 STATUS 不是完全相同的概念。**

例如：

```
NAME   READY   STATUS
api    0/1     CrashLoopBackOff
```

`CrashLoopBackOff` 并不是 Pod Phase。

它描述的是 Container 反复启动失败后的状态。

------

### 4.4.1 Pod 创建阶段

执行：

```
kubectl apply -f pod.yaml
```

首先：

```
kubectl
   ↓
API Server
```

API Server 接收 Pod 对象。

然后 Kubernetes 将对象状态保存到 etcd：

```
API Server
   ↓
etcd
```

此时 Kubernetes 已经知道：

```
需要存在一个 Pod
```

但这时候 Pod 还不一定已经运行。

------

### 4.4.2 Pod 调度阶段

Scheduler 发现：

```
存在一个还没有 Node 的 Pod
```

然后根据调度规则选择 Worker Node。

例如：

```
Pod-A
   ↓
Scheduler
   ↓
Node-02
```

此时 Pod 被绑定到 Node-02。

------

### 4.4.3 Pod 启动阶段

Node-02 上的 kubelet 发现：

```
Pod-A 应该运行在 Node-02
```

然后调用 Container Runtime：

```
kubelet
   ↓
Container Runtime
   ↓
Pull Image
   ↓
Create Container
   ↓
Start Container
```

最终：

```
Pod
 ↓
Container Running
 ↓
Pod Running
```

------

### 4.4.4 Pod 终止阶段

删除 Pod：

```
kubectl delete pod nginx
```

Pod 通常不会瞬间消失，而是进入：

```
Terminating
```

随后 Kubernetes 会尝试优雅停止应用：

```
Pod
 ↓
SIGTERM
 ↓
Application Graceful Shutdown
 ↓
Container Stop
 ↓
Pod Deleted
```

如果应用在规定时间内没有退出，Kubernetes 最终会强制终止进程。

------

## 4.5 Pod IP

Pod 通常拥有一个独立的 IP 地址。

例如：

```
Pod-A
10.244.1.10

Pod-B
10.244.1.11
```

查看 Pod IP：

```
kubectl get pods -o wide
```

例如：

```
NAME   READY   STATUS    IP            NODE
api    1/1     Running   10.244.1.10   node-01
web    1/1     Running   10.244.1.11   node-02
```

这里：

```
10.244.1.10
```

就是 Pod-A 的 IP。

------

### 4.5.1 Pod IP 用于 Pod 间通信

例如：

```
Pod-A
10.244.1.10
   │
   │ HTTP
   ▼
Pod-B
10.244.1.11
```

在正常的 Kubernetes 网络模型下，Pod 可以直接与其他 Pod 通信。

具体网络实现由 CNI 网络插件负责。

------

### 4.5.2 Pod IP 不是稳定地址

这是生产环境必须掌握的概念。

假设：

```
Pod-A
10.244.1.10
```

Pod 被删除：

```
Pod-A
   ↓
Deleted
```

然后重新创建：

```
Pod-B
10.244.1.35
```

新的 Pod 很可能得到新的 IP。

因此不能依赖：

```
10.244.1.10
```

作为应用永久访问地址。

这也是 Kubernetes Service 存在的重要原因：

```
Client
   ↓
Service
   ↓
Pod
```

Service 提供稳定的访问入口，而 Pod 可以自由创建、删除和替换。

------

## 4.6 Pod 网络模型

Kubernetes Pod 网络模型建立在一个重要原则上：

> **每个 Pod 都拥有自己的网络命名空间，并通常拥有一个独立的 Pod IP。**

例如：

```
Node-01
│
├── Pod-A
│   IP: 10.244.1.10
│
└── Pod-B
    IP: 10.244.1.11
```

Pod-A 可以访问 Pod-B：

```
Pod-A
   │
   │ 10.244.1.11
   ▼
Pod-B
```

------

### 4.6.1 Pod 是网络通信的基本单位

从应用开发角度来看：

```
Pod
```

拥有自己的网络身份。

例如：

```
Pod IP = 10.244.1.10
```

Pod 中的 Container 共享这个网络环境。

因此：

```
Pod
├── App :8080
└── Sidecar :9090
```

对外表现为一个网络实体：

```
Pod IP
10.244.1.10
```

------

### 4.6.2 Pod 中多个 Container 共享网络

例如：

```
Pod
├── Application :8080
└── Sidecar     :9090
```

因为共享网络命名空间：

```
Application
    │
    └── localhost:9090
             ↓
          Sidecar
```

而 Sidecar：

```
Sidecar
    │
    └── localhost:8080
             ↓
        Application
```

这也是 Sidecar 模式能够工作的基础。

------

### 4.6.3 Pod 网络由 CNI 实现

Kubernetes 本身定义了 Pod 网络模型，但具体网络实现通常由 CNI 插件完成。

常见 CNI 包括：

- Cilium
- Calico
- Flannel

可以简单理解为：

```
Kubernetes
    ↓
定义网络模型
    ↓
CNI
    ↓
具体实现 Pod 网络
```

因此不同 Kubernetes 集群的底层网络实现可能完全不同。

------

## 4.7 Pod 中多个 Container

Pod 可以包含多个 Container：

```
Pod
├── Container-A
└── Container-B
```

这些 Container 具有非常紧密的运行关系。

它们通常：

- 运行在同一个 Node
- 共享网络
- 可以共享 Volume
- 属于同一个 Pod 生命周期

------

### 4.7.1 多 Container Pod 的典型场景

例如：

```
Pod
├── Application
└── Log Collector
```

Application：

```
负责业务
```

Log Collector：

```
负责日志采集
```

又例如：

```
Pod
├── Application
└── Proxy
```

Proxy：

```
负责网络代理
```

这些都是比较合理的多 Container Pod。

------

### 4.7.2 不应该把多个独立业务放进一个 Pod

例如：

```
Pod
├── Frontend
├── Backend
├── MySQL
└── Redis
```

这种设计通常是不合理的。

更常见的生产设计：

```
Pod
└── Frontend

Pod
└── Backend

Pod
└── MySQL

Pod
└── Redis
```

然后通过 Kubernetes 网络机制进行通信。

原因是不同应用通常具有不同的：

- 扩容需求
- 更新频率
- 资源需求
- 生命周期
- 故障影响范围

因此：

> **只有需要强绑定运行的 Container 才适合放在同一个 Pod。**

------

## 4.8 Init Container

Init Container 是专门用于 Pod 初始化工作的 Container。

它与普通 Container 最大的区别：

> **Init Container 必须先成功执行完成，普通 Container 才会启动。**

例如：

```
Pod
│
├── Init Container
│       ↓
│    初始化
│       ↓
│     完成
│
└── Application Container
```

------

### 4.8.1 Init Container 的典型用途

常见用途：

- 初始化文件
- 下载配置
- 检查依赖服务
- 等待数据库
- 执行初始化脚本
- 准备共享 Volume

例如：

```
Init Container
      ↓
检查数据库
      ↓
检查成功
      ↓
Application Container 启动
```

------

### 4.8.2 Init Container 示例

```
apiVersion: v1
kind: Pod

metadata:
  name: api

spec:
  initContainers:
    - name: init
      image: busybox
      command:
        - sh
        - -c
        - "echo initialization complete"

  containers:
    - name: api
      image: my-api:1.0
```

执行顺序：

```
init
 ↓
Completed
 ↓
api
 ↓
Running
```

如果 Init Container 失败：

```
init
 ↓
Failed
 ↓
重新执行
 ↓
Failed
 ↓
...
```

那么业务 Container 不会正常启动。

------

## 4.9 Sidecar Container

Sidecar 是 Kubernetes 多 Container Pod 中非常典型的一种设计模式。

基本结构：

```
Pod
├── Main Container
└── Sidecar Container
```

Main Container：

```
负责主要业务
```

Sidecar：

```
负责辅助功能
```

------

### 4.9.1 Sidecar 的典型用途

例如日志采集：

```
Pod
├── Application
└── Log Collector
```

Application：

```
产生日志
```

Log Collector：

```
读取日志
   ↓
发送到日志平台
```

又例如：

```
Pod
├── Application
└── Proxy
```

Proxy 可以负责：

- 流量代理
- TLS
- 服务间通信
- 监控
- 安全策略

------

### 4.9.2 Sidecar 为什么适合放在 Pod 中

因为 Sidecar 通常需要和主应用：

```
共享网络
共享文件
紧密协作
```

如果它们属于不同 Pod：

```
Pod-A
└── Application

Pod-B
└── Sidecar
```

就需要额外处理：

```
网络通信
服务发现
数据共享
生命周期协调
```

而放在同一个 Pod：

```
Pod
├── Application
└── Sidecar
```

这些问题会简单很多。

------

## 4.10 Pod YAML 结构

一个最简单的 Pod YAML：

```
apiVersion: v1
kind: Pod

metadata:
  name: nginx

spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

Pod YAML 的核心结构：

```
apiVersion
kind
metadata
spec
```

------

### 4.10.1 `apiVersion`

```
apiVersion: v1
```

表示使用的 Kubernetes API 版本。

Pod 属于 Kubernetes Core API，因此通常：

```
v1
```

------

### 4.10.2 `kind`

```
kind: Pod
```

表示 Kubernetes Object 类型。

这里：

```
kind = Pod
```

------

### 4.10.3 `metadata`

```
metadata:
  name: nginx
```

用于描述对象的元数据。

常见字段：

```
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
```

其中：

```
name
```

表示对象名称。

```
namespace
```

表示对象所属 Namespace。

```
labels
```

用于对象分类和 Selector 选择。

------

### 4.10.4 `spec`

```
spec:
```

定义 Pod 的期望状态。

例如：

```
spec:
  containers:
```

表达的意思是：

> 我希望这个 Pod 中运行这些 Container。

这就是 Kubernetes Desired State 思想在 Pod 上的体现。

------

### 4.10.5 `containers`

```
containers:
  - name: nginx
    image: nginx:1.27
```

定义 Container。

其中：

```
name
```

是 Container 名称。

```
image
```

是 Container 使用的镜像。

------

### 4.10.6 一个完整的 Pod 示例

```
apiVersion: v1
kind: Pod

metadata:
  name: nginx
  labels:
    app: nginx

spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

创建：

```
kubectl apply -f nginx.yaml
```

查看：

```
kubectl get pod nginx
```

查看详细信息：

```
kubectl describe pod nginx
```

查看日志：

```
kubectl logs nginx
```

------

## 4.11 Pod 创建、运行、终止过程

执行：

```
kubectl apply -f pod.yaml
```

从整个 Kubernetes 集群角度看：

```
kubectl
   ↓
API Server
   ↓
etcd
   ↓
Scheduler
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

------

### 4.11.1 `kubectl apply` 到 API Server

用户执行：

```
kubectl apply -f nginx.yaml
```

kubectl 向 API Server 发起请求：

```
kubectl
   ↓
API Server
```

API Server 负责：

- 身份认证
- 权限检查
- 请求验证
- 对象处理

------

### 4.11.2 API Server 到 etcd

API Server 将 Kubernetes 对象状态持久化到 etcd：

```
API Server
   ↓
etcd
```

etcd 保存：

```
Pod nginx
```

的 Kubernetes 状态信息。

------

### 4.11.3 Scheduler 选择 Node

Scheduler 发现：

```
存在一个尚未分配 Node 的 Pod
```

然后选择合适的 Worker Node：

```
Pod
 ↓
Scheduler
 ↓
Node-02
```

------

### 4.11.4 kubelet 创建 Container

Node-02 上的 kubelet 发现：

```
nginx Pod 应该运行在 Node-02
```

于是调用 Container Runtime：

```
kubelet
   ↓
Container Runtime
```

Container Runtime 负责：

```
Pull Image
Create Container
Start Container
```

------

### 4.11.5 Pod 进入 Running

最终：

```
Image
 ↓
Container
 ↓
Application
 ↓
Pod Running
```

这就是从：

```
kubectl apply
```

到：

```
Pod Running
```

的大致过程。

------

## 4.12 Pod 常见状态

查看：

```
kubectl get pods
```

可能看到：

```
NAME    READY   STATUS
api     1/1     Running
job     0/1     Completed
test    0/1     Error
nginx   0/1     Pending
redis   0/1     CrashLoopBackOff
```

需要区分两类概念。

第一类是 Pod Phase：

```
Pending
Running
Succeeded
Failed
Unknown
```

第二类是 `kubectl` 的状态展示：

```
CrashLoopBackOff
ImagePullBackOff
ErrImagePull
ContainerCreating
Terminating
Completed
```

所以：

```
CrashLoopBackOff
```

不能简单理解为 Pod Phase。

------

## 4.13 `Pending`

`Pending` 表示：

> Pod 已经被 Kubernetes 接受，但还没有进入正常运行状态。

常见原因：

```
Pod 尚未完成调度
Node 资源不足
PVC 未绑定
Node Selector 不匹配
Affinity 不满足
Taint / Toleration 不匹配
```

例如：

```
NAME   READY   STATUS
api    0/1     Pending
```

第一步通常执行：

```
kubectl describe pod api
```

重点查看：

```
Events
```

例如：

```
0/3 nodes are available: insufficient cpu
```

说明：

```
没有 Node 拥有足够的 CPU 资源
```

------

### 4.13.1 Pending 的基本排查方法

形成固定排查习惯：

```
Pod Pending
    ↓
kubectl describe pod
    ↓
查看 Events
    ↓
判断调度失败原因
    ↓
检查 Node / CPU / Memory / PVC / Affinity
```

------

## 4.14 `Running`

`Running` 表示：

> Pod 已经被绑定到 Node，并且至少有一个 Container 正在运行或正在启动。

例如：

```
NAME   READY   STATUS
api    1/1     Running
```

这里：

```
1/1
```

表示：

```
1 个 Container
1 个 Container Ready
```

但是：

> **Running 不代表业务一定正常。**

例如：

```
Pod
 ↓
Container Running
 ↓
Application
 ↓
HTTP 500
```

Pod 依然可能是：

```
Running
```

因此生产环境还需要：

```
Readiness Probe
Liveness Probe
Startup Probe
```

判断应用是否真正健康。

------

## 4.15 `Succeeded`

`Succeeded` 表示：

> Pod 中所有 Container 都成功执行完成，并且不会再重新启动。

典型应用场景：

```
Job
CronJob
```

例如：

```
数据库初始化任务
       ↓
执行成功
       ↓
Exit Code = 0
       ↓
Succeeded
```

`kubectl get pods` 中可能显示：

```
NAME       READY   STATUS      RESTARTS
db-init    0/1     Completed   0
```

这里的：

```
Completed
```

表示 Pod 已经成功完成。

------

## 4.16 `Failed`

`Failed` 表示：

> Pod 中至少有一个 Container 以失败状态结束，并且 Pod 已经进入终止状态。

例如：

```
Pod
 ↓
执行任务
 ↓
程序异常退出
 ↓
Exit Code ≠ 0
 ↓
Failed
```

典型场景：

```
Job
 ↓
执行数据库迁移
 ↓
迁移失败
 ↓
Pod Failed
```

排查：

```
kubectl describe pod <pod-name>
```

以及：

```
kubectl logs <pod-name>
```

------

## 4.17 `CrashLoopBackOff`

`CrashLoopBackOff` 是 Kubernetes 生产环境排障中非常常见的状态。

它表示：

> Container 启动后不断退出，kubelet 不断尝试重新启动，并在重启之间采用逐渐增加的等待时间。

典型过程：

```
Container 启动
     ↓
Application 崩溃
     ↓
Container Exit
     ↓
kubelet Restart
     ↓
Application 再次崩溃
     ↓
不断重启
     ↓
CrashLoopBackOff
```

------

### 4.17.1 CrashLoopBackOff 的常见原因

常见原因包括：

```
应用启动失败
配置错误
环境变量缺失
数据库连接失败
启动命令错误
程序 Bug
OOMKilled
Liveness Probe 失败
```

------

### 4.17.2 CrashLoopBackOff 的排查方法

首先：

```
kubectl logs <pod-name>
```

如果 Container 已经重启：

```
kubectl logs <pod-name> --previous
```

然后：

```
kubectl describe pod <pod-name>
```

重点观察：

```
Last State
Reason
Exit Code
Events
```

例如：

```
Last State:     Terminated
Reason:         OOMKilled
Exit Code:      137
```

说明 Container 很可能因为内存限制而被杀死。

------

## 4.18 `ImagePullBackOff`

`ImagePullBackOff` 表示：

> Kubernetes 无法成功拉取 Container Image，并开始采用退避策略重新尝试。

过程：

```
Pod 创建
   ↓
kubelet
   ↓
Pull Image
   ↓
失败
   ↓
重试
   ↓
失败
   ↓
ImagePullBackOff
```

------

### 4.18.1 ImagePullBackOff 的常见原因

#### 镜像名称错误

例如：

```
image: nginx-not-exist:1.0
```

------

#### Tag 不存在

例如：

```
image: my-api:999.999
```

Registry 中没有这个 Tag。

------

#### 私有 Registry 认证失败

例如：

```
Private Registry
      ↓
需要认证
      ↓
kubelet
```

这种情况下通常需要配置：

```
imagePullSecrets
```

------

#### Registry 无法访问

可能是：

- DNS 问题
- 网络问题
- 防火墙
- Registry 故障
- 代理配置问题

------

### 4.18.2 ImagePullBackOff 的排查方法

执行：

```
kubectl describe pod <pod-name>
```

重点查看：

```
Events
```

通常可以看到：

```
Failed to pull image
```

或者：

```
pull access denied
```

根据具体错误进一步检查：

```
Image
Tag
Registry
Credentials
Network
DNS
```

------

## 4.19 Pod 为什么会重启

执行：

```
kubectl get pods
```

可能看到：

```
NAME   READY   STATUS    RESTARTS
api    1/1     Running   15
```

这里的：

```
RESTARTS = 15
```

通常表示：

> Pod 中的 Container 已经重启过 15 次。

并不一定意味着：

```
Pod 被创建了 15 次
```

这是两个不同的概念。

------

### 4.19.1 Container 进程崩溃

最常见情况：

```
Application
    ↓
Exception
    ↓
Process Exit
    ↓
Container Exit
    ↓
kubelet Restart
```

例如应用启动时：

```
数据库连接失败
```

导致进程退出：

```
Exit Code ≠ 0
```

kubelet 可能重新启动 Container。

------

### 4.19.2 Liveness Probe 失败

如果配置了：

```
Liveness Probe
```

Kubernetes 会定期检查应用。

如果持续失败：

```
Application
    ↓
Liveness Probe Failed
    ↓
kubelet
    ↓
Restart Container
```

因此：

> **错误的 Liveness Probe 配置也可能造成应用不断重启。**

------

### 4.19.3 OOMKilled

Container 如果超过配置的内存限制：

```
Container
    ↓
Memory Usage
    ↓
超过 Memory Limit
    ↓
OOMKilled
    ↓
Container Restart
```

这是生产环境非常常见的重启原因。

------

### 4.19.4 Node 故障

如果 Node 出现问题：

```
Node
 ↓
NotReady
```

Pod 可能受到影响。

如果由 Deployment、StatefulSet 等 Workload Controller 管理，Kubernetes 可以根据控制器的期望状态重新创建应用实例。

因此要区分：

```
Container Restart
```

和：

```
Pod Recreated
```

这两个问题的排查方向不同。

------

## 4.20 Pod 为什么不是生产环境直接部署应用的最佳方式

虽然可以直接创建 Pod：

```
apiVersion: v1
kind: Pod
```

但是生产环境通常不会直接使用裸 Pod 部署长期运行的业务。

核心原因：

> **Pod 主要负责描述一个应用实例如何运行，而不是负责完整的应用生命周期管理。**

------

### 4.20.1 Pod 不负责副本数量管理

假设我们需要：

```
3 个 API 实例
```

理想状态：

```
Pod-A
Pod-B
Pod-C
```

裸 Pod 本身并没有：

```
Desired Replicas = 3
```

这样的副本管理能力。

这属于 ReplicaSet、Deployment 等控制器的职责。

------

### 4.20.2 Pod 不负责滚动更新

生产环境发布：

```
v1
 ↓
v2
```

通常希望：

```
旧 Pod
旧 Pod
旧 Pod
   ↓
逐步替换
   ↓
新 Pod
新 Pod
新 Pod
```

并且控制：

- 最大不可用 Pod 数量
- 最大额外 Pod 数量
- 更新过程
- 更新失败后的处理

这些属于 Deployment 等 Workload Controller 的职责。

------

### 4.20.3 Pod 不负责版本回滚

例如：

```
v1
 ↓
v2
 ↓
发现 v2 有 Bug
 ↓
回滚
 ↓
v1
```

裸 Pod 本身不具备完整的版本管理和回滚能力。

Deployment 则可以管理应用版本历史并执行回滚。

------

### 4.20.4 生产环境通常通过 Workload 管理 Pod

典型结构：

```
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Container
    ↓
Application
```

Deployment 负责更高层的应用生命周期：

```
副本管理
滚动更新
回滚
扩缩容
故障恢复
```

Pod 负责：

```
实际运行应用实例
```

所以可以记住：

> **Pod 是 Kubernetes 最核心的运行单元，但通常不是生产环境直接管理业务应用的最高层对象。**

------

## Pod 设计中的一个常见误区

初学 Kubernetes 时，很容易把 Pod 理解成：

```
Pod = Docker Container
```

这是不准确的。

更准确的理解是：

```
Image
  ↓
Container
  ↓
Pod
  ↓
Workload Controller
  ↓
Application
```

其中每一层解决不同的问题：

```
Image
→ 应用运行所需要的文件和环境

Container
→ 实际运行应用进程

Pod
→ 为 Container 提供 Kubernetes 运行边界

Deployment
→ 管理应用实例和版本

Service
→ 提供稳定的网络访问入口
```

这张关系非常重要，后面学习 Deployment、Service 时会反复用到。

------

## 本章核心认知

### Pod 的核心定位

可以用一句话记忆：

> **Pod 是 Kubernetes 最小的调度和运行单元，用来组织一个或多个需要紧密协作的 Container。**

### Pod 与 Container 的关系

```
Pod
├── Container
├── Network
└── Volume
```

Container 是实际执行应用的地方，而 Pod 是 Kubernetes 管理这些 Container 的边界。

### Pod 与生产环境的关系

不要形成：

```
生产应用
   ↓
直接创建 Pod
```

而应该逐渐形成：

```
生产应用
   ↓
Workload
   ↓
Pod
   ↓
Container
```

最典型的是：

```
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Container
    ↓
Application
```

### Pod 排障的基本思路

以后看到 Pod 异常，可以首先按照这个思路判断：

```
Pod 异常
   │
   ├── Pending
   │      ↓
   │   调度 / 资源 / PVC / Affinity
   │
   ├── CrashLoopBackOff
   │      ↓
   │   logs / previous / Events / Exit Code
   │
   ├── ImagePullBackOff
   │      ↓
   │   Image / Tag / Registry / Secret
   │
   └── RESTARTS 增加
          ↓
       应用崩溃
       Probe 失败
       OOMKilled
       Node 异常
```

最终建立这张完整的心智模型：

```
                    Kubernetes
                        │
                        ▼
                   Workload
                        │
                        ▼
                       Pod
                ┌───────┴───────┐
                ▼               ▼
           Container        Container
                │               │
                └───────┬───────┘
                        │
                  Shared Network
                  Shared Volume
                        │
                        ▼
                   Application
```

**Pod 是理解 Kubernetes 应用运行机制的核心。**后续学习 Deployment、StatefulSet、DaemonSet、Job、Service 等对象时，最终都会回到一个核心问题：

> **这些 Kubernetes 对象究竟是如何创建、管理和访问 Pod 的。**

# 第 5 章：Deployment——部署无状态应用

Deployment 是 Kubernetes 中最常用的 Workload 资源之一。

如果上一章学习的 Pod 解决的是：

> **“一个应用实例应该如何运行？”**

那么 Deployment 解决的是：

> **“一个无状态应用应该运行多少个实例、如何更新、如何扩缩容，以及出现问题后如何恢复？”**

在实际生产环境中，我们通常不会直接创建裸 Pod，而是通过 Deployment 管理 Pod。

最典型的关系是：

```
Deployment
    │
    ▼
ReplicaSet
    │
    ▼
Pod
    │
    ▼
Container
    │
    ▼
Application
```

------

## 5.1 Deployment 是什么

Deployment 是 Kubernetes 用于管理**无状态应用（Stateless Application）**的 Workload 资源。

它主要负责：

- 管理 Pod 副本数量
- 自动创建和删除 Pod
- 应用扩缩容
- 滚动更新
- 发布版本管理
- 回滚
- Pod 故障后的自动恢复

例如我们希望 API 服务始终运行 3 个实例：

```
Deployment
    │
    ├── Pod-A
    ├── Pod-B
    └── Pod-C
```

如果 Pod-A 因为某种原因被删除：

```
Deployment
    │
    ├── Pod-A ❌
    ├── Pod-B
    └── Pod-C
```

Deployment 会通过 ReplicaSet 发现：

```
期望：3 个 Pod
实际：2 个 Pod
```

然后创建新的 Pod：

```
Deployment
    │
    ├── Pod-B
    ├── Pod-C
    └── Pod-D
```

最终恢复：

```
3 个 Pod
```

因此 Deployment 的核心思想依然是 Kubernetes 的：

> **Desired State → Actual State → Reconciliation**

------

### 5.1.1 Deployment 主要解决什么问题

如果我们只使用裸 Pod：

```
Pod-A
Pod-B
Pod-C
```

需要自己处理：

```
Pod 数量
Pod 故障
扩容
缩容
更新
回滚
```

而使用 Deployment：

```
Deployment
    │
    ├── ReplicaSet
    │       ├── Pod
    │       ├── Pod
    │       └── Pod
```

Deployment 可以帮助我们管理这些事情。

因此生产环境的典型思维是：

```
不要直接管理 Pod
        ↓
管理 Deployment
        ↓
Deployment 管理 Pod
```

------

## 5.2 Deployment → ReplicaSet → Pod

这是本章最重要的知识点之一。

完整关系：

```
Deployment
      │
      ▼
 ReplicaSet
      │
      ▼
    Pod
      │
      ▼
 Container
```

这几个对象各自负责不同的事情。

------

### 5.2.1 Deployment 负责应用版本和发布策略

Deployment 位于最高层。

例如：

```
Deployment
├── replicas: 3
├── image: api:v2
└── strategy: RollingUpdate
```

它描述：

```
我要运行 3 个 API 实例
使用 v2 镜像
更新时采用滚动更新
```

------

### 5.2.2 ReplicaSet 负责 Pod 副本数量

ReplicaSet 更关注：

> **当前应该存在多少个符合条件的 Pod。**

例如：

```
ReplicaSet
replicas = 3
```

它会持续检查：

```
Desired = 3
Actual  = 3
```

如果：

```
Desired = 3
Actual  = 2
```

ReplicaSet 就会创建 Pod。

如果：

```
Desired = 3
Actual = 4
```

则会删除多余 Pod。

------

### 5.2.3 Pod 负责真正运行应用

最终：

```
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Container
    ↓
Application
```

Deployment 不直接启动 Container。

ReplicaSet 也不直接启动 Container。

真正运行应用的是：

```
Pod
  ↓
Container
```

------

### 5.2.4 为什么需要 ReplicaSet

可能会产生一个问题：

> Deployment 为什么不直接管理 Pod？

因为 Kubernetes 将职责进行了拆分。

```
Deployment
    ↓
负责发布和版本
    ↓
ReplicaSet
    ↓
负责副本数量
    ↓
Pod
    ↓
负责运行
```

这样可以让 Deployment 在更新过程中同时管理多个 ReplicaSet。

例如：

```
Deployment
│
├── ReplicaSet-v1
│     ├── Pod
│     └── Pod
│
└── ReplicaSet-v2
      ├── Pod
      └── Pod
```

这正是滚动更新和回滚能够实现的基础。

------

## 5.3 Replicas

`replicas` 表示：

> **Deployment 期望运行多少个 Pod 副本。**

例如：

```
spec:
  replicas: 3
```

表示：

```
Desired Pods = 3
```

最终 Kubernetes 会努力维持：

```
Pod-A
Pod-B
Pod-C
```

------

### 5.3.1 Replicas 与高可用

假设：

```
replicas: 1
```

只有：

```
Pod-A
```

如果 Pod-A 出现故障：

```
Pod-A ❌
```

应用会出现短暂甚至较长时间的不可用。

如果：

```
replicas: 3
```

则：

```
Pod-A
Pod-B
Pod-C
```

即使一个 Pod 出现故障：

```
Pod-A ❌

Pod-B ✅
Pod-C ✅
```

仍然存在两个实例。

但需要注意：

> **多个 replicas 不等于完整的高可用。**

如果 3 个 Pod 全部运行在同一个 Node：

```
Node-01
├── Pod-A
├── Pod-B
└── Pod-C
```

Node-01 故障时：

```
3 个 Pod
   ↓
全部受影响
```

因此生产环境还需要结合：

- Node 调度
- Pod Anti-Affinity
- Topology Spread Constraints
- 多节点
- 多可用区

等机制设计真正的高可用。

------

### 5.3.2 Replicas 是期望状态

例如：

```
replicas: 5
```

并不是简单表示：

> “现在有 5 个 Pod。”

而是：

> **“我希望最终保持 5 个符合要求的 Pod。”**

例如当前：

```
Desired = 5
Actual = 3
```

Deployment / ReplicaSet 会继续创建 Pod。

最终：

```
Desired = 5
Actual = 5
```

这就是 Kubernetes 的控制循环思想。

------

## 5.4 Deployment YAML

一个最基础的 Deployment：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

这是生产环境学习 Deployment 时必须掌握的基本结构。

------

### 5.4.1 `apiVersion`

```
apiVersion: apps/v1
```

Deployment 使用：

```
apps/v1
```

API Group。

------

### 5.4.2 `kind`

```
kind: Deployment
```

表示 Kubernetes Object 类型是：

```
Deployment
```

------

### 5.4.3 `metadata`

```
metadata:
  name: nginx
```

表示 Deployment 名称：

```
nginx
```

还可以配置：

```
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
```

------

### 5.4.4 `spec.replicas`

```
spec:
  replicas: 3
```

表示期望：

```
3 个 Pod
```

------

### 5.4.5 `spec.selector`

```
selector:
  matchLabels:
    app: nginx
```

表示：

> Deployment 管理哪些 Pod？

它通过 Label Selector 找到自己的 Pod。

这里的：

```
app: nginx
```

必须与 Pod Template 中的 Label 对应：

```
template:
  metadata:
    labels:
      app: nginx
```

也就是：

```
Deployment Selector
        │
        │ app=nginx
        ▼
Pod Label
        │
        ▼
Pod
```

这是 Deployment YAML 中非常重要的关系。

------

### 5.4.6 `spec.template`

```
template:
  metadata:
    labels:
      app: nginx

  spec:
    containers:
      - name: nginx
        image: nginx:1.27
```

`template` 是：

> **Pod Template，也就是 Pod 模板。**

Deployment 并不是让你一个一个定义 Pod。

而是告诉 Kubernetes：

```
以后创建出来的 Pod
应该按照这个模板创建
```

所以：

```
Deployment
   │
   ▼
Pod Template
   │
   ▼
多个 Pod
```

------

### 5.4.7 Deployment 与 Pod Template 的关系

例如：

```
spec:
  replicas: 3

  template:
    spec:
      containers:
        - name: api
          image: my-api:v1
```

最终可能得到：

```
Deployment
    │
    └── ReplicaSet
          │
          ├── Pod-1 → my-api:v1
          ├── Pod-2 → my-api:v1
          └── Pod-3 → my-api:v1
```

------

## 5.5 创建和扩缩容

创建 Deployment：

```
kubectl apply -f deployment.yaml
```

查看：

```
kubectl get deployment
```

例如：

```
NAME    READY   UP-TO-DATE   AVAILABLE
nginx   3/3     3            3
```

查看 ReplicaSet：

```
kubectl get rs
```

查看 Pod：

```
kubectl get pods
```

可能得到：

```
NAME                     READY   STATUS
nginx-7c8f7d6b5d-abcde   1/1     Running
nginx-7c8f7d6b5d-fghij   1/1     Running
nginx-7c8f7d6b5d-klmno   1/1     Running
```

------

### 5.5.1 使用 `kubectl scale` 扩容

当前：

```
replicas = 3
```

扩容到 5：

```
kubectl scale deployment nginx --replicas=5
```

然后：

```
kubectl get pods
```

最终：

```
Pod-1
Pod-2
Pod-3
Pod-4
Pod-5
```

------

### 5.5.2 使用 `kubectl scale` 缩容

从 5 缩减到 2：

```
kubectl scale deployment nginx --replicas=2
```

Kubernetes 会逐步删除多余 Pod。

最终：

```
Pod-1
Pod-2
```

------

### 5.5.3 直接修改 YAML

也可以修改：

```
spec:
  replicas: 5
```

然后：

```
kubectl apply -f deployment.yaml
```

这种方式更适合 IaC / GitOps 工作流，因为最终配置文件可以作为应用部署状态的源代码。

------

## 5.6 滚动更新

滚动更新（Rolling Update）是 Deployment 最重要的能力之一。

假设当前：

```
v1
v1
v1
```

现在发布：

```
v2
```

我们希望：

```
v1
v1
v1

↓
```

逐渐变成：

```
v1
v1
v2

↓

v1
v2
v2

↓

v2
v2
v2
```

而不是一次性：

```
v1
v1
v1
   ↓
全部删除
   ↓
v2
v2
v2
```

这种逐步替换就是 Rolling Update。

------

### 5.6.1 滚动更新的核心目标

滚动更新主要解决：

```
旧版本
   ↓
逐步替换
   ↓
新版本
```

过程中尽可能：

- 保持服务可用
- 控制同时运行的 Pod 数量
- 控制不可用 Pod 数量
- 降低发布风险

------

## 5.7 Rollout

Rollout 可以理解为：

> **Deployment 对应用新版本进行发布和逐步替换的过程。**

例如：

```
v1
 ↓
开始 Rollout
 ↓
v2
 ↓
逐步替换 Pod
 ↓
Rollout 完成
```

查看：

```
kubectl rollout status deployment/nginx
```

可能看到：

```
deployment "nginx" successfully rolled out
```

------

### 5.7.1 查看 Rollout 状态

```
kubectl rollout status deployment/nginx
```

如果更新过程中：

```
3 个旧 Pod
```

逐渐变成：

```
2 个旧 Pod
1 个新 Pod
```

再变成：

```
1 个旧 Pod
2 个新 Pod
```

最终：

```
3 个新 Pod
```

------

### 5.7.2 Rollout 与 ReplicaSet

Deployment 更新时，不会简单修改所有现有 Pod。

而是创建新的 ReplicaSet：

```
Deployment
│
├── ReplicaSet-v1
│     ├── Pod
│     ├── Pod
│     └── Pod
│
└── ReplicaSet-v2
      ├── Pod
      └── Pod
```

然后逐渐：

```
v1 replicas: 3 → 2 → 1 → 0

v2 replicas: 0 → 1 → 2 → 3
```

因此滚动更新实际上是：

> **Deployment 控制两个版本的 ReplicaSet，逐步调整它们的副本数量。**

------

## 5.8 Rollback

Rollback 就是：

> **将 Deployment 回滚到之前的版本。**

例如：

```
v1
 ↓
发布 v2
 ↓
发现 Bug
 ↓
Rollback
 ↓
v1
```

执行：

```
kubectl rollout undo deployment/nginx
```

Deployment 会尝试恢复之前的版本。

------

### 5.8.1 为什么需要 Rollback

生产环境发布非常容易遇到：

```
代码 Bug
配置错误
依赖错误
环境变量错误
数据库兼容问题
```

如果新版本上线：

```
v2
```

导致大量请求失败，可以快速：

```
kubectl rollout undo deployment/nginx
```

恢复旧版本。

因此：

> **滚动更新负责发布，Rollback 负责快速恢复。**

------

## 5.9 Deployment Strategy

Deployment 的更新策略主要有：

```
RollingUpdate
Recreate
```

配置方式：

```
strategy:
  type: RollingUpdate
```

或者：

```
strategy:
  type: Recreate
```

两者最大的区别：

```
RollingUpdate
旧版本和新版本可以同时存在
```

而：

```
Recreate
先删除旧版本，再创建新版本
```

------

## 5.10 `RollingUpdate`

`RollingUpdate` 是 Deployment 默认策略。

例如：

```
strategy:
  type: RollingUpdate
```

发布过程中：

```
Old Pods
   +
New Pods
```

可以同时存在。

例如：

```
v1
v1
v1
```

逐渐变成：

```
v1
v1
v2
```

再：

```
v1
v2
v2
```

最后：

```
v2
v2
v2
```

------

### 5.10.1 RollingUpdate 的优点

主要优点：

- 发布过程中可以保持服务
- 不需要一次性删除全部 Pod
- 可以控制更新速度
- 发布风险较低

因此：

> **无状态 API、Web 服务通常优先使用 RollingUpdate。**

------

## 5.11 `Recreate`

Recreate 策略：

```
strategy:
  type: Recreate
```

更新时：

```
旧 Pod
旧 Pod
旧 Pod
   ↓
全部删除
   ↓
新 Pod
新 Pod
新 Pod
```

也就是说：

> **先删除旧版本所有 Pod，再创建新版本 Pod。**

------

### 5.11.1 Recreate 的特点

优点：

```
旧版本和新版本不会同时存在
```

缺点：

```
发布过程中会存在服务不可用
```

例如：

```
v1
v1
v1

↓ 删除

无 Pod

↓ 创建

v2
v2
v2
```

因此一般不适合要求持续可用的在线 API 服务。

------

### 5.11.2 什么场景可能使用 Recreate

例如某些应用：

```
v1 和 v2 不能同时运行
```

可能因为：

- 使用独占资源
- 两个版本不兼容
- 旧版本和新版本同时运行会产生数据问题

这种情况下可以考虑 Recreate。

------

## 5.12 `maxSurge`

`maxSurge` 用于控制：

> **滚动更新过程中，最多可以比期望副本数多创建多少个 Pod。**

假设：

```
replicas: 3
```

配置：

```
maxSurge: 1
```

意味着更新期间最多可以：

```
3 + 1 = 4
```

个 Pod。

例如：

```
旧 Pod × 3
```

更新开始：

```
旧 × 3
新 × 1
```

总数：

```
4
```

然后再逐步删除旧 Pod。

------

### 5.12.1 `maxSurge` 可以使用整数

例如：

```
maxSurge: 2
```

表示：

```
最多额外增加 2 个 Pod
```

如果：

```
replicas = 5
```

那么最多：

```
5 + 2 = 7
```

个 Pod。

------

### 5.12.2 `maxSurge` 可以使用百分比

例如：

```
maxSurge: 25%
```

如果：

```
replicas = 4
```

则最多允许一定比例的额外 Pod。

生产环境经常使用：

```
maxSurge: 25%
```

------

## 5.13 `maxUnavailable`

`maxUnavailable` 用于控制：

> **滚动更新期间，最多允许多少个 Pod 不可用。**

例如：

```
replicas: 4

strategy:
  rollingUpdate:
    maxUnavailable: 1
```

意味着更新期间最多允许：

```
1 个 Pod
```

处于不可用状态。

例如：

```
旧 × 4
```

更新过程中：

```
旧 × 3
新 × 1
```

或者某个阶段：

```
旧 × 2
新 × 2
```

但 Kubernetes 会按照策略控制不可用数量。

------

### 5.13.1 `maxUnavailable` 的作用

可以简单理解：

```
maxSurge
    ↓
最多允许多多少 Pod

maxUnavailable
    ↓
最多允许少多少可用 Pod
```

记忆：

```
Surge = 超出的
Unavailable = 不可用的
```

------

### 5.13.2 `maxSurge` 与 `maxUnavailable` 配合

例如：

```
replicas: 10

strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 2
    maxUnavailable: 1
```

可以理解成：

```
期望：10 个 Pod

最多：
10 + 2 = 12 个 Pod

同时：
最多允许 1 个 Pod 不可用
```

这两个参数共同控制 Rolling Update 的节奏。

------

## 5.14 Deployment 更新过程中发生了什么

这是本章最重要的实战理解之一。

假设现在 Deployment：

```
replicas: 3
image: api:v1
```

当前：

```
Deployment
    │
    ▼
ReplicaSet-v1
    │
    ├── Pod-v1-A
    ├── Pod-v1-B
    └── Pod-v1-C
```

现在修改：

```
api:v1
```

为：

```
api:v2
```

执行：

```
kubectl apply -f deployment.yaml
```

------

### 5.14.1 Deployment 发现 Pod Template 发生变化

Deployment Controller 发现：

```
旧 Template
image: api:v1
```

变成：

```
新 Template
image: api:v2
```

由于 Pod Template 发生变化，Deployment 创建新的 ReplicaSet：

```
Deployment
│
├── ReplicaSet-v1
│     ├── Pod-v1
│     ├── Pod-v1
│     └── Pod-v1
│
└── ReplicaSet-v2
```

------

### 5.14.2 新 ReplicaSet 开始创建 Pod

例如：

```
ReplicaSet-v2
    ↓
创建 Pod-v2
```

变成：

```
v1
v1
v1
v2
```

如果：

```
maxSurge: 1
```

最多允许额外一个 Pod。

------

### 5.14.3 新 Pod Ready 后继续更新

假设 Pod-v2 已经 Ready：

```
v1
v1
v1
v2 ✅
```

Deployment 可以继续减少旧 ReplicaSet：

```
v1
v1
v2
v2
```

再：

```
v1
v2
v2
v2
```

最终：

```
v2
v2
v2
```

------

### 5.14.4 最终状态

最终：

```
Deployment
│
├── ReplicaSet-v1
│     └── replicas: 0
│
└── ReplicaSet-v2
      └── replicas: 3
```

注意：

> **旧 ReplicaSet 通常不会立即被删除，而是保留用于发布历史和回滚。**

------

## 5.15 如何查看发布历史

查看 Deployment 的 Rollout History：

```
kubectl rollout history deployment/nginx
```

例如：

```
deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```

表示 Deployment 有多个修订版本。

------

### 5.15.1 查看某一个版本

例如：

```
kubectl rollout history deployment/nginx --revision=2
```

可以查看这个 Revision 的详细信息。

------

### 5.15.2 Revision 是什么

可以简单理解：

```
Revision 1
    ↓
v1

Revision 2
    ↓
v2

Revision 3
    ↓
v3
```

它是 Deployment 发布历史中的一个版本记录。

需要注意：

> Revision 并不简单等于“镜像版本号”。

例如：

```
Revision 3
```

并不意味着：

```
image:v3
```

两者是不同概念。

------

## 5.16 如何回滚

查看历史：

```
kubectl rollout history deployment/nginx
```

然后回滚到指定版本：

```
kubectl rollout undo deployment/nginx --to-revision=2
```

或者直接回滚到上一个版本：

```
kubectl rollout undo deployment/nginx
```

------

### 5.16.1 回滚并不是删除 Deployment

执行：

```
kubectl rollout undo deployment/nginx
```

不会删除：

```
Deployment
```

而是让 Deployment 恢复之前的 Pod Template。

例如：

```
当前：

Deployment
 ↓
api:v3
```

回滚：

```
Deployment
 ↓
api:v2
```

然后 Deployment 再执行一次 Rollout：

```
v3
 ↓
v2
```

------

### 5.16.2 回滚后应该检查状态

执行：

```
kubectl rollout status deployment/nginx
```

然后：

```
kubectl get pods
```

再根据需要：

```
kubectl describe deployment nginx
```

查看事件：

```
kubectl describe pod <pod-name>
```

生产环境不要仅仅执行：

```
kubectl rollout undo
```

然后认为问题已经解决。

还需要验证：

```
Pod 是否 Ready
应用是否正常
接口是否正常
错误率是否恢复
```

------

## 5.17 无状态 API 服务的标准部署方式

这是 Deployment 在实际工作中最典型的场景。

假设我们有一个：

```
.NET / Java / Node.js / Go
```

编写的 HTTP API。

例如：

```
api:v1.0.0
```

标准 Kubernetes 部署通常不是：

```
直接创建 Pod
```

而是：

```
Deployment
    ↓
ReplicaSet
    ↓
多个 Pod
    ↓
Container
    ↓
API Application
```

然后：

```
Client
   ↓
Service
   ↓
Deployment 管理的 Pod
```

------

### 5.17.1 标准架构

一个简单的生产架构：

```
                    Client
                      │
                      ▼
                   Ingress
                      │
                      ▼
                   Service
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
           Pod      Pod      Pod
             │        │        │
             └────────┼────────┘
                      │
                 API Container
```

Deployment 位于 Pod 管理层：

```
Deployment
     │
     ▼
ReplicaSet
     │
     ├── Pod
     ├── Pod
     └── Pod
```

------

### 5.17.2 一个基础的 API Deployment

例如：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: api

spec:
  replicas: 3

  strategy:
    type: RollingUpdate

    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: api

  template:
    metadata:
      labels:
        app: api

    spec:
      containers:
        - name: api
          image: my-api:1.0.0

          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: 100m
              memory: 128Mi

            limits:
              cpu: 500m
              memory: 512Mi
```

这个配置表达了：

```
3 个 API 实例
```

使用：

```
my-api:1.0.0
```

更新策略：

```
RollingUpdate
```

并且：

```
maxSurge = 1
maxUnavailable = 0
```

意味着更新过程中尽量保证已有服务能力不下降。

------

### 5.17.3 为什么无状态 API 非常适合 Deployment

假设：

```
Pod-A
Pod-B
Pod-C
```

三个 Pod 都运行：

```
api:v1
```

任何一个 Pod 被删除：

```
Pod-A ❌
```

Deployment / ReplicaSet 会创建：

```
Pod-D
```

因此应用实例可以被替换。

这就是无状态应用的重要特征：

> **请求处理不依赖某一个固定 Pod 的本地状态。**

例如用户请求：

```
Request-1 → Pod-A
Request-2 → Pod-B
Request-3 → Pod-C
```

都可以正常处理。

------

### 5.17.4 无状态应用的关键特征

典型无状态 API：

```
Pod-A
Pod-B
Pod-C
```

这些实例之间应该尽可能做到：

```
代码相同
配置一致
运行环境一致
可以互相替换
```

不要依赖：

```
Pod-A 本地文件
Pod-A 内存中的 Session
Pod-A 固定 IP
Pod-A 固定名称
```

如果确实需要持久化状态，应使用：

```
Database
Redis
Object Storage
Persistent Volume
```

等专门的数据存储机制。

------

### 5.17.5 Deployment、Service、Ingress 的职责

学习到这里，可以先建立一个非常重要的架构模型：

```
                Ingress
                   │
                   ▼
                Service
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
         Pod      Pod      Pod
          ▲        ▲        ▲
          └────────┼────────┘
                   │
              Deployment
```

它们的职责不同：

| 对象       | 核心职责                       |
| ---------- | ------------------------------ |
| Deployment | 管理应用版本、副本、更新和回滚 |
| ReplicaSet | 保证 Pod 副本数量              |
| Pod        | 运行应用实例                   |
| Service    | 为 Pod 提供稳定的网络访问入口  |
| Ingress    | 管理 HTTP/HTTPS 外部访问入口   |

因此不要把这些概念混在一起。

------

## Deployment 中一个非常重要的认知

### Deployment 管理的是“应用期望状态”，不是某几个固定 Pod

例如：

```
Deployment
replicas: 3
image: api:v2
```

它表达的是：

> **我希望系统最终拥有 3 个运行 api:v2 的 Pod。**

至于具体是哪三个 Pod：

```
Pod-A
Pod-B
Pod-C
```

并不重要。

Pod-A 删除后：

```
Pod-B
Pod-C
Pod-D
```

仍然满足：

```
3 个 api:v2 Pod
```

这正是 Kubernetes 与传统服务器运维思维非常不同的地方。

------

## Deployment 的核心认知

把本章浓缩成一张图：

```
                    Deployment
                         │
              ┌──────────┴──────────┐
              │                     │
          Replicas             Deployment Strategy
              │                     │
              ▼                     ├── RollingUpdate
         ReplicaSet                 └── Recreate
              │
              ▼
             Pods
              │
              ▼
          Containers
              │
              ▼
         Applications
```

更新时：

```
Deployment
    │
    ├── ReplicaSet-v1
    │      ├── Pod-v1
    │      ├── Pod-v1
    │      └── Pod-v1
    │
    └── ReplicaSet-v2
           ├── Pod-v2
           ├── Pod-v2
           └── Pod-v2
```

RollingUpdate：

```
v1 × 3
  ↓
v1 × 2 + v2 × 1
  ↓
v1 × 1 + v2 × 2
  ↓
v2 × 3
```

如果新版本出现问题：

```
v1
 ↓
v2 ❌
 ↓
kubectl rollout undo
 ↓
v1
```

所以对于一个**无状态 API 服务**，可以形成最基本的生产思维：

```
Application
    ↓
Container Image
    ↓
Deployment
    ↓
ReplicaSet
    ↓
Multiple Pods
    ↓
Service
    ↓
External Traffic
```

其中 Deployment 是负责**“应用实例生命周期管理”**的核心对象。

# 第 6 章：StatefulSet——部署有状态应用

前面第 4 章学习了 Pod，第 5 章学习了 Deployment。

Deployment 非常适合部署这样的应用：

> “给我运行 3 个完全一样的 Web Pod，哪个 Pod 挂了就重新创建一个。”

例如：

```
                Deployment
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       web-xxx    web-xxx    web-xxx
          │         │         │
       无状态      无状态      无状态
```

但数据库、Redis、Kafka 等应用通常不是这种模式。

它们往往需要：

- Pod 有稳定、可预测的身份
- Pod 重启后仍然能够识别自己
- 每个 Pod 拥有自己的数据
- Pod 之间存在固定的角色或顺序
- 创建、删除 Pod 时需要遵循一定顺序

这就是 StatefulSet 解决的问题。

------

## 6.1 StatefulSet 是什么

### 6.1.1 StatefulSet 的定义

StatefulSet 是 Kubernetes 用来管理**有状态应用（Stateful Application）**的一种工作负载控制器。

它主要解决有状态应用的几个核心问题：

```
                    StatefulSet
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
  稳定的身份          稳定的网络          稳定的存储
       │                 │                 │
       ▼                 ▼                 ▼
  mysql-0             mysql-0            mysql-0
  mysql-1             mysql-1            mysql-1
  mysql-2             mysql-2            mysql-2
```

与 Deployment 最大的区别在于：

> **StatefulSet 管理的 Pod 具有稳定的、与 Pod 生命周期相对独立的身份。**

例如一个 StatefulSet：

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: nginx
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

创建：

```
kubectl apply -f nginx-statefulset.yaml
```

查看：

```
kubectl get pods
```

可能看到：

```
NAME      READY   STATUS    RESTARTS
nginx-0   1/1     Running   0
nginx-1   1/1     Running   0
nginx-2   1/1     Running   0
```

注意这里的 Pod 名称：

```
nginx-0
nginx-1
nginx-2
```

这不是随机生成的。

这正是 StatefulSet 最重要的特征之一。

------

## 6.2 StatefulSet 与 Deployment 的区别

### 6.2.1 Pod 名称不同

Deployment 创建的 Pod 通常类似：

```
web-7d9f8c7f5b-x2k4m
web-7d9f8c7f5b-p8q7r
web-7d9f8c7f5b-h9j2k
```

Pod 名称包含随机后缀。

如果删除：

```
kubectl delete pod web-7d9f8c7f5b-x2k4m
```

Deployment 会创建一个新的 Pod：

```
web-7d9f8c7f5b-z6t8p
```

旧 Pod：

```
web-7d9f8c7f5b-x2k4m
```

消失以后，不会再回来。

Deployment 关注的是：

> **保持 Pod 数量和版本符合期望状态。**

------

StatefulSet 则不同。

例如：

```
mysql-0
mysql-1
mysql-2
```

删除：

```
kubectl delete pod mysql-1
```

StatefulSet 会重新创建：

```
mysql-1
```

而不是：

```
mysql-xxxxx
```

因此可以认为：

```
Deployment：

Pod A ──删除──> Pod B
身份发生变化


StatefulSet：

Pod A ──删除──> Pod A'
身份保持稳定
```

这里的 `A'` 虽然是一个全新的 Pod 对象，但它继承了 StatefulSet 的稳定身份。

------

### 6.2.2 Deployment 与 StatefulSet 的核心区别

可以用一个表格理解：

| 特性       | Deployment                | StatefulSet             |
| ---------- | ------------------------- | ----------------------- |
| 主要用途   | 无状态应用                | 有状态应用              |
| Pod 名称   | 随机后缀                  | 固定序号                |
| Pod 身份   | 不稳定                    | 稳定                    |
| 网络身份   | 通常不固定                | 可以固定                |
| 存储       | 通常共享或不依赖 Pod 身份 | 每个 Pod 可拥有独立存储 |
| Pod 顺序   | 一般不关注                | 可以严格控制            |
| Pod 删除后 | 新 Pod 身份通常变化       | 原序号恢复              |
| 扩缩容顺序 | 通常不强调                | 有明确顺序              |
| 典型应用   | Web、API、前端            | 数据库、集群型中间件    |

但是要特别注意：

> **StatefulSet 并不意味着“自动拥有数据库能力”。**

Kubernetes 只负责提供：

- 稳定身份
- 生命周期管理
- 网络身份
- 存储绑定
- 顺序控制

数据库本身的：

- 主从复制
- Leader 选举
- 数据一致性
- 故障转移
- 集群协议

仍然需要数据库自身或者 Operator 来解决。

------

## 6.3 稳定的 Pod 名称

### 6.3.1 StatefulSet 的 Pod 命名规则

假设：

```
metadata:
  name: mysql
```

并且：

```
spec:
  replicas: 3
```

那么 Pod 名称为：

```
mysql-0
mysql-1
mysql-2
```

基本规则：

```
<StatefulSet名称>-<序号>
```

序号从：

```
0
```

开始。

------

### 6.3.2 删除 Pod 后名称保持不变

例如：

```
kubectl delete pod mysql-1
```

查看：

```
kubectl get pods -w
```

可能看到：

```
mysql-1   Terminating
mysql-1   Pending
mysql-1   ContainerCreating
mysql-1   Running
```

最终还是：

```
mysql-1
```

这对于有状态系统非常重要。

例如某个数据库集群知道：

```
mysql-0 = 主节点
mysql-1 = 从节点
mysql-2 = 从节点
```

那么即使 `mysql-1` 的 Pod 被删除并重新创建，它仍然可以继续使用：

```
mysql-1
```

这个稳定身份。

------

### 6.3.3 StatefulSet 的序号具有实际意义

StatefulSet 的序号不是简单的名字。

例如：

```
redis-0
redis-1
redis-2
```

很多分布式系统可以利用这个稳定身份建立：

```
redis-0
    │
    ├── redis-1
    │
    └── redis-2
```

或者：

```
node-0
node-1
node-2
```

对应：

```
节点 0
节点 1
节点 2
```

因此 StatefulSet 的 Pod 身份是整个模型的一部分。

------

## 6.4 稳定的网络身份

稳定的 Pod 名称还不够。

假设：

```
mysql-0
mysql-1
mysql-2
```

如果每次 Pod 重建以后 IP 地址发生变化，那么其他 Pod 仍然无法稳定访问：

```
mysql-0
```

因此 StatefulSet 还需要解决：

> **如何让每个 Pod 拥有稳定的网络身份？**

这通常需要：

**Headless Service。**

------

## 6.5 稳定的存储

### 6.5.1 为什么 StatefulSet 需要稳定存储

这是有状态应用最关键的问题之一。

假设：

```
mysql-0
```

里面有：

```
/data/mysql/
```

保存数据库：

```
users
orders
products
```

现在 Pod 被删除。

如果重新创建一个 Pod 时，数据也消失：

```
mysql-0
   │
   ├── users
   ├── orders
   └── products
          ↓
       Pod 删除
          ↓
       数据消失
```

那么数据库就无法正常工作。

所以我们需要：

```
Pod
 │
 └── PVC
      │
      └── PV
           │
           └── 实际存储
```

StatefulSet 可以为每一个 Pod 创建独立的 PVC。

------

### 6.5.2 volumeClaimTemplates

例如：

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql

spec:
  serviceName: mysql
  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8.4

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "example-password"

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: mysql-data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 10Gi
```

这里最重要的是：

```
volumeClaimTemplates:
```

它相当于告诉 Kubernetes：

> 每创建一个 StatefulSet Pod，就为它创建一个属于自己的 PVC。

最终类似：

```
mysql-0
   │
   └── mysql-data-mysql-0

mysql-1
   │
   └── mysql-data-mysql-1

mysql-2
   │
   └── mysql-data-mysql-2
```

可以查看：

```
kubectl get pvc
```

可能看到：

```
NAME                  STATUS   VOLUME
mysql-data-mysql-0    Bound    pvc-xxx
mysql-data-mysql-1    Bound    pvc-yyy
mysql-data-mysql-2    Bound    pvc-zzz
```

这意味着：

```
mysql-0 → PVC-0
mysql-1 → PVC-1
mysql-2 → PVC-2
```

而不是：

```
所有 Pod → 同一个 PVC
```

------

### 6.5.3 删除 Pod 不等于删除 PVC

这是生产环境非常重要的概念。

执行：

```
kubectl delete pod mysql-0
```

一般情况下：

```
mysql-0      → 删除
mysql-data-mysql-0 → 保留
```

新的：

```
mysql-0
```

重新启动以后，可以重新使用：

```
mysql-data-mysql-0
```

所以：

```
Pod 生命周期
      ≠
数据生命周期
```

这正是 StatefulSet 适合有状态应用的重要原因。

------

### 6.5.4 StatefulSet 删除后的存储处理

需要特别注意：

> **删除 StatefulSet 不应该简单理解为“数据也会自动删除”。**

具体 PVC/PV 是否删除，还涉及：

- PVC 生命周期
- PV 生命周期
- StorageClass
- reclaimPolicy
- Kubernetes 版本及 StatefulSet PVC 保留策略

生产环境删除数据库 StatefulSet 前，必须明确：

```
哪些东西可以删除？
哪些数据必须保留？
备份在哪里？
恢复流程是什么？
```

不要直接：

```
kubectl delete statefulset mysql
```

然后再考虑数据怎么办。

------

## 6.6 Ordered Deployment

StatefulSet 一个非常重要的特点是：

> **Pod 可以按照严格顺序创建。**

默认情况下，StatefulSet 使用：

```
podManagementPolicy: OrderedReady
```

------

### 6.6.1 OrderedReady 是什么

假设：

```
replicas: 3
```

StatefulSet：

```
mysql-0
mysql-1
mysql-2
```

创建时默认按照：

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
   ↓
Ready
```

而不是：

```
mysql-0 ─┐
mysql-1 ─┼── 同时创建
mysql-2 ─┘
```

------

### 6.6.2 为什么需要有序创建

一些分布式系统需要：

```
第一节点
    ↓
初始化
    ↓
第二节点加入
    ↓
第三节点加入
```

例如某些数据库集群初始化过程中：

```
node-0
  ↓
初始化集群
  ↓
node-1 加入
  ↓
node-2 加入
```

如果所有节点同时启动：

```
node-0 ─┐
node-1 ─┼── 同时启动
node-2 ─┘
```

可能出现初始化竞争。

StatefulSet 可以帮助应用建立这样的启动顺序。

------

### 6.6.3 podManagementPolicy

StatefulSet 支持：

```
podManagementPolicy: OrderedReady
```

或者：

```
podManagementPolicy: Parallel
```

默认：

```
podManagementPolicy: OrderedReady
```

如果设置：

```
podManagementPolicy: Parallel
```

那么 StatefulSet 可以并行创建 Pod。

例如：

```
mysql-0 ─┐
mysql-1 ─┼── 并行
mysql-2 ─┘
```

这可以加快启动速度。

但是：

> **Parallel 并不意味着 StatefulSet 就变成 Deployment。**

Pod 依然具有：

- 固定名称
- 固定序号
- 稳定网络身份
- 独立存储

只是生命周期管理不再严格按照 Ready 顺序进行。

------

## 6.7 Ordered Termination

StatefulSet 不仅可以控制创建顺序，也可以控制删除顺序。

默认情况下，StatefulSet 删除 Pod 时遵循：

> **从高序号到低序号。**

例如：

```
mysql-0
mysql-1
mysql-2
```

删除时：

```
mysql-2
   ↓
mysql-1
   ↓
mysql-0
```

而不是：

```
mysql-0
mysql-1
mysql-2
```

------

### 6.7.1 为什么要反向删除

考虑一个集群：

```
node-0
node-1
node-2
```

假设：

```
node-0 = 集群初始化节点
node-1 = 成员
node-2 = 成员
```

如果关闭：

```
node-0
```

之前就把：

```
node-1
node-2
```

处理掉，可能影响整个集群的生命周期管理。

因此 Kubernetes 默认采用：

```
高序号 → 低序号
```

的方式终止 StatefulSet Pod。

------

### 6.7.2 一个容易误解的问题

Ordered Termination 并不是：

> “Kubernetes 知道数据库应该怎么关闭。”

Kubernetes 只知道：

```
mysql-2
    ↓
mysql-1
    ↓
mysql-0
```

这样的 Pod 生命周期顺序。

至于：

```
数据库 flush 数据
关闭复制
解除集群成员关系
执行 checkpoint
```

这些属于数据库自身的职责。

------

## 6.8 Headless Service

### 6.8.1 Headless Service 是什么

普通 Service 通常提供一个稳定的虚拟 IP：

```
Client
   ↓
Service
   ↓
Pod
```

例如：

```
mysql-service
10.96.100.20
```

客户端访问：

```
mysql-service
```

Service 再把请求转发给后面的 Pod。

但 StatefulSet 经常需要的是：

> **能够直接找到某一个具体 Pod。**

例如：

```
mysql-0
mysql-1
mysql-2
```

这时候就需要 Headless Service。

------

### 6.8.2 Headless Service 的定义

Headless Service：

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None

  selector:
    app: mysql

  ports:
    - port: 3306
      targetPort: 3306
```

关键配置：

```
clusterIP: None
```

这表示：

> 这个 Service 不分配普通的 ClusterIP。

------

### 6.8.3 Headless Service 的工作方式

假设：

```
StatefulSet
    │
    ├── mysql-0
    ├── mysql-1
    └── mysql-2
```

Headless Service：

```
mysql
clusterIP: None
```

DNS 可以提供：

```
mysql-0.mysql
mysql-1.mysql
mysql-2.mysql
```

完整域名通常是：

```
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

其中：

```
mysql-0
```

是 Pod 名称。

```
mysql
```

是 Service 名称。

```
default
```

是 Namespace。

```
svc.cluster.local
```

是 Kubernetes 集群 DNS 域。

------

### 6.8.4 为什么 Headless Service 对 StatefulSet 很重要

它建立了：

```
Pod
 ↓
稳定名称
 ↓
DNS
 ↓
稳定网络身份
```

例如：

```
mysql-0.mysql
```

即使 `mysql-0` 的 IP 地址发生变化：

```
旧 IP
  ↓
Pod 重建
  ↓
新 IP
```

客户端仍然可以使用：

```
mysql-0.mysql
```

而不需要记住具体 IP。

------

### 6.8.5 StatefulSet 的 serviceName

StatefulSet 通常需要：

```
spec:
  serviceName: mysql
```

配合：

```
kind: Service
spec:
  clusterIP: None
```

形成：

```
StatefulSet
    │
    │ serviceName
    ▼
Headless Service
    │
    ├── mysql-0
    ├── mysql-1
    └── mysql-2
```

因此常见完整结构是：

```
                    Headless Service
                         mysql
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          mysql-0       mysql-1       mysql-2
             │             │             │
            PVC           PVC           PVC
             │             │             │
            PV            PV            PV
```

这就是 StatefulSet 的经典架构。

------

## 6.9 StatefulSet 常见使用场景

StatefulSet 并不是：

> “只要应用使用数据库，就必须使用 StatefulSet。”

真正应该判断的是：

> **应用实例是否需要稳定身份、稳定存储、稳定网络身份或者有序生命周期。**

常见场景包括：

### 6.9.1 数据库

例如：

```
MySQL
PostgreSQL
MongoDB
```

数据库实例通常需要：

```
稳定存储
稳定身份
```

------

### 6.9.2 Redis

例如 Redis Cluster：

```
redis-0
redis-1
redis-2
...
```

不同节点需要拥有自己的数据和稳定身份。

------

### 6.9.3 Kafka

Kafka Broker 通常有稳定的 Broker ID。

例如：

```
broker-0
broker-1
broker-2
```

节点身份对于集群非常重要。

------

### 6.9.4 Elasticsearch

例如：

```
es-0
es-1
es-2
```

节点之间需要发现彼此，并保持一定的节点身份。

------

### 6.9.5 ZooKeeper

ZooKeeper 集群节点具有明确的：

```
server identity
```

例如：

```
server.1
server.2
server.3
```

StatefulSet 的稳定身份非常适合这种模型。

------

### 6.9.6 需要特别注意

并不是：

```
数据库
  ↓
必须 StatefulSet
```

而是：

```
应用需求
  ↓
需要稳定身份？
需要稳定存储？
需要稳定网络？
需要有序生命周期？
  ↓
是
  ↓
考虑 StatefulSet
```

在生产环境中，更常见的选择甚至可能是：

```
数据库
   ↓
云数据库
```

或者：

```
数据库
   ↓
Kubernetes Operator
   ↓
StatefulSet
```

因此 StatefulSet 是底层工作负载机制，并不等于完整的数据库高可用方案。

------

## 6.10 Redis / MySQL / PostgreSQL 等应用的部署思路

这一部分非常重要。

初学者很容易产生一个误区：

> “既然 StatefulSet 可以部署有状态应用，那我直接写一个 StatefulSet 就可以把 MySQL 做成生产数据库了。”

这是错误的。

StatefulSet 只解决 Kubernetes 层面的部分问题。

------

### 6.10.1 Redis 的部署思路

简单 Redis 实例：

```
StatefulSet
    │
    └── redis-0
          │
          └── PVC
```

如果只是开发环境：

```
redis-0
   │
   └── 数据盘
```

可能已经足够。

但是生产环境 Redis 还要考虑：

```
持久化
   ├── RDB
   └── AOF

高可用
   ├── Redis Sentinel
   └── Redis Cluster

备份
监控
故障恢复
容量
网络
安全
```

所以：

```
StatefulSet ≠ Redis 高可用
```

StatefulSet 只是其中的一层。

------

### 6.10.2 MySQL 的部署思路

一个简单 MySQL：

```
MySQL
  │
StatefulSet
  │
mysql-0
  │
PVC
```

但生产环境 MySQL 还需要考虑：

```
数据库复制
        ↓
主从 / InnoDB Cluster 等方案
        ↓
故障转移
        ↓
备份
        ↓
恢复
        ↓
监控
        ↓
数据一致性
```

所以不能简单地：

```
replicas: 3
```

就认为：

```
MySQL 变成三节点高可用集群
```

这是非常危险的理解。

------

### 6.10.3 PostgreSQL 的部署思路

PostgreSQL 同样如此。

简单部署：

```
StatefulSet
    │
postgres-0
    │
   PVC
```

但生产 PostgreSQL 通常还需要：

```
Primary
   │
   ├── Replica
   └── Replica

备份
恢复
故障转移
连接管理
监控
```

因此生产环境更常见：

```
PostgreSQL
     ↓
Operator
     ↓
StatefulSet
     ↓
Pods + PVC
```

Operator 负责更高层的数据库运维逻辑。

------

### 6.10.4 为什么生产环境经常使用 Operator

StatefulSet 能解决：

```
Pod
存储
网络
顺序
```

但数据库还需要：

```
初始化
复制
故障检测
Leader 选举
故障转移
备份
恢复
扩缩容
版本升级
```

这些已经超出了 StatefulSet 本身的职责。

因此生产环境经常采用：

```
Operator
   │
   ├── StatefulSet
   ├── Service
   ├── PVC
   ├── ConfigMap
   ├── Secret
   └── 数据库自身的高可用逻辑
```

也就是说：

> **StatefulSet 是基础设施层能力，Operator 是应用运维自动化层能力。**

------

## 6.11 StatefulSet 为什么不能简单理解成“有状态版 Deployment”

这是本章最重要的总结。

很多初学者会形成这样的认识：

```
Deployment = 无状态

StatefulSet = 有状态
```

虽然这样理解入门时没有完全错误，但它太简单。

更准确的理解应该是：

```
Deployment
    ↓
关注：
“我需要 N 个可以互相替换的 Pod”
```

而 StatefulSet：

```
StatefulSet
    ↓
关注：
“我需要 N 个具有稳定身份的 Pod”
```

------

### 6.11.1 Deployment 的核心思想

Deployment：

```
Pod A
Pod B
Pod C
```

这三个 Pod 通常是：

```
可替换的
```

例如：

```
Web Pod A
Web Pod B
Web Pod C
```

A 挂了：

```
删除 A
   ↓
创建 D
```

最终：

```
B
C
D
```

完全可以。

因为：

```
A ≈ B ≈ C ≈ D
```

它们没有必须保留的独立身份。

------

### 6.11.2 StatefulSet 的核心思想

StatefulSet：

```
Pod 0
Pod 1
Pod 2
```

它们并不一定可以互相替换。

例如：

```
mysql-0
mysql-1
mysql-2
```

可能分别对应：

```
节点 0
节点 1
节点 2
```

即使：

```
mysql-1
```

被删除，也需要重新得到：

```
mysql-1
```

并重新连接自己的：

```
PVC
```

以及恢复自己的：

```
网络身份
```

所以 StatefulSet 的核心不是：

> “Pod 里面有数据。”

而是：

> **Pod 的身份、网络和存储具有稳定性，并且这些属性与 Pod 生命周期紧密相关。**

------

### 6.11.3 两者可以这样理解

```
Deployment
│
├── Pod 是可替换的
├── 身份通常不重要
├── IP 通常不重要
├── 顺序通常不重要
└── 典型：Web/API
```

而：

```
StatefulSet
│
├── Pod 身份重要
├── Pod 名称稳定
├── 网络身份稳定
├── 存储可以稳定绑定
├── 生命周期可以有序
└── 典型：数据库/分布式系统
```

------

## 一个完整的 StatefulSet 示例

下面用一个 Nginx 示例，把本章几个核心概念串起来。

### Headless Service

```
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None

  selector:
    app: web

  ports:
    - port: 80
      targetPort: 80
```

------

### StatefulSet

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web

spec:
  serviceName: web
  replicas: 3

  podManagementPolicy: OrderedReady

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
        - name: nginx
          image: nginx:1.27

          ports:
            - containerPort: 80

          volumeMounts:
            - name: web-data
              mountPath: /usr/share/nginx/html

  volumeClaimTemplates:
    - metadata:
        name: web-data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 1Gi
```

应用：

```
kubectl apply -f web.yaml
```

查看：

```
kubectl get pods
```

应该看到类似：

```
web-0
web-1
web-2
```

查看 Service：

```
kubectl get svc web
```

查看 PVC：

```
kubectl get pvc
```

应该类似：

```
web-data-web-0
web-data-web-1
web-data-web-2
```

------

### 验证 Pod 的稳定身份

删除：

```
kubectl delete pod web-1
```

然后：

```
kubectl get pods -w
```

最终仍然会得到：

```
web-0
web-1
web-2
```

------

### 验证稳定存储

查看：

```
kubectl get pvc
```

可以看到：

```
web-data-web-0
web-data-web-1
web-data-web-2
```

删除：

```
kubectl delete pod web-1
```

PVC：

```
web-data-web-1
```

仍然存在。

新的：

```
web-1
```

重新启动后，可以继续使用原来的存储。

------

## 常见问题

### StatefulSet 是不是数据库专用？

不是。

数据库只是典型场景。

只要应用需要：

```
稳定身份
稳定网络
稳定存储
有序生命周期
```

都可以考虑 StatefulSet。

------

### Deployment 能不能挂 PVC？

可以。

例如 Deployment 完全可以：

```
Deployment
    │
    └── PVC
```

所以：

> **“使用 PVC”并不是 StatefulSet 的唯一判断标准。**

关键是：

```
Pod 是否需要独立、稳定、与身份绑定的存储？
```

------

### StatefulSet 的 replicas 设置成 3，是不是数据库自动变成三副本？

不是。

例如：

```
replicas: 3
```

只表示 Kubernetes 创建：

```
3 个 Pod
```

并不意味着：

```
MySQL 自动复制
PostgreSQL 自动复制
Redis 自动组成集群
```

应用层的复制机制必须另外配置。

------

### StatefulSet 删除 Pod 后，数据一定还在吗？

不能简单地说“一定”。

通常：

```
删除 Pod
   ↓
PVC 保留
```

但真正的数据是否安全，还取决于：

```
PVC
PV
StorageClass
底层存储
reclaimPolicy
备份机制
```

所以生产环境不能把：

```
PVC
```

当成：

```
备份
```

PVC 是存储，不是备份。

------

### 可以扩容吗？

可以。

例如：

```
kubectl scale statefulset web --replicas=5
```

原来：

```
web-0
web-1
web-2
```

扩容后：

```
web-0
web-1
web-2
web-3
web-4
```

并且新增 Pod 可以拥有自己的 PVC：

```
web-data-web-3
web-data-web-4
```

------

### StatefulSet 可以缩容吗？

可以：

```
kubectl scale statefulset web --replicas=2
```

会从高序号开始减少：

```
web-4
   ↓
web-3
   ↓
web-2
```

最终：

```
web-0
web-1
```

需要特别注意：

> 缩容删除 Pod，并不等于对应 PVC 自动删除。

所以缩容后可能仍然存在：

```
web-data-web-2
web-data-web-3
web-data-web-4
```

这在生产环境的数据管理中非常重要。

------

## 本章核心知识总结

StatefulSet 最重要的不是记住 YAML，而是建立下面这张模型：

```
                    StatefulSet
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
      稳定身份        稳定网络        稳定存储
          │              │              │
          ▼              ▼              ▼
       Pod 名称         DNS            PVC
          │              │              │
          ▼              ▼              ▼
      mysql-0         mysql-0.mysql   mysql-data-0
      mysql-1         mysql-1.mysql   mysql-data-1
      mysql-2         mysql-2.mysql   mysql-data-2
```

再加上生命周期：

```
创建：
0 → 1 → 2

删除：
2 → 1 → 0
```

因此 StatefulSet 的核心可以浓缩成：

> **稳定身份 + 稳定网络 + 稳定存储 + 有序生命周期。**

而 Deployment 的核心则是：

> **维持指定数量的、可替换的无状态 Pod。**

最终不要把 StatefulSet 简单记成：

```
Deployment + 数据库
```

更准确的理解是：

```
Deployment
    ↓
管理“可替换的 Pod 副本”


StatefulSet
    ↓
管理“具有稳定身份的 Pod 副本”
    │
    ├── 稳定名称
    ├── 稳定网络身份
    ├── 稳定存储
    └── 有序生命周期
```

这才是 StatefulSet 与 Deployment 最本质的区别。

# 第 7 章：DaemonSet、Job、CronJob

Kubernetes 中常见的工作负载控制器，可以按照应用生命周期大致分成几类：

```
Deployment
    ↓
长期运行的无状态应用
    ↓
Web / API / 前端


StatefulSet
    ↓
长期运行的有状态应用
    ↓
MySQL / Redis / PostgreSQL


DaemonSet
    ↓
每个 Node 运行一个 Pod
    ↓
日志 / 监控 / 网络


Job
    ↓
一次性任务
    ↓
数据处理 / 初始化 / 批处理


CronJob
    ↓
定时执行任务
    ↓
备份 / 清理 / 报表
```

本章重点学习后三种工作负载。

------

## 7.1 DaemonSet

### 7.1.1 DaemonSet 是什么

DaemonSet 是 Kubernetes 中用于保证：

> **符合条件的每个 Node 上运行一个 Pod。**

例如集群有：

```
Node-1
Node-2
Node-3
```

创建 DaemonSet 后：

```
Node-1 → Pod
Node-2 → Pod
Node-3 → Pod
```

如果以后新增：

```
Node-4
```

DaemonSet 会自动在 Node-4 上创建：

```
Node-4 → Pod
```

所以 DaemonSet 的核心不是：

> “我要运行 3 个 Pod。”

而是：

> **“我要让每个符合条件的 Node 都运行一个 Pod。”**

------

### 7.1.2 DaemonSet 的工作原理

Deployment 的思路是：

```
Deployment
    ↓
我要 N 个 Pod
```

DaemonSet 的思路则是：

```
DaemonSet
    ↓
检查集群中的 Node
    ↓
每个符合条件的 Node
    ↓
运行一个 Pod
```

例如：

```
                    DaemonSet
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Node-1        Node-2       Node-3
          │            │            │
          ▼            ▼            ▼
        Pod-1        Pod-2        Pod-3
```

如果 Node-2 被删除：

```
Node-2
   ↓
删除
   ↓
Pod-2 也随之消失
```

如果新增 Node-4：

```
Node-4
   ↓
DaemonSet 检测到
   ↓
创建 Pod-4
```

因此 DaemonSet 会随着集群 Node 的变化自动调整 Pod 分布。

------

## 7.2 每个 Node 运行一个 Pod

### 7.2.1 基本示例

下面创建一个最简单的 DaemonSet：

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent

spec:
  selector:
    matchLabels:
      app: node-agent

  template:
    metadata:
      labels:
        app: node-agent

    spec:
      containers:
        - name: node-agent
          image: nginx:1.27
          ports:
            - containerPort: 80
```

保存为：

```
daemonset.yaml
```

执行：

```
kubectl apply -f daemonset.yaml
```

查看：

```
kubectl get daemonset
```

可能看到：

```
NAME         DESIRED   CURRENT   READY
node-agent   3         3         3
```

查看 Pod：

```
kubectl get pods -o wide
```

例如：

```
NAME               NODE
node-agent-abc12   worker-1
node-agent-def34   worker-2
node-agent-ghi56   worker-3
```

这里需要注意：

> DaemonSet 的 Pod 名称并不像 StatefulSet 那样具有 `xxx-0`、`xxx-1` 的稳定序号。

DaemonSet 关注的是：

```
Node ↔ Pod
```

而不是：

```
Pod ↔ 固定身份
```

------

### 7.2.2 新增 Node 后会发生什么

假设现在：

```
worker-1 → node-agent
worker-2 → node-agent
worker-3 → node-agent
```

新增：

```
worker-4
```

DaemonSet Controller 会发现：

```
worker-4 没有 node-agent Pod
```

于是自动创建：

```
worker-4 → node-agent
```

最终：

```
worker-1 → Pod
worker-2 → Pod
worker-3 → Pod
worker-4 → Pod
```

这就是 DaemonSet 最核心的能力。

------

### 7.2.3 为什么不是 Deployment + replicas

假设集群有 5 个 Node：

```
Node-1
Node-2
Node-3
Node-4
Node-5
```

你创建：

```
replicas: 5
```

并不能保证：

```
Node-1 → 1
Node-2 → 1
Node-3 → 1
Node-4 → 1
Node-5 → 1
```

因为 Deployment 的目标是：

> 保持 5 个 Pod。

它并不天然要求：

> 每个 Node 恰好一个 Pod。

例如调度器完全可能安排成：

```
Node-1 → 2
Node-2 → 1
Node-3 → 2
Node-4 → 0
Node-5 → 0
```

只要满足 Deployment 的副本数量，调度层面并没有违反 Deployment 的基本目标。

而 DaemonSet 的目标就是：

```
每个符合条件的 Node → 一个 Pod
```

所以这两种模型完全不同。

------

## 7.3 日志采集

### 7.3.1 为什么日志采集适合 DaemonSet

生产环境中，Node 上经常存在容器日志：

```
Node
 │
 ├── Pod A
 │    └── Container
 │
 ├── Pod B
 │    └── Container
 │
 └── Pod C
      └── Container
```

我们希望在每个 Node 上部署一个日志采集 Agent：

```
Node-1
 ├── Application Pods
 └── Fluent Bit

Node-2
 ├── Application Pods
 └── Fluent Bit

Node-3
 ├── Application Pods
 └── Fluent Bit
```

然后：

```
Application
    ↓
Node 本地日志
    ↓
日志 Agent
    ↓
Elasticsearch / Loki / 云日志平台
```

因为日志 Agent 需要：

> **每个 Node 都运行一个实例。**

所以 DaemonSet 非常合适。

------

### 7.3.2 常见日志采集组件

生产环境中常见：

```
Fluent Bit
Fluentd
Filebeat
Vector
```

它们通常以 DaemonSet 方式运行。

典型架构：

```
                 Kubernetes Cluster
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
      Node-1          Node-2          Node-3
        │               │               │
   Application      Application      Application
        │               │               │
        ▼               ▼               ▼
   Log Agent        Log Agent        Log Agent
        │               │               │
        └───────────────┼───────────────┘
                        ▼
                  Log Backend
```

------

### 7.3.3 日志采集 Agent 如何读取 Node 日志

通常需要把 Node 上的目录挂载到 Pod。

例如：

```
volumeMounts:
  - name: varlog
    mountPath: /var/log

volumes:
  - name: varlog
    hostPath:
      path: /var/log
```

这里：

```
hostPath
```

表示访问 Node 本机文件系统。

结构类似：

```
Node
└── /var/log
       ↑
       │
    hostPath
       │
       ↓
DaemonSet Pod
└── /var/log
```

因此每个 Node 上的日志 Agent 都可以读取本机日志。

生产环境使用 `hostPath` 时需要非常谨慎，因为它让 Pod 能够访问 Node 文件系统。

------

## 7.4 Node 监控

### 7.4.1 为什么 Node 监控适合 DaemonSet

我们通常需要监控：

```
CPU
内存
磁盘
网络
文件系统
Load
```

这些信息与：

```
Node
```

本身有关。

因此每个 Node 部署一个监控 Agent 是非常自然的方案：

```
Node-1 → Monitoring Agent
Node-2 → Monitoring Agent
Node-3 → Monitoring Agent
```

------

### 7.4.2 常见 Node 监控方案

例如 Prometheus 体系中的：

```
node-exporter
```

常见部署方式就是 DaemonSet。

架构：

```
              Prometheus
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Node-1      Node-2      Node-3
       │          │          │
   exporter    exporter    exporter
```

Prometheus 定期访问：

```
node-exporter
```

获取 Node 指标。

------

### 7.4.3 Node 监控和 Pod 监控的区别

这是初学者容易混淆的地方。

Node 监控关注：

```
Node CPU
Node Memory
Node Disk
Node Network
```

而 Pod / Container 监控关注：

```
Pod CPU
Pod Memory
Container CPU
Container Memory
```

两者通常都会存在。

------

## 7.5 网络插件

### 7.5.1 为什么网络插件需要 DaemonSet

Kubernetes 集群中，每个 Node 都需要参与：

```
Pod 网络
Node 网络
跨 Node Pod 通信
Service 网络
```

因此网络插件往往需要在每个 Node 上运行一个 Agent。

例如：

```
Node-1 → CNI Agent
Node-2 → CNI Agent
Node-3 → CNI Agent
```

典型 Kubernetes 网络方案包括：

```
Calico
Cilium
Flannel
```

它们的具体实现不同，但很多组件都会采用 DaemonSet 部署 Node 级 Agent。

------

### 7.5.2 CNI 与 DaemonSet 的关系

可以简单理解：

```
Kubernetes
    │
    ↓
需要 Pod 网络
    │
    ↓
CNI
    │
    ↓
每个 Node 都需要网络组件
    │
    ↓
DaemonSet
```

因此你在生产集群中执行：

```
kubectl get pods -A
```

经常会看到类似：

```
calico-node-xxxxx
cilium-xxxxx
kube-proxy-xxxxx
```

其中很多都是 DaemonSet 管理的。

------

## 7.6 Job

### 7.6.1 Job 是什么

Deployment 和 StatefulSet 都主要负责：

> **长期运行的应用。**

但是有些任务只需要执行一次。

例如：

```
数据库初始化
数据迁移
批量数据处理
生成报表
执行脚本
数据导入
```

这些任务完成以后：

```
进程退出
```

并不是失败。

例如：

```
python migrate.py
```

执行：

```
开始迁移
 ↓
迁移完成
 ↓
程序退出
```

这是一个成功的任务。

Job 就是 Kubernetes 用来管理这种：

> **一次性、最终需要完成的任务。**

------

## 7.7 一次性任务

### 7.7.1 最简单的 Job

```
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job

spec:
  template:
    spec:
      restartPolicy: Never

      containers:
        - name: hello
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "Hello Kubernetes"
```

执行：

```
kubectl apply -f job.yaml
```

查看：

```
kubectl get jobs
```

可能：

```
NAME         COMPLETIONS   DURATION
hello-job    1/1           5s
```

查看 Pod：

```
kubectl get pods
```

可能：

```
hello-job-xxxxx   Completed
```

查看日志：

```
kubectl logs job/hello-job
```

得到：

```
Hello Kubernetes
```

------

### 7.7.2 Job 完成后为什么 Pod 还是存在

这是正常行为。

Job 完成：

```
Job
 ↓
Pod
 ↓
任务成功
 ↓
Completed
```

Kubernetes 通常不会因为任务完成就立即删除 Pod。

保留 Pod 有一个重要价值：

> **方便查看日志和排查问题。**

例如：

```
kubectl logs job/hello-job
```

就可以查看任务输出。

------

## 7.8 Job 的完成条件

### 7.8.1 completions

Job 可以指定：

```
spec:
  completions: 5
```

含义是：

> **需要成功完成 5 次任务。**

例如：

```
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job

spec:
  completions: 5

  template:
    spec:
      restartPolicy: Never

      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "processing"; sleep 2
```

查看：

```
kubectl get job
```

可能逐渐变成：

```
NAME         COMPLETIONS
batch-job    1/5
batch-job    2/5
batch-job    3/5
batch-job    4/5
batch-job    5/5
```

最终：

```
5/5
```

Job 才算完成。

------

### 7.8.2 parallelism

还可以指定：

```
parallelism: 2
```

表示：

> 同时最多运行 2 个任务 Pod。

例如：

```
spec:
  completions: 5
  parallelism: 2
```

大致过程：

```
第一批：

Job-1 ─┐
Job-2 ─┘

完成后：

Job-3 ─┐
Job-4 ─┘

最后：

Job-5
```

最终需要：

```
成功完成 5 次
```

------

### 7.8.3 completions + parallelism

这是生产批处理任务中非常常见的组合：

```
spec:
  completions: 100
  parallelism: 10
```

含义：

```
总共成功 100 个任务
同时最多执行 10 个
```

可以理解成：

```
100 个任务
     │
     ├── 同时处理 10 个
     │
     ├── 完成后继续
     │
     └── 最终累计完成 100 个
```

------

## 7.9 Job 的失败重试

### 7.9.1 为什么 Job 需要重试

实际任务可能失败：

```
Pod
 ↓
执行任务
 ↓
连接数据库失败
 ↓
进程退出
```

如果任务本身是临时性失败：

```
网络抖动
数据库暂时不可用
临时 API 错误
```

我们希望：

```
失败
 ↓
重新执行
```

Job 可以做到这一点。

------

### 7.9.2 backoffLimit

例如：

```
spec:
  backoffLimit: 3
```

表示 Job 允许一定次数的失败重试。

例如：

```
第一次失败
    ↓
重试

第二次失败
    ↓
重试

第三次失败
    ↓
重试

持续失败
    ↓
Job 最终失败
```

可以查看：

```
kubectl describe job hello-job
```

关注：

```
Completions
Failed
Conditions
```

------

### 7.9.3 restartPolicy

Job 的 Pod 通常需要：

```
restartPolicy: Never
```

或者：

```
restartPolicy: OnFailure
```

例如：

```
spec:
  template:
    spec:
      restartPolicy: OnFailure
```

`OnFailure` 表示容器失败后，Pod 内可以重新启动容器。

而 `Never` 表示失败后不在原 Pod 中重新启动，而由 Job 根据策略创建新的 Pod。

因此：

```
restartPolicy
```

和：

```
backoffLimit
```

不是同一个概念。

可以简单理解为：

```
restartPolicy
    ↓
Pod 内怎么处理容器失败


backoffLimit
    ↓
Job 层面允许失败多少次
```

------

## 7.10 CronJob

### 7.10.1 CronJob 是什么

Job 是：

```
执行一次
```

CronJob 则是：

> **按照时间计划，周期性创建 Job。**

例如：

```
每天凌晨 2 点
    ↓
创建一个 Job
    ↓
执行备份
    ↓
Job 完成
```

第二天：

```
凌晨 2 点
    ↓
再创建一个 Job
```

因此：

```
CronJob
   │
   ├── Job-1
   ├── Job-2
   ├── Job-3
   └── Job-4
```

CronJob 本身负责：

> **什么时候创建 Job。**

Job 负责：

> **任务怎么执行、什么时候算完成。**

------

## 7.11 定时任务

### 7.11.1 最简单的 CronJob

例如每分钟执行一次：

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron

spec:
  schedule: "* * * * *"

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never

          containers:
            - name: hello
              image: busybox:1.36
              command:
                - sh
                - -c
                - date; echo "Hello from CronJob"
```

应用：

```
kubectl apply -f cronjob.yaml
```

查看：

```
kubectl get cronjob
```

查看 Job：

```
kubectl get jobs
```

可以看到：

```
hello-cron-29384720
hello-cron-29384721
hello-cron-29384722
```

每次 CronJob 到达调度时间，就创建一个新的 Job。

------

### 7.11.2 Cron 表达式

CronJob 的：

```
schedule:
```

使用 Cron 表达式。

常见例子：

```
* * * * *
```

每分钟执行。

```
0 * * * *
```

每小时执行一次。

```
0 2 * * *
```

每天凌晨 2 点执行。

```
0 0 * * 0
```

每周日凌晨执行。

```
0 3 1 * *
```

每月 1 日凌晨 3 点执行。

基本结构：

```
分钟 小时 日 月 星期
```

例如：

```
0 2 * * *
│ │ │ │ │
│ │ │ │ └── 星期
│ │ │ └──── 月
│ │ └────── 日
│ └──────── 小时
└────────── 分钟
```

------

### 7.11.3 CronJob 并不直接执行 Pod

这一点很重要：

```
CronJob
   ↓
创建 Job
   ↓
Job 创建 Pod
   ↓
Pod 执行容器
```

完整关系：

```
CronJob
   │
   ├── Job
   │    └── Pod
   │         └── Container
   │
   ├── Job
   │    └── Pod
   │         └── Container
   │
   └── Job
        └── Pod
             └── Container
```

因此：

> CronJob 是 Job 的调度器，而不是另一种 Pod。

------

## 7.12 Job / CronJob 的生产实践

### 7.12.1 数据库备份

这是最典型的生产场景。

例如：

```
每天凌晨 2 点
       ↓
CronJob
       ↓
Job
       ↓
mysqldump
       ↓
对象存储
```

架构：

```
                CronJob
                   │
             每天凌晨 2 点
                   ↓
                  Job
                   ↓
              Backup Pod
                   │
             mysqldump
                   │
                   ▼
             Object Storage
```

例如：

```
S3
OSS
COS
MinIO
```

备份任务执行完成以后：

```
Pod → Completed
Job → Complete
```

下一次时间到达后重新执行。

------

### 7.12.2 数据清理

例如每天清理：

```
30 天以前的数据
```

可以：

```
CronJob
   ↓
Job
   ↓
清理程序
   ↓
DELETE old records
```

这种任务非常适合 CronJob。

------

### 7.12.3 数据同步

例如：

```
每天凌晨
     ↓
从 A 系统同步数据
     ↓
B 系统
```

可以使用：

```
CronJob
```

------

### 7.12.4 报表生成

例如每天：

```
01:00
 ↓
生成日报
 ↓
上传对象存储
 ↓
发送通知
```

可以使用 CronJob。

------

### 7.12.5 数据迁移

数据库 Schema Migration 也是 Job 的典型用途。

例如：

```
Deployment
    │
    │
数据库
    │
    ▲
    │
Migration Job
```

发布新版本之前：

```
Job
 ↓
执行 migration
 ↓
成功
 ↓
再继续应用发布
```

不过生产环境需要特别注意迁移脚本的：

```
幂等性
向前兼容
向后兼容
失败回滚
锁
执行时间
```

不能简单认为：

```
Job 成功
=
数据库迁移一定安全
```

------

### 7.12.6 设置历史 Job 保留数量

CronJob 长时间运行后，会不断产生 Job。

例如：

```
每天一次
一年
≈ 365 个 Job
```

如果不管理历史任务，集群中会积累大量历史对象。

可以设置：

```
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 5
```

例如：

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup

spec:
  schedule: "0 2 * * *"

  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never

          containers:
            - name: backup
              image: busybox:1.36
              command:
                - sh
                - -c
                - echo "backup"
```

表示：

```
成功 Job
最多保留 3 个

失败 Job
最多保留 5 个
```

生产环境非常建议配置。

------

### 7.12.7 防止任务重复执行

这是 CronJob 生产环境最重要的问题之一。

假设：

```
Job A
```

本来应该：

```
执行 5 分钟
```

但实际上由于数据库或者网络问题：

```
执行 30 分钟
```

与此同时下一次调度时间到了。

这时候可能出现：

```
Job A
正在执行

Job B
开始执行
```

如果任务不是幂等的，就可能造成：

```
重复扣款
重复发送
重复写入
重复备份
重复数据处理
```

所以 CronJob 需要考虑：

```
concurrencyPolicy:
```

常见配置：

```
concurrencyPolicy: Forbid
```

表示：

> 如果上一次 Job 还没有完成，就不要启动新的 Job。

例如：

```
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid
```

------

### 7.12.8 Replace

还可以：

```
concurrencyPolicy: Replace
```

表示：

> 如果旧 Job 还在运行，则终止旧 Job，然后启动新的 Job。

这种策略需要非常谨慎。

对于数据库备份、数据处理等任务，通常不能简单地因为新一轮时间到了就杀掉旧任务。

所以生产环境不能机械地使用：

```
Replace
```

而应该根据业务特性决定。

------

### 7.12.9 Job 必须考虑幂等性

所谓幂等，可以简单理解成：

> **任务执行一次和因为重试执行多次，最终结果应该保持正确。**

例如：

```
INSERT INTO orders ...
```

如果重复执行可能产生两条数据。

而：

```
INSERT ... ON CONFLICT DO UPDATE
```

或者通过唯一键设计，就可以降低重复执行风险。

因此生产 Job 应尽量做到：

```
任务失败
    ↓
可以安全重试
```

而不是：

```
任务失败
    ↓
重试
    ↓
数据重复
```

------

### 7.12.10 设置 activeDeadlineSeconds

对于可能卡死的任务，可以设置：

```
activeDeadlineSeconds: 3600
```

表示：

> Job 最长运行 3600 秒。

例如：

```
spec:
  activeDeadlineSeconds: 3600
```

如果任务一直没有结束：

```
开始
 ↓
运行
 ↓
超过 1 小时
 ↓
Job 超时
```

这可以避免某些异常任务无限占用资源。

------

### 7.12.11 设置资源限制

Job 和普通 Pod 一样，也应该配置：

```
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"

  limits:
    cpu: "1"
    memory: "512Mi"
```

例如：

```
containers:
  - name: worker
    image: busybox:1.36

    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"

      limits:
        cpu: "1"
        memory: "512Mi"
```

生产环境不要认为：

> “任务只运行几分钟，所以不用限制资源。”

如果任务出现异常：

```
无限循环
内存泄漏
异常数据处理
```

同样可能影响整个 Node。

------

### 7.12.12 设置合理的重试策略

不要看到：

```
backoffLimit: 100
```

就认为：

> “失败就多重试，总会成功。”

如果任务本身是永久错误：

```
配置错误
权限错误
SQL 错误
镜像错误
参数错误
```

那么无限重试只会：

```
失败
 ↓
重试
 ↓
失败
 ↓
重试
 ↓
失败
```

浪费集群资源。

生产环境应该区分：

```
临时性错误
    ↓
适合重试


永久性错误
    ↓
应该快速失败 + 告警
```

------

### 7.12.13 CronJob 的时间问题

生产环境特别容易忽略时区。

例如：

```
schedule: "0 2 * * *"
```

你以为：

```
每天台湾时间 02:00
```

但实际执行时间必须结合 Kubernetes 版本、CronJob 的 `timeZone` 配置以及集群环境确认。

现代 Kubernetes 可以显式指定：

```
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Taipei"
```

这样表达：

> 按 Asia/Taipei 时区每天 02:00 执行。

生产环境推荐明确指定时区，而不是依赖环境默认行为。

------

### 7.12.14 Job / CronJob 生产排查

任务失败时，可以按照这个顺序：

```
kubectl get cronjob
```

查看 CronJob。

然后：

```
kubectl get jobs
```

查看生成的 Job。

然后：

```
kubectl describe job <job-name>
```

查看 Job 状态和事件。

然后：

```
kubectl get pods
```

找到 Job 对应的 Pod。

最后：

```
kubectl logs <pod-name>
```

查看真正的程序日志。

完整排查链路：

```
CronJob
   ↓
Job
   ↓
Pod
   ↓
Container
   ↓
Application Log
```

如果只看：

```
kubectl get cronjob
```

通常无法定位真正的问题。

------

## 本章核心知识总结

这一章需要形成三种工作负载的清晰模型：

```
DaemonSet
    ↓
每个 Node 一个 Pod
    ↓
Node 级 Agent
    ↓
日志 / 监控 / 网络
Job
    ↓
执行一次任务
    ↓
成功完成
    ↓
结束
CronJob
    ↓
按照时间调度
    ↓
创建 Job
    ↓
Job 创建 Pod
    ↓
执行任务
```

最关键的区别可以记成：

| 工作负载  | 核心目标                     | 典型场景                 |
| --------- | ---------------------------- | ------------------------ |
| DaemonSet | 每个符合条件的 Node 一个 Pod | 日志、监控、网络         |
| Job       | 一次性完成任务               | 数据迁移、批处理、初始化 |
| CronJob   | 按时间周期创建 Job           | 备份、清理、报表         |

其中最重要的三个工作模型是：

```
Deployment
    ↓
我要 N 个长期运行的 Pod


StatefulSet
    ↓
我要 N 个具有稳定身份的 Pod


DaemonSet
    ↓
我要每个 Node 一个 Pod


Job
    ↓
我要这个任务最终成功完成


CronJob
    ↓
我要按照时间反复创建 Job
```

这五种工作负载分别解决的是**不同的生命周期问题**，不能简单互相替代。
