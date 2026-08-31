# 第 27 章：Kubernetes 集群部署

前面的章节主要学习的是：

```
Kubernetes 如何运行应用
        ↓
Pod / Deployment / StatefulSet
        ↓
Service / Ingress
        ↓
ConfigMap / Secret / Storage
        ↓
Resources / Scheduling / HPA / HA
        ↓
Logging / Monitoring / Troubleshooting
        ↓
RBAC / Security
        ↓
Helm
```

现在开始进入**生产环境 Kubernetes**。

这一章的重点不是简单地执行几条 `kubeadm` 命令，而是理解：

> **一个真正可用的 Kubernetes Cluster 到底由哪些部分组成，以及生产环境应该如何规划和部署。**

------

## 27.1 本地 K8S

### 27.1.1 什么是本地 Kubernetes

本地 Kubernetes 指在自己的电脑或开发服务器上运行 Kubernetes Cluster。

常见方案：

```
Minikube
Kind
K3s
Docker Desktop Kubernetes
```

它们主要用于：

```
学习
开发
测试
CI
验证 YAML
验证 Helm Chart
```

而不是直接作为生产集群。

------

### 27.1.2 为什么需要本地 Kubernetes

生产 Kubernetes 集群通常涉及：

```
多个 Node
网络
存储
Load Balancer
DNS
TLS
监控
日志
权限
```

如果每次学习都使用云 Kubernetes：

```
成本高
创建慢
实验不方便
容易误操作生产资源
```

本地 Kubernetes 可以让我们快速验证：

```
kubectl apply
helm install
Service
Ingress
PVC
HPA
RBAC
NetworkPolicy
```

------

### 27.1.3 本地 K8S 与生产 K8S 的区别

本地：

```
开发者电脑
    ↓
单机 / 少量 Node
    ↓
学习和测试
```

生产：

```
多个物理机 / VM
       ↓
Control Plane
       ↓
Worker Nodes
       ↓
网络 / 存储 / LB
       ↓
监控 / 日志 / 安全
```

因此：

> **本地 Kubernetes 能帮助你学习 Kubernetes，但不能简单认为“本地能运行 = 生产架构已经准备好了”。**

------

## 27.2 Minikube

### 27.2.1 Minikube 是什么

Minikube 是一个非常适合 Kubernetes 学习的本地集群工具。

它可以在：

```
Linux
macOS
Windows
```

上创建本地 Kubernetes Cluster。

官方文档：

[Minikube 官方文档](https://minikube.sigs.k8s.io/docs/?utm_source=chatgpt.com)

------

### 27.2.2 为什么使用 Minikube

Minikube 非常适合：

```
Kubernetes 初学
kubectl 实验
Service
Ingress
Storage
Deployment
Helm
```

例如：

```
minikube start
```

查看：

```
kubectl get nodes
```

通常可以看到：

```
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane   ...
```

------

### 27.2.3 部署测试应用

```
kubectl create deployment nginx \
  --image=nginx
```

查看：

```
kubectl get pods
```

创建 Service：

```
kubectl expose deployment nginx \
  --port=80 \
  --type=NodePort
```

查看：

```
minikube service nginx
```

这就可以在本地快速验证：

```
Deployment
   ↓
Pod
   ↓
Service
   ↓
本地访问
```

------

### 27.2.4 Minikube 的局限

Minikube 非常适合学习，但不要把它当作生产集群。

主要原因：

```
单机资源有限
网络模型与生产可能不同
存储通常是本地模拟
HA 能力有限
Load Balancer 行为与云环境不同
```

所以：

> **Minikube 的目标是学习和开发，而不是生产。**

------

## 27.3 Kind

### 27.3.1 Kind 是什么

Kind：

> **Kubernetes IN Docker**

它通过 Docker Container 运行 Kubernetes Node。

官方项目：

[Kind 官方项目](https://kind.sigs.k8s.io/?utm_source=chatgpt.com)

例如：

```
Docker
 │
 ├── kind-control-plane
 ├── kind-worker
 └── kind-worker
```

从 Kubernetes 的角度看：

```
Container
   ↓
Node
```

------

### 27.3.2 为什么使用 Kind

Kind 非常适合：

```
CI/CD
自动化测试
Kubernetes API 测试
多 Node 实验
开发 Kubernetes Controller
测试 Helm Chart
```

例如创建 Cluster：

```
kind create cluster
```

查看：

```
kubectl get nodes
```

------

### 27.3.3 创建多 Node Cluster

例如：

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

保存为：

```
kind-config.yaml
```

执行：

```
kind create cluster \
  --config kind-config.yaml
```

查看：

```
kubectl get nodes
```

可能看到：

```
kind-control-plane
kind-worker
kind-worker2
```

这对于学习：

```
Scheduler
Node Affinity
Pod Anti-Affinity
Topology
DaemonSet
```

非常方便。

------

## 27.4 K3s

### 27.4.1 K3s 是什么

K3s 是一个轻量级 Kubernetes 发行版。

官方项目：

[K3s 官方项目](https://k3s.io/?utm_source=chatgpt.com)

它针对：

```
边缘计算
资源受限环境
IoT
实验环境
小型生产环境
```

进行了大量简化。

------

### 27.4.2 K3s 为什么轻量

标准 Kubernetes 包含很多组件。

K3s 将很多组件进行了：

```
集成
裁剪
简化
```

因此部署和运行所需资源相对较少。

------

### 27.4.3 K3s 与 Minikube / Kind 的区别

可以简单理解：

| 工具     | 主要用途                              |
| -------- | ------------------------------------- |
| Minikube | 学习 Kubernetes                       |
| Kind     | Docker 中创建 Kubernetes，适合测试/CI |
| K3s      | 轻量 Kubernetes                       |
| kubeadm  | 正式构建 Kubernetes 集群              |

注意：

> K3s 并不是“只能学习用”。

它可以用于实际生产场景，但具体是否适合生产，要根据规模、可用性、生态和运维能力评估。

------

## 27.5 kubeadm

### 27.5.1 kubeadm 是什么

`kubeadm` 是 Kubernetes 官方提供的集群引导工具。

它主要解决：

> **如何把多台 Linux 主机初始化成 Kubernetes Cluster。**

官方文档：

[kubeadm 官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/?utm_source=chatgpt.com)

------

### 27.5.2 kubeadm 不是什么

非常容易误解。

`kubeadm`：

```
不是 Kubernetes 本身
不是完整的集群管理平台
不是 CNI
不是 CSI
不是 Ingress Controller
```

它主要负责：

```
初始化 Control Plane
加入 Worker Node
生成必要的集群配置
```

------

### 27.5.3 kubeadm 集群结构

例如：

```
                 Load Balancer
                      │
          ┌───────────┴───────────┐
          │                       │
   Control Plane 1        Control Plane 2
          │                       │
          └───────────┬───────────┘
                      │
                Worker Nodes
              ┌───────┼───────┐
              │       │       │
            Worker  Worker  Worker
```

生产环境通常还需要：

```
CNI
CSI
Ingress
Metrics
DNS
Monitoring
Logging
```

所以：

> **`kubeadm init` 成功，并不代表生产集群已经完成。**

------

### 27.5.4 kubeadm 初始化

典型流程：

```
kubeadm init
```

初始化完成后，通常需要配置：

```
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf \
  $HOME/.kube/config

sudo chown "$(id -u)":"$(id -g)" \
  $HOME/.kube/config
```

然后：

```
kubectl get nodes
```

此时 Node 可能还是：

```
NotReady
```

这是正常现象之一，因为通常还没有安装 CNI。

------

### 27.5.5 Worker 加入集群

Control Plane 初始化后会得到类似：

```
kubeadm join <CONTROL_PLANE>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

然后在 Worker Node 执行。

查看：

```
kubectl get nodes
```

最终：

```
NAME       STATUS   ROLES           AGE
master-01  Ready    control-plane   ...
worker-01  Ready    <none>          ...
worker-02  Ready    <none>          ...
```

实际生产部署时，版本、网络、container runtime、内核参数以及 kubeadm 配置必须与目标 Kubernetes 版本匹配，不能简单复制一套旧教程命令。

------

## 27.6 云 Kubernetes

### 27.6.1 什么是云 Kubernetes

云厂商提供的 Kubernetes 服务通常称为：

> **Managed Kubernetes**

例如：

```
AWS  → EKS
Microsoft Azure → AKS
Google Cloud → GKE
```

它们会替你管理 Kubernetes 的部分基础设施。

------

### 27.6.2 为什么生产环境经常使用云 Kubernetes

自己维护：

```
Control Plane
etcd
API Server
Scheduler
Controller Manager
升级
证书
高可用
```

运维成本很高。

Managed Kubernetes 可以将大量 Control Plane 运维工作交给云厂商。

于是团队主要关注：

```
Worker Nodes
Application
Network
Storage
Security
Monitoring
CI/CD
```

------

### 27.6.3 Managed Kubernetes 不是“完全不用运维”

这是生产环境非常重要的认识。

云厂商管理 Control Plane：

```
Cloud Provider
      │
      ▼
Control Plane
```

但是：

```
你的应用
你的容器
你的 Service
你的 Ingress
你的 RBAC
你的 NetworkPolicy
你的数据
你的 Worker Node
```

仍然需要管理。

因此：

> **Managed Kubernetes 降低了 Kubernetes Control Plane 运维成本，但不会消灭 Kubernetes 运维。**

------

## 27.7 EKS

### 27.7.1 EKS 是什么

EKS：

> **Amazon Elastic Kubernetes Service**

是 AWS 提供的 Managed Kubernetes 服务。

Amazon Web Services 提供 Kubernetes Control Plane 的托管能力。

官方文档：

[Amazon EKS 官方文档](https://docs.aws.amazon.com/eks/?utm_source=chatgpt.com)

------

### 27.7.2 EKS 的典型结构

```
AWS
│
├── EKS Control Plane
│
├── Worker Nodes
│     ├── EC2
│     └── 其他计算方案
│
├── VPC
│
├── Load Balancer
│
├── EBS
│
└── EFS
```

因此 EKS 不是一个孤立的 Kubernetes。

它通常和 AWS：

```
VPC
IAM
EC2
ELB
EBS
EFS
Route 53
CloudWatch
```

等服务结合。

------

## 27.8 AKS

### 27.8.1 AKS 是什么

AKS：

> **Azure Kubernetes Service**

是 Microsoft Azure 提供的 Managed Kubernetes。

Microsoft Azure 提供托管 Kubernetes Control Plane。

官方文档：

[Azure Kubernetes Service 官方文档](https://learn.microsoft.com/azure/aks/?utm_source=chatgpt.com)

------

### 27.8.2 AKS 典型生态

```
Azure
 │
 ├── AKS
 ├── Virtual Network
 ├── Load Balancer
 ├── Managed Identity
 ├── Azure Disk
 ├── Azure Files
 └── Azure Monitor
```

如果企业已经大量使用 Azure：

```
Azure AD / Entra ID
Azure Networking
Azure Storage
Azure Monitor
```

那么 AKS 通常比较自然。

------

## 27.9 GKE

### 27.9.1 GKE 是什么

GKE：

> **Google Kubernetes Engine**

是 Google Cloud 提供的 Kubernetes 服务。

Google Cloud 提供 GKE。

官方文档：

[Google Kubernetes Engine 官方文档](https://cloud.google.com/kubernetes-engine/docs?utm_source=chatgpt.com)

------

### 27.9.2 GKE 的特点

GKE 与 Google Cloud 的基础设施结合紧密：

```
GKE
 │
 ├── VPC
 ├── Load Balancing
 ├── Persistent Disk
 ├── Cloud Monitoring
 ├── Cloud Logging
 └── IAM
```

Google 本身长期深度参与 Kubernetes 生态，因此 GKE 在 Kubernetes 能力和 Google Cloud 服务之间具有很强的整合能力。

------

## 27.10 节点规划

生产环境部署 Kubernetes，不能简单：

```
找几台服务器
↓
kubeadm init
↓
结束
```

首先必须规划 Node。

------

### 27.10.1 Node 的基本分类

最基础：

```
Control Plane
Worker Node
```

Control Plane 负责：

```
API Server
Scheduler
Controller Manager
etcd
```

Worker Node 负责：

```
Application Pods
```

------

### 27.10.2 典型生产规划

例如：

```
Control Plane:
cp-01
cp-02
cp-03

Worker:
worker-01
worker-02
worker-03
worker-04
worker-05
worker-06
```

------

### 27.10.3 为什么 Control Plane 通常使用奇数

尤其是 `etcd`。

例如：

```
1
3
5
```

更常见。

因为 etcd 使用 quorum。

3 个节点：

```
需要至少 2 个正常
```

5 个节点：

```
需要至少 3 个正常
```

因此：

```
3 Control Plane
```

通常比：

```
2 Control Plane
```

更适合作为高可用设计。

------

### 27.10.4 Worker Node 规划考虑什么

主要考虑：

```
CPU
Memory
Disk
Network
Pod 数量
应用类型
可用区
故障域
```

例如 API 服务：

```
4 CPU
16 GB RAM
```

数据库：

```
更高内存
高速磁盘
低延迟存储
```

日志系统：

```
大量磁盘
较高 IO
```

因此不能简单让所有 Node 都使用同样规格。

------

## 27.11 Control Plane 高可用

### 27.11.1 什么是 Control Plane HA

如果只有：

```
Control Plane 1
```

那么它挂掉：

```
kubectl
   ↓
API Server
   ↓
不可用
```

虽然已经运行的 Pod 不一定立即全部停止，但集群控制能力会受到严重影响。

生产环境通常需要：

```
Control Plane 1
Control Plane 2
Control Plane 3
```

------

### 27.11.2 etcd HA

etcd 是 Kubernetes 非常关键的数据存储。

保存：

```
Cluster State
Objects
Configuration
```

例如：

```
Pod
Deployment
Service
Secret
ConfigMap
```

等 Kubernetes 对象的状态。

因此：

> **etcd 是 Control Plane 高可用的核心组成部分。**

------

### 27.11.3 API Server 前面需要统一入口

多个 API Server：

```
API Server 1
API Server 2
API Server 3
```

通常前面需要：

```
Load Balancer
```

形成：

```
kubectl
   │
   ▼
Load Balancer
   │
 ┌─┼─────────┐
 ▼ ▼         ▼
API1 API2    API3
```

这样单个 API Server 故障不会导致整个 API Endpoint 消失。

------

### 27.11.4 HA 的关键

真正的 Control Plane HA 至少需要考虑：

```
多个 Control Plane
多个 etcd
API Server Load Balancing
网络冗余
故障域
数据备份
证书
```

------

## 27.12 Worker Node

### 27.12.1 Worker Node 是什么

Worker Node 是：

> **真正运行业务 Pod 的节点。**

例如：

```
Worker Node
 │
 ├── kubelet
 ├── container runtime
 ├── kube-proxy / 网络组件
 └── Pods
```

------

### 27.12.2 kubelet

`kubelet` 是 Node 上非常重要的组件。

它负责：

```
监听 Pod Spec
创建 / 管理 Container
健康检查
汇报 Node / Pod 状态
```

简单理解：

```
API Server
    ↓
Pod Spec
    ↓
kubelet
    ↓
Container Runtime
    ↓
Container
```

------

### 27.12.3 Container Runtime

Worker Node 需要 Container Runtime。

现代 Kubernetes 常见：

```
containerd
CRI-O
```

例如：

```
kubelet
   ↓
CRI
   ↓
containerd
   ↓
Container
```

------

### 27.12.4 Worker Node 故障

如果：

```
worker-02
```

突然宕机：

```
worker-02
   X
```

Kubernetes Controller 会发现 Node 状态异常。

如果应用：

```
spec:
  replicas: 3
```

并且其他 Node 有足够资源，Pod 可以重新调度到其他 Node。

这也是前面学习：

```
Deployment
Replica
Scheduler
Pod Anti-Affinity
```

的生产意义。

------

## 27.13 CNI

### 27.13.1 CNI 是什么

CNI：

> **Container Network Interface**

它定义容器网络插件与容器运行环境之间的接口。

Kubernetes 本身不会直接实现完整的 Pod 网络。

因此必须安装 CNI。

------

### 27.13.2 为什么需要 CNI

没有 CNI，可能出现：

```
Node Ready
```

但：

```
Pod 网络不可用
```

例如：

```
Pod A
  X
Pod B
```

Service、DNS、Pod-to-Pod 通信也可能无法正常工作。

------

### 27.13.3 常见 CNI

前面已经学习过：

```
Flannel
Calico
Cilium
```

生产环境需要根据：

```
NetworkPolicy
性能
可观测性
eBPF
路由模式
云环境
运维能力
```

选择。

------

### 27.13.4 kubeadm 与 CNI

使用 kubeadm 创建集群时：

```
kubeadm init
```

并不意味着完整网络已经配置完成。

通常需要：

```
kubeadm
   ↓
Control Plane
   ↓
安装 CNI
   ↓
Pod Network Ready
```

例如安装某个 CNI 后：

```
kubectl get pods -n kube-system
```

检查网络组件是否正常。

------

## 27.14 CSI

### 27.14.1 CSI 是什么

CSI：

> **Container Storage Interface**

用于 Kubernetes 与存储系统之间进行集成。

例如：

```
Kubernetes
    ↓
CSI Driver
    ↓
Cloud Disk / NAS / Ceph
```

------

### 27.14.2 为什么需要 CSI

假设 Pod 需要：

```
PersistentVolume
```

Kubernetes 需要知道：

```
如何创建磁盘
如何挂载磁盘
如何卸载磁盘
如何删除磁盘
```

这些工作通常由 CSI Driver 实现。

------

### 27.14.3 云环境中的 CSI

例如：

```
AWS
 └── EBS CSI Driver

Azure
 └── Azure Disk CSI Driver

Google Cloud
 └── Persistent Disk CSI
```

实际生产环境中：

> **不要只关注 Kubernetes YAML，还必须确认底层 CSI Driver 是否正确安装和配置。**

------

### 27.14.4 CSI 故障

例如：

```
kubectl get pvc
```

看到：

```
Pending
```

就不能只检查 PVC。

需要进一步检查：

```
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Node
 ↓
Storage Backend
```

这就是生产环境真正的存储排障思维。

------

## 27.15 Ingress Controller

### 27.15.1 为什么需要 Ingress Controller

前面学习过：

```
Ingress
```

但要注意：

> **Ingress 本身不会处理网络流量。**

Ingress 是：

```
规则
```

真正执行规则的是：

> **Ingress Controller**

架构：

```
Internet
   │
   ▼
Load Balancer
   │
   ▼
Ingress Controller
   │
   ├── api.example.com
   │       ↓
   │     api-service
   │
   └── web.example.com
           ↓
         web-service
```

------

### 27.15.2 常见 Ingress Controller

例如：

```
NGINX Ingress Controller
Traefik
HAProxy
云厂商 Ingress Controller
```

此外，现代 Kubernetes 生产环境也越来越需要关注：

> **Gateway API**

因为它正在成为更丰富的 Kubernetes 流量管理 API。

------

### 27.15.3 生产环境入口

生产环境通常不是：

```
Internet
   ↓
Pod
```

而是：

```
Internet
   ↓
DNS
   ↓
Cloud Load Balancer / LB
   ↓
Ingress / Gateway
   ↓
Service
   ↓
Pod
```

------

## 27.16 Metrics

### 27.16.1 Metrics 是什么

Kubernetes 需要指标来知道：

```
CPU
Memory
```

例如 HPA：

```
CPU 使用率上升
       ↓
HPA
       ↓
增加 Pod
```

前面第 17 章已经学习过 HPA。

它依赖 Metrics 数据。

------

### 27.16.2 Metrics Server

最基础的 Kubernetes Metrics 方案之一：

```
Metrics Server
```

可以提供：

```
Node CPU
Node Memory
Pod CPU
Pod Memory
```

例如：

```
kubectl top nodes
```

以及：

```
kubectl top pods
```

如果出现：

```
error: Metrics API not available
```

就需要检查 Metrics Server。

------

### 27.16.3 Metrics 与 Prometheus 的区别

不要混淆：

```
Metrics Server
```

主要用于：

```
基础资源指标
HPA
kubectl top
```

而：

```
Prometheus
```

主要用于：

```
监控
历史指标
查询
告警体系
Application Metrics
```

生产环境通常会需要两者承担不同职责。

------

## 27.17 Cluster Addons

### 27.17.1 什么是 Addons

Kubernetes Cluster 创建以后，并不是所有生产能力都自动具备。

还需要根据实际需求安装：

```
CNI
CSI
Ingress
Metrics
DNS
Monitoring
Logging
Autoscaling
Certificate
```

这些都可以理解为 Cluster Addons / 平台组件。

------

### 27.17.2 CoreDNS

Kubernetes 内部 DNS 非常重要。

例如：

```
api-service.shop.svc.cluster.local
```

Pod 可以通过 Service Name 找到 Service。

架构：

```
Application Pod
      ↓
DNS Query
      ↓
CoreDNS
      ↓
Service
      ↓
Pod
```

检查：

```
kubectl get pods -n kube-system
```

通常可以看到 CoreDNS。

------

### 27.17.3 一个生产 Cluster 的典型组成

可以把整套 Kubernetes 理解成：

```
                     Internet
                         │
                         ▼
                  Load Balancer
                         │
                         ▼
                 Ingress Controller
                         │
                         ▼
                     Service
                         │
                         ▼
                       Pods
                         │
          ┌──────────────┼──────────────┐
          │              │              │
       ConfigMap       Secret          PVC
                                         │
                                         ▼
                                    CSI Storage


Control Plane
├── API Server
├── Scheduler
├── Controller Manager
└── etcd

Worker Nodes
├── kubelet
├── Container Runtime
├── CNI
└── Pods

Cluster Addons
├── CoreDNS
├── Metrics Server
├── Monitoring
├── Logging
└── Ingress
```

------

## 生产环境 Kubernetes 集群的整体部署思路

一个比较典型的生产环境，可以按照下面的顺序设计：

```
① 基础设施
   │
   ├── VM / Cloud
   ├── Network
   ├── Subnet
   ├── Security
   └── DNS

② Control Plane
   │
   ├── API Server
   ├── Scheduler
   ├── Controller Manager
   └── etcd

③ Worker Nodes
   │
   ├── kubelet
   └── Container Runtime

④ CNI
   │
   └── Pod Network

⑤ CSI
   │
   └── Persistent Storage

⑥ CoreDNS
   │
   └── Service Discovery

⑦ Ingress Controller
   │
   └── External Traffic

⑧ Metrics
   │
   └── Resource Metrics

⑨ Monitoring / Logging
   │
   ├── Prometheus
   ├── Grafana
   └── Log System

⑩ Application
   │
   ├── Deployment
   ├── Service
   ├── Ingress
   ├── ConfigMap
   ├── Secret
   └── HPA
```

因此，生产 Kubernetes 并不是：

```
kubeadm init
```

这么简单。

更准确地说：

> **Kubernetes Cluster = Control Plane + Worker Nodes + Network + Storage + Traffic Entry + Metrics + Addons + 运维体系。**

------

## 生产环境集群规划示例

假设我们需要部署一个中小型生产 API 平台，可以设计：

```
                    Internet
                        │
                        ▼
                 Cloud Load Balancer
                        │
                        ▼
                Ingress Controller
                        │
                ┌───────┴───────┐
                │               │
             API Service     Web Service
                │               │
          ┌─────┴─────┐   ┌─────┴─────┐
          ▼           ▼   ▼           ▼
        API Pod     API Pod Web Pod   Web Pod
```

Kubernetes Cluster：

```
Control Plane
├── cp-01
├── cp-02
└── cp-03

Worker
├── worker-01
├── worker-02
├── worker-03
├── worker-04
├── worker-05
└── worker-06
```

基础设施：

```
CNI
CSI
CoreDNS
Ingress Controller
Metrics Server
```

平台能力：

```
Prometheus
Grafana
Log System
Alertmanager
```

应用：

```
Deployment
Service
Ingress
ConfigMap
Secret
HPA
PVC
```

这才逐渐形成一个能够承载实际业务的 Kubernetes 平台。

------

## 本章核心知识总结

这一章最重要的不是记住每一个工具，而是建立**集群组成和选型思维**。

### 本地环境

```
Minikube
    ↓
学习 Kubernetes

Kind
    ↓
测试 / CI / 多 Node 实验

K3s
    ↓
轻量 Kubernetes
```

### 自建生产

```
kubeadm
    ↓
构建 Kubernetes Cluster
```

### 云生产

```
EKS
AKS
GKE
    ↓
Managed Kubernetes
```

### 集群核心组成

```
Control Plane
    ↓
管理 Cluster

Worker Node
    ↓
运行应用

CNI
    ↓
网络

CSI
    ↓
存储

Ingress Controller
    ↓
外部流量

Metrics
    ↓
资源指标

Cluster Addons
    ↓
补充平台能力
```

最终需要形成一个非常重要的生产认知：

> **部署 Kubernetes 集群只是生产环境的起点。真正的生产集群必须同时考虑高可用、网络、存储、入口流量、资源指标、安全、监控、日志、备份以及故障恢复。**

# 第 28 章：生产环境集群规划

第 27 章解决的是：

> **Kubernetes 集群怎么部署。**

第 28 章解决的是更重要的问题：

> **生产环境到底应该部署成什么样。**

生产环境 Kubernetes 规划不能从“我要几台服务器”开始，而应该从业务需求倒推：

```
业务需求
   ↓
可用性要求
   ↓
集群规模
   ↓
Node 规格
   ↓
网络 / 存储
   ↓
Ingress / DNS / Certificate
   ↓
镜像 / 日志 / 监控 / 告警
   ↓
权限 / 备份 / 灾备
```

------

## 28.1 集群规模规划

### 28.1.1 什么是集群规模规划

集群规模规划就是确定：

```
Control Plane 数量
Worker Node 数量
Node 规格
Pod 数量
业务数量
资源余量
```

例如一个中小型生产集群：

```
Control Plane
├── cp-01
├── cp-02
└── cp-03

Worker
├── worker-01
├── worker-02
├── worker-03
├── worker-04
├── worker-05
└── worker-06
```

------

### 28.1.2 为什么不能只按照当前业务规划

假设现在业务需要：

```
CPU: 8 Core
Memory: 16 GB
```

不能因此直接部署：

```
2 × 4 Core / 8 GB
```

因为生产环境还需要考虑：

```
业务增长
Node 故障
Pod 重启
滚动更新
DaemonSet
系统组件
资源峰值
维护窗口
```

------

### 28.1.3 生产规划应该保留余量

例如 Worker 总资源：

```
24 CPU
96 GB Memory
```

不要把：

```
24 CPU
96 GB Memory
```

全部分配给业务。

应该预留：

```
Kubernetes 系统组件
CNI
CSI
监控
日志
DaemonSet
故障余量
```

因此：

> **集群总资源 ≠ 应用可使用资源。**

------

### 28.1.4 从 Pod 反推集群规模

例如：

```
API:
6 Pods
每 Pod:
500m CPU
1Gi Memory
```

理论 Requests：

```
CPU:
6 × 0.5 = 3 CPU

Memory:
6 × 1Gi = 6Gi
```

如果还有：

```
Frontend: 2 Pods
Worker:   4 Pods
Monitoring
Logging
Ingress
CNI
CSI
```

就需要继续累加。

最终：

```
业务 Requests
+
系统组件
+
故障余量
+
增长空间
=
Cluster 规划容量
```

------

### 28.1.5 生产环境常见规模

没有所谓“所有生产环境统一规格”。

可以粗略理解：

```
小型生产：
3 Control Plane
3~5 Worker

中型生产：
3 Control Plane
5~20+ Worker

大型生产：
多个 Cluster
每个 Cluster 多个 Worker Pool
```

大型企业通常不会把所有业务都塞进一个无限扩大的 Cluster。

可能采用：

```
Production Cluster A
Production Cluster B
Production Cluster C
```

按照：

```
业务
团队
环境
区域
合规要求
故障域
```

进行拆分。

------

## 28.2 Node 规格

### 28.2.1 什么是 Node 规格

Node 规格主要包括：

```
CPU
Memory
Disk
Network
```

例如：

```
8 CPU
32 GB RAM
200 GB SSD
10 Gbps Network
```

------

### 28.2.2 为什么不能所有 Node 使用同一种规格

不同工作负载特点不同。

例如：

```
API
    ↓
CPU / Memory

数据库
    ↓
Memory / Disk IOPS

日志系统
    ↓
Disk / Network

监控系统
    ↓
Memory / Disk
```

因此生产环境可能采用不同 Node Pool：

```
General Pool
 ├── worker-01
 ├── worker-02
 └── worker-03

Compute Pool
 ├── worker-04
 └── worker-05

Storage / Logging Pool
 ├── worker-06
 └── worker-07
```

再通过：

```
Taints
Tolerations
Node Affinity
Node Selector
```

控制 Pod 放在哪里。

------

### 28.2.3 不建议一开始把 Node 做得过大

例如：

```
1 台
64 CPU / 512 GB
```

看起来很强，但存在一个问题：

> **单个 Node 故障时，损失的容量非常大。**

例如：

```
Node A
64 CPU
```

突然宕机：

```
64 CPU
    ↓
同时消失
```

如果换成：

```
8 × 8 CPU
```

单个 Node 故障影响会小很多。

所以生产规划需要在：

```
Node 数量
Node 大小
成本
故障影响
```

之间平衡。

------

## 28.3 CPU / Memory 规划

### 28.3.1 CPU 规划

Kubernetes 中最重要的是理解：

```
CPU Request
CPU Limit
```

例如：

```
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"
```

表示：

```
Request:
500m = 0.5 CPU

Limit:
1 CPU
```

Scheduler 主要依据 Request 判断：

> **这个 Pod 能不能放到某个 Node。**

------

### 28.3.2 Memory 规划

例如：

```
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

生产环境不能随意写：

```
requests:
  memory: "1Mi"
```

然后认为：

```
“这样就可以部署更多 Pod。”
```

因为这只是降低了调度器看到的资源需求，并没有降低应用实际内存需求。

最终可能：

```
Node Memory 不够
      ↓
OOM
      ↓
Pod 被杀
```

------

### 28.3.3 如何估算应用资源

不要拍脑袋。

建议观察：

```
CPU P50
CPU P95
CPU P99

Memory P50
Memory P95
Memory Peak
```

例如应用长期：

```
CPU:
P95 = 300m

Memory:
P95 = 450Mi
Peak = 600Mi
```

可以从类似：

```
resources:
  requests:
    cpu: "300m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

开始，然后通过生产监控持续调整。

------

### 28.3.4 资源规划原则

生产环境建议：

```
Requests
    ↓
反映正常运行需要的资源

Limits
    ↓
反映允许的最大资源
```

不要把：

```
Requests = Limit = 随便一个数字
```

当成通用答案。

不同应用应该根据真实指标调整。

------

## 28.4 Namespace 规划

### 28.4.1 为什么需要 Namespace

Namespace 可以提供逻辑隔离：

```
production
staging
development
```

例如：

```
production
├── api
├── frontend
└── worker

staging
├── api
├── frontend
└── worker
```

------

### 28.4.2 Namespace 不等于真正的安全隔离

这是生产环境非常重要的一点。

Namespace 可以隔离：

```
资源名称
部分权限
ResourceQuota
NetworkPolicy
```

但是它不是：

```
VM 隔离
物理机隔离
完整安全边界
```

例如错误配置：

```
NetworkPolicy
RBAC
Secret
ServiceAccount
```

仍然可能导致跨 Namespace 的安全问题。

------

### 28.4.3 推荐的 Namespace 思路

例如：

```
production
staging
monitoring
logging
ingress
```

也可以根据团队 / 业务拆分：

```
production-payment
production-user
production-order
```

不要为了“看起来整齐”创建几十上百个 Namespace。

规划原则应该是：

```
权限边界
资源边界
生命周期
团队边界
环境边界
```

------

### 28.4.4 ResourceQuota

生产 Namespace 可以配置资源上限：

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
```

这样可以防止某个团队错误创建大量 Pod，把整个 Cluster 的资源吃光。

------

## 28.5 网络规划

### 28.5.1 为什么生产环境网络规划很重要

Kubernetes 至少存在几个重要网络概念：

```
Node Network
Pod Network
Service Network
External Network
```

例如：

```
Node:
10.0.0.0/16

Pod:
10.244.0.0/16

Service:
10.96.0.0/12
```

实际 CIDR 只是示例。

------

### 28.5.2 CIDR 不能随便选

必须避免：

```
Pod CIDR
Service CIDR
VPC CIDR
公司内网 CIDR
VPN CIDR
```

发生冲突。

例如公司网络：

```
10.0.0.0/8
```

而 Kubernetes Pod Network 又规划成：

```
10.0.0.0/8
```

后续连接：

```
Kubernetes
   ↕
公司 VPN
```

就可能出现路由冲突。

------

### 28.5.3 生产网络需要考虑

至少考虑：

```
VPC / VLAN
Subnet
Pod CIDR
Service CIDR
Node CIDR
Load Balancer
DNS
Firewall
Security Group
NetworkPolicy
```

以及：

```
Pod → Pod
Pod → Service
Pod → Internet
Internet → Ingress
Pod → Database
```

这些流量路径。

------

### 28.5.4 多可用区

如果云环境支持：

```
AZ-A
AZ-B
AZ-C
```

生产环境通常应该尽可能跨 AZ：

```
             Cluster
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
     AZ-A      AZ-B      AZ-C
      │         │         │
   Worker     Worker     Worker
```

这样：

```
AZ-A 故障
```

不会导致整个业务消失。

------

## 28.6 Storage 规划

### 28.6.1 首先区分数据类型

生产环境不要看到 PVC 就全部使用同一种 Storage。

例如：

```
临时缓存
    ↓
emptyDir

数据库
    ↓
高可靠 Persistent Storage

共享文件
    ↓
NAS / NFS / EFS 等

对象数据
    ↓
Object Storage
```

------

### 28.6.2 StorageClass

生产环境一般优先使用：

```
StorageClass
    ↓
Dynamic Provisioning
    ↓
CSI
    ↓
Storage Backend
```

而不是管理员手动创建大量 PV。

例如：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
```

Kubernetes 可以根据 StorageClass 动态创建对应存储。

------

### 28.6.3 Storage 规划重点

必须确认：

```
容量
IOPS
吞吐量
延迟
Access Mode
备份
快照
跨 AZ 能力
数据保留策略
故障恢复
```

尤其数据库：

> **“PVC 存储成功”不等于“数据库已经具备灾备能力”。**

PVC 是存储机制，不是完整 Backup 方案。

------

## 28.7 Ingress 规划

生产环境入口通常：

```
Internet
   ↓
DNS
   ↓
Load Balancer
   ↓
Ingress Controller
   ↓
Service
   ↓
Pod
```

------

### 28.7.1 不建议每个 Service 一个公网 IP

假设：

```
api-service
user-service
order-service
payment-service
```

不建议简单变成：

```
4 个 Service
↓
4 个公网 IP
```

可以使用统一入口：

```
api.example.com
user.example.com
order.example.com
payment.example.com
        ↓
Ingress
        ↓
各 Service
```

这样可以统一：

```
TLS
认证
限流
日志
WAF
访问控制
```

------

### 28.7.2 Ingress Controller 高可用

不要只有：

```
1 个 Ingress Controller Pod
```

生产环境应该至少多个副本：

```
spec:
  replicas: 3
```

并结合：

```
Pod Anti-Affinity
Topology Spread Constraints
```

避免全部运行在同一个 Node。

------

### 28.7.3 Ingress 层需要考虑什么

```
TLS
HTTP → HTTPS
域名
Path
限流
Timeout
Connection
Client IP
X-Forwarded-For
健康检查
WAF
```

------

## 28.8 DNS

### 28.8.1 外部 DNS

例如：

```
api.example.com
```

解析到：

```
Load Balancer
```

流量：

```
User
 ↓
DNS
 ↓
Load Balancer
 ↓
Ingress
```

------

### 28.8.2 Kubernetes 内部 DNS

Cluster 内部：

```
api.production.svc.cluster.local
```

通常由 CoreDNS 提供解析。

应用不要把：

```
10.20.31.15
```

这样的 Pod IP 写死。

应该使用：

```
Service DNS
```

例如：

```
postgres.production.svc.cluster.local
```

------

### 28.8.3 DNS 生产规划

需要考虑：

```
External DNS
Internal DNS
CoreDNS
域名 TTL
故障切换
DNS Provider
```

如果使用云环境，还需要考虑：

```
Private DNS
Public DNS
```

的区别。

------

## 28.9 Certificate

### 28.9.1 为什么需要 Certificate

HTTPS：

```
https://api.example.com
```

需要 TLS Certificate。

否则用户可能看到：

```
Certificate Error
```

------

### 28.9.2 Kubernetes 中 Certificate 的基本关系

通常：

```
Certificate
    ↓
TLS Secret
    ↓
Ingress
    ↓
HTTPS
```

例如：

```
tls:
  - hosts:
      - api.example.com
    secretName: api-tls
```

------

### 28.9.3 生产环境不要手工长期维护证书

如果：

```
证书 90 天
```

每次都人工：

```
生成
上传
替换
```

很容易发生：

```
证书过期
```

生产环境通常需要自动化证书管理，例如：

```
cert-manager
ACME
Let's Encrypt
企业 CA
```

核心目标：

> **Certificate 应该自动签发、自动续期、自动部署。**

------

## 28.10 镜像仓库

### 28.10.1 为什么需要镜像仓库

Kubernetes Node 不应该依赖开发者电脑：

```
Developer PC
   ↓
Docker Image
```

生产环境应该：

```
CI/CD
 ↓
Build Image
 ↓
Registry
 ↓
Kubernetes
 ↓
Pod
```

------

### 28.10.2 生产镜像仓库

可以使用：

```
Docker Hub
Harbor
AWS ECR
Azure Container Registry
Google Artifact Registry
```

具体选择取决于基础设施和安全要求。

------

### 28.10.3 镜像仓库需要考虑

```
私有仓库
访问控制
ImagePullSecrets
漏洞扫描
镜像签名
Retention
版本管理
高可用
备份
```

生产环境尤其不要：

```
image: myapp:latest
```

更推荐明确版本：

```
image: registry.example.com/myapp:1.8.3
```

或者更严格：

```
image: registry.example.com/myapp@sha256:...
```

这样可以保证部署的是确定版本。

------

## 28.11 日志

### 28.11.1 为什么生产环境需要集中日志

如果有：

```
50 Nodes
500 Pods
```

发生问题时不能：

```
ssh worker-01
docker logs ...
```

然后一个个找。

生产环境需要：

```
Application
    ↓
stdout / stderr
    ↓
Log Agent
    ↓
Central Log System
    ↓
Search / Dashboard
```

------

### 28.11.2 常见架构

例如：

```
Pod
 ↓
Container stdout
 ↓
Node
 ↓
Fluent Bit
 ↓
Loki / Elasticsearch
 ↓
Grafana / Kibana
```

------

### 28.11.3 日志规划需要考虑

```
采集
解析
结构化
索引
存储周期
Retention
查询
权限
成本
```

生产环境建议应用输出结构化日志，例如 JSON：

```
{
  "level": "error",
  "service": "order-api",
  "traceId": "abc123",
  "message": "database timeout"
}
```

这样更方便检索和关联。

------

## 28.12 监控

生产环境监控至少应该覆盖：

```
Cluster
Node
Pod
Container
Application
Storage
Network
```

------

### 28.12.1 Cluster 指标

例如：

```
Node Ready
Pod 数量
Pending Pod
API Server
Scheduler
Controller
etcd
```

------

### 28.12.2 Node 指标

```
CPU
Memory
Disk
Disk I/O
Network
Load
Filesystem
```

------

### 28.12.3 Pod 指标

```
CPU
Memory
Restart
Network
状态
```

------

### 28.12.4 Application Metrics

这是生产环境特别重要的一层。

例如 API：

```
Request Rate
Error Rate
Latency
```

也就是常见的：

```
RED
Rate
Errors
Duration
```

例如：

```
HTTP Requests/sec
HTTP 5xx rate
P95 latency
P99 latency
```

只监控：

```
CPU 20%
Memory 40%
```

并不能说明：

> **业务是健康的。**

------

## 28.13 告警

### 28.13.1 监控不等于告警

监控：

```
发现 CPU = 90%
```

告警：

```
CPU 持续 > 90%
5 分钟
→ Alert
```

------

### 28.13.2 不要什么都告警

如果配置：

```
CPU > 70%
→ Alert
```

那么生产环境可能每天收到大量：

```
CPU Warning
CPU Warning
CPU Warning
```

最终：

> **运维人员开始忽略告警。**

这就是 Alert Fatigue。

------

### 28.13.3 好的告警应该关注业务影响

例如：

```
API 5xx > 5%
持续 5 分钟
```

比：

```
CPU > 80%
```

更接近用户真正感受到的问题。

推荐告警层级：

```
用户体验
    ↓
Application
    ↓
Pod
    ↓
Node
    ↓
Cluster
```

------

### 28.13.4 告警需要明确处理动作

一个好的告警应该包含：

```
发生了什么
影响什么
严重程度
什么时候发生
Dashboard
Runbook
处理方法
```

例如：

```
[CRITICAL]
Order API 5xx > 5%
持续 5 分钟

Impact:
订单创建失败

Dashboard:
...

Runbook:
...
```

这样告警才真正有运维价值。

------

## 28.14 权限

生产 Kubernetes 必须遵循：

> **Least Privilege，最小权限原则。**

------

### 28.14.1 用户权限

不要所有人都：

```
cluster-admin
```

例如开发人员：

```
production:
get pods
get logs
describe pods
```

测试环境：

```
create
update
delete
```

管理员：

```
cluster-admin
```

------

### 28.14.2 ServiceAccount 权限

Pod 也可能访问 Kubernetes API。

例如：

```
Pod
 ↓
ServiceAccount
 ↓
Role / ClusterRole
 ↓
Kubernetes API
```

默认情况下：

> **不要给业务 Pod 不必要的 Kubernetes API 权限。**

------

### 28.14.3 权限规划

生产环境至少考虑：

```
Authentication
Authorization
RBAC
ServiceAccount
Namespace
NetworkPolicy
Secret
Audit Log
```

并且定期检查：

```
谁可以访问生产
谁可以修改 Deployment
谁可以读取 Secret
谁可以删除 Pod
谁拥有 cluster-admin
```

------

## 28.15 备份

### 28.15.1 Kubernetes 需要备份什么

首先要区分：

```
Cluster State
```

和：

```
Application Data
```

------

### 28.15.2 etcd 备份

自建 Kubernetes 中：

```
etcd
```

非常关键。

因为其中保存大量 Cluster State。

因此生产环境必须有：

```
etcd Backup
```

并且不能只：

```
备份
```

还必须：

```
Restore Test
```

------

### 28.15.3 应用数据备份

例如 PostgreSQL：

```
PostgreSQL
    ↓
Database Backup
```

例如 PVC：

```
PVC
 ↓
Snapshot / Backup
```

不能因为：

```
PVC 存在
```

就认为：

```
数据已经备份
```

------

### 28.15.4 配置备份

还应该考虑：

```
Helm Values
Kubernetes YAML
Git
Secrets 管理系统
Infrastructure as Code
```

推荐：

```
Git
 ↓
Manifest / Helm
 ↓
Cluster
```

而不是：

```
管理员手工 kubectl edit
```

然后没人知道生产环境到底改过什么。

------

## 28.16 灾备

### 28.16.1 什么是灾备

灾备：

> **Disaster Recovery，DR**

关注的是：

> 如果整个生产环境发生严重故障，如何恢复业务。

这和高可用不是一回事。

------

### 28.16.2 HA 与 DR 的区别

高可用：

```
Node A
  X
  ↓
Pod → Node B
```

目标：

```
业务继续运行
```

灾备：

```
整个 Cluster
      X
      ↓
Backup / Secondary Region
      ↓
恢复 Cluster
      ↓
恢复 Application
      ↓
恢复 Data
```

目标：

```
灾难后恢复业务
```

------

### 28.16.3 DR 需要定义 RPO / RTO

#### RPO

Recovery Point Objective：

> **最多可以丢多少数据。**

例如：

```
RPO = 1 hour
```

意味着发生灾难时，最多接受约 1 小时的数据丢失窗口。

------

#### RTO

Recovery Time Objective：

> **最多允许业务中断多久。**

例如：

```
RTO = 30 minutes
```

意味着：

```
灾难发生
 ↓
30 分钟内恢复服务
```

------

### 28.16.4 灾备方案示例

例如：

```
Primary Region
      │
      ├── Kubernetes
      ├── Database
      └── Object Storage
             │
             │ Backup / Replication
             ▼
Secondary Region
      │
      ├── Kubernetes
      └── Database
```

如果 Primary Region 整体故障：

```
DNS / Traffic Manager
          ↓
Secondary Region
```

------

### 28.16.5 灾备最容易犯的错误

最典型的是：

```
“我们每天都有 Backup。”
```

然后真正故障时发现：

```
Backup 无法恢复
```

所以生产环境必须进行：

> **Restore Drill / Disaster Recovery Drill**

例如定期验证：

```
删除测试环境
    ↓
从 Backup 恢复
    ↓
恢复 Kubernetes
    ↓
恢复数据库
    ↓
恢复应用
    ↓
验证用户访问
```

**没有验证过的 Backup，只能称为“看起来存在的备份”。**

------

## 生产环境集群整体规划示例

把本章全部内容组合起来，可以得到一个比较完整的生产架构：

```
                         Internet
                            │
                            ▼
                       Public DNS
                            │
                            ▼
                    Cloud Load Balancer
                            │
                            ▼
                  Ingress / Gateway
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          API Service   Web Service   Other Service
              │             │
              ▼             ▼
          ┌───────┐      ┌───────┐
          │ Pods  │      │ Pods  │
          └───────┘      └───────┘
              │
       ┌──────┼──────────────┐
       ▼      ▼              ▼
    ConfigMap Secret         PVC
                              │
                              ▼
                         Storage System


             Kubernetes Cluster
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     Control       Worker       Worker
      Plane         Pool         Pool
        │            │            │
        │            └─────┬──────┘
        │                  │
        │             CNI / CSI
        │
      etcd


Platform Layer
├── CoreDNS
├── Metrics Server
├── Prometheus
├── Grafana
├── Alertmanager
├── Fluent Bit
├── Loki / Elasticsearch
└── Certificate Management


Security
├── RBAC
├── ServiceAccount
├── NetworkPolicy
├── Secret Management
└── Image Security


Operations
├── Backup
├── Restore
├── Disaster Recovery
└── Runbook
```

------

## 生产环境规划的核心方法

面对一个新的 Kubernetes 项目，不应该直接问：

> “需要几台服务器？”

应该按照下面的顺序：

```
① 业务规模
   ↓
② SLA / 可用性要求
   ↓
③ RPO / RTO
   ↓
④ Cluster 数量
   ↓
⑤ AZ / Region 规划
   ↓
⑥ Control Plane
   ↓
⑦ Worker Node / Node Pool
   ↓
⑧ CPU / Memory
   ↓
⑨ Network
   ↓
⑩ Storage
   ↓
⑪ Ingress / DNS / Certificate
   ↓
⑫ Registry
   ↓
⑬ Logging
   ↓
⑭ Monitoring
   ↓
⑮ Alerting
   ↓
⑯ RBAC / Security
   ↓
⑰ Backup
   ↓
⑱ Disaster Recovery
```

最终形成：

> **容量规划 + 故障域规划 + 网络规划 + 存储规划 + 平台能力规划 + 安全规划 + 运维规划 + 灾备规划。**

这才是生产环境 Kubernetes 的“集群规划”，而不是单纯决定 Node 数量。