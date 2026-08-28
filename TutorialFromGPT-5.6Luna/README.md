提示词：

```
你是一个专业的K8S运维师。我是一个准备入门K8S进行应用部署的新手，请你把使用K8S需要用到的概念、知识和操作方法给我详细介绍。接下来你的回复我将会进行笔记，并用于我以后K8S的入门到生产环境的使用。首先列出一份课程的目录，然后后续会按照这个目录去逐个开展。注意课程内容避免重复啰嗦，但要保持知识点的完整性。
```

正文：

可以。我们把这套课程定位成一套 **“Kubernetes 从 0 到生产实战”** 的系统课程，而不是只教你 `kubectl apply`。

考虑到你是后端开发出身，我会尽量把 K8S 的知识和你熟悉的 **应用、进程、网络、容器、服务、数据库、配置、发布** 联系起来。后续每一章都按照“**概念 → 原理 → YAML → 命令 → 实战 → 常见问题 → 生产实践**”展开，避免重复。

------

# Kubernetes 从入门到生产环境课程目录

# 第一阶段：K8S 基础认知

### 第 1 章：为什么需要 Kubernetes

1. 传统应用部署方式的问题
2. Docker 解决了什么问题
3. Docker 和 Kubernetes 的关系
4. Kubernetes 解决什么问题
5. Kubernetes 能做什么、不能做什么
6. K8S 的典型应用场景
7. K8S 与虚拟机、Docker Compose 的区别
8. 从开发者视角理解 K8S
9. 一次完整的 K8S 应用部署流程
10. K8S 学习路线和核心知识地图

------

### 第 2 章：Kubernetes 核心架构

1. Kubernetes Cluster
2. Control Plane
3. Worker Node
4. API Server
5. etcd
6. Scheduler
7. Controller Manager
8. kubelet
9. kube-proxy
10. Container Runtime
11. Kubernetes API
12. Controller / Reconciliation Loop
13. Desired State 与 Actual State
14. K8S 对象（Object）的概念
15. 从 `kubectl apply` 到 Pod 启动发生了什么

**重点：建立 K8S 整体脑图。**

------

### 第 3 章：Kubernetes 基础操作

1. `kubectl` 是什么
2. kubeconfig
3. Context
4. Cluster / User / Context
5. `kubectl get`
6. `kubectl describe`
7. `kubectl create`
8. `kubectl apply`
9. `kubectl delete`
10. `kubectl logs`
11. `kubectl exec`
12. `kubectl port-forward`
13. `kubectl explain`
14. Label 和 Selector
15. Namespace
16. 常用 `kubectl` 命令体系

------

# 第二阶段：Pod 与应用运行

### 第 4 章：Pod——Kubernetes 最核心的概念

1. Pod 是什么
2. 为什么不是直接管理 Container
3. Pod 与 Container 的关系
4. Pod 的生命周期
5. Pod IP
6. Pod 网络模型
7. Pod 中多个 Container
8. Init Container
9. Sidecar Container
10. Pod YAML 结构
11. Pod 创建、运行、终止过程
12. Pod 常见状态
13. `Pending`
14. `Running`
15. `Succeeded`
16. `Failed`
17. `CrashLoopBackOff`
18. `ImagePullBackOff`
19. Pod 为什么会重启
20. Pod 为什么不是生产环境直接部署应用的最佳方式

------

### 第 5 章：Deployment——部署无状态应用

1. Deployment 是什么
2. Deployment → ReplicaSet → Pod
3. Replicas
4. Deployment YAML
5. 创建和扩缩容
6. 滚动更新
7. Rollout
8. Rollback
9. Deployment Strategy
10. `RollingUpdate`
11. `Recreate`
12. `maxSurge`
13. `maxUnavailable`
14. Deployment 更新过程中发生了什么
15. 如何查看发布历史
16. 如何回滚
17. 无状态 API 服务的标准部署方式

------

### 第 6 章：StatefulSet——部署有状态应用

1. StatefulSet 是什么
2. StatefulSet 与 Deployment 的区别
3. 稳定的 Pod 名称
4. 稳定的网络身份
5. 稳定的存储
6. Ordered Deployment
7. Ordered Termination
8. Headless Service
9. StatefulSet 常见使用场景
10. Redis / MySQL / PostgreSQL 等应用的部署思路
11. StatefulSet 为什么不能简单理解成“有状态版 Deployment”

------

### 第 7 章：DaemonSet、Job、CronJob

1. DaemonSet
2. 每个 Node 运行一个 Pod
3. 日志采集
4. Node 监控
5. 网络插件
6. Job
7. 一次性任务
8. Job 的完成条件
9. Job 的失败重试
10. CronJob
11. 定时任务
12. Job / CronJob 的生产实践

------

# 第三阶段：Kubernetes 网络

### 第 8 章：Kubernetes 网络模型

1. Linux 网络基础回顾
2. Container Network
3. Pod Network
4. Pod IP
5. Container 到 Container 通信
6. Pod 到 Pod 通信
7. Node 到 Pod 通信
8. Cluster 网络
9. CNI 是什么
10. Flannel
11. Calico
12. Cilium
13. Kubernetes 网络模型背后的原理

------

### 第 9 章：Service——让应用可以被访问

1. 为什么需要 Service
2. Service 与 Pod 的关系
3. Selector
4. Service ClusterIP
5. Service Endpoint / EndpointSlice
6. Service 类型
   - ClusterIP
   - NodePort
   - LoadBalancer
   - ExternalName
7. Service DNS
8. Kubernetes 内部服务发现
9. Service → Pod 流量过程
10. Headless Service
11. Service 常见故障排查

------

### 第 10 章：Ingress——让外部用户访问应用

1. 为什么需要 Ingress
2. Ingress 与 Service 的关系
3. Ingress Controller
4. Nginx Ingress
5. Traefik
6. Gateway API 的基本认知
7. 域名路由
8. Path 路由
9. HTTPS
10. TLS Certificate
11. HTTP → HTTPS
12. 多域名部署
13. 多服务统一入口
14. Ingress 常见问题
15. 生产环境流量入口设计

------

# 第四阶段：配置、存储与安全

### 第 11 章：ConfigMap——管理应用配置

1. ConfigMap 是什么
2. 环境变量
3. 配置文件
4. ConfigMap YAML
5. `env`
6. `envFrom`
7. Volume Mount
8. ConfigMap 更新机制
9. 配置与镜像解耦
10. Spring/.NET/Node.js 等应用配置实践

------

### 第 12 章：Secret——管理敏感信息

1. Secret 是什么
2. Secret 与 ConfigMap 的区别
3. Secret 类型
4. `Opaque`
5. Docker Registry Secret
6. TLS Secret
7. Secret 环境变量
8. Secret Volume
9. Secret 的安全问题
10. Base64 ≠ 加密
11. 生产环境 Secret 管理
12. External Secrets / Vault 的基本认知

------

### 第 13 章：Kubernetes 存储

1. Container 为什么不适合直接保存数据
2. Volume
3. `emptyDir`
4. `hostPath`
5. PersistentVolume
6. PersistentVolumeClaim
7. StorageClass
8. Dynamic Provisioning
9. CSI
10. 云盘 / NAS / Ceph
11. StatefulSet + PVC
12. 数据持久化生命周期
13. 删除 Pod 后数据还在吗？
14. 删除 PVC 后数据还在吗？
15. Kubernetes 存储故障排查

------

# 第五阶段：资源管理与调度

### 第 14 章：资源限制——CPU 和 Memory

1. Kubernetes Resource
2. CPU
3. Memory
4. Requests
5. Limits
6. QoS Class
7. Guaranteed
8. Burstable
9. BestEffort
10. OOMKilled
11. CPU Throttling
12. Pod 为什么调度不上去
13. 如何合理设置 Requests / Limits
14. 生产环境资源管理策略

------

### 第 15 章：Kubernetes 调度机制

1. Scheduler
2. Node Selector
3. Node Affinity
4. Pod Affinity
5. Pod Anti-Affinity
6. Taints
7. Tolerations
8. Topology Spread Constraints
9. 多可用区部署
10. 高可用调度
11. 如何让 Pod 不要集中到同一个 Node
12. 调度失败排查

------

### 第 16 章：Pod 生命周期与健康检查

1. Pod Lifecycle
2. Container Lifecycle
3. Restart Policy
4. Startup Probe
5. Liveness Probe
6. Readiness Probe
7. 三种 Probe 的区别
8. HTTP Probe
9. TCP Probe
10. Exec Probe
11. Graceful Shutdown
12. `preStop`
13. `terminationGracePeriodSeconds`
14. 应用优雅退出
15. K8S 如何避免把流量发送给异常 Pod

------

# 第六阶段：扩缩容与高可用

### 第 17 章：Horizontal Pod Autoscaler

1. HPA 是什么
2. 为什么需要自动扩容
3. CPU HPA
4. Memory HPA
5. Custom Metrics
6. HPA 工作原理
7. Metrics Server
8. HPA YAML
9. 扩容与缩容行为
10. HPA 常见问题

------

### 第 18 章：高可用设计

1. Kubernetes 高可用到底是什么
2. Pod 高可用
3. Node 高可用
4. Service 高可用
5. Deployment 高可用
6. 多副本
7. Pod Anti-Affinity
8. PodDisruptionBudget
9. Rolling Update
10. 多节点
11. 多可用区
12. 故障场景演练
13. 如何设计一个真正高可用的 API 服务

------

# 第七阶段：镜像与应用发布

### 第 19 章：Docker 镜像与 Kubernetes

1. Image 基础
2. Registry
3. Docker Hub
4. Harbor
5. 私有 Registry
6. ImagePullSecrets
7. 镜像 Tag
8. `latest` 的问题
9. Image Digest
10. 镜像版本管理
11. 多阶段构建
12. 镜像大小优化
13. K8S 拉取镜像失败排查

------

### 第 20 章：从代码到 Kubernetes

完整走一遍：

```
源代码
 ↓
编译
 ↓
Docker Image
 ↓
Registry
 ↓
Deployment
 ↓
Pod
 ↓
Service
 ↓
Ingress
 ↓
用户访问
```

重点实践：

- .NET API
- Vue 前端
- Redis
- PostgreSQL/MySQL
- Nginx
- 配置文件
- Secret
- HTTPS

最终形成一个完整可运行的业务系统。

------

# 第八阶段：Kubernetes 运维

### 第 21 章：日志系统

1. Kubernetes 日志模型
2. `kubectl logs`
3. Container Logs
4. Pod Logs
5. Node Logs
6. 日志轮转
7. Fluent Bit
8. Fluentd
9. Loki
10. Elasticsearch
11. Kibana
12. 日志采集架构
13. 生产环境日志方案

------

### 第 22 章：监控与指标

1. Metrics
2. Metrics Server
3. Prometheus
4. Grafana
5. kube-state-metrics
6. Node Exporter
7. Pod 指标
8. Node 指标
9. Application Metrics
10. CPU / Memory / Network / Disk
11. 告警
12. Alertmanager
13. 生产监控体系

------

### 第 23 章：Kubernetes 故障排查

重点建立一套固定排障思维。

1. Pod 起不来怎么办
2. Pod 一直 Pending
3. ImagePullBackOff
4. CrashLoopBackOff
5. OOMKilled
6. Service 无法访问
7. Ingress 无法访问
8. DNS 解析失败
9. Pod 网络异常
10. PVC 挂载失败
11. Node NotReady
12. Deployment 更新失败
13. 应用偶发 502 / 503
14. CPU 飙高
15. Memory 飙高
16. 网络延迟
17. 磁盘满
18. 从 Event → Pod → Node → Application 逐层排查

------

# 第九阶段：Kubernetes 安全

### 第 24 章：Kubernetes 权限模型

1. Authentication
2. Authorization
3. RBAC
4. Role
5. ClusterRole
6. RoleBinding
7. ClusterRoleBinding
8. ServiceAccount
9. 最小权限原则
10. 如何限制用户只能操作某个 Namespace
11. 如何限制 Pod 权限

------

### 第 25 章：Pod 与 Cluster 安全

1. SecurityContext
2. runAsUser
3. runAsNonRoot
4. Linux Capabilities
5. Privileged Container
6. ReadOnlyRootFilesystem
7. Pod Security Standards
8. NetworkPolicy
9. Namespace 隔离
10. Secret 安全
11. Container 安全
12. Supply Chain Security

------

# 第十阶段：Helm 与 Kubernetes 应用管理

### 第 26 章：Helm

1. Helm 是什么
2. 为什么需要 Helm
3. Chart
4. Release
5. Repository
6. Chart 目录结构
7. `values.yaml`
8. Template
9. Helm Functions
10. Helm Variables
11. Helm Install
12. Helm Upgrade
13. Helm Rollback
14. Helm Uninstall
15. 环境配置
16. 开发 / 测试 / 生产 values
17. 自己制作 Helm Chart

------

# 第十一阶段：生产环境 Kubernetes

### 第 27 章：Kubernetes 集群部署

1. 本地 K8S
2. Minikube
3. Kind
4. K3s
5. kubeadm
6. 云 Kubernetes
7. EKS
8. AKS
9. GKE
10. 节点规划
11. Control Plane 高可用
12. Worker Node
13. CNI
14. CSI
15. Ingress Controller
16. Metrics
17. Cluster Addons

------

### 第 28 章：生产环境集群规划

1. 集群规模规划
2. Node 规格
3. CPU / Memory 规划
4. Namespace 规划
5. 网络规划
6. Storage 规划
7. Ingress 规划
8. DNS
9. Certificate
10. 镜像仓库
11. 日志
12. 监控
13. 告警
14. 权限
15. 备份
16. 灾备

------

# 第十二阶段：CI/CD 与 Kubernetes

### 第 29 章：CI/CD

建立完整发布链：

```
Git
 ↓
CI
 ↓
Build
 ↓
Test
 ↓
Docker Build
 ↓
Push Registry
 ↓
Deploy Kubernetes
 ↓
Health Check
 ↓
Release
```

学习：

1. CI/CD 基础
2. GitHub Actions
3. GitLab CI
4. Jenkins
5. Docker Build
6. Registry
7. Kubernetes Deployment
8. 自动更新 Image
9. 环境管理
10. 发布失败处理

------

### 第 30 章：GitOps 与 Argo CD

1. GitOps 是什么
2. 为什么需要 GitOps
3. Desired State
4. Argo CD
5. Git Repository
6. Application
7. Sync
8. Auto Sync
9. Rollback
10. 多环境管理
11. Helm + Argo CD
12. Kustomize + Argo CD

------

# 第十三阶段：高级 Kubernetes

### 第 31 章：Kustomize

1. Kustomize 是什么
2. Base
3. Overlay
4. Patch
5. ConfigMap Generator
6. Secret Generator
7. 多环境部署
8. Helm vs Kustomize
9. Helm + Kustomize

------

### 第 32 章：CRD 与 Operator

1. Kubernetes API Extension
2. CRD
3. Custom Resource
4. Controller
5. Operator
6. Operator 工作原理
7. 为什么 Prometheus / Database Operator 都大量使用 CRD
8. 如何理解 Operator 模式

------

### 第 33 章：Service Mesh

1. Service Mesh 是什么
2. 为什么需要 Service Mesh
3. Sidecar
4. Traffic Management
5. mTLS
6. Observability
7. Istio
8. Linkerd
9. Service Mesh 的代价
10. 什么情况下不要使用 Service Mesh

------

# 第十四阶段：生产实战项目

### 第 34 章：完整项目——部署一个真实业务系统

最终做一个接近生产环境的项目：

```
                    Internet
                       │
                       ▼
                  LoadBalancer
                       │
                       ▼
                    Ingress
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
     Vue Frontend              .NET API
                                    │
                     ┌──────────────┼──────────────┐
                     ▼              ▼              ▼
                   Redis        PostgreSQL      Other API
```

同时加入：

- Deployment
- Service
- Ingress
- ConfigMap
- Secret
- PVC
- StatefulSet
- HPA
- Probe
- Resource Limits
- RBAC
- NetworkPolicy
- Helm
- Prometheus
- Grafana
- Loki
- CI/CD
- Argo CD

------

# 第十五阶段：生产事故演练

### 第 35 章：Kubernetes 实战故障实验室

故意制造问题，然后按照标准运维流程解决：

1. Pod 无法启动
2. 镜像无法拉取
3. Pod 无限重启
4. 内存 OOM
5. CPU 100%
6. Service 无法访问
7. DNS 故障
8. Ingress 502
9. PVC 无法挂载
10. Node NotReady
11. Deployment 滚动发布失败
12. HPA 不扩容
13. Secret 配置错误
14. Readiness Probe 配置错误
15. 网络策略导致服务无法通信
16. 节点宕机
17. Pod 大面积迁移
18. 应用发布导致线上故障

最终形成一套：

> **现象 → 判断 → 定位 → 处理 → 验证 → 复盘**

的 Kubernetes 排障方法论。

------

# 最终知识体系

学完以后，你应该能够把 Kubernetes 理解成下面这张地图：

```
                         Kubernetes
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
       Cluster              Workload            Network
          │                   │                   │
   ┌──────┼──────┐      ┌─────┼────────┐    ┌─────┼─────┐
   │      │      │      │     │        │    │     │     │
Control Worker  Node  Pod Deployment Stateful Service Ingress
Plane    Node         │                 │
                      │                 └── Storage
                      │
               Config / Secret
                      │
                 Resource
                      │
                  Scheduler
                      │
                  Autoscaling
                      │
                  Monitoring
                      │
                    Logs
                      │
                    RBAC
                      │
                  Security
                      │
                    Helm
                      │
                    CI/CD
                      │
                  GitOps
                      │
                Production
```

## 我建议我们的学习顺序

虽然上面按照完整知识体系分成了 **35 章**，但实际学习时我会控制每章的深度和顺序，不会一开始就把大量高级概念塞给你。

我们会遵循：

**认识 → 能跑 → 能部署 → 能访问 → 能配置 → 能存储 → 能扩缩 → 能排障 → 能监控 → 能发布 → 能保证安全 → 能生产运维**

也就是说，前面学到的每个知识点，都会逐渐汇总到最终的生产项目中，而不是学完一个概念就丢掉。

### 第一阶段真正需要掌握的核心链路

你刚入门时，最重要的不是背几十个 Kubernetes Object，而是先彻底理解：

```
Docker Image
     ↓
     Pod
     ↓
Deployment
     ↓
Service
     ↓
Ingress
     ↓
用户
```

以及：

```
Deployment
    ↓
ReplicaSet
    ↓
  Pod
    ↓
Container
```

和：

```
用户
 ↓
Ingress
 ↓
Service
 ↓
Pod
 ↓
Container
 ↓
你的应用
```

这三条链路理解以后，K8S 就不再是“一堆 YAML 配置”，后面的 **ConfigMap、Secret、Storage、HPA、RBAC、Helm、监控、CI/CD、GitOps** 都会有明确的归属。

------

**接下来我们就从第 1 章开始：`为什么需要 Kubernetes`。**

我会以一个真实后端项目为背景，从“**一台服务器上直接运行 .NET API**”开始，一步步演进到 **Docker → Docker Compose → Kubernetes → 生产集群**，让你理解 K8S 到底是在解决什么问题，而不是先背概念。