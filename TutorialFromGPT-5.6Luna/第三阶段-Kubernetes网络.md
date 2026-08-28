# 第 8 章：Kubernetes 网络模型

Kubernetes 网络是初学者比较容易产生“能用，但不知道为什么”的领域。

前面我们已经多次看到：

```
Pod
Service
Node
CNI
```

但这些对象之间到底是怎么通信的？

例如：

```
Pod A
  ↓
Pod B
```

数据包究竟经过了什么？

又例如：

```
Pod
  ↓
Service
  ↓
Pod
```

Service 本身到底是不是一个真正的网络设备？

再比如：

```
Node A 上的 Pod
        ↓
Node B 上的 Pod
```

跨 Node 的数据包又是怎么走的？

这一章先把 Kubernetes 网络的底层模型建立起来。重点不是背命令，而是理解：

> **Kubernetes 网络最终还是 Linux 网络，只是在 Linux 网络能力之上增加了 Pod 网络、Service 网络以及 CNI 等机制。**

------

## 8.1 Linux 网络基础回顾

### 8.1.1 IP 地址

IP 地址用于标识网络中的一个通信端点。

例如：

```
192.168.1.10
```

可以理解成：

```
这台机器 / 网络接口在这个网络中的地址
```

在 Kubernetes 中，你会同时遇到多种 IP：

```
Node IP
Pod IP
Service IP
```

它们的用途完全不同。

例如：

```
Node IP
192.168.1.10


Pod IP
10.244.1.5


Service IP
10.96.0.10
```

不要看到都是 IP，就认为它们属于同一种网络。

------

### 8.1.2 网卡

Linux 中可以通过：

```
ip addr
```

查看网络接口。

例如：

```
eth0
lo
```

其中：

```
lo
```

是 Loopback。

而：

```
eth0
```

通常是实际网络接口。

查看：

```
ip link
```

可以看到接口状态。

------

### 8.1.3 路由

Linux 根据路由表决定：

> 一个数据包下一步应该发送到哪里。

查看：

```
ip route
```

例如：

```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0
```

可以简单理解成：

```
目标地址
   ↓
查路由表
   ↓
决定出口网卡 / 下一跳
```

这对理解 Kubernetes Pod 网络非常重要。

------

### 8.1.4 Linux Network Namespace

Linux Network Namespace 是 Kubernetes 网络非常重要的底层基础。

它可以让不同进程拥有相互隔离的：

```
网络接口
IP 地址
路由表
iptables / nftables 网络规则
```

例如：

```
Node
│
├── Namespace A
│     ├── eth0
│     └── route
│
└── Namespace B
      ├── eth0
      └── route
```

两个 Namespace 可以拥有各自独立的网络环境。

容器网络大量依赖这个能力。

------

### 8.1.5 veth pair

Linux 中还有一个非常重要的网络设备：

```
veth pair
```

可以把它理解成：

> **一根虚拟网线的两端。**

例如：

```
Network Namespace
       │
     veth0
       │
       │
     veth1
       │
       ▼
     Node
```

一端放进容器 / Pod 的 Network Namespace：

```
Pod Namespace
    │
   eth0
```

另一端留在 Node：

```
Node
    │
   vethXXXX
```

于是：

```
Pod eth0
    │
    │ veth pair
    │
Node vethXXXX
```

两端之间就形成了虚拟连接。

这是理解 Pod 网络的关键。

------

## 8.2 Container Network

### 8.2.1 容器为什么需要网络

容器中的应用通常不是孤立运行的。

例如：

```
Frontend Container
       ↓
Backend Container
       ↓
Database
```

容器需要：

```
发送数据
接收数据
监听端口
访问其他服务
```

因此容器必须拥有自己的网络环境。

------

### 8.2.2 容器网络 Namespace

Linux 容器通常拥有自己的 Network Namespace。

例如：

```
Host
│
├── Container A
│     └── Network Namespace A
│
└── Container B
      └── Network Namespace B
```

Container A 和 Container B 可以拥有：

```
不同的 IP
不同的网络接口
不同的路由表
```

例如：

```
Container A
10.0.0.2

Container B
10.0.0.3
```

------

### 8.2.3 Docker bridge 模型

以常见的 Linux Bridge 模型理解最容易。

例如：

```
Container A
10.0.0.2
    │
   veth
    │
    ▼
 docker0
    │
    ├──────── Container B
    │            10.0.0.3
    │
    └──────── Host
```

这里：

```
docker0
```

可以理解成一个虚拟交换机。

数据包从：

```
Container A
```

进入：

```
veth
```

然后进入：

```
bridge
```

最终到达：

```
Container B
```

Kubernetes Pod 网络在底层也使用了类似的 Linux 网络能力，只是具体实现由 CNI 决定。

------

## 8.3 Pod Network

### 8.3.1 Pod 为什么需要网络

Pod 是 Kubernetes 的基本运行单元。

一个 Pod 可以包含多个 Container：

```
Pod
├── Container A
└── Container B
```

这些 Container 默认共享：

```
Network Namespace
```

因此一个 Pod 通常只有一个 Pod IP。

例如：

```
Pod
10.244.1.10
│
├── Container A
└── Container B
```

而不是：

```
Container A → 10.244.1.10
Container B → 10.244.1.11
```

------

### 8.3.2 Pod 内多个 Container 为什么共享网络

因为这些 Container 使用同一个 Network Namespace。

因此：

```
Pod
└── Network Namespace
       │
       └── eth0
```

多个 Container 都通过这个网络环境通信。

例如：

```
Container A
localhost:8080
       │
       │
Container B
localhost:9090
```

Container A 可以访问：

```
localhost:9090
```

因为：

> **它们位于同一个网络 Namespace。**

这是 Sidecar 模式非常重要的基础。

例如：

```
Pod
├── Application
│     └── localhost:8080
│
└── Envoy
      └── localhost:15001
```

两个 Container 可以直接通过 localhost 通信。

------

### 8.3.3 Pod 的 eth0

从 Pod 内部执行：

```
ip addr
```

通常可以看到：

```
eth0@ifXX
```

以及类似：

```
10.244.1.10/24
```

这个：

```
eth0
```

就是 Pod 的主要网络接口。

其背后通常连接到 Node 上的：

```
veth pair
```

结构可以理解成：

```
Pod Network Namespace
        │
       eth0
        │
     veth pair
        │
        ▼
       Node
```

------

## 8.4 Pod IP

### 8.4.1 Pod IP 是什么

Pod IP 是 Kubernetes 为 Pod 网络分配的 IP 地址。

例如：

```
kubectl get pods -o wide
```

可能看到：

```
NAME       READY   IP           NODE
web-xxx    1/1     10.244.1.10  worker-1
api-xxx    1/1     10.244.2.20  worker-2
```

这里：

```
10.244.1.10
```

就是 Pod IP。

------

### 8.4.2 Pod IP 通常不是固定的

这是非常重要的概念。

例如：

```
web Pod
10.244.1.10
```

删除 Pod：

```
kubectl delete pod web-xxx
```

新的 Pod 可能得到：

```
10.244.1.15
```

因此：

> **Pod IP 默认不是稳定身份。**

这也是为什么应用通常不应该直接把 Pod IP 写死。

这也是 Service 存在的重要原因之一。

------

### 8.4.3 Pod IP 与 Node IP

例如：

```
Node-1
192.168.1.10
    │
    ├── Pod A
    │   10.244.1.10
    │
    └── Pod B
        10.244.1.11
```

这里：

```
192.168.1.10
```

是 Node IP。

而：

```
10.244.1.10
10.244.1.11
```

是 Pod IP。

它们属于不同的网络层次。

------

### 8.4.4 Pod CIDR

Kubernetes 集群通常会规划一个 Pod 网络地址范围。

例如：

```
10.244.0.0/16
```

可以进一步分配给不同 Node：

```
worker-1 → 10.244.1.0/24
worker-2 → 10.244.2.0/24
worker-3 → 10.244.3.0/24
```

于是：

```
worker-1
10.244.1.0/24

worker-2
10.244.2.0/24
```

不同 Node 上的 Pod 就可以拥有不同的 Pod IP。

不过：

> **具体地址范围和分配方式取决于集群安装方式及 CNI 实现。**

不要认为所有 Kubernetes 集群一定使用 `10.244.0.0/16`。

------

## 8.5 Container 到 Container 通信

这里需要区分：

> **同一个 Pod 内的 Container。**

假设：

```
Pod
IP = 10.244.1.10

├── Container A
└── Container B
```

两个 Container 共享 Network Namespace。

因此：

```
Container A
      │
      │ localhost
      ▼
Container B
```

例如：

Container B：

```
listen :8080
```

Container A：

```
curl http://localhost:8080
```

即可访问。

------

### 8.5.1 容器之间不能监听同一个端口

由于共享 Network Namespace：

```
Container A → :8080
Container B → :8080
```

会产生端口冲突。

因为实际上是：

```
同一个网络 Namespace
       │
       └── :8080
```

不能让两个进程同时监听同一个 IP:Port。

所以 Sidecar Container 通常会使用不同端口：

```
Application → 8080
Envoy       → 15001
```

------

## 8.6 Pod 到 Pod 通信

### 8.6.1 Kubernetes 网络模型的核心要求

Kubernetes 对 Pod 网络有一个非常重要的设计原则：

> **一个 Pod 可以直接与另一个 Pod 通信，而不需要经过 NAT。**

例如：

```
Pod A
10.244.1.10
    │
    ▼
Pod B
10.244.2.20
```

Pod A 应该能够直接访问：

```
10.244.2.20
```

而不需要：

```
Pod A
  ↓
NAT
  ↓
某个中间地址
  ↓
Pod B
```

这也是 Kubernetes 网络模型的核心思想之一。

------

### 8.6.2 同一个 Node 上的 Pod

例如：

```
Node-1
│
├── Pod A
│   10.244.1.10
│
└── Pod B
    10.244.1.11
```

两个 Pod 的网络 Namespace 不同：

```
Pod A Namespace
Pod B Namespace
```

但它们通过 Node 上的 Linux 网络设备连接起来。

简化理解：

```
Pod A
  │
 veth
  │
  ▼
Linux Bridge / CNI
  │
  ▲
 veth
  │
Pod B
```

具体使用 Bridge、eBPF、路由等方式，由 CNI 决定。

------

### 8.6.3 不同 Node 上的 Pod

例如：

```
Node-1                     Node-2
192.168.1.10               192.168.1.11
   │                           │
   │                           │
Pod A                       Pod B
10.244.1.10                 10.244.2.20
```

Pod A 访问 Pod B：

```
10.244.1.10
      │
      ▼
Node-1
      │
      │ Cluster Network
      ▼
Node-2
      │
      ▼
10.244.2.20
```

这就是：

> **跨 Node Pod-to-Pod 通信。**

具体怎么走，取决于 CNI。

------

## 8.7 Node 到 Pod 通信

### 8.7.1 为什么 Node 需要访问 Pod

Node 上运行的 Kubernetes 组件需要与 Pod 通信。

例如：

```
kubelet
container runtime
监控 Agent
日志 Agent
```

都可能需要访问 Pod。

例如：

```
Node
 │
 └── Pod
      10.244.1.10
```

Node 必须能够正确路由到：

```
10.244.1.10
```

------

### 8.7.2 Node 路由到 Pod

例如 Node：

```
192.168.1.10
```

Pod：

```
10.244.1.10
```

Linux 路由表可能存在类似：

```
10.244.1.0/24 dev ...
```

或者：

```
10.244.1.0/24 via ...
```

具体形式由 CNI 决定。

基本逻辑：

```
Node
 ↓
查路由表
 ↓
找到 Pod 网络
 ↓
发送数据包
 ↓
Pod
```

因此 Kubernetes 网络最终仍然依赖 Linux 的：

```
网卡
路由
veth
bridge
iptables / nftables
eBPF
```

等基础能力。

------

## 8.8 Cluster 网络

### 8.8.1 什么是 Cluster 网络

所谓 Kubernetes Cluster Network，可以理解为：

> **整个 Kubernetes 集群中用于连接 Node、Pod、Service 等网络对象的网络体系。**

至少需要考虑：

```
Node ↔ Node
Node ↔ Pod
Pod ↔ Pod
Pod ↔ Service
```

例如：

```
                  Cluster Network
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
      Node-1          Node-2          Node-3
        │               │               │
      Pod A           Pod B           Pod C
```

------

### 8.8.2 Cluster 网络并不是一个单独的网络设备

初学者容易理解成：

```
Kubernetes Cluster Network
        ↓
一台交换机
```

实际上不是。

它是一套由多个组件共同构建出来的网络体系：

```
Linux Network Namespace
veth
Routing
Bridge
iptables / nftables
CNI
Service rules
eBPF
```

不同 Kubernetes 网络方案采用的技术组合也不同。

------

### 8.8.3 Pod 网络和 Service 网络不是一回事

这是非常重要的区分。

Pod 网络：

```
Pod A
  ↓
Pod B
```

使用的是：

```
Pod IP
```

Service 网络：

```
Client
  ↓
Service
  ↓
Pod
```

使用的是：

```
Service IP
```

例如：

```
Pod A
10.244.1.10

Service
10.96.0.10

Pod B
10.244.2.20
```

这三个 IP 的角色完全不同。

------

## 8.9 CNI 是什么

### 8.9.1 CNI 的定义

CNI 是：

> **Container Network Interface**

即：

> 容器网络接口标准。

它定义了容器运行时与网络插件之间的一套接口规范。

可以简单理解为：

```
Container Runtime
       │
       │ CNI
       ▼
Network Plugin
       │
       ▼
Linux Network
```

Kubernetes 本身并没有直接规定：

> “Pod 网络必须使用 Flannel。”

也没有规定：

> “必须使用 Calico。”

而是通过 CNI 机制让不同网络插件实现 Pod 网络。

------

### 8.9.2 CNI 负责什么

CNI 网络插件通常负责：

```
创建 Pod 网络接口
分配 Pod IP
配置路由
配置网络连接
实现跨 Node Pod 通信
```

例如创建 Pod：

```
Kubernetes
    ↓
Container Runtime
    ↓
CNI
    ↓
创建网络 Namespace
    ↓
创建 veth
    ↓
分配 Pod IP
    ↓
配置路由
```

最终：

```
Pod
10.244.1.10
```

就可以正常通信。

------

### 8.9.3 CNI 不只是“分配 IP”

这是一个常见误区。

很多初学者认为：

```
CNI = IP 地址分配器
```

实际上远不止。

它可能需要负责：

```
IPAM
网络接口
路由
跨 Node 通信
NetworkPolicy
Service 配合
加密
可观测性
```

不同 CNI 的能力差异很大。

------

### 8.9.4 CNI 的基本工作流程

Pod 创建时：

```
1. Kubernetes 创建 Pod
        ↓
2. kubelet 请求容器运行时创建容器
        ↓
3. 创建 Network Namespace
        ↓
4. 调用 CNI
        ↓
5. CNI 创建网络接口
        ↓
6. 分配 Pod IP
        ↓
7. 配置路由 / 网络规则
        ↓
8. Pod 获得网络能力
```

所以：

> Pod 能联网，不是 Kubernetes API Server 直接给它“发了一张网卡”。

而是底层 CNI 完成了实际网络配置。

------

## 8.10 Flannel

### 8.10.1 Flannel 是什么

Flannel 是 Kubernetes 生态中较早、非常经典的网络方案。

它的核心目标是：

> **为 Kubernetes Pod 提供跨 Node 的网络连接。**

例如：

```
Node-1
Pod A
10.244.1.10

        ↓ Flannel ↓

Node-2
Pod B
10.244.2.20
```

让：

```
10.244.1.10
```

可以访问：

```
10.244.2.20
```

------

### 8.10.2 Flannel 的基本思路

可以简化理解为：

```
Node-1
Pod Network
10.244.1.0/24
       │
       ▼
   Flannel
       │
       │ Overlay / Routing
       ▼
   Flannel
       │
       ▼
Node-2
Pod Network
10.244.2.0/24
```

Flannel 可以通过不同后端实现 Node 之间的 Pod 网络通信。

经典模式之一是 VXLAN。

------

### 8.10.3 VXLAN 的基本概念

VXLAN 可以理解为：

> **在现有 IP 网络之上，再构建一个逻辑二层网络。**

例如：

```
Pod A
10.244.1.10
    │
    ▼
封装
    │
    ▼
Node-1 IP
192.168.1.10
    │
    │ 物理网络
    ▼
Node-2 IP
192.168.1.11
    │
    ▼
解封装
    │
    ▼
Pod B
10.244.2.20
```

外层使用 Node 网络：

```
192.168.1.x
```

内层承载 Pod 网络：

```
10.244.x.x
```

------

### 8.10.4 Flannel 的特点

Flannel 的优点：

```
简单
成熟
部署相对容易
适合基础 Pod 网络
```

但它的定位比较专注。

如果你需要非常丰富的：

```
NetworkPolicy
高级可观测性
复杂网络安全
高性能 eBPF
```

通常会考虑其他方案。

------

## 8.11 Calico

### 8.11.1 Calico 是什么

Calico 是 Kubernetes 生态中非常成熟的网络与网络安全方案。

它不仅可以提供：

```
Pod 网络
```

还可以提供：

```
NetworkPolicy
网络安全
路由
```

等能力。

------

### 8.11.2 Calico 的基本思路

Calico 一个重要特点是：

> **可以使用纯三层路由的方式实现 Pod 网络。**

例如：

```
Node-1
Pod CIDR
10.244.1.0/24

Node-2
Pod CIDR
10.244.2.0/24
```

通过路由：

```
10.244.1.0/24 → Node-1
10.244.2.0/24 → Node-2
```

Node 就知道：

```
去 10.244.2.x
    ↓
应该发送给 Node-2
```

这样可以不一定依赖传统 Overlay。

------

### 8.11.3 Calico 的重要能力：NetworkPolicy

例如：

```
Frontend
   ↓
Backend
   ↓
Database
```

你可能希望：

```
Frontend → Backend   允许
Backend → Database   允许
Frontend → Database   禁止
```

Calico 可以利用 Kubernetes NetworkPolicy 等机制实现网络访问控制。

所以：

> Calico 不只是“让 Pod 能联网”。

它还是重要的：

> **Kubernetes 网络安全方案。**

------

### 8.11.4 Calico 的特点

可以简单理解：

```
Calico
├── Pod 网络
├── 路由
├── NetworkPolicy
└── 网络安全
```

因此生产环境中非常常见。

------

## 8.12 Cilium

### 8.12.1 Cilium 是什么

Cilium 是现代 Kubernetes 网络方案之一。

它最大的技术特点之一是：

> **大量使用 eBPF。**

eBPF 是 Linux 内核中的一种机制，可以让程序在内核网络路径等位置执行高效逻辑。

因此 Cilium 可以实现：

```
网络
安全
负载均衡
可观测性
```

等能力。

------

### 8.12.2 eBPF 为什么重要

传统网络处理可能大量依赖：

```
iptables
```

当规则非常多时：

```
规则
规则
规则
规则
...
```

管理和性能都会变得复杂。

Cilium 可以利用 eBPF 将部分网络处理逻辑放到内核中更高效的位置。

简单理解：

```
传统：

Packet
  ↓
iptables
  ↓
大量规则
  ↓
后续处理


Cilium：

Packet
  ↓
eBPF
  ↓
高效处理
```

实际实现远比这个复杂，但这个模型足够帮助初学者建立概念。

------

### 8.12.3 Cilium 的能力

Cilium 不只是 CNI。

它可以提供：

```
Pod 网络
NetworkPolicy
Service 负载均衡
网络安全
可观测性
eBPF
```

并且可以与：

```
Hubble
```

等组件结合，对 Kubernetes 网络流量进行可观测。

------

### 8.12.4 Cilium 与 Calico 的区别

可以先建立一个粗略认识：

| 特性          | Flannel      | Calico        | Cilium   |
| ------------- | ------------ | ------------- | -------- |
| Pod 网络      | ✅            | ✅             | ✅        |
| Overlay       | 常见         | 可选          | 可选     |
| NetworkPolicy | 基础能力有限 | 强            | 强       |
| 路由能力      | 有           | 强            | 强       |
| eBPF          | ❌            | 可选/部分场景 | 核心技术 |
| 网络可观测性  | 基础         | 较强          | 很强     |
| 技术复杂度    | 较低         | 中            | 较高     |
| 生产使用      | 常见         | 很常见        | 很常见   |

这里不要简单得出：

```
Flannel 最差
Calico 第二
Cilium 最好
```

正确理解应该是：

> **不同 CNI 针对的能力范围和架构目标不同。**

如果只是需要简单 Pod 网络：

```
Flannel
```

可能已经足够。

如果需要成熟的网络策略：

```
Calico
```

很常见。

如果希望使用：

```
eBPF
高级网络能力
可观测性
```

那么：

```
Cilium
```

是非常重要的选择。

------

## 8.13 Kubernetes 网络模型背后的原理

这一节把前面的知识全部串起来。

### 8.13.1 从 Pod 创建开始

假设创建：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

Kubernetes 创建 Pod 后，大致经历：

```
Pod 创建
   ↓
Scheduler 选择 Node
   ↓
kubelet 发现 Pod
   ↓
Container Runtime 创建容器
   ↓
创建 Network Namespace
   ↓
调用 CNI
   ↓
创建网络接口
   ↓
分配 Pod IP
   ↓
配置路由
   ↓
启动容器
```

最终：

```
Pod
10.244.1.10
```

------

### 8.13.2 Pod 内部

Pod 内部：

```
Pod
│
└── Network Namespace
      │
      └── eth0
            │
            └── Pod IP
```

如果有两个 Container：

```
Pod
│
├── Container A
│
└── Container B
       │
       └── 共享 Network Namespace
```

因此：

```
A → localhost → B
```

可以直接通信。

------

### 8.13.3 Pod 到 Node

网络接口背后通常存在：

```
Pod eth0
    │
    │ veth
    ▼
Node 网络
```

可以理解为：

```
Pod Namespace
     │
    eth0
     │
   veth pair
     │
     ▼
Node
```

这是 Linux Network Namespace + veth 的组合。

------

### 8.13.4 同 Node Pod 到 Pod

例如：

```
Pod A
10.244.1.10
     │
     ▼
Node 网络
     │
     ▼
Pod B
10.244.1.11
```

CNI 根据自己的实现完成：

```
二层转发
或者
三层路由
或者
eBPF
```

最终数据包到达 Pod B。

------

### 8.13.5 跨 Node Pod 到 Pod

例如：

```
Node-1                    Node-2
192.168.1.10              192.168.1.11
   │                          │
Pod A                      Pod B
10.244.1.10                10.244.2.20
```

数据：

```
Pod A
  ↓
Node-1
  ↓
Cluster Network
  ↓
Node-2
  ↓
Pod B
```

其中：

```
Cluster Network
```

可能由：

```
Flannel
Calico
Cilium
```

等 CNI 实现。

------

### 8.13.6 Pod 到 Service

这里又增加了一层。

例如：

```
Pod A
10.244.1.10
     │
     ▼
Service
10.96.0.10
     │
     ▼
Pod B
10.244.2.20
```

Service IP：

```
10.96.0.10
```

通常不是某个真实 Pod 的网络接口。

它是 Kubernetes Service 提供的一个虚拟访问地址。

具体流量转发可能涉及：

```
kube-proxy
iptables
IPVS
eBPF
```

具体取决于集群网络实现。

------

### 8.13.7 Service 为什么能够找到 Pod

Service：

```
selector:
  app: nginx
```

对应：

```
Pod A
labels:
  app: nginx

Pod B
labels:
  app: nginx
```

Kubernetes 根据 Label 建立 Endpoint / EndpointSlice 信息：

```
Service
   │
   ▼
EndpointSlice
   │
   ├── 10.244.1.10
   ├── 10.244.1.11
   └── 10.244.2.20
```

网络组件再根据这些后端地址进行转发。

因此：

```
Client
  ↓
Service
  ↓
EndpointSlice
  ↓
Pod
```

这是后面学习 Service 时必须建立的网络基础。

------

### 8.13.8 为什么 Pod IP 不适合作为服务地址

假设：

```
api Pod
10.244.1.10
```

其他应用直接访问：

```
10.244.1.10
```

Pod 重建以后：

```
旧：
10.244.1.10

新：
10.244.2.15
```

客户端保存的：

```
10.244.1.10
```

就失效了。

因此应用之间通常使用：

```
Service DNS
```

而不是直接依赖 Pod IP。

例如：

```
api.default.svc.cluster.local
```

------

### 8.13.9 Kubernetes 网络的几个基本原则

可以把 Kubernetes 网络模型浓缩成几个原则。

**原则一：Pod 拥有自己的 IP**

```
Pod → Pod IP
```

**原则二：Pod 之间应该可以直接通信**

```
Pod A → Pod B
```

**原则三：Pod 与 Node 之间应该可以通信**

```
Node → Pod
```

**原则四：Pod IP 不应该承担稳定服务身份**

```
Pod IP
   ↓
不稳定

Service
   ↓
稳定访问入口
```

**原则五：具体网络实现由 CNI 完成**

```
Kubernetes Network Model
          ↓
         CNI
          ↓
Flannel / Calico / Cilium / ...
```

------

### 8.13.10 一张图理解整个 Kubernetes 网络

把这一章的知识放到一起：

```
                         Kubernetes Cluster
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   Node-1                                  Node-2              │
│  192.168.1.10                            192.168.1.11        │
│      │                                        │              │
│      │                                        │              │
│   ┌──┴──────────┐                         ┌───┴──────────┐   │
│   │ Pod A       │                         │ Pod B        │   │
│   │10.244.1.10  │                         │10.244.2.20   │   │
│   │             │                         │              │   │
│   │ Container A │                         │ Container B  │   │
│   │ Container C │                         │              │   │
│   └──────┬──────┘                         └──────┬───────┘   │
│          │                                       │           │
│       Network                                  Network       │
│      Namespace                                Namespace      │
│          │                                       │           │
│         eth0                                    eth0          │
│          │                                       │           │
│        veth                                    veth           │
│          │                                       │           │
│          └────────────── CNI ───────────────────┘           │
│                         │                                    │
│               Flannel / Calico / Cilium                     │
│                                                              │
│                         │                                    │
│                      Service                                 │
│                    10.96.0.10                                │
│                         │                                    │
│                    EndpointSlice                             │
│                         │                                    │
│                  ┌──────┴──────┐                             │
│                  ▼             ▼                             │
│               Pod A          Pod B                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

最重要的是理解这几个层次：

```
Linux
  ↓
Network Namespace
  ↓
veth
  ↓
Pod Network
  ↓
CNI
  ↓
Cluster Pod Network
  ↓
Service Network
```

而不同 CNI：

```
Flannel
Calico
Cilium
```

本质上是在解决：

> **如何把这些 Pod 连接成一个可用、可扩展、可控制的 Kubernetes 网络。**

## 本章核心知识

这一章不要急着背 Flannel、Calico、Cilium 的参数，首先把网络路径记清楚：

```
同 Pod：

Container A
    ↓
localhost
    ↓
Container B
同 Node：

Pod A
  ↓
veth / CNI
  ↓
Node 网络
  ↓
Pod B
跨 Node：

Pod A
  ↓
Node A
  ↓
CNI / Cluster Network
  ↓
Node B
  ↓
Pod B
访问 Service：

Client Pod
   ↓
Service IP / DNS
   ↓
Service 转发机制
   ↓
EndpointSlice
   ↓
Backend Pod
```

而整个体系的底层基础是：

```
Network Namespace
veth
Linux Routing
Bridge
iptables / nftables
eBPF
```

CNI 则负责把这些 Linux 网络能力组合起来，为 Kubernetes 提供真正可用的 Pod 网络。

**真正理解这一章的标志，是以后看到 `10.244.x.x`、`Pod IP`、`veth`、`CNI`、`Service IP`、`iptables` 或 `eBPF` 时，能够知道它们分别处于 Kubernetes 网络的哪一层，以及一个 Pod 发出的数据包大致会经过什么路径。**

# 第 9 章：Service——让应用可以被访问

在第 8 章中，我们已经知道：

- Pod 有自己的 IP。
- Pod IP 通常是不稳定的。
- Pod 可以因为 Deployment 扩缩容、滚动更新、故障重建而被创建和删除。
- Kubernetes 网络允许 Pod 之间直接通信。
- CNI 负责提供底层 Pod 网络。

那么现在会出现一个非常现实的问题：

```
Deployment
    │
    ├── Pod A → 10.244.1.10
    ├── Pod B → 10.244.1.11
    └── Pod C → 10.244.2.15
```

如果其他应用需要访问这个 Deployment：

```
应该访问哪个 Pod？
```

更麻烦的是：

```
Pod A 删除
    ↓
新 Pod
10.244.3.20
```

原来的 IP 已经不存在。

所以 Kubernetes 引入了：

> **Service——为一组动态变化的 Pod 提供稳定的访问入口。**

------

## 9.1 为什么需要 Service

### 9.1.1 Pod IP 为什么不能直接作为应用入口

假设有一个 Web 应用：

```
web-1
10.244.1.10

web-2
10.244.1.11

web-3
10.244.2.10
```

如果客户端直接访问：

```
10.244.1.10
```

那么实际上只访问了：

```
web-1
```

如果 `web-1` 被删除：

```
10.244.1.10
```

就失效了。

------

### 9.1.2 Pod 扩缩容会进一步增加问题

例如：

```
Deployment
    │
    ├── Pod A
    ├── Pod B
    └── Pod C
```

扩容：

```
Deployment
    │
    ├── Pod A
    ├── Pod B
    ├── Pod C
    ├── Pod D
    └── Pod E
```

客户端不应该每次都重新获取：

```
Pod A IP
Pod B IP
Pod C IP
Pod D IP
Pod E IP
```

然后自己实现负载均衡。

Kubernetes 应该提供一个稳定的访问入口。

这就是 Service。

------

### 9.1.3 Service 解决什么问题

Service 主要解决：

```
Pod IP 不稳定
Pod 数量动态变化
Pod 负载均衡
服务发现
稳定访问入口
```

例如：

```
             Service
          10.96.10.100
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
      Pod A    Pod B    Pod C
      10.244   10.244   10.244
```

客户端只需要访问：

```
10.96.10.100
```

而不需要知道具体 Pod IP。

------

## 9.2 Service 与 Pod 的关系

### 9.2.1 Service 不是 Pod

这是第一个必须明确的概念。

例如：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

创建：

```
kubectl apply -f service.yaml
```

然后：

```
kubectl get svc
```

可能看到：

```
NAME    TYPE        CLUSTER-IP      PORT(S)
nginx   ClusterIP   10.96.10.100    80/TCP
```

这里的：

```
10.96.10.100
```

不是某个 Pod 的 IP。

Service 是一个 Kubernetes API 对象。

它描述：

> **应该如何访问一组 Pod。**

------

### 9.2.2 Service 与 Deployment 的关系

通常结构是：

```
Deployment
    │
    ├── Pod
    ├── Pod
    └── Pod
          ▲
          │
       Selector
          │
       Service
```

例如 Deployment 创建：

```
metadata:
  labels:
    app: nginx
```

Service：

```
spec:
  selector:
    app: nginx
```

Service 就能找到这些 Pod。

注意：

> **Service 并不直接绑定 Deployment。**

它实际上是通过：

```
Selector → Pod Label
```

找到后端 Pod。

所以一个 Service 完全可以对应：

- Deployment 创建的 Pod
- StatefulSet 创建的 Pod
- Job 创建的 Pod
- 手工创建的 Pod

------

### 9.2.3 Service 的核心作用

可以把 Service 理解成：

```
稳定入口
    ↓
动态 Pod 集合
```

例如：

```
               nginx Service
              10.96.10.100:80
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Pod A        Pod B        Pod C
```

Service 提供稳定的：

```
IP
DNS
端口
访问语义
```

而 Pod 可以自由变化。

------

## 9.3 Selector

### 9.3.1 Selector 是什么

Selector 是 Service 找 Pod 的核心机制。

例如：

```
spec:
  selector:
    app: nginx
```

意思是：

> 找出 Label 中包含 `app=nginx` 的 Pod。

------

### 9.3.2 Pod Label

例如：

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

Pod：

```
labels:
  app=nginx
```

Service：

```
selector:
  app: nginx
```

于是：

```
Service
   │
   │ selector app=nginx
   ▼
Pod
app=nginx
```

------

### 9.3.3 Selector 不匹配会发生什么

这是 Service 最常见的问题之一。

例如 Pod：

```
labels:
  app: nginx
```

Service：

```
selector:
  app: web
```

那么：

```
Service
   │
   │ app=web
   ▼
没有 Pod
```

Service 仍然存在：

```
kubectl get svc
```

也可能显示正常。

但是：

```
Service
    ↓
没有后端
```

最终访问失败。

所以：

> **Service 存在，不代表 Service 有可用后端。**

------

### 9.3.4 查看 Pod Label

```
kubectl get pods --show-labels
```

例如：

```
NAME    READY   STATUS    LABELS
nginx   1/1     Running   app=nginx
```

查看 Service：

```
kubectl get svc nginx -o yaml
```

重点看：

```
spec:
  selector:
    app: nginx
```

这两个必须匹配。

------

## 9.4 Service ClusterIP

### 9.4.1 ClusterIP 是什么

ClusterIP 是 Service 最常见的访问方式。

例如：

```
Service
ClusterIP = 10.96.10.100
Port = 80
```

这个 IP：

> **主要用于 Kubernetes 集群内部访问。**

例如：

```
Pod A
  │
  │ http://10.96.10.100
  ▼
Service
  │
  ▼
Pod B
```

集群外部通常不能直接访问这个 ClusterIP。

------

### 9.4.2 ClusterIP 是虚拟 IP

ClusterIP 很容易让初学者误解。

例如：

```
10.96.10.100
```

并不一定对应：

```
某台 Node 的真实网卡
```

也不对应：

```
某个 Pod 的 eth0
```

它是一个：

> **Service 的虚拟访问地址。**

流量到达 ClusterIP 后，再由 Kubernetes 的 Service 转发机制找到后端 Pod。

------

### 9.4.3 创建 ClusterIP Service

例如：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

应用：

```
kubectl apply -f nginx-service.yaml
```

查看：

```
kubectl get svc nginx
```

可能得到：

```
NAME    TYPE        CLUSTER-IP      PORT(S)
nginx   ClusterIP   10.96.10.100    80/TCP
```

------

### 9.4.4 `port` 和 `targetPort`

这是 Service 初学者最容易混淆的两个字段。

```
ports:
  - port: 80
    targetPort: 8080
```

表示：

```
客户端
   │
   │ Service:80
   ▼
Service
   │
   │ 转发
   ▼
Pod:8080
```

所以：

```
port
```

是：

> Service 暴露的端口。

而：

```
targetPort
```

是：

> 后端 Pod 接收流量的端口。

例如应用实际监听：

```
8080
```

那么可以：

```
ports:
  - port: 80
    targetPort: 8080
```

客户端访问：

```
http://nginx:80
```

最终到：

```
Pod:8080
```

------

## 9.5 Service Endpoint / EndpointSlice

### 9.5.1 Endpoint 是什么

Service 本身并不知道应该把流量发送到哪些具体 Pod。

例如：

```
Service
10.96.10.100
```

后端可能是：

```
10.244.1.10:80
10.244.1.11:80
10.244.2.10:80
```

这些后端地址就是 Service 的 Endpoint 信息。

------

### 9.5.2 查看 Endpoint

传统方式：

```
kubectl get endpoints nginx
```

可能看到：

```
NAME    ENDPOINTS
nginx   10.244.1.10:80,10.244.1.11:80
```

这意味着：

```
Service
   │
   ├── 10.244.1.10:80
   └── 10.244.1.11:80
```

------

### 9.5.3 EndpointSlice

现代 Kubernetes 更推荐 EndpointSlice。

查看：

```
kubectl get endpointslice
```

或者：

```
kubectl get endpointslice -l kubernetes.io/service-name=nginx
```

EndpointSlice 用于保存 Service 后端 Endpoint 信息。

可以理解成：

```
Service
   │
   ▼
EndpointSlice
   │
   ├── Pod A
   ├── Pod B
   └── Pod C
```

------

### 9.5.4 为什么需要 EndpointSlice

假设一个 Service 后面有很多 Pod：

```
Pod 1
Pod 2
...
Pod 1000
```

如果所有 Endpoint 都集中到一个巨大对象中：

```
一个 Service
     ↓
大量 Endpoint
```

随着规模扩大，管理效率会受到影响。

EndpointSlice 可以把 Endpoint 分成多个 Slice：

```
Service
   │
   ├── EndpointSlice 1
   ├── EndpointSlice 2
   ├── EndpointSlice 3
   └── ...
```

更适合大型集群。

------

### 9.5.5 EndpointSlice 与 Service 的关系

完整关系：

```
Pod
 │
 │ Label
 ▼
Service Selector
 │
 ▼
EndpointSlice Controller
 │
 ▼
EndpointSlice
 │
 ├── Pod A IP
 ├── Pod B IP
 └── Pod C IP
```

然后网络组件根据这些后端信息完成实际流量转发。

------

## 9.6 Service 类型

Service 有四种主要类型：

```
ClusterIP
NodePort
LoadBalancer
ExternalName
```

可以先建立整体认识：

| 类型         | 主要用途                       |
| ------------ | ------------------------------ |
| ClusterIP    | 集群内部访问                   |
| NodePort     | 通过 Node IP + 端口访问        |
| LoadBalancer | 通过外部负载均衡器访问         |
| ExternalName | 将 Service 映射到外部 DNS 名称 |

------

### 9.6.1 ClusterIP

默认类型就是：

```
type: ClusterIP
```

例如：

```
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

访问方式：

```
集群内部
    ↓
backend:80
    ↓
Pod:8080
```

典型用途：

```
Frontend → Backend
Backend → Redis
Backend → MySQL
```

只需要集群内部访问时，优先考虑 ClusterIP。

------

### 9.6.2 NodePort

NodePort 在 ClusterIP 的基础上，额外提供：

```
Node IP:NodePort
```

例如：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

于是可以通过：

```
Node-1:30080
Node-2:30080
Node-3:30080
```

访问 Service。

流量逻辑可以理解为：

```
Client
  │
  ▼
Node IP:30080
  │
  ▼
Service
  │
  ▼
Pod:80
```

------

### 9.6.3 NodePort 的端口范围

默认情况下，NodePort 通常使用：

```
30000-32767
```

例如：

```
30080
```

但实际范围可以通过 Kubernetes API Server 配置调整。

------

### 9.6.4 NodePort 的生产注意事项

NodePort 很方便，但生产环境通常不会直接让用户访问：

```
NodeIP:30080
```

因为：

```
Node IP
NodePort
```

暴露方式比较原始。

更常见的生产结构是：

```
Internet
   ↓
Load Balancer
   ↓
Ingress / Gateway
   ↓
Service
   ↓
Pod
```

或者：

```
Internet
   ↓
Cloud Load Balancer
   ↓
NodePort
   ↓
Service
   ↓
Pod
```

NodePort 更适合作为底层暴露机制，而不是最终用户体验入口。

------

### 9.6.5 LoadBalancer

LoadBalancer 用于：

> **通过外部负载均衡器暴露 Service。**

例如：

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

在支持云负载均衡的环境中，Kubernetes 可以请求云平台创建：

```
External Load Balancer
```

例如：

```
Internet
    │
    ▼
Cloud Load Balancer
    │
    ▼
Kubernetes Service
    │
    ▼
Pod
```

------

### 9.6.6 LoadBalancer 并不是 Kubernetes 自己“凭空创建”硬件

这是一个重要概念。

在云环境：

```
AWS
Azure
GCP
阿里云
腾讯云
```

等平台中，通常由对应的云控制器 / 集成机制调用云平台 API。

因此：

```
Service type=LoadBalancer
```

背后可能发生：

```
Kubernetes
    ↓
Cloud Controller
    ↓
Cloud Provider API
    ↓
创建 Load Balancer
```

在裸机环境中：

```
type: LoadBalancer
```

不一定自动就能获得公网 IP。

通常需要额外的负载均衡实现。

------

### 9.6.7 ExternalName

ExternalName 的思路和前面三种不同。

它不是把流量转发到 Pod，而是：

> **让 Kubernetes Service 指向一个外部 DNS 名称。**

例如：

```
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

集群内部访问：

```
external-db
```

可以解析到：

```
db.example.com
```

它主要依赖 DNS CNAME 语义。

------

### 9.6.8 ExternalName 与普通 Service 的区别

普通 Service：

```
Service
  ↓
EndpointSlice
  ↓
Pod
```

ExternalName：

```
Service
  ↓
DNS CNAME
  ↓
外部域名
```

因此它通常没有普通 ClusterIP Service 那样的 Pod Endpoint 转发模型。

------

## 9.7 Service DNS

### 9.7.1 为什么需要 DNS

假设 Service IP：

```
10.96.10.100
```

让应用直接写：

```
http://10.96.10.100
```

存在两个问题：

1. IP 不够直观。
2. 应用配置与具体 Service IP 强耦合。

所以 Kubernetes 提供 Service DNS。

例如 Service：

```
backend
```

应用可以访问：

```
http://backend
```

------

### 9.7.2 Service 的完整 DNS 名称

一个 Service 的完整 DNS 名称通常类似：

```
<service>.<namespace>.svc.cluster.local
```

例如：

```
backend.default.svc.cluster.local
```

组成：

```
backend
   │
   └── Service 名称

default
   │
   └── Namespace

svc
   │
   └── Service 域

cluster.local
   │
   └── Cluster DNS 域
```

------

### 9.7.3 同 Namespace 下访问

假设：

```
Service:
backend

Namespace:
default
```

那么同一个 Namespace 的 Pod 通常可以直接：

```
curl http://backend
```

访问。

也可以：

```
curl http://backend.default.svc.cluster.local
```

------

### 9.7.4 跨 Namespace 访问

假设：

```
backend
Namespace = production
```

那么其他 Namespace 可以使用：

```
backend.production.svc.cluster.local
```

访问。

例如：

```
curl http://backend.production.svc.cluster.local
```

这也是生产环境中非常重要的服务发现机制。

------

### 9.7.5 DNS 由谁提供

Kubernetes 集群通常部署：

```
CoreDNS
```

CoreDNS 负责 Kubernetes Service 等 DNS 信息的解析。

例如：

```
Pod
  │
  │ DNS Query
  ▼
CoreDNS
  │
  ▼
Service DNS
  │
  ▼
Service
```

可以查看：

```
kubectl get pods -n kube-system
```

通常可以看到 CoreDNS Pod。

------

## 9.8 Kubernetes 内部服务发现

### 9.8.1 服务发现是什么

服务发现的核心问题：

> 应用如何找到另一个应用？

例如：

```
Frontend
   ↓
Backend
```

Frontend 不应该知道：

```
Backend Pod A IP
Backend Pod B IP
Backend Pod C IP
```

它只需要知道：

```
backend
```

然后 Kubernetes DNS 帮它找到 Service。

------

### 9.8.2 典型访问过程

例如：

```
Frontend Pod
    │
    │ 请求 backend
    ▼
CoreDNS
    │
    │ 解析
    ▼
backend.default.svc.cluster.local
    │
    ▼
Service ClusterIP
    │
    ▼
Backend Pod
```

应用因此不需要自己管理 Pod IP。

------

### 9.8.3 服务发现的真正价值

Service + DNS 将：

```
应用身份
```

与：

```
Pod 实例
```

分离。

例如：

```
backend
```

代表：

```
Backend 服务
```

而不是：

```
10.244.1.10
```

这样 Backend Pod 怎么扩缩容、怎么滚动更新，都不会影响调用方的配置。

这是微服务架构非常重要的基础。

------

## 9.9 Service → Pod 流量过程

这一部分非常重要。

假设：

```
Service:
backend

ClusterIP:
10.96.10.100

Backend Pods:

10.244.1.10:8080
10.244.1.11:8080
10.244.2.20:8080
```

客户端：

```
Frontend Pod
```

执行：

```
curl http://backend
```

大致经历以下过程。

------

### 9.9.1 第一步：DNS 解析

Frontend Pod 查询：

```
backend
```

CoreDNS 返回：

```
10.96.10.100
```

于是：

```
backend
    ↓
10.96.10.100
```

------

### 9.9.2 第二步：访问 ClusterIP

Frontend Pod 发起：

```
10.96.10.100:80
```

注意：

```
10.96.10.100
```

不是 Backend Pod IP。

它是 Service 的 ClusterIP。

------

### 9.9.3 第三步：Service 转发

Kubernetes 网络组件识别：

```
目标：
10.96.10.100:80
```

然后根据 Service 的后端 Endpoint：

```
10.244.1.10:8080
10.244.1.11:8080
10.244.2.20:8080
```

选择一个后端。

例如：

```
10.244.1.11:8080
```

------

### 9.9.4 第四步：数据包到达 Pod

最终：

```
Frontend
   │
   ▼
10.96.10.100:80
   │
   ▼
10.244.1.11:8080
   │
   ▼
Backend Pod
```

------

### 9.9.5 kube-proxy 在这里做什么

传统 Kubernetes 网络中，经常会看到：

```
kube-proxy
```

它负责实现 Service 的部分网络转发逻辑。

常见实现包括：

```
iptables
IPVS
```

现代网络方案也可能使用：

```
eBPF
```

直接实现或替代部分传统 kube-proxy 功能。

所以不要死记：

```
Service = kube-proxy
```

更准确的是：

> **Service 是 Kubernetes API 对象；实际 Service 流量转发由集群网络数据面实现。**

------

### 9.9.6 一个完整的数据流

最终可以理解为：

```
┌──────────────┐
│ Frontend Pod │
└──────┬───────┘
       │
       │ DNS Query
       ▼
┌──────────────┐
│   CoreDNS    │
└──────┬───────┘
       │
       │ ClusterIP
       ▼
┌──────────────┐
│   Service    │
│10.96.10.100  │
└──────┬───────┘
       │
       │ Service forwarding
       ▼
┌──────────────────────────┐
│       EndpointSlice      │
│ 10.244.1.10              │
│ 10.244.1.11              │
│ 10.244.2.20              │
└────────────┬─────────────┘
             │
             ▼
       ┌───────────┐
       │ Backend   │
       │   Pod     │
       └───────────┘
```

这就是 Kubernetes 内部服务发现和 Service 转发的核心流程。

------

## 9.10 Headless Service

### 9.10.1 什么是 Headless Service

Headless Service 是：

> **不分配 ClusterIP 的 Service。**

定义：

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

关键：

```
clusterIP: None
```

------

### 9.10.2 普通 Service

普通 Service：

```
Service
ClusterIP
10.96.10.100
       │
       ├── Pod A
       ├── Pod B
       └── Pod C
```

客户端访问：

```
backend
```

通常得到：

```
ClusterIP
```

然后再进行 Service 流量转发。

------

### 9.10.3 Headless Service

Headless Service：

```
Service
clusterIP = None
       │
       ├── Pod A
       ├── Pod B
       └── Pod C
```

DNS 不再主要提供一个普通 Service ClusterIP，而是可以直接返回后端 Pod 地址。

例如：

```
mysql.default.svc.cluster.local
```

可能解析得到：

```
10.244.1.10
10.244.1.11
10.244.2.20
```

因此：

> **Headless Service 更像是通过 DNS 把服务背后的实例直接暴露给客户端发现。**

------

### 9.10.4 为什么需要 Headless Service

一些有状态系统需要知道：

```
具体是哪一个实例
```

例如：

```
database-0
database-1
database-2
```

它们不是完全等价的 Web Pod。

可能存在：

```
主节点
从节点
Leader
Follower
数据分片
成员发现
```

这时普通 ClusterIP Service 的：

```
一个虚拟 IP
```

反而可能不够。

Headless Service 可以帮助应用直接发现具体 Pod。

------

### 9.10.5 StatefulSet 与 Headless Service

这也是为什么 StatefulSet 经常和 Headless Service 一起出现。

例如：

```
StatefulSet
   │
   ├── db-0
   ├── db-1
   └── db-2
```

Headless Service：

```
db
```

可以形成稳定的 DNS 身份，例如：

```
db-0.db.default.svc.cluster.local
db-1.db.default.svc.cluster.local
db-2.db.default.svc.cluster.local
```

这样有状态应用就能够识别：

```
具体实例
```

而不是只知道：

```
某一个随机 Backend
```

------

## 9.11 Service 常见故障排查

Service 故障排查一定不要一上来就：

```
kubectl delete svc
```

正确方法是按照网络链路逐层排查。

------

### 9.11.1 第一步：确认 Service 是否存在

```
kubectl get svc
```

例如：

```
NAME      TYPE        CLUSTER-IP      PORT(S)
backend   ClusterIP   10.96.10.100    80/TCP
```

查看详细信息：

```
kubectl describe svc backend
```

重点关注：

```
Type
IP
Port
TargetPort
Selector
Endpoints
```

------

### 9.11.2 第二步：检查 Selector

查看：

```
kubectl get svc backend -o yaml
```

例如：

```
selector:
  app: backend
```

然后查看 Pod：

```
kubectl get pods --show-labels
```

确认：

```
Pod:
app=backend
```

如果没有匹配：

```
Service
    ↓
Selector
    ↓
0 Pod
```

那么问题已经找到了。

------

### 9.11.3 第三步：检查 EndpointSlice

执行：

```
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

如果没有后端地址：

```
ENDPOINTS
---------
```

说明 Service 没有发现可用后端。

可以进一步：

```
kubectl describe endpointslice <name>
```

------

### 9.11.4 第四步：确认 Pod 是否 Ready

这是生产环境非常重要的一点。

假设有：

```
Pod A   Running   Ready
Pod B   Running   NotReady
```

Pod B 即使进程还活着，也可能不会作为正常 Service 后端接收流量。

查看：

```
kubectl get pods
```

重点看：

```
READY
```

例如：

```
NAME       READY   STATUS
backend-1  1/1     Running
backend-2  0/1     Running
```

进一步：

```
kubectl describe pod backend-2
```

检查：

```
Readiness Probe
Events
Container status
```

------

### 9.11.5 第五步：确认 `port` 和 `targetPort`

例如：

```
ports:
  - port: 80
    targetPort: 8080
```

意味着：

```
Service:80
     ↓
Pod:8080
```

如果应用实际监听：

```
8081
```

那么：

```
Service
80
 ↓
Pod
8080
 ↓
没有进程监听
```

自然会失败。

可以进入 Pod 检查：

```
kubectl exec -it <pod> -- sh
```

然后：

```
ss -lnt
```

确认应用到底监听哪个端口。

------

### 9.11.6 第六步：从集群内部测试 DNS

可以启动临时测试 Pod：

```
kubectl run net-test \
  --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- sh
```

进入后：

```
nslookup backend
```

或者：

```
nslookup backend.default.svc.cluster.local
```

如果 DNS 无法解析：

```
backend
  ↓
DNS 失败
```

就不要继续排查 Service 转发，应该先排查 CoreDNS。

------

### 9.11.7 第七步：直接访问 ClusterIP

在测试 Pod 中：

```
wget -qO- http://10.96.10.100
```

或者：

```
wget -qO- http://backend
```

如果：

```
DNS 正常
Pod IP 正常
Service IP 访问失败
```

说明问题可能位于：

```
Service
EndpointSlice
Service 数据面
网络策略
CNI
```

等层面。

------

### 9.11.8 第八步：绕过 Service，直接访问 Pod

先查看 Pod IP：

```
kubectl get pods -o wide
```

例如：

```
backend-xxx
10.244.1.20
```

然后从测试 Pod：

```
wget -qO- http://10.244.1.20:8080
```

如果：

```
Pod IP 可以访问
Service IP 不能访问
```

那么应用本身大概率没问题。

重点检查：

```
Service Selector
EndpointSlice
port / targetPort
Service 数据面
NetworkPolicy
CNI
```

反过来，如果：

```
Pod IP 也访问不了
```

则应该优先检查：

```
Pod
应用监听
Readiness
Pod 网络
NetworkPolicy
CNI
```

------

### 9.11.9 Service 故障排查路径

生产环境建议形成固定习惯：

```
客户端
  │
  ▼
DNS
  │
  ▼
Service
  │
  ▼
Selector
  │
  ▼
EndpointSlice
  │
  ▼
Pod Ready
  │
  ▼
Pod IP:targetPort
  │
  ▼
应用进程
```

逐层确认。

可以使用：

```
kubectl get svc
kubectl describe svc <service>
kubectl get endpointslice
kubectl get pods --show-labels
kubectl get pods -o wide
kubectl describe pod <pod>
```

再配合集群内部测试：

```
kubectl run net-test \
  --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- sh
```

然后测试：

```
nslookup <service>
wget -qO- http://<service>:<port>
wget -qO- http://<pod-ip>:<target-port>
```

这比反复删除、重建 Service 有效得多。

------

## 本章核心知识

Service 最核心的思想可以浓缩成一句话：

> **Pod 负责运行实例，Service 负责提供稳定的访问入口。**

完整关系：

```
                Service
           backend:80
                 │
                 ▼
          ClusterIP
         10.96.10.100
                 │
                 ▼
           EndpointSlice
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
    Pod A      Pod B      Pod C
```

其中：

```
Selector
```

负责：

```
Service → 找到哪些 Pod
```

而：

```
EndpointSlice
```

负责记录：

```
当前有哪些后端 Endpoint
```

Service 类型：

```
ClusterIP
    ↓
集群内部访问

NodePort
    ↓
Node IP + NodePort

LoadBalancer
    ↓
外部负载均衡器

ExternalName
    ↓
外部 DNS 名称
```

服务发现：

```
应用
  ↓
Service DNS
  ↓
ClusterIP
  ↓
EndpointSlice
  ↓
Pod
```

Headless Service 则是另一种模型：

```
Service
clusterIP=None
     ↓
DNS
     ↓
具体 Pod IP
```

因此，真正需要掌握的不是某几个 YAML 字段，而是这条完整链路：

```
                Service
                   │
                   │ Selector
                   ▼
              EndpointSlice
                   │
                   ▼
                 Pod
                   │
                   ▼
            targetPort
```

以及：

```
应用
 ↓
DNS
 ↓
Service
 ↓
EndpointSlice
 ↓
Pod IP
 ↓
Pod
 ↓
应用进程
```

**一旦 Service 访问出现问题，就沿着这条链路逐层排查，而不是把 Service 当成一个简单的“虚拟 IP”。**

# 第 10 章：Ingress——让外部用户访问应用

在上一章中，我们解决了一个核心问题：

> **Service 如何让 Kubernetes 集群内部的应用稳定地被访问。**

例如：

```
Frontend Pod
    │
    ▼
backend.default.svc.cluster.local
    │
    ▼
Service
    │
    ▼
Backend Pod
```

但生产环境还有一个更直接的问题：

> **互联网用户如何访问 Kubernetes 里面的应用？**

用户不会访问：

```
10.96.10.100
```

也不会希望记住：

```
NodeIP:30080
```

而是希望：

```
https://www.example.com
https://api.example.com
```

并且多个应用能够共用入口：

```
                    Internet
                       │
                       ▼
                Ingress Controller
                  /      |       \
                 /       |        \
                ▼        ▼         ▼
             Service   Service   Service
               │         │         │
              Web       API       Admin
```

这就是 Ingress 要解决的问题。

------

## 10.1 为什么需要 Ingress

### 10.1.1 直接使用 Service 暴露应用的问题

假设我们有三个应用：

```
Web
API
Admin
```

如果全部使用 LoadBalancer：

```
Web
 ↓
LoadBalancer 1

API
 ↓
LoadBalancer 2

Admin
 ↓
LoadBalancer 3
```

这会带来：

- 多个公网入口
- 多个公网 IP / LB
- 成本增加
- TLS 配置分散
- 路由规则分散
- 运维复杂度增加

生产环境更常见的设计是：

```
                    Internet
                       │
                       ▼
              一个统一流量入口
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
             Web       API     Admin
```

Ingress 就是 Kubernetes 中用于描述这种 HTTP/HTTPS 路由的核心 API。

------

### 10.1.2 Ingress 解决什么问题

Ingress 主要解决：

```
外部 HTTP/HTTPS 请求
        ↓
根据域名 / Path
        ↓
转发到 Kubernetes Service
        ↓
Pod
```

例如：

```
www.example.com
        ↓
      Web

api.example.com
        ↓
      API

admin.example.com
        ↓
      Admin
```

或者：

```
example.com/
        ↓
      Web

example.com/api
        ↓
      API

example.com/admin
        ↓
      Admin
```

------

### 10.1.3 Ingress 主要解决的是 HTTP/HTTPS

这是一个非常重要的边界。

Ingress API 的传统模型主要面向：

```
HTTP
HTTPS
```

因此它特别适合：

```
Web
REST API
HTTP 微服务
HTTPS 网站
```

而不是所有 TCP/UDP 服务的通用入口。

例如：

```
MySQL:3306
Redis:6379
```

不能简单理解成：

```
Ingress = 所有网络流量入口
```

生产环境中，TCP/UDP 等其他协议通常使用其他暴露方式或 Gateway / LB 能力。

------

## 10.2 Ingress 与 Service 的关系

### 10.2.1 两者解决不同层次的问题

可以把两者分成：

```
Ingress
    ↓
外部 HTTP/HTTPS 流量入口

Service
    ↓
Kubernetes 内部服务访问
```

完整结构：

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

------

### 10.2.2 一个典型例子

假设有：

```
frontend Service
backend Service
```

Ingress：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

流量：

```
www.example.com
       │
       ▼
    Ingress
       │
       ▼
frontend Service
       │
       ▼
Frontend Pods
```

而：

```
api.example.com
       │
       ▼
    Ingress
       │
       ▼
backend Service
       │
       ▼
Backend Pods
```

所以：

> **Ingress 通常不直接把流量发送给 Pod，而是将请求路由到 Service。**

------

### 10.2.3 Ingress 不等于 Service

这是初学阶段非常容易混淆的地方。

```
Ingress
```

解决：

> HTTP/HTTPS 请求应该进入哪个 Service？

而：

```
Service
```

解决：

> 请求进入这个 Service 后，应该发送给哪些 Pod？

因此：

```
Ingress
   ↓
选择 Service

Service
   ↓
选择 Pod
```

两层职责不同。

------

## 10.3 Ingress Controller

### 10.3.1 为什么光有 Ingress 不够

这是理解 Ingress 最重要的概念之一。

你创建：

```
kind: Ingress
```

并不会自动出现一个 Web 服务器。

例如：

```
kubectl apply -f ingress.yaml
```

只是创建了一个 Kubernetes API 对象。

它描述：

```
www.example.com
       ↓
frontend Service
```

但是：

> **谁真正监听 80/443？谁接收 HTTP 请求？谁执行路由？**

答案就是：

> **Ingress Controller。**

------

### 10.3.2 Ingress 和 Controller 的关系

可以理解成：

```
Ingress
  │
  │ 声明“我要什么”
  ▼
Ingress Controller
  │
  │ 真正执行
  ▼
HTTP/HTTPS 流量
```

Ingress 更像：

```
配置 / 规则
```

Controller 更像：

```
实际运行的软件
```

------

### 10.3.3 Ingress Controller 做什么

Ingress Controller 通常负责：

```
监听 HTTP/HTTPS
读取 Ingress 资源
解析域名
解析 Path
选择 Service
处理 TLS
转发请求
```

例如：

```
Client
  │
  │ HTTPS
  ▼
Ingress Controller
  │
  ├── www.example.com → frontend
  │
  ├── api.example.com → backend
  │
  └── admin.example.com → admin
```

------

### 10.3.4 Ingress Controller 的实现并不唯一

Ingress 是 Kubernetes API。

具体怎么实现，可以有不同的软件。

常见实现包括：

- NGINX Ingress Controller
- Traefik
- HAProxy
- Kong
- 其他实现

因此：

> **Ingress 是 API/配置模型，Ingress Controller 是实现这个模型的软件。**

------

## 10.4 Nginx Ingress

### 10.4.1 什么是 Nginx Ingress

NGINX Ingress Controller 是一种非常常见的 Ingress Controller 实现。

它基于 NGINX 处理：

```
HTTP
HTTPS
```

请求。

整体结构：

```
Internet
   │
   ▼
NGINX Ingress Controller
   │
   ├── example.com
   │       ↓
   │    frontend
   │
   └── api.example.com
           ↓
        backend
```

------

### 10.4.2 Ingress Controller 本身也需要被暴露

这是一个非常重要的现实问题。

假设：

```
Ingress Controller
```

运行在 Pod 中：

```
nginx-ingress-controller Pod
```

互联网怎么访问这个 Pod？

通常需要再通过：

```
LoadBalancer
NodePort
Host Network
云 LB
```

等方式把它暴露出去。

例如：

```
Internet
    │
    ▼
Cloud Load Balancer
    │
    ▼
NGINX Ingress Controller
    │
    ▼
Service
    │
    ▼
Pod
```

所以：

> **Ingress 不是凭空创造公网入口的。**

Ingress Controller 本身仍然需要一个可以接收外部流量的网络入口。

------

### 10.4.3 NGINX Ingress 的生产价值

它可以集中处理：

```
域名
Path
TLS
HTTP → HTTPS
请求转发
```

例如：

```
               Internet
                   │
                   ▼
             NGINX Ingress
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
     Web           API        Admin
```

这样应用 Service 可以保持：

```
ClusterIP
```

而不需要每个应用都暴露公网。

------

## 10.5 Traefik

### 10.5.1 Traefik 是什么

Traefik 也是 Kubernetes 常见的入口控制器 / 反向代理实现。

它同样可以处理：

```
HTTP
HTTPS
域名
Path
TLS
Service 路由
```

基本结构：

```
Internet
   ↓
Traefik
   ↓
Service
   ↓
Pod
```

------

### 10.5.2 Traefik 与 NGINX Ingress 的关系

它们不是：

```
Ingress + NGINX
Ingress + Traefik
```

这样的 API 关系。

更准确地说：

```
Kubernetes Ingress API
       │
       ├── NGINX Controller
       │
       ├── Traefik
       │
       └── 其他 Controller
```

也就是说：

> **它们都是 Ingress API 的不同实现。**

------

### 10.5.3 如何选择

初学阶段不需要同时深入学习多个 Controller。

生产选型时可以考虑：

```
团队经验
社区生态
云平台支持
性能需求
配置方式
扩展能力
可观测性
安全能力
运维复杂度
```

核心原则是：

> **先理解 Kubernetes 的 Ingress 模型，再学习具体 Controller 的实现。**

否则很容易把：

```
Ingress API
```

和：

```
某个 Controller 的专有功能
```

混在一起。

------

## 10.6 Gateway API 的基本认知

### 10.6.1 为什么会出现 Gateway API

传统 Ingress API 比较简单。

例如：

```
域名
Path
Service
TLS
```

这些需求能够很好地表达。

但是随着生产环境复杂化，会出现：

```
不同团队管理路由
更复杂的流量治理
TCP / UDP
更丰富的路由规则
角色分离
Gateway 生命周期管理
```

传统 Ingress 的表达能力开始显得有限。

于是 Kubernetes 社区发展出了：

> **Gateway API。**

------

### 10.6.2 Gateway API 的核心思想

传统 Ingress：

```
Ingress
   ↓
Ingress Controller
```

Gateway API 则更强调：

```
GatewayClass
      ↓
Gateway
      ↓
HTTPRoute
      ↓
Service
      ↓
Pod
```

可以先记住这几个核心资源：

```
GatewayClass
Gateway
HTTPRoute
```

------

### 10.6.3 GatewayClass

GatewayClass 可以理解为：

> **定义由哪一类 Gateway Controller 来实现 Gateway。**

类似：

```
Ingress
   ↓
Ingress Controller
```

对应到 Gateway API：

```
GatewayClass
   ↓
Gateway Controller
```

------

### 10.6.4 Gateway

Gateway 表示实际的流量入口。

例如：

```
Internet
   ↓
Gateway
   ↓
HTTPRoute
```

可以理解为：

> **“我要一个这样的网络入口。”**

------

### 10.6.5 HTTPRoute

HTTPRoute 用于描述 HTTP 路由：

```
域名
Path
Header
Method
Backend
```

例如：

```
api.example.com
    ↓
HTTPRoute
    ↓
backend Service
```

------

### 10.6.6 Gateway API 与 Ingress 的关系

当前学习阶段可以这样理解：

```
Ingress
    ↓
较简单的 HTTP/HTTPS 入口模型

Gateway API
    ↓
更加结构化、可扩展的流量入口模型
```

不要简单理解为：

```
Gateway API = 新版 Ingress
```

它更准确地代表 Kubernetes 对网络流量入口和路由模型的一次扩展。

------

## 10.7 域名路由

这是 Ingress 最核心的使用场景之一。

假设：

```
www.example.com
api.example.com
admin.example.com
```

分别对应：

```
frontend
backend
admin
```

可以通过 Host 路由。

------

### 10.7.1 基本 YAML

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

请求：

```
https://www.example.com
```

进入：

```
frontend
```

请求：

```
https://api.example.com
```

进入：

```
backend
```

------

### 10.7.2 DNS 还需要正确配置

Ingress YAML 写了：

```
www.example.com
```

并不意味着公网 DNS 自动知道它在哪里。

你还需要：

```
DNS
  ↓
Ingress Controller 的公网 IP / LB
```

例如：

```
www.example.com
       ↓
203.0.113.10
       ↓
Load Balancer
       ↓
Ingress Controller
```

因此生产环境必须同时考虑：

```
DNS
+
Load Balancer
+
Ingress Controller
+
Ingress Rule
```

四者缺一不可。

------

## 10.8 Path 路由

除了根据域名路由，还可以根据 URL Path。

例如：

```
example.com/
example.com/api
example.com/admin
```

分别进入：

```
frontend
backend
admin
```

------

### 10.8.1 Prefix 路由

例如：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

于是：

```
example.com/
```

→ frontend

```
example.com/api
```

→ backend

```
example.com/api/users
```

也匹配：

```
/api
```

因此：

> `Prefix` 表示前缀匹配。

------

### 10.8.2 PathType

Kubernetes Ingress 常见的 PathType：

```
Prefix
Exact
ImplementationSpecific
```

初学阶段重点掌握：

```
Prefix
Exact
```

例如：

```
path: /api
pathType: Prefix
```

匹配：

```
/api
/api/
/api/users
/api/orders/123
```

而：

```
path: /api
pathType: Exact
```

主要匹配：

```
/api
```

不会按照 Prefix 那样匹配：

```
/api/users
```

------

## 10.9 HTTPS

生产环境几乎不应该让用户长期使用：

```
http://example.com
```

而应该使用：

```
https://example.com
```

因此 Ingress 通常需要承担 TLS 终止。

------

### 10.9.1 HTTPS 流量结构

典型结构：

```
Client
   │
   │ HTTPS
   ▼
Load Balancer
   │
   ▼
Ingress Controller
   │
   │ TLS termination
   ▼
HTTP / HTTPS
   │
   ▼
Service
   │
   ▼
Pod
```

常见模式是：

```
客户端
  │ HTTPS
  ▼
Ingress Controller
  │
  │ 解密
  ▼
Service
  │
  ▼
Pod
```

这叫：

> **TLS Termination（TLS 终止）**

------

### 10.9.2 为什么在 Ingress 终止 TLS

集中管理证书：

```
Certificate
    ↓
Ingress
    ↓
多个后端 Service
```

而不是：

```
Web Pod → 自己管理证书
API Pod → 自己管理证书
Admin Pod → 自己管理证书
```

这样可以减少：

```
证书配置
证书更新
证书续期
密钥管理
```

的复杂度。

------

## 10.10 TLS Certificate

### 10.10.1 Kubernetes 如何保存 TLS 证书

Ingress 通常使用 Kubernetes Secret 保存：

```
TLS Certificate
Private Key
```

例如：

```
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key
```

查看：

```
kubectl get secret example-tls
```

类型通常是：

```
kubernetes.io/tls
```

------

### 10.10.2 Ingress 使用 TLS Secret

例如：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls

  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

这里：

```
secretName: example-tls
```

表示：

> `example.com` 使用 `example-tls` 中保存的证书。

------

### 10.10.3 证书与域名必须匹配

例如用户访问：

```
https://example.com
```

证书应该覆盖：

```
example.com
```

如果证书只覆盖：

```
api.example.com
```

那么浏览器会出现证书域名不匹配。

生产环境必须关注：

```
证书有效期
证书域名
SAN
私钥
证书链
自动续期
```

------

### 10.10.4 生产环境不要手工频繁更新证书

生产环境常见做法是：

```
cert-manager
```

配合：

```
Let's Encrypt
```

等 CA 自动申请和续期证书。

整体：

```
Ingress
   │
   ▼
Certificate
   │
   ▼
cert-manager
   │
   ▼
CA
```

证书续期后自动更新 Kubernetes Secret。

这比人工：

```
kubectl create secret tls ...
```

然后几个月后再手动更换，更适合生产环境。

------

## 10.11 HTTP → HTTPS

### 10.11.1 为什么需要重定向

如果用户访问：

```
http://example.com
```

希望自动变成：

```
https://example.com
```

也就是：

```
HTTP
 ↓
301/308
 ↓
HTTPS
```

这样可以避免用户继续使用明文 HTTP。

------

### 10.11.2 HTTP → HTTPS 的实现

具体配置方式取决于 Ingress Controller。

例如某些 NGINX Ingress 配置可以通过 TLS 和 Controller 行为实现 HTTP 自动重定向。

核心逻辑是：

```
80
 ↓
redirect
 ↓
443
```

生产环境应该明确：

```
HTTP 是否允许
HTTP 是否仅用于重定向
HTTPS 是否强制
HSTS 是否启用
```

------

### 10.11.3 为什么不应该把所有 HTTP 都直接禁掉

通常：

```
HTTP
 ↓
HTTPS Redirect
```

比直接：

```
HTTP → connection refused
```

更友好。

因为：

- 老链接可能仍然使用 HTTP。
- 用户可能手工输入 HTTP。
- 搜索引擎或第三方系统可能仍保存旧 URL。

因此常见做法：

```
HTTP → 301/308 → HTTPS
```

------

## 10.12 多域名部署

生产环境中非常常见：

```
www.example.com
api.example.com
admin.example.com
```

共用一个 Ingress Controller。

结构：

```
                         Internet
                            │
                            ▼
                    Load Balancer
                            │
                            ▼
                  Ingress Controller
                   /        |        \
                  /         |         \
                 ▼          ▼          ▼
             www.xxx     api.xxx    admin.xxx
                 │          │          │
                 ▼          ▼          ▼
              frontend    backend     admin
```

------

### 10.12.1 多域名 YAML

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80

    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin
                port:
                  number: 80
```

------

### 10.12.2 多域名 + TLS

可以为多个域名配置证书。

例如：

```
tls:
  - hosts:
      - www.example.com
      - api.example.com
      - admin.example.com
    secretName: example-tls
```

前提是：

```
example-tls
```

中的证书能够覆盖这些域名。

生产环境也可以根据证书管理策略拆分多个 Secret。

------

## 10.13 多服务统一入口

这是 Ingress 最重要的生产价值之一。

假设一个系统：

```
Web
API
Admin
File
```

可以统一使用：

```
example.com
```

通过 Path：

```
example.com/
example.com/api
example.com/admin
example.com/files
```

路由到：

```
frontend
backend
admin
file-service
```

结构：

```
                         Internet
                            │
                            ▼
                    Ingress Controller
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       /                  /api              /admin
          │                 │                 │
          ▼                 ▼                 ▼
      frontend           backend            admin
```

------

### 10.13.1 为什么这种方式非常重要

统一入口可以集中处理：

```
TLS
认证
访问日志
限流
域名
路由
安全策略
```

例如：

```
Internet
    │
    ▼
Ingress
    │
    ├── TLS
    ├── Access Log
    ├── Authentication
    ├── Rate Limit
    └── Routing
            │
            ├── Web
            ├── API
            └── Admin
```

这也是生产环境中 API Gateway / Ingress Gateway 类组件非常重要的原因。

------

## 10.14 Ingress 常见问题

Ingress 出问题时，最忌讳直接认为：

```
Ingress YAML 写错了
```

实际上可能涉及：

```
DNS
LoadBalancer
Ingress Controller
Ingress Rule
Service
EndpointSlice
Pod
TLS
NetworkPolicy
CNI
```

因此必须逐层排查。

------

### 10.14.1 Ingress 对象是否存在

```
kubectl get ingress
```

详细信息：

```
kubectl describe ingress <name>
```

重点检查：

```
Rules
Address
TLS
Backend
Events
```

------

### 10.14.2 Ingress Controller 是否运行

例如：

```
kubectl get pods -A | grep -i ingress
```

确认 Controller：

```
Running
Ready
```

进一步：

```
kubectl logs <ingress-controller-pod>
```

查看是否存在：

```
configuration error
upstream error
certificate error
```

------

### 10.14.3 DNS 是否正确

例如：

```
nslookup api.example.com
```

确认解析结果是否指向：

```
Ingress Controller / LoadBalancer
```

而不是：

```
Pod IP
Service ClusterIP
错误 IP
```

生产环境经常出现：

```
Ingress 配置完全正确
但 DNS 指错了
```

最终表现仍然是：

```
访问失败
```

------

### 10.14.4 Ingress 是否找到了 Service

例如：

```
kubectl describe ingress example
```

确认：

```
backend Service
```

存在。

然后：

```
kubectl get svc backend
```

------

### 10.14.5 Service 是否有 Endpoint

```
kubectl get endpointslice
```

或者：

```
kubectl get endpoints
```

如果 Service 没有后端：

```
Ingress
  ↓
Service
  ↓
没有 Endpoint
```

Ingress Controller 即使工作正常，也无法把请求发送给应用。

------

### 10.14.6 Pod 是否 Ready

```
kubectl get pods
```

例如：

```
backend-xxx   0/1   Running
```

这时候重点检查：

```
kubectl describe pod backend-xxx
```

尤其是：

```
Readiness Probe
```

------

### 10.14.7 `port` 是否正确

Ingress：

```
backend:
  service:
    name: backend
    port:
      number: 80
```

必须对应：

```
backend Service port
```

注意这里通常是：

```
Ingress → Service port
```

而不是直接填写：

```
Pod targetPort
```

完整关系：

```
Ingress
   │
   │ Service:80
   ▼
Service
   │
   │ targetPort:8080
   ▼
Pod:8080
```

------

### 10.14.8 TLS 证书问题

常见现象：

```
ERR_CERTIFICATE
```

检查：

```
kubectl get secret
```

查看：

```
kubectl describe ingress <name>
```

确认：

```
TLS
Secret Name
Hosts
```

常见原因：

```
证书过期
域名不匹配
Secret 不存在
Secret 类型错误
证书链不完整
Ingress Controller 没有加载新证书
```

------

### 10.14.9 访问 404

Ingress 404 不一定代表应用不存在。

需要区分：

```
Ingress Controller 404
```

和：

```
Backend Application 404
```

例如：

```
Client
 ↓
Ingress Controller
 ↓
没有匹配 Host/Path
 ↓
404
```

这是路由问题。

另一种：

```
Client
 ↓
Ingress
 ↓
Service
 ↓
Pod
 ↓
应用返回 404
```

这是应用自身的问题。

因此排查时要确认：

> **404 到底是谁返回的。**

------

### 10.14.10 访问超时

如果：

```
DNS 正常
Ingress Controller 正常
```

但：

```
请求一直 timeout
```

继续向后检查：

```
Service
EndpointSlice
Pod
NetworkPolicy
CNI
防火墙
LoadBalancer
```

特别是生产环境中的：

```
NetworkPolicy
```

可能允许：

```
Pod → Pod
```

却不允许：

```
Ingress Controller → Backend Pod
```

最终表现就是：

```
Ingress → Service
       ↓
     Timeout
```

------

## 10.15 生产环境流量入口设计

Ingress 真正进入生产环境后，不能只考虑：

```
Ingress YAML
```

而需要设计完整的入口链路。

------

### 10.15.1 典型生产架构

一个比较典型的 Kubernetes Web 应用入口：

```
                         Internet
                            │
                            ▼
                         DNS
                            │
                            ▼
                  Cloud Load Balancer
                            │
                            ▼
                 Ingress Controller
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
           frontend       backend       admin
           Service        Service       Service
               │            │            │
               ▼            ▼            ▼
              Pods         Pods         Pods
```

每一层职责不同：

```
DNS
 ↓
找到入口

Load Balancer
 ↓
把公网流量送入 Kubernetes

Ingress Controller
 ↓
HTTP/HTTPS 路由

Service
 ↓
找到后端 Pod

Pod
 ↓
运行应用
```

------

### 10.15.2 为什么应用 Service 通常使用 ClusterIP

例如：

```
frontend Service
type: ClusterIP
```

而不是：

```
type: LoadBalancer
```

原因是：

```
Internet
   ↓
LoadBalancer
   ↓
Ingress Controller
   ↓
ClusterIP Service
   ↓
Pod
```

已经有统一入口。

如果每个应用都：

```
LoadBalancer
```

那么就会重新产生：

```
多个公网入口
多个 IP
多个 TLS
多个安全策略
```

失去统一入口的意义。

------

### 10.15.3 入口层应该做什么

生产环境入口层通常承担：

```
DNS
TLS
HTTP → HTTPS
域名路由
Path 路由
访问日志
基础安全策略
限流
认证
请求头处理
```

但是不要把所有业务逻辑都塞进去。

例如：

```
Ingress
```

适合：

```
/api → backend
```

不应该负责：

```
用户订单计算
库存逻辑
支付业务
```

入口层负责：

> **流量治理，不负责业务逻辑。**

------

### 10.15.4 高可用入口

生产环境不能设计成：

```
Internet
   ↓
一个 Ingress Controller Pod
```

如果这个 Pod 挂了：

```
整个网站无法访问
```

更合理的是：

```
             Load Balancer
              /    |    \
             ▼     ▼     ▼
         Ingress Ingress Ingress
         Pod 1   Pod 2   Pod 3
```

同时：

```
Ingress Controller
```

本身需要：

- 多副本
- 合理的资源 requests/limits
- Pod 调度策略
- PodDisruptionBudget（视场景）
- 节点高可用
- 健康检查

这样入口层才不会成为单点故障。

------

### 10.15.5 生产环境不要只看 Ingress

一个真正可靠的入口架构是：

```
                    DNS
                     │
                     ▼
              Load Balancer
                     │
                     ▼
          ┌───────────────────┐
          │ Ingress Controller │
          │      replicas      │
          └─────────┬─────────┘
                    │
             ┌──────┴──────┐
             ▼             ▼
          Service        Service
             │             │
             ▼             ▼
           Pods           Pods
```

还应该考虑：

```
TLS Certificate
      │
      ▼
cert-manager

Logs
      │
      ▼
Logging System

Metrics
      │
      ▼
Monitoring

Tracing
      │
      ▼
Observability

Security
      │
      ▼
WAF / NetworkPolicy / Authentication
```

Ingress 只是入口架构中的一个组成部分。

------

### 10.15.6 一个比较实用的生产设计

对于典型互联网 Web 应用，可以采用：

```
                        Internet
                           │
                           ▼
                    Public DNS
                           │
                           ▼
                  Cloud Load Balancer
                           │
                    TCP 443 / 80
                           │
                           ▼
                Ingress Controller
                  ┌────────┼────────┐
                  │        │        │
                  ▼        ▼        ▼
                Web       API      Admin
                  │        │        │
                  ▼        ▼        ▼
              ClusterIP ClusterIP ClusterIP
                  │        │        │
                  ▼        ▼        ▼
                Pods      Pods      Pods
```

其中：

```
公网
 ↓
只进入入口层
```

应用 Service：

```
ClusterIP
```

尽量不直接暴露公网。

这样可以形成比较清晰的安全边界：

```
Internet
    │
    ▼
[入口层]
    │
    ▼
[Service 层]
    │
    ▼
[应用 Pod]
```

------

## 本章核心知识

本章最重要的是建立完整的 **Kubernetes 外部流量入口模型**。

### 入口层的完整关系

```
                         Internet
                            │
                            ▼
                           DNS
                            │
                            ▼
                    Load Balancer
                            │
                            ▼
                  Ingress Controller
                            │
                            ▼
                         Ingress
                     ┌──────┼──────┐
                     │      │      │
                  Host/Path Host/Path
                     │      │      │
                     ▼      ▼      ▼
                  Service Service Service
                     │      │      │
                     ▼      ▼      ▼
                    Pod    Pod    Pod
```

其中：

```
Ingress
```

负责描述：

> **请求应该根据 Host / Path 去哪个 Service。**

```
Ingress Controller
```

负责：

> **真正接收并处理 HTTP/HTTPS 流量。**

```
Service
```

负责：

> **把流量进一步交给后端 Pod。**

------

### 生产环境最重要的几个概念

**第一：Ingress 不是公网 IP。**

```
Ingress
≠
公网入口
```

通常还需要：

```
DNS
+
LoadBalancer
+
Ingress Controller
```

------

**第二：Ingress 不是 Ingress Controller。**

```
Ingress
   ↓
规则 / API 对象

Ingress Controller
   ↓
真正执行规则的软件
```

------

**第三：Ingress 与 Service 是上下游关系。**

```
Ingress
   ↓
Service
   ↓
Pod
```

而不是：

```
Ingress
   ↓
Pod
```

------

**第四：TLS 通常在入口层终止。**

```
Client
   │ HTTPS
   ▼
Ingress Controller
   │
   │ TLS Termination
   ▼
Service
   ▼
Pod
```

证书通常通过：

```
TLS Secret
```

提供，生产环境则常配合自动化证书管理。

------

**第五：Ingress 最核心的路由方式就是：**

```
Host Routing
```

例如：

```
api.example.com
```

以及：

```
Path Routing
```

例如：

```
example.com/api
```

最终：

```
域名 / Path
      ↓
Ingress
      ↓
Service
      ↓
Pod
```

------

**第六：生产环境不要把每个应用都直接暴露到公网。**

更合理的是：

```
                         Internet
                            │
                            ▼
                         DNS
                            │
                            ▼
                     Load Balancer
                            │
                            ▼
                  Ingress Controller
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Web Service   API Service   Admin Service
             │              │              │
             ▼              ▼              ▼
            Pods           Pods           Pods
```

这就是 Kubernetes 中非常典型的：

> **统一流量入口 + Service + Pod**

架构。

而对于更复杂的现代 Kubernetes 网络入口，则需要逐步理解：

```
Ingress
   ↓
Gateway API
   ↓
Gateway / HTTPRoute
```

核心目标始终没有改变：

> **让外部用户通过稳定、安全、可管理的网络入口访问 Kubernetes 中不断变化的应用实例。**