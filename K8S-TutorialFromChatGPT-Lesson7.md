## 第七阶段 第一章：Service Mesh 入门——为什么 Kubernetes 还需要 Istio？

> **关键词：Service Mesh、Sidecar、Envoy、East-West Traffic、North-South Traffic、Data Plane、Control Plane**

------

提示词：

```
开始下一章
```

正文：

很好。

到这里，我们正式进入 **Kubernetes 高级生态**。

从这一章开始，你学习的内容，不再只是 Kubernetes 本身，而是 **现代云原生架构（Cloud Native Architecture）**。

很多人学习 Kubernetes 后都会有一个疑问：

> Kubernetes 已经有 Service 了，为什么还要 Istio？
>
> Kubernetes 已经有 Ingress 了，为什么还要 Service Mesh？

这是一个非常好的问题。

实际上，**Service、Ingress、Service Mesh 分别解决的是三个不同层次的问题。**

这一章，我们先不讲 Istio 的安装，也不讲各种 CRD，而是先彻底理解：

> **为什么会出现 Service Mesh。**

如果这一章理解了，后面的 Istio、Envoy、Linkerd 等工具都会非常容易理解。

### 本章学习目标

完成本章后，你应该能够回答：

- 什么是 Service Mesh？
- Kubernetes Service 为什么还不够？
- 微服务到底遇到了什么问题？
- Sidecar 为什么要和业务容器放在一起？
- Envoy 为什么会成为 Service Mesh 的核心？
- Istio 到底负责什么？

------

### 第一节：从单体架构说起

先来看最简单的应用。

```
用户
 │
 ▼
Web 应用
 │
 ▼
数据库
```

这里只有一个程序。

所有逻辑：

- 登录
- 下单
- 支付
- 商品查询

全部都在一个进程里。

网络通信很简单。

------

#### 后来变成微服务

随着业务增长。

系统拆分成：

```
订单服务
支付服务
用户服务
库存服务
商品服务
```

于是：

```
订单
 │
 ├────► 用户
 │
 ├────► 库存
 │
 ├────► 商品
 │
 └────► 支付
```

以前：

程序内部：

函数调用。

现在：

全部：

HTTP

gRPC

RPC

网络请求。

------

### 第二节：微服务真正复杂的不是业务，而是通信

很多新人觉得：

微服务难。

是因为：

服务多。

其实：

真正复杂的是：

> **服务之间的通信。**

例如：

订单服务调用支付服务。

需要考虑：

- 地址在哪里？
- 怎么负载均衡？
- 超时多久？
- 如果支付服务挂了怎么办？
- 要不要重试？
- 调用日志怎么记录？
- 链路追踪怎么做？
- 怎么统计成功率？
- 怎么限流？
- 怎么熔断？

这些：

和业务：

完全没有关系。

却必须：

每个服务：

都实现。

------

#### 一个例子

假设：

Order Service：

调用：

Payment Service。

代码：

```
paymentClient.pay();
```

实际上：

背后可能发生：

```
DNS 查询

↓

建立 TCP

↓

TLS 握手

↓

HTTP 请求

↓

重试

↓

超时

↓

记录日志

↓

统计耗时
```

如果：

几十个服务。

每个：

都写：

这些代码。

维护：

几乎：

不可能。

------

### 第三节：最早的解决方案——SDK

以前：

很多公司：

会写：

公共 SDK。

例如：

```
CompanyRPCClient
```

里面：

实现：

- 重试
- 熔断
- 超时
- 日志

业务：

直接：

调用：

```
client.call()
```

看起来：

很好。

但是：

问题来了。

------

#### SDK 的问题

假设：

今天：

新增：

限流。

所有服务：

都要：

升级 SDK。

例如：

```
SDK v1

↓

SDK v2

↓

SDK v3
```

几十个：

服务：

全部：

重新：

发布。

成本：

非常高。

------

### 第四节：Service Mesh 的思想

于是：

社区提出：

一个新理念。

既然：

这些功能：

都和业务：

无关。

为什么：

一定：

放：

业务程序里面？

能不能：

单独：

放：

一个代理？

于是：

变成：

```
订单服务
     │
     ▼
 Envoy Proxy
     │
     ▼
支付服务
```

注意：

业务程序：

不再：

直接：

访问：

支付。

而是：

先经过：

代理。

------

### 第五节：什么是 Sidecar？

这是 Service Mesh 最重要的概念。

假设：

以前：

一个 Pod：

```
Pod

├── order-api
```

现在：

变成：

```
Pod

├── order-api

└── Envoy
```

Envoy：

和：

业务容器：

一起：

运行。

这个模式：

叫：

> **Sidecar（边车模式）**

为什么叫边车？

可以想象：

摩托车：

```
摩托车

+

旁边：

一个边车
```

边车：

不会：

开车。

但是：

负责：

运输。

Envoy：

也是：

这样。

业务：

继续：

写业务。

代理：

负责：

网络。

------

### 第六节：Sidecar 如何工作？

假设：

订单服务：

发送：

HTTP 请求。

以前：

```
Order

↓

Payment
```

现在：

变成：

```
Order

↓

本地 Envoy

↓

远程 Envoy

↓

Payment
```

也就是说：

所有流量：

都会：

经过：

Envoy。

业务：

完全：

不知道。

------

### 第七节：Envoy 是什么？

很多人认为：

Istio：

就是代理。

其实：

不是。

真正：

处理流量的是：

> **Envoy Proxy。**

Envoy：

是一个高性能代理。

它负责：

- HTTP
- gRPC
- TCP
- TLS
- Load Balance
- Retry
- Circuit Breaker
- Metrics

可以理解为：

它是：

Service Mesh 的"数据处理器"。

------

### 第八节：Istio 又是什么？

既然：

Envoy：

已经能代理。

为什么：

还需要：

Istio？

原因很简单。

假设：

1000 个 Pod。

每个：

都有：

一个：

Envoy。

如果：

修改：

超时时间：

```
5 秒

↓

3 秒
```

难道：

登录：

1000 个：

Envoy？

当然：

不可能。

所以：

需要：

统一管理。

这就是：

Istio。

------

#### Istio 的职责

Istio：

负责：

```
配置 Envoy

↓

下发规则

↓

证书管理

↓

安全策略

↓

流量管理
```

真正：

转发：

数据。

仍然：

是：

Envoy。

------

### 第九节：Data Plane 与 Control Plane

这是 Service Mesh 的核心架构。

```
               Istio
         (Control Plane)
                │
        下发配置、策略
                │
 ┌──────────────┼──────────────┐
 ▼              ▼              ▼
Envoy         Envoy         Envoy
(Data Plane) (Data Plane) (Data Plane)
 │              │              │
App A         App B         App C
```

这里有两个重要概念：

##### Data Plane（数据平面）

负责：

真正处理网络流量。

例如：

- 转发请求
- TLS 加密
- 负载均衡
- 重试
- 熔断

典型代表：

**Envoy。**

------

##### Control Plane（控制平面）

负责：

统一管理配置。

例如：

- 下发路由规则
- 更新证书
- 配置流量策略

典型代表：

**Istio Control Plane**（现代版本主要由 `istiod` 负责）。

------

### 第十节：East-West 与 North-South Traffic

这是企业面试经常问的问题。

##### North-South（南北流量）

表示：

**集群外 ↔ 集群内**

例如：

```
浏览器

↓

Ingress

↓

订单服务
```

用户：

进入：

集群。

这叫：

North-South。

------

##### East-West（东西流量）

表示：

**集群内部服务之间**

例如：

```
订单

↓

支付

↓

库存

↓

商品
```

全部：

属于：

East-West。

Service Mesh：

主要：

解决：

East-West。

Ingress：

主要：

解决：

North-South。

------

### 第十一节：Service、Ingress、Service Mesh 的职责

这是最容易混淆的地方。

| 组件         | 主要职责               |
| ------------ | ---------------------- |
| Service      | 服务发现、基础负载均衡 |
| Ingress      | 外部流量进入集群       |
| Service Mesh | 服务之间通信治理       |

它们并不是替代关系。

而是：

不同层次：

共同工作。

------

### 第十二节：Service Mesh 能解决什么问题？

有了 Sidecar 和 Envoy。

很多能力：

变成：

平台提供。

例如：

- 自动重试
- 超时控制
- 熔断
- 限流
- 灰度发布
- 金丝雀发布
- mTLS
- 链路追踪
- 指标采集
- 流量镜像

业务程序：

几乎：

不用：

关心。

------

### 第十三节：本章知识关系图

```
          用户请求
              │
              ▼
          Ingress
              │
              ▼
      ┌────────────────┐
      │ Order Pod       │
      │ ┌────────────┐ │
      │ │ Order App  │ │
      │ ├────────────┤ │
      │ │ Envoy      │ │
      │ └────────────┘ │
      └────────────────┘
              │
              ▼
      ┌────────────────┐
      │ Payment Pod     │
      │ ┌────────────┐ │
      │ │ Payment App│ │
      │ ├────────────┤ │
      │ │ Envoy      │ │
      │ └────────────┘ │
      └────────────────┘
              ▲
              │
          Istio（Control Plane）
        下发配置、策略、证书
```

------

### 第十四节：本章总结（建议牢记）

请重点记住以下几点：

1. **微服务真正复杂的是服务间通信，而不是业务代码。**
2. **Service Mesh 将网络治理能力从业务代码中剥离出来。**
3. **Sidecar 是一种架构模式，Envoy 是最常见的 Sidecar Proxy。**
4. **Envoy 负责处理数据流量（Data Plane），Istio 负责统一管理策略（Control Plane）。**
5. **North-South 流量通常由 Ingress 管理，East-West 流量通常由 Service Mesh 管理。**
6. **Service、Ingress、Service Mesh 各自负责不同层次的问题，它们通常协同工作。**

------

### 🌟 企业经验：是不是所有 Kubernetes 集群都需要 Istio？

答案是：

**不是。**

很多团队一上来就安装 Istio，结果发现：

- 集群资源占用增加；
- 每个 Pod 多了一个 Sidecar，CPU 和内存开销上升；
- 网络排障复杂度提高；
- 学习成本增加。

因此，一般可以参考下面的经验：

| 场景                                     | 是否建议使用 Service Mesh |
| ---------------------------------------- | ------------------------- |
| 单体应用、少量服务                       | ❌ 一般不需要              |
| 5~10 个简单微服务                        | ⚠️ 视需求而定              |
| 数十到上百个微服务                       | ✅ 非常适合                |
| 对安全（mTLS）、灰度发布、流量治理要求高 | ✅ 强烈推荐                |

**不要为了使用 Istio 而使用 Istio。** 当服务规模和治理需求达到一定程度时，Service Mesh 才会真正体现价值。

------

#### 下一章预告：Istio 核心流量管理——VirtualService、DestinationRule 与 Gateway

理解了 Service Mesh 为什么存在之后，下一章我们将进入 Istio 的核心能力：

- 为什么一个请求会先经过 Gateway？
- VirtualService 到底控制什么？
- DestinationRule 为什么决定流量去哪里？
- Header、Cookie、URI 如何做路由？
- 金丝雀发布（Canary）和蓝绿发布（Blue-Green）如何配置？
- 熔断、超时、重试是如何在 Istio 中实现的？

这一章结束后，你将真正理解 Istio 最核心的流量治理能力，也是企业最常用的部分。

## 第七阶段 第二章：Istio 核心流量管理——Gateway、VirtualService 与 DestinationRule

> **关键词：Gateway、VirtualService、DestinationRule、Subset、Canary、Blue-Green、Retry、Timeout、Circuit Breaker**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，是 **整个 Istio 学习中最重要的一章**。

如果说上一章回答的是：

> **为什么需要 Service Mesh？**

那么这一章回答的是：

> **Istio 到底是如何控制流量的？**

很多刚接触 Istio 的人都会觉得：

- Gateway
- VirtualService
- DestinationRule
- ServiceEntry
- Sidecar
- EnvoyFilter

这些 CRD（自定义资源）太多了。

其实不用害怕。

**真正每天都会用的只有三个：**

1. **Gateway**（入口）
2. **VirtualService**（路由规则）
3. **DestinationRule**（目标策略）

只要把这三个理解透了，Istio 的核心就掌握了。

### 本章学习目标

完成本章后，你应该能够回答：

- Gateway 和 Kubernetes Ingress 有什么关系？
- VirtualService 到底控制什么？
- DestinationRule 为什么必须和 VirtualService 配合？
- 什么是 Subset？
- 如何实现灰度发布？
- 如何实现蓝绿发布？
- Retry、Timeout、Circuit Breaker 是在哪里配置的？

------

### 第一节：先理解一次完整请求

假设：

浏览器访问：

```
https://api.example.com/orders
```

在 Istio 中，这个请求通常会经过：

```
浏览器
    │
    ▼
Istio Gateway
    │
    ▼
VirtualService
    │
    ▼
DestinationRule
    │
    ▼
Envoy
    │
    ▼
Order Pod
```

很多新人一开始觉得：

为什么要绕这么多层？

其实：

每一层负责的事情都不同。

------

### 第二节：Gateway——进入集群的大门

还记得我们学习 Ingress 时吗？

它负责：

> **让集群外部访问 Kubernetes。**

Istio 也有 Gateway。

但是：

要注意一点。

**Gateway 不是应用程序。**

它只是：

> **告诉 Istio：监听哪些端口、哪些域名。**

例如：

```
servers:
- port:
    number: 80
  hosts:
  - api.example.com
```

它表达的是：

```
监听：

80

允许：

api.example.com
```

注意：

这里没有写：

```
订单服务
```

Gateway：

**不知道请求最终去哪里。**

------

#### 一个生活中的例子

可以把 Gateway 想成：

> **商场的大门。**

商场的大门只负责：

- 开门
- 验证入口
- 允许顾客进入

它不会决定：

顾客去：

- 超市
- 餐厅
- 电影院

真正决定去哪的是：

后面的导购。

Istio 里：

这个导购就是：

VirtualService。

------

### 第三节：VirtualService——真正的路由规则

VirtualService 是 Istio 中最重要的资源。

它负责回答：

> **这个请求应该去哪里？**

例如：

浏览器访问：

```
/orders
```

VirtualService 可以配置：

```
/orders

↓

Order Service
```

访问：

```
/payment
```

配置：

```
/payment

↓

Payment Service
```

所以：

VirtualService：

就是：

**路由规则。**

------

### 第四节：VirtualService 能匹配什么？

不仅可以匹配：

URI。

还可以匹配：

- Host
- Header
- Cookie
- Query 参数
- HTTP Method
- Source IP（部分场景）

例如：

Header：

```
Version: beta
```

可以：

路由：

到：

Beta 服务。

Cookie：

```
user=test
```

也可以：

进入：

测试环境。

这也是灰度发布的重要基础。

------

### 第五节：DestinationRule——决定"怎么访问"

很多新人会问：

VirtualService 已经决定去 Order Service。

为什么还需要：

DestinationRule？

因为：

VirtualService：

只决定：

> **去哪。**

DestinationRule：

决定：

> **怎么去。**

例如：

Order Service：

有：

三个 Pod。

DestinationRule 可以定义：

- 负载均衡策略
- 熔断
- TLS
- 子集（Subset）
- 连接池

所以：

它更像：

"交通规则"。

------

### 第六节：什么是 Subset？

这是 Istio 最重要的概念之一。

假设：

订单服务：

有两个版本。

```
v1

v2
```

Pod：

Label：

```
version: v1
```

以及：

```
version: v2
```

DestinationRule：

可以定义：

```
subset:

v1

v2
```

以后：

VirtualService：

就可以：

指定：

流量：

去：

哪个版本。

------

#### 一个生活中的例子

假设：

一家酒店。

房间：

```
普通房

豪华房
```

酒店：

就是：

Service。

Subset：

就是：

不同房型。

VirtualService：

负责：

安排：

客人：

入住：

哪一种房型。

------

### 第七节：灰度发布（Canary Release）

这是企业使用 Istio 最多的功能之一。

假设：

目前：

```
100%

↓

v1
```

今天：

上线：

v2。

不敢：

全部：

切过去。

于是：

配置：

```
90%

↓

v1

10%

↓

v2
```

如果：

稳定。

再：

修改：

```
50%

↓

50%
```

最后：

```
0%

↓

100%

↓

v2
```

整个过程：

无需：

修改：

Deployment。

只是：

修改：

VirtualService。

------

### 第八节：蓝绿发布（Blue-Green）

和 Canary 不一样。

蓝绿：

不是：

按比例。

而是：

两套环境：

同时：

存在。

例如：

```
Blue

↓

v1
```

以及：

```
Green

↓

v2
```

平时：

全部：

进入：

Blue。

发布时：

瞬间：

切换：

Green。

好处：

切换：

非常快。

回滚：

也非常快。

缺点：

资源：

翻倍。

------

### 第九节：Retry（自动重试）

以前：

业务代码：

自己：

写：

```
try {
    ...
} catch (...) {
    retry();
}
```

用了：

Istio。

可以：

平台：

统一：

配置。

例如：

```
失败

↓

自动：

重试：

3 次
```

业务：

不用：

关心。

------

### 第十节：Timeout（超时）

例如：

支付服务：

超过：

2 秒。

自动：

失败。

而不是：

一直：

等待。

Timeout：

避免：

请求：

无限阻塞。

通常：

和 Retry：

一起：

配置。

------

### 第十一节：Circuit Breaker（熔断）

假设：

支付服务：

已经：

挂了。

如果：

订单服务：

还：

疯狂：

请求。

结果：

整个系统：

都会：

被拖垮。

于是：

Istio：

可以：

配置：

```
失败率：

超过：

50%

↓

停止：

继续：

请求
```

等待：

恢复。

再：

开放。

这就是：

熔断。

------

### 第十二节：流量治理关系图

```
浏览器
    │
    ▼
Gateway
    │
    ▼
VirtualService
    │
决定去哪里
    ▼
DestinationRule
    │
决定怎么访问
    ▼
Envoy
    │
    ▼
Pod
```

请牢记：

- **Gateway**：入口。
- **VirtualService**：路由。
- **DestinationRule**：访问策略。

------

### 第十三节：企业常见组合

在企业里，一个典型配置通常是：

```
Gateway

↓

VirtualService

↓

DestinationRule

↓

Deployment(v1)

↓

Deployment(v2)
```

VirtualService：

控制：

比例。

DestinationRule：

定义：

Subset。

Deployment：

真正：

运行：

Pod。

------

### 第十四节：Istio 与 Kubernetes Service 的关系

很多新人认为：

用了 Istio。

Service：

可以：

不要。

这是错误的。

实际上：

流量：

仍然：

先到：

Service。

然后：

由 Envoy 和 Istio 的规则进一步控制。

也就是说：

**Istio 建立在 Kubernetes Service 之上，而不是替代 Service。**

------

### 第十五节：本章知识关系图

```
             Browser
                 │
                 ▼
            Istio Gateway
                 │
                 ▼
         VirtualService
        （决定去哪里）
                 │
                 ▼
       DestinationRule
      （决定怎么访问）
                 │
        ┌────────┴────────┐
        ▼                 ▼
   Subset:v1         Subset:v2
        │                 │
        ▼                 ▼
      Pod v1           Pod v2
```

------

### 第十六节：本章总结（建议牢记）

请重点记住以下几点：

1. **Gateway 负责接收进入集群的流量，本身不负责路由到具体服务。**
2. **VirtualService 决定请求应该路由到哪个服务或哪个版本。**
3. **DestinationRule 决定访问策略，包括负载均衡、TLS、连接池、熔断和 Subset。**
4. **Subset 是基于 Pod Label 划分出的逻辑版本集合，是灰度发布的基础。**
5. **Canary（灰度发布）通过调整流量比例逐步切换版本；Blue-Green（蓝绿发布）通过两套完整环境快速切换。**
6. **Retry、Timeout、Circuit Breaker 等流量治理能力可以由 Istio 统一提供，而无需业务代码重复实现。**
7. **Istio 建立在 Kubernetes Service 之上，不会替代 Service。**

------

### 🌟 企业经验：不要把 Istio 当成"银弹"

很多团队刚接触 Istio 时，希望把所有网络问题都交给它解决。

实际上，Istio 的价值在于：

- 统一流量治理；
- 提供可观测性（指标、日志、链路追踪）；
- 支持安全通信（mTLS）；
- 支持高级发布策略（灰度、蓝绿、流量镜像）。

但它并不能替代：

- 应用本身合理的错误处理；
- 数据库设计；
- 缓存策略；
- 消息队列等架构能力。

一个成熟的微服务平台，通常是 **Kubernetes + Istio + Prometheus + Grafana + Jaeger + Argo CD** 等组件共同协作，而不是依赖某一个工具。

------

#### 下一章预告：Service Mesh 安全体系——mTLS、SPIFFE、AuthorizationPolicy 与零信任网络

上一章我们学习了**流量如何走**。

下一章我们将学习：

- 为什么 Service Mesh 可以做到服务间自动加密？
- mTLS 与普通 TLS 有什么区别？
- SPIFFE、SPIRE 是什么？
- Istio 如何给每个 Workload 分配身份？
- PeerAuthentication、RequestAuthentication、AuthorizationPolicy 的作用
- 什么是零信任网络（Zero Trust）？为什么越来越多的企业采用它？

这一章结束后，你将理解 Service Mesh 在安全方面的核心能力，也是 Istio 在大型企业中广泛应用的重要原因。

## 第七阶段 第三章：Service Mesh 安全体系——mTLS、SPIFFE、AuthorizationPolicy 与零信任网络

> **关键词：Zero Trust、mTLS、Identity、SPIFFE、Certificate、PeerAuthentication、AuthorizationPolicy**

------

提示词：

```
开始下一章
```

正文：

很好。

上一章我们学习了：

> **Istio 如何控制流量。**

我们知道：

- Gateway 负责入口；
- VirtualService 负责路由；
- DestinationRule 负责访问策略；
- Envoy 负责真正转发。

但是，在生产环境中，还有一个更重要的问题：

> **服务之间的通信安全吗？**

------

假设你的 Kubernetes 集群里面有：

```
订单服务

↓

支付服务

↓

用户服务
```

如果没有额外保护：

任何一个 Pod：

理论上：

都可能访问：

其他服务。

那么问题来了：

- 订单服务真的应该访问用户数据库吗？
- 一个被攻击的服务，能不能调用支付接口？
- 服务之间的数据是否被窃听？
- 如何确认请求到底来自哪个服务？

传统网络安全：

通常采用：

```
防火墙

↓

边界保护
```

但是 Kubernetes 微服务环境：

大量通信发生在：

**集群内部。**

传统边界安全：

已经不足。

于是产生：

> **Zero Trust（零信任安全）**

而 Service Mesh 正是实现零信任的重要技术之一。

### 本章学习目标

完成本章后，你应该能够理解：

- 为什么 Kubernetes 需要服务身份认证？
- TLS 和 mTLS 有什么区别？
- Istio 如何给服务自动颁发身份？
- SPIFFE 是什么？
- PeerAuthentication 做什么？
- AuthorizationPolicy 如何控制服务访问权限？
- 什么叫零信任架构？

------

### 第一节：传统网络安全的问题

过去：

企业网络：

通常这样设计：

```
互联网

↓

防火墙

↓

内部网络

↓

应用服务器
```

核心思想：

> 外部不可信，内部可信。

也就是：

"进入公司网络的人，默认比较安全。"

------

但是 Kubernetes 改变了这个模型。

现在：

一个集群：

可能：

运行：

几百个服务。

例如：

```
namespace: shop

├── order-service

├── payment-service

├── user-service

├── inventory-service

└── report-service
```

问题：

如果：

payment-service 被攻击。

攻击者进入：

payment Pod。

那么：

他可能：

直接访问：

其他服务。

因为：

传统网络：

认为：

"已经进入内部了。"

------

这就是：

传统安全模型：

最大的缺陷。

------

### 第二节：什么是 Zero Trust（零信任）

零信任的核心思想：

一句话：

> **永远不要默认信任任何请求。**

包括：

- 外部请求；
- 内部服务；
- 管理员；
- 已经进入集群的 Pod。

------

传统模型：

```
内部 = 信任
外部 = 不信任
```

零信任：

```
所有请求

↓

必须验证身份

↓

验证权限

↓

允许访问
```

------

例如：

订单服务调用支付服务：

不是：

```
order-service

↓

payment-service

↓

直接允许
```

而是：

```
order-service

↓

证明身份

↓

检查权限

↓

payment-service

↓

允许
```

------

### 第三节：为什么需要服务身份？

人访问系统：

有：

用户名。

例如：

```
andy

admin

user001
```

那么：

服务呢？

例如：

```
order-service
```

它怎么证明：

"我是订单服务"？

这就是：

Service Identity。

------

Istio 中：

每个 Workload：

都会拥有：

自己的身份。

例如：

```
spiffe://cluster.local/ns/shop/sa/order-service
```

这个：

就是：

服务身份证。

------

### 第四节：SPIFFE 是什么？

SPIFFE：

全称：

> Secure Production Identity Framework For Everyone

中文可以理解：

> 服务身份标准。

它定义：

服务应该如何拥有：

统一身份。

------

例如：

一个服务：

身份：

```
spiffe://cluster.local/ns/shop/sa/order
```

拆开：

```
spiffe://

固定协议


cluster.local

集群


ns/shop

namespace


sa/order

service account
```

------

类似：

人的身份证：

```
国家

城市

身份证号码
```

服务：

也需要：

唯一身份。

------

### 第五节：Istio 如何给服务发身份？

Istio：

通过：

mTLS。

自动：

完成：

身份认证。

流程：

```
Pod 启动

↓

Istio Sidecar 注入

↓

Envoy 获取证书

↓

Envoy 使用证书通信

↓

双方验证身份
```

开发人员：

不需要：

修改代码。

------

### 第六节：TLS 和 mTLS 区别

这是重点。

------

#### 普通 TLS

例如：

浏览器访问：

HTTPS：

```
浏览器

↓

服务器
```

服务器：

证明：

自己是谁。

例如：

网站证书：

```
example.com
```

但是：

浏览器：

通常：

不会：

提供身份证。

所以：

是：

单向认证。

------

#### mTLS

Mutual TLS。

意思：

双向 TLS。

通信：

双方：

都证明身份。

例如：

```
服务 A

↓

证明：

我是 order-service


服务 B

↓

证明：

我是 payment-service
```

双方：

互相验证。

------

简单理解：

TLS：

> 服务器证明自己。

mTLS：

> 双方互相证明。

------

### 第七节：Istio mTLS 工作流程

假设：

Order 调用 Payment。

实际：

不是：

```
Order App

↓

Payment App
```

而是：

```
Order App

↓

Order Envoy

↓

Payment Envoy

↓

Payment App
```

认证发生：

在：

Envoy 之间。

流程：

```
Order Envoy

发送请求

+

客户端证书


        ↓


Payment Envoy

验证证书


        ↓


验证通过


        ↓


转发给 Payment App
```

------

### 第八节：PeerAuthentication ——控制 mTLS 模式

Istio 使用：

PeerAuthentication：

控制：

服务之间：

如何认证。

------

例如：

关闭：

```
mtls:
  mode: DISABLE
```

表示：

不使用 mTLS。

------

允许：

```
mtls:
  mode: PERMISSIVE
```

表示：

兼容模式。

允许：

- mTLS
- 非 mTLS

------

强制：

```
mtls:
  mode: STRICT
```

表示：

必须：

mTLS。

------

生产环境：

通常：

推荐：

```
STRICT
```

------

### 第九节：为什么不是一开始全部 STRICT？

因为：

迁移过程：

比较复杂。

例如：

旧服务：

没有：

Sidecar。

如果：

突然：

开启：

STRICT。

结果：

旧服务：

无法通信。

所以：

企业通常：

分阶段：

迁移。

例如：

阶段1：

```
PERMISSIVE
```

观察：

流量。

------

阶段2：

所有服务：

注入 Sidecar。

------

阶段3：

改：

```
STRICT
```

------

### 第十节：AuthorizationPolicy ——访问控制

mTLS：

解决：

一个问题：

> 你是谁？

但是：

还有：

第二个问题：

> 你有没有权限？

例如：

订单服务：

可以：

访问：

支付服务。

但是：

库存服务：

不能：

访问：

支付服务。

怎么办？

AuthorizationPolicy。

------

例如：

规则：

```
允许：

order-service

访问：

payment-service
```

其它：

拒绝。

------

### 第十一节：身份 + 权限完整流程

一个请求：

需要：

经过：

两个检查。

------

第一步：

身份认证。

```
你是谁？
```

mTLS：

解决。

------

第二步：

权限检查。

```
你能访问什么？
```

AuthorizationPolicy：

解决。

------

完整：

```
请求

↓

mTLS 身份认证

↓

AuthorizationPolicy 权限判断

↓

允许 / 拒绝
```

------

### 第十二节：RequestAuthentication ——用户身份认证

前面：

讲的是：

服务身份。

但是：

还有一种：

用户身份。

例如：

用户访问：

API。

携带：

JWT：

```
Authorization:

Bearer xxxx
```

Istio：

可以：

验证 JWT。

这就是：

RequestAuthentication。

------

区别：

| 类型                  | 认证对象   |
| --------------------- | ---------- |
| mTLS                  | 服务到服务 |
| RequestAuthentication | 用户请求   |

------

### 第十三节：企业零信任架构

一个成熟 Kubernetes 平台：

通常：

这样设计：

```
             用户
              │
              ▼
          API Gateway
              │
              ▼
          服务 Mesh
              │
     ┌────────┼────────┐
     ▼        ▼        ▼
  Order    Payment   User

     │        │        │

   Envoy    Envoy    Envoy

     │        │        │

   mTLS + AuthorizationPolicy
```

所有：

服务通信：

都有：

身份。

所有：

访问：

都有：

权限。

------

### 第十四节：Service Mesh 安全能力总结

| 能力     | 组件                  |
| -------- | --------------------- |
| 服务身份 | SPIFFE                |
| 自动证书 | Istio CA              |
| 加密通信 | mTLS                  |
| 双向认证 | PeerAuthentication    |
| 用户 JWT | RequestAuthentication |
| 服务权限 | AuthorizationPolicy   |

------

### 第十五节：本章知识关系图

```
             Service Request

                    │

                    ▼

              Envoy Sidecar

                    │

        ┌───────────┴───────────┐

        ▼                       ▼

    mTLS认证              Authorization

        │                       │

   你是谁？                能访问吗？

        │                       │

        └───────────┬───────────┘

                    ▼

              允许访问服务
```

------

### 第十六节：本章总结（建议牢记）

请重点记住：

1. **零信任的核心思想是不默认信任任何请求。**
2. **Service Mesh 通过 Sidecar 实现服务间安全通信。**
3. **mTLS 是双向身份认证，服务双方都需要证明自己的身份。**
4. **SPIFFE 为服务提供标准化身份标识。**
5. **PeerAuthentication 控制服务间 mTLS 模式。**
6. **AuthorizationPolicy 控制服务之间的访问权限。**
7. **认证解决"你是谁"，授权解决"你能做什么"。**

------

### 🌟 企业经验：为什么大型企业越来越重视 mTLS？

随着企业微服务数量增加：

传统方式：

- 防火墙；
- IP 白名单；
- 网络隔离；

越来越难维护。

因为：

Pod：

会不断变化：

- IP 改变；
- 扩缩容；
- 服务迁移；
- 多集群部署。

而 mTLS 使用：

**服务身份**

而不是：

**IP 地址**

作为安全依据。

这使得安全策略可以随着 Kubernetes 动态环境自动适配。

这也是 Service Mesh 在金融、电商、大型互联网企业中越来越常见的重要原因。
