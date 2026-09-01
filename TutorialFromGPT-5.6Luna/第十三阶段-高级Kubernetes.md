# 第 31 章：Kustomize

Kustomize 是 Kubernetes 生态中非常重要的配置管理工具。

前面学习 Helm 时，我们解决的是：

```
同一个应用
    ↓
不同参数
    ↓
生成不同 Kubernetes Manifest
```

Kustomize 解决问题的思路不同：

```
Base
 ↓
Overlay
 ↓
Patch
 ↓
不同环境的 Kubernetes Manifest
```

它最大的特点是：

> **在不修改原始 YAML 的情况下，通过 Overlay 和 Patch 定制 Kubernetes 配置。**

Kustomize 已经集成到 `kubectl` 中，因此通常不需要额外安装。

可以直接检查：

```
kubectl kustomize --help
```

也可以直接渲染：

```
kubectl kustomize .
```

------

## 31.1 Kustomize 是什么

### 31.1.1 什么是 Kustomize

Kustomize 是 Kubernetes 配置定制工具。

它的核心思想不是：

```
Template
```

而是：

```
Base
 +
Overlay
 +
Patch
```

例如我们有一个基础 Deployment：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi

spec:
  replicas: 2

  selector:
    matchLabels:
      app: myapi

  template:
    metadata:
      labels:
        app: myapi

    spec:
      containers:
        - name: myapi
          image: registry.example.com/myapi:1.0.0
```

开发环境可能需要：

```
replicas = 1
```

生产环境：

```
replicas = 5
```

传统方式可能复制两份 YAML：

```
dev-deployment.yaml
prod-deployment.yaml
```

这会造成：

```
重复
配置漂移
维护困难
```

Kustomize：

```
              Base
               │
        ┌──────┴──────┐
        ▼             ▼
       Dev           Prod
     Overlay        Overlay
        │             │
        ▼             ▼
    replicas=1    replicas=5
```

这样只维护一份基础配置。

------

### 31.1.2 Kustomize 的工作原理

基本流程：

```
Base YAML
    ↓
Kustomization
    ↓
Overlay
    ↓
Patch / Generator
    ↓
Rendered Manifest
    ↓
kubectl apply
```

例如：

```
kubectl kustomize overlays/production
```

只负责生成最终 YAML：

```
Kustomize
    ↓
YAML
```

然后：

```
kubectl apply -k overlays/production
```

则会直接：

```
Kustomize
    ↓
Kubernetes
```

------

### 31.1.3 Kustomize 与模板的区别

Helm 通常：

```
replicas: {{ .Values.replicaCount }}
```

Kustomize 通常保留正常 Kubernetes YAML：

```
replicas: 2
```

然后 Overlay 修改：

```
replicas: 5
```

因此 Kustomize 对 Kubernetes YAML 的理解更加直接。

------

## 31.2 Base

### 31.2.1 什么是 Base

Base 是：

> **多个环境共同使用的基础 Kubernetes 配置。**

例如：

```
myapi/
└── base/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml
```

Base 可以包含：

```
Deployment
Service
ConfigMap
Ingress
ServiceAccount
HPA
```

等资源。

------

### 31.2.2 Deployment

`base/deployment.yaml`：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi

spec:
  replicas: 2

  selector:
    matchLabels:
      app: myapi

  template:
    metadata:
      labels:
        app: myapi

    spec:
      containers:
        - name: myapi
          image: registry.example.com/myapi:1.0.0
          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: 200m
              memory: 256Mi

            limits:
              cpu: 1
              memory: 512Mi
```

------

### 31.2.3 Service

`base/service.yaml`：

```
apiVersion: v1
kind: Service

metadata:
  name: myapi

spec:
  selector:
    app: myapi

  ports:
    - port: 80
      targetPort: 8080
```

------

### 31.2.4 kustomization.yaml

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

目录：

```
myapi/
└── base/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml
```

执行：

```
kubectl kustomize base/
```

可以看到最终生成的 YAML。

------

### 31.2.5 Base 不一定是“生产环境配置”

这是一个很重要的概念。

Base：

```
共同配置
```

而不是：

```
Production 配置
```

例如：

```
Base:
  image
  containerPort
  Service
  probes
  resources

Production:
  replicas
  domain
  resource size
```

因此 Base 应尽量放：

> **跨环境稳定、共同的配置。**

------

## 31.3 Overlay

### 31.3.1 什么是 Overlay

Overlay 是：

> **基于 Base 对某个环境进行定制的配置。**

典型目录：

```
myapi/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
│
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    │
    ├── staging/
    │   └── kustomization.yaml
    │
    └── production/
        └── kustomization.yaml
```

结构非常清晰：

```
Base
 │
 ├── Dev
 ├── Staging
 └── Production
```

------

### 31.3.2 Dev Overlay

`overlays/dev/kustomization.yaml`：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - ../../base

replicas:
  - name: myapi
    count: 1
```

这里：

```
Base:
replicas = 2

Dev:
replicas = 1
```

------

### 31.3.3 Production Overlay

`overlays/production/kustomization.yaml`：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

replicas:
  - name: myapi
    count: 5
```

于是：

```
Base:
2 replicas

Dev:
1 replica

Production:
5 replicas
```

执行：

```
kubectl kustomize overlays/production
```

检查生成结果。

确认无误后：

```
kubectl apply -k overlays/production
```

------

### 31.3.4 Overlay 的生产价值

Overlay 最适合解决：

```
环境差异
```

例如：

```
Dev
 ├── replicas: 1
 ├── resources: small
 └── log level: debug

Staging
 ├── replicas: 2
 ├── resources: medium
 └── log level: info

Production
 ├── replicas: 5
 ├── resources: large
 └── log level: warn
```

应用本身仍然是：

```
同一个应用
```

而不是复制三份。

------

## 31.4 Patch

### 31.4.1 什么是 Patch

Patch 用于：

> **修改 Base 中已有资源的部分字段。**

例如 Base：

```
spec:
  replicas: 2
```

Production：

```
spec:
  replicas: 5
```

不需要复制整个 Deployment。

只写：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi

spec:
  replicas: 5
```

------

### 31.4.2 Strategic Merge Patch

传统 Kubernetes/Kustomize 中常见的 Patch 方式之一是 Strategic Merge Patch。

例如：

`replicas-patch.yaml`：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi

spec:
  replicas: 5
```

然后：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - path: replicas-patch.yaml
```

结果：

```
Base Deployment
      ↓
Patch
      ↓
replicas = 5
```

------

### 31.4.3 JSON 6902 Patch

另一种方式是 JSON 6902 Patch。

例如：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: myapi
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

这里：

```
op: replace
```

表示替换。

```
path:
/spec/replicas
```

表示修改：

```
Deployment.spec.replicas
```

------

### 31.4.4 Patch 应该修改什么

适合：

```
replicas
resources
image
environment
labels
annotations
nodeSelector
affinity
tolerations
```

例如生产环境增加 Node Affinity：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi

spec:
  template:
    spec:
      nodeSelector:
        workload: production
```

------

### 31.4.5 Patch 不应该滥用

如果 Overlay 里存在大量 Patch：

```
patch-1.yaml
patch-2.yaml
patch-3.yaml
patch-4.yaml
patch-5.yaml
...
```

最终会很难理解：

```
Base
 ↓
Patch A
 ↓
Patch B
 ↓
Patch C
 ↓
最终 YAML
```

因此生产环境建议：

> **共同配置放 Base，真正的环境差异放 Overlay，Patch 只修改必要字段。**

------

## 31.5 ConfigMap Generator

### 31.5.1 为什么需要 ConfigMap Generator

前面学习 ConfigMap 时，可以手动写：

```
apiVersion: v1
kind: ConfigMap

metadata:
  name: myapi-config

data:
  LOG_LEVEL: info
  APP_ENV: production
```

Kustomize 可以自动根据文件或字面量生成 ConfigMap。

例如：

```
config/
└── application.yaml
```

内容：

```
server:
  port: 8080

logging:
  level: info
```

------

### 31.5.2 文件生成 ConfigMap

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

configMapGenerator:
  - name: myapi-config
    files:
      - config/application.yaml
```

Kustomize 会生成 ConfigMap。

需要注意：

> Kustomize 默认会为 Generator 生成的 ConfigMap 添加内容哈希后缀。

例如可能得到：

```
myapi-config-8h7k5m9g2
```

这样配置内容变化时，ConfigMap 名称也会变化。

------

### 31.5.3 为什么 Hash 很重要

假设：

```
ConfigMap:
version A
```

Pod 使用：

```
myapi-config
```

如果 ConfigMap 内容改变，应用是否自动重新加载取决于应用和挂载方式。

Kustomize 的哈希名称机制可以让：

```
ConfigMap A
 ↓
ConfigMap B
```

变成不同资源。

如果 Pod 的 PodTemplate 中引用了新的 ConfigMap 名称，那么：

```
PodTemplate 改变
 ↓
Deployment Rollout
 ↓
新 Pod
```

这对于配置变更触发滚动更新非常有价值。

------

### 31.5.4 literals

也可以：

```
configMapGenerator:
  - name: myapi-config
    literals:
      - APP_ENV=production
      - LOG_LEVEL=info
```

生成：

```
APP_ENV=production
LOG_LEVEL=info
```

------

### 31.5.5 ConfigMap Generator 的常见问题

不要把：

```
数据库密码
API Token
Access Key
```

放进：

```
configMapGenerator
```

ConfigMap 不是敏感信息存储机制。

敏感数据应该使用 Secret 管理体系。

------

## 31.6 Secret Generator

### 31.6.1 什么是 Secret Generator

Kustomize 同样提供：

```
secretGenerator
```

例如：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

secretGenerator:
  - name: myapi-secret
    literals:
      - DB_USER=app
      - DB_PASSWORD=example-password
```

Kustomize 会生成 Secret。

------

### 31.6.2 为什么使用 Secret Generator

和 ConfigMap Generator 类似：

```
Secret
 ↓
内容变化
 ↓
Secret 名称 Hash 变化
 ↓
引用发生变化
 ↓
Pod Template 变化
 ↓
Deployment Rollout
```

可以帮助处理配置变更后的滚动更新。

------

### 31.6.3 重要安全问题

下面这种方式：

```
secretGenerator:
  - name: myapi-secret
    literals:
      - DB_PASSWORD=production-password
```

如果直接提交 Git：

```
Git Repository
      ↓
production-password
```

仍然是危险的。

因为：

> **Kustomize Secret Generator 并不会让 Git 中的明文 Secret 自动变得安全。**

生产环境应结合：

```
External Secrets
Vault
Cloud Secret Manager
SOPS
Sealed Secrets
```

等机制。

------

## 31.7 多环境部署

### 31.7.1 推荐目录结构

一个比较实用的生产结构：

```
myapi/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
│
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    │
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    │
    └── production/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── resources-patch.yaml
```

------

### 31.7.2 部署 Dev

```
kubectl apply -k overlays/dev/
```

查看：

```
kubectl -n dev get all
```

------

### 31.7.3 部署 Staging

```
kubectl apply -k overlays/staging/
```

查看：

```
kubectl -n staging get all
```

------

### 31.7.4 部署 Production

先检查最终 YAML：

```
kubectl kustomize overlays/production/
```

这是生产环境非常重要的习惯。

确认：

```
Image
Replicas
Namespace
Resources
Ingress
ConfigMap
Secret
```

都正确之后再：

```
kubectl apply -k overlays/production/
```

------

### 31.7.5 Kustomize + GitOps

结合上一章的 Argo CD：

```
Git
 │
 ├── base
 │
 └── overlays
       ├── dev
       ├── staging
       └── production
             │
             ▼
          Argo CD
             │
             ▼
        Kubernetes
```

Argo CD Application：

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: myapi-production
  namespace: argocd

spec:
  source:
    repoURL: https://github.com/example/gitops.git
    targetRevision: main
    path: myapi/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

这里 Argo CD 不需要自己理解：

```
replicas 应该是多少
```

而是调用 Kustomize 生成：

```
最终 Kubernetes Manifest
```

然后进行 Sync。

------

## 31.8 Helm vs Kustomize

Helm 和 Kustomize 都可以解决：

```
Kubernetes 配置管理
多环境部署
应用发布
```

但设计理念不同。

### 31.8.1 Helm

Helm：

```
Template
+
Values
=
Manifest
```

例如：

```
replicas: {{ .Values.replicaCount }}
```

Values：

```
replicaCount: 5
```

最终：

```
replicas: 5
```

Helm 更像：

> **应用模板 + 参数化打包系统。**

------

### 31.8.2 Kustomize

Kustomize：

```
Base
+
Overlay
+
Patch
=
Manifest
```

它更像：

> **Kubernetes YAML 的环境定制系统。**

------

### 31.8.3 对比

| 能力                   | Helm                       | Kustomize      |
| ---------------------- | -------------------------- | -------------- |
| 模板                   | 强                         | 不以模板为核心 |
| Values                 | 强                         | 无传统 Values  |
| Overlay                | 不属于核心模型             | 核心能力       |
| Patch                  | 有相关能力，但不是主要模型 | 核心能力       |
| 应用打包               | 强                         | 较弱           |
| 第三方 Chart           | 非常丰富                   | 不适用         |
| Kubernetes YAML 原生感 | 中                         | 高             |
| 多环境                 | 很适合                     | 很适合         |
| 学习成本               | 中                         | 较低           |
| GitOps                 | 很适合                     | 很适合         |

------

### 31.8.4 什么时候选择 Helm

比较适合：

```
复杂应用
需要大量参数
需要作为可复用 Package
需要发布给其他团队
使用第三方 Kubernetes 应用
```

例如：

```
Prometheus
Grafana
Ingress Controller
数据库 Operator
```

很多 Kubernetes 软件都以 Helm Chart 形式提供。

------

### 31.8.5 什么时候选择 Kustomize

比较适合：

```
团队自己维护 YAML
环境之间只有少量差异
希望尽量保持原生 Kubernetes YAML
需要大量 Overlay / Patch
GitOps Repository
```

例如：

```
myapi
frontend
worker
internal-service
```

------

## 31.9 Helm + Kustomize

Helm 和 Kustomize **不是完全互斥的技术**。

在生产环境中可以组合使用。

### 31.9.1 组合方式一：Helm 负责生成，Kustomize 负责定制

流程：

```
Helm Chart
    ↓
Helm Render
    ↓
Kubernetes YAML
    ↓
Kustomize Overlay
    ↓
Final Manifest
```

概念上：

```
Helm
 ↓
Package / Template
 ↓
Kustomize
 ↓
Environment Customization
```

例如：

```
Helm Chart
 ├── Deployment
 ├── Service
 └── ConfigMap
        ↓
      Base
        ↓
Production Overlay
        ↓
Patch replicas
        ↓
Patch resources
```

------

### 31.9.2 组合方式二：Argo CD + Helm + Kustomize

完整生产链可以是：

```
Git
 │
 ▼
Argo CD
 │
 ├── Helm
 │     ↓
 │   Render
 │
 └── Kustomize
       ↓
   Final Manifest
       ↓
   Kubernetes
```

但这里必须控制复杂度。

如果一个简单 API：

```
Helm
 +
Kustomize
 +
多个 Patch
 +
多个 Values
```

最终可能变成：

```
开发人员：
“这个 replicas 到底在哪里定义的？”
```

所以：

> **工具越多不代表架构越高级。**

应该根据配置复杂度选择合适的方案。

------

### 31.9.3 一个实际选择建议

对于自己维护的简单业务：

```
Kubernetes YAML
        +
   Kustomize
        +
     Argo CD
```

通常已经足够。

对于需要参数化、复用和打包的复杂应用：

```
Helm
 +
Argo CD
```

比较合适。

对于复杂平台型场景：

```
Helm
 +
Kustomize
 +
Argo CD
```

也可以使用，但必须明确每一层的职责。

------

## Kustomize 生产实践

一个比较推荐的结构：

```
GitOps Repository
│
└── myapi/
    │
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── ingress.yaml
    │   └── kustomization.yaml
    │
    └── overlays/
        │
        ├── dev/
        │   ├── kustomization.yaml
        │   └── patches/
        │
        ├── staging/
        │   ├── kustomization.yaml
        │   └── patches/
        │
        └── production/
            ├── kustomization.yaml
            └── patches/
```

发布：

```
Developer
   ↓
Application Git
   ↓
CI
   ↓
Docker Image
   ↓
Registry
   ↓
GitOps Repository
   ↓
Kustomize Overlay
   ↓
Argo CD
   ↓
Kubernetes
```

------

## 生产环境使用 Kustomize 的几个原则

### 1. Base 放共同配置

```
Base
 ↓
所有环境共享
```

不要把大量 Production 特有配置塞进 Base。

### 2. Overlay 表达环境差异

```
dev
staging
production
```

应该清晰可见。

### 3. Patch 保持小而明确

不要让：

```
10 个 Patch
```

共同修改同一个 Deployment 的同一个字段。

### 4. 先 Render，再 Apply

生产部署前：

```
kubectl kustomize overlays/production/
```

确认最终 YAML。

然后：

```
kubectl apply -k overlays/production/
```

### 5. 不要把明文 Secret 放进 Git

特别注意：

```
secretGenerator
```

并不意味着：

```
Secret = 安全
```

它只是生成 Kubernetes Secret。

### 6. GitOps 中让 Git 保持唯一事实来源

使用 Argo CD 时，不要形成：

```
Git
 ↓
Argo CD

同时

管理员
 ↓
kubectl edit
```

两套系统共同管理同一资源。

应该尽量：

```
Git
 ↓
Argo CD
 ↓
Kubernetes
```

------

## 本章核心知识总结

Kustomize 最重要的不是记住命令，而是理解这张关系图：

```
                    Base
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
         Dev       Staging    Production
          │          │          │
          ▼          ▼          ▼
       Overlay     Overlay    Overlay
          │          │          │
          └──────────┼──────────┘
                     │
             Patch / Generator
                     │
                     ▼
             Final Manifest
                     │
                     ▼
                Kubernetes
```

核心概念可以记成：

```
Base
 ↓
公共配置

Overlay
 ↓
环境差异

Patch
 ↓
修改已有资源

ConfigMap Generator
 ↓
生成 ConfigMap

Secret Generator
 ↓
生成 Secret

Kustomize
 ↓
最终 Kubernetes Manifest
```

而与 Helm、Argo CD 的关系则是：

```
Helm
 ↓
模板 / 打包

Kustomize
 ↓
定制 / Overlay

Argo CD
 ↓
GitOps / Sync / Reconciliation

Kubernetes
 ↓
运行应用
```

因此在生产环境中，Kustomize 最有价值的能力不是“少写几份 YAML”，而是建立一种清晰的配置继承关系：

```
共同配置
   ↓
环境 Overlay
   ↓
必要 Patch
   ↓
最终 Manifest
   ↓
Argo CD
   ↓
Kubernetes
```

这样既能避免复制大量 YAML，又能让 **Dev / Staging / Production 的差异明确、可审计、可版本控制**。

# 第 32 章：CRD 与 Operator

前面我们学习 Kubernetes 内置资源，例如：

```
Pod
Deployment
Service
ConfigMap
Secret
StatefulSet
Job
```

这些资源都是 Kubernetes API 原生提供的。

但是生产环境中经常会出现一个问题：

> **如果 Kubernetes 原生资源无法表达我们的业务需求怎么办？**

例如，我们希望 Kubernetes 能够理解：

```
PostgreSQL
Redis
Kafka
Prometheus
MySQL
```

并且不仅仅是创建一个 Pod，而是能够理解：

```
数据库集群
主从关系
故障切换
备份
恢复
扩容
版本升级
```

这时候就需要：

```
CRD
 ↓
Custom Resource
 ↓
Controller
 ↓
Operator
```

这是理解现代 Kubernetes 生态非常重要的一组概念。

------

## 32.1 Kubernetes API Extension

### 32.1.1 什么是 Kubernetes API Extension

Kubernetes 本质上不仅仅是一个：

```
容器运行平台
```

更重要的是，它提供了一个：

```
声明式 API + Controller
```

体系。

我们之前执行：

```
kubectl apply -f deployment.yaml
```

实际上是在告诉 Kubernetes：

> “我希望集群最终达到这个状态。”

例如：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
spec:
  replicas: 3
```

这里的：

```
kind: Deployment
```

就是 Kubernetes API 中的一种资源类型。

------

### 32.1.2 Kubernetes API 的资源

可以查看集群支持哪些 API Resource：

```
kubectl api-resources
```

例如可能看到：

```
pods
services
deployments
statefulsets
configmaps
secrets
jobs
cronjobs
```

查看 API Version：

```
kubectl api-versions
```

例如：

```
apps/v1
batch/v1
v1
```

------

### 32.1.3 为什么需要扩展 Kubernetes API

假设我们要管理 PostgreSQL。

原生 Kubernetes 能提供：

```
Deployment
StatefulSet
Service
PVC
ConfigMap
Secret
```

但是 Kubernetes 并不知道：

```
PostgreSQL 主节点是谁？
什么时候可以进行数据库升级？
如何执行数据库备份？
如何进行主从切换？
数据库集群是否健康？
```

如果全部依靠人工脚本：

```
kubectl
+
Shell
+
定时任务
+
监控
```

系统会越来越复杂。

因此我们希望直接告诉 Kubernetes：

```
kind: PostgreSQL
spec:
  version: "16"
  replicas: 3
  backup:
    enabled: true
```

然后由 Kubernetes 中的控制器负责实现。

这就是 Kubernetes API Extension 的核心思想。

------

## 32.2 CRD

### 32.2.1 什么是 CRD

CRD：

> **CustomResourceDefinition**

中文可以理解为：

> 自定义资源定义。

它允许我们向 Kubernetes API 增加一种新的资源类型。

例如 Kubernetes 原生有：

```
kind: Deployment
```

我们可以通过 CRD 增加：

```
kind: PostgreSQL
```

或者：

```
kind: Redis
```

甚至：

```
kind: MyDatabase
```

------

### 32.2.2 一个简单 CRD

下面创建一个非常简单的 `Website` CRD：

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com

spec:
  group: example.com

  names:
    plural: websites
    singular: website
    kind: Website
    shortNames:
      - web

  scope: Namespaced

  versions:
    - name: v1
      served: true
      storage: true

      schema:
        openAPIV3Schema:
          type: object

          properties:
            spec:
              type: object

              properties:
                replicas:
                  type: integer
                  minimum: 1

                image:
                  type: string
```

保存：

```
website-crd.yaml
```

然后：

```
kubectl apply -f website-crd.yaml
```

查看：

```
kubectl get crd
```

应该可以看到：

```
websites.example.com
```

------

### 32.2.3 CRD 做了什么

CRD 本身并没有部署应用。

它只是告诉 Kubernetes：

```
以后 API 中允许出现：

kind: Website
```

也就是说：

```
CRD
 ↓
定义新的 API Resource
```

而不是：

```
CRD
 ↓
自动创建 Pod
```

这一点非常重要。

------

## 32.3 Custom Resource

### 32.3.1 什么是 Custom Resource

CR：

> **Custom Resource**

是基于 CRD 创建出来的实际资源对象。

刚才 CRD 定义了：

```
Website
```

现在可以创建一个：

```
apiVersion: example.com/v1
kind: Website

metadata:
  name: my-website

spec:
  replicas: 3
  image: nginx:1.29
```

保存：

```
website.yaml
```

然后：

```
kubectl apply -f website.yaml
```

查看：

```
kubectl get websites
```

也可以：

```
kubectl get web
```

因为我们在 CRD 中定义了：

```
shortNames:
  - web
```

------

### 32.3.2 CRD 与 CR 的关系

可以把它理解成：

```
CRD
 ↓
定义“数据结构 / API 类型”
 ↓
Website

CR
 ↓
Website 的一个具体实例
 ↓
my-website
```

类似：

```
Class
 ↓
Object
```

但不要把它简单等同于编程语言的 Class/Object；这里只是帮助理解。

------

### 32.3.3 CR 本身不会自动产生业务效果

这是初学者非常容易产生的误解。

创建：

```
kind: Website
```

之后：

```
kubectl get website
```

可以看到：

```
my-website
```

但是：

```
kubectl get pods
```

**不会因为 CRD 自动出现 Pod。**

原因是：

```
CRD
 ↓
只定义 API
```

还缺少：

```
Controller
```

------

## 32.4 Controller

### 32.4.1 什么是 Controller

Controller 可以理解为：

> **持续观察 Kubernetes 当前状态，并努力把它调整到期望状态的控制循环。**

这是 Kubernetes 最核心的设计思想之一。

例如 Deployment：

```
spec:
  replicas: 3
```

Controller 观察：

```
期望：
3 Pods

实际：
2 Pods
```

于是：

```
Controller
 ↓
发现实际状态 ≠ 期望状态
 ↓
创建 1 个 Pod
```

最终：

```
期望 = 3
实际 = 3
```

------

### 32.4.2 Controller 的基本工作循环

可以抽象成：

```
        Kubernetes API
              │
              ▼
         Controller
              │
       Observe State
              │
              ▼
      Desired ≠ Actual?
          │          │
         Yes         No
          │           │
          ▼           ▼
       Reconcile     等待
          │
          ▼
      修改资源
          │
          ▼
   Kubernetes API
```

这个过程通常称为：

> **Reconciliation（调谐 / 协调）**

------

### 32.4.3 Controller 不是一次性脚本

错误理解：

```
Controller
 ↓
执行一次
 ↓
结束
```

正确理解：

```
Controller
 ↓
持续 Watch
 ↓
发现变化
 ↓
Reconcile
 ↓
再次 Watch
 ↓
持续运行
```

例如：

```
Deployment replicas = 3
```

某个 Pod 因为 Node 故障消失：

```
实际 = 2
```

Controller 发现：

```
Desired = 3
Actual = 2
```

于是重新创建 Pod。

------

## 32.5 Operator

### 32.5.1 什么是 Operator

Operator 可以理解为：

> **把一个领域专家的运维知识编码成 Kubernetes Controller。**

例如一个 PostgreSQL Operator，可以把数据库管理员通常执行的操作自动化：

```
部署数据库
配置主从
故障检测
主从切换
备份
恢复
扩容
升级
```

最终用户只需要声明：

```
kind: PostgreSQL
```

例如：

```
apiVersion: database.example.com/v1
kind: PostgreSQL

metadata:
  name: production-db

spec:
  version: "16"
  replicas: 3
```

Operator 负责把它转换成实际 Kubernetes 资源和数据库操作。

------

### 32.5.2 Operator 与 Controller 的关系

可以这样理解：

```
Controller
    ↓
通用控制循环概念

Operator
    ↓
针对某个具体应用 / 领域的 Controller
    ↓
通常结合 CRD
```

所以：

> **Operator 通常是 Controller + CRD + 领域运维逻辑。**

但严格来说，Operator 并不是 Kubernetes API 中的一个特殊资源类型，而是一种软件设计模式。

------

## 32.6 Operator 工作原理

### 32.6.1 整体架构

一个 Operator 通常类似：

```
                 Kubernetes API
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
        Custom Resource      Pod/Service
             │
             ▼
          Operator
             │
         Watch CR
             │
             ▼
        Reconcile()
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
    Create Update Delete
       │     │     │
       └─────┼─────┘
             ▼
       Desired State
```

------

### 32.6.2 用户提交 CR

例如：

```
apiVersion: database.example.com/v1
kind: PostgreSQL

metadata:
  name: mydb

spec:
  version: "16"
  replicas: 3
```

执行：

```
kubectl apply -f postgres.yaml
```

Kubernetes API Server 保存：

```
PostgreSQL/mydb
```

------

### 32.6.3 Operator Watch

Operator 一直 Watch：

```
PostgreSQL resources
```

发现：

```
mydb created
```

于是触发：

```
Reconcile(mydb)
```

------

### 32.6.4 Operator 创建底层资源

例如 Operator 可能创建：

```
StatefulSet
Service
ConfigMap
Secret
PVC
```

最终：

```
PostgreSQL CR
      ↓
PostgreSQL Operator
      ↓
StatefulSet
Service
PVC
Secret
      ↓
PostgreSQL Pods
```

------

### 32.6.5 Operator 持续修复状态

假设：

```
期望：
PostgreSQL = 3 replicas

实际：
PostgreSQL = 2 replicas
```

Operator 会发现：

```
Desired ≠ Actual
```

然后执行：

```
Reconcile
```

修复集群。

所以 Operator 的核心不是：

```
创建资源
```

而是：

> **持续让实际状态趋近于用户声明的期望状态。**

------

### 32.6.6 Status

成熟的 Operator 通常还会更新 CR 的：

```
status:
```

例如：

```
status:
  phase: Ready
  readyReplicas: 3
```

于是：

```
kubectl get postgres
```

可能看到：

```
NAME   STATUS   READY
mydb   Ready    3
```

这形成完整的：

```
spec
 ↓
用户想要什么

status
 ↓
当前实际怎么样
```

这也是 Kubernetes API 非常重要的设计模式。

------

## 32.7 为什么 Prometheus / Database Operator 都大量使用 CRD

### 32.7.1 Prometheus 为什么适合 Operator

Prometheus 本身不是简单的：

```
Deployment + Service
```

生产环境还涉及：

```
配置
监控目标
告警规则
存储
高可用
升级
扩缩容
```

如果全部手工维护 YAML：

```
Deployment
ConfigMap
Service
PVC
Rules
ServiceMonitor
```

会越来越复杂。

Operator 可以把这些知识封装起来。

例如用户只需要定义类似：

```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus

metadata:
  name: prometheus

spec:
  replicas: 2
```

Operator 负责：

```
Prometheus CR
      ↓
Prometheus Operator
      ↓
Deployment / StatefulSet
Service
Config
Rules
```

------

### 32.7.2 ServiceMonitor

Prometheus Operator 生态中还会定义新的 CRD，例如：

```
ServiceMonitor
```

用户可以声明：

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:
  name: myapi

spec:
  selector:
    matchLabels:
      app: myapi

  endpoints:
    - port: metrics
```

表达：

> Prometheus 应该监控哪些 Service。

这样就把：

```
Prometheus 配置
```

变成了：

```
Kubernetes Resource
```

可以使用：

```
kubectl get servicemonitor
```

统一管理。

------

### 32.7.3 Database Operator 为什么更需要 CRD

数据库比普通 Web API 更复杂。

例如 PostgreSQL：

```
Primary
Replica
WAL
Backup
Restore
Failover
Storage
Upgrade
```

用户如果手动管理：

```
StatefulSet
Service
PVC
ConfigMap
Secret
Backup Job
Failover Script
```

运维复杂度非常高。

Operator 可以把这些操作封装成：

```
PostgreSQL CR
      ↓
Operator
      ↓
完整数据库集群
```

例如用户声明：

```
spec:
  replicas: 3
  backup:
    enabled: true
```

Operator 负责实际实现。

这就是：

> **把“数据库管理员经验”转换成 Kubernetes 自动化控制逻辑。**

------

## 32.8 如何理解 Operator 模式

这是本章最重要的部分。

### 32.8.1 不要把 Operator 理解成“安装工具”

错误理解：

```
Operator
=
帮我安装软件
```

这只是其中很小的一部分。

Operator 的核心是：

```
声明
 ↓
观察
 ↓
判断
 ↓
调谐
 ↓
持续维护
```

------

### 32.8.2 Operator = 软件化的运维人员

可以用一个非常直观的方式理解。

传统数据库运维：

```
运维工程师
 │
 ├── 部署数据库
 ├── 检查状态
 ├── 创建副本
 ├── 故障切换
 ├── 扩容
 ├── 备份
 └── 升级
```

Operator：

```
Kubernetes Operator
 │
 ├── 部署数据库
 ├── 检查状态
 ├── 创建副本
 ├── 故障切换
 ├── 扩容
 ├── 备份
 └── 升级
```

区别只是：

```
人工运维经验
      ↓
代码化
      ↓
Controller
      ↓
Operator
```

------

### 32.8.3 声明式 vs 命令式

没有 Operator 时：

```
创建 StatefulSet
创建 Service
创建 PVC
执行初始化
配置复制
配置备份
```

更接近：

```
告诉系统“怎么做”
```

Operator 模式：

```
kind: PostgreSQL

spec:
  replicas: 3
  backup:
    enabled: true
```

告诉系统：

```
“我要什么”
```

Operator 自己决定：

```
“应该怎么做”
```

这就是 Kubernetes 最核心的：

> **Declarative Configuration + Reconciliation**

------

### 32.8.4 Operator 的完整模型

最终可以把整个机制理解成：

```
                 用户
                  │
                  │ kubectl apply
                  ▼
             Custom Resource
                  │
                  ▼
             Kubernetes API
                  │
                  ▼
              Operator
                  │
              Watch / Reconcile
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
   StatefulSet  Service      PVC
       │          │          │
       └──────────┼──────────┘
                  ▼
              Application
                  │
                  ▼
             Actual State
                  │
                  └──────────────┐
                                 │
                                 ▼
                           Operator 再次观察
                                 │
                                 ▼
                              Reconcile
```

整个系统不断循环：

```
Desired State
      ↓
Observe
      ↓
Compare
      ↓
Reconcile
      ↓
Actual State
      ↓
Observe
      ↓
Compare
      ↓
Reconcile
      ↓
...
```

------

## CRD、CR、Controller、Operator 的关系

最后把几个容易混淆的概念放在一起：

| 概念            | 作用                                  |
| --------------- | ------------------------------------- |
| Kubernetes API  | 提供资源管理接口                      |
| CRD             | 扩展 Kubernetes API，定义新的资源类型 |
| Custom Resource | CRD 定义出来的具体资源实例            |
| Controller      | 观察资源并进行 Reconcile              |
| Operator        | 面向特定应用/领域的 Controller 模式   |
| `spec`          | 用户声明期望状态                      |
| `status`        | 系统反映当前状态                      |

最重要的关系：

```
CRD
 ↓
定义资源类型

Custom Resource
 ↓
创建资源实例

Controller
 ↓
处理资源

Operator
 ↓
把复杂应用运维逻辑自动化
```

------

## 生产环境中的 Operator

在生产 Kubernetes 中，你会经常遇到：

```
Prometheus Operator
Database Operator
Kafka Operator
Redis Operator
Cert Operator
Storage Operator
```

它们的共同思路基本都是：

```
把复杂系统
      ↓
抽象成 Kubernetes Resource
      ↓
通过 CR 声明需求
      ↓
Operator Controller
      ↓
自动完成部署、配置、维护、故障处理
```

因此看到一个陌生 Operator 时，不要首先问：

> “这个 Operator 有哪些命令？”

应该首先问：

1. **它定义了哪些 CRD？**
2. **每种 CR 表达什么业务含义？**
3. **Controller 监听什么资源？**
4. **Reconcile 会创建/修改哪些 Kubernetes 资源？**
5. **CR 的 `status` 如何反映实际状态？**
6. **故障时 Operator 如何进行恢复？**

掌握这套思维，你就不再只是“会使用某个 Operator”，而是真正理解了 **Operator 模式**。

# 第 33 章：Service Mesh

Service Mesh（服务网格）是 Kubernetes 进入**大型微服务生产环境**后经常会遇到的技术。

在前面的章节中，我们主要关注：

```
用户
 ↓
Ingress
 ↓
Service
 ↓
Pod
```

但当系统变成几十、几百个微服务以后，服务之间会产生大量内部通信：

```
                    ┌──► user-service
                    │
api-gateway ────────┼──► order-service
                    │
                    ├──► payment-service
                    │
                    └──► inventory-service
```

这时候问题开始出现：

```
服务发现
流量控制
重试
超时
熔断
TLS
身份认证
监控
分布式追踪
```

如果每个微服务自己实现这些功能：

```
.NET Service
 ├── Retry
 ├── Timeout
 ├── TLS
 ├── Metrics
 └── Tracing

Java Service
 ├── Retry
 ├── Timeout
 ├── TLS
 ├── Metrics
 └── Tracing

Node.js Service
 ├── Retry
 ├── Timeout
 ├── TLS
 ├── Metrics
 └── Tracing
```

会产生大量重复代码。

Service Mesh 的核心思想就是：

> **把大量服务间通信能力从业务代码中抽离出来，由基础设施层统一处理。**

------

## 33.1 Service Mesh 是什么

### 33.1.1 什么是 Service Mesh

Service Mesh 可以理解成：

> **专门负责微服务之间网络通信的基础设施层。**

传统微服务：

```
Service A
    │
    │ HTTP/gRPC
    ▼
Service B
```

Service Mesh：

```
Service A
    │
    ▼
Proxy A
    │
    │
    ▼
Proxy B
    │
    ▼
Service B
```

业务程序主要负责：

```
业务逻辑
```

Proxy 负责：

```
网络通信治理
```

例如：

```
Retry
Timeout
Load Balancing
mTLS
Metrics
Tracing
Traffic Routing
```

------

### 33.1.2 Service Mesh 解决的核心问题

可以把 Service Mesh 的能力分成几个类别：

```
Traffic Management
        │
        ├── Routing
        ├── Retry
        ├── Timeout
        ├── Load Balancing
        └── Traffic Splitting

Security
        │
        ├── mTLS
        ├── Identity
        └── Authorization

Observability
        │
        ├── Metrics
        ├── Logs
        └── Tracing
```

因此 Service Mesh 本质上不是：

```
另一个 Kubernetes
```

也不是：

```
另一个 Ingress
```

而是：

> **主要解决 Service-to-Service 通信治理问题。**

------

### 33.1.3 Service Mesh 与 Kubernetes Service 的区别

这是非常容易混淆的地方。

Kubernetes Service：

```
Service
 ↓
提供稳定的服务访问地址
 ↓
Pod
```

主要解决：

```
服务发现
负载均衡
稳定访问入口
```

Service Mesh：

```
Service A
 ↓
Proxy
 ↓
Proxy
 ↓
Service B
```

主要解决：

```
流量治理
安全
可观测性
```

两者不是互相替代关系。

生产环境中经常是：

```
Kubernetes Service
        +
Service Mesh
```

一起使用。

------

## 33.2 为什么需要 Service Mesh

### 33.2.1 微服务数量增加

假设只有：

```
frontend
backend
database
```

通信关系很简单：

```
frontend
   ↓
backend
   ↓
database
```

通常没有必要引入 Service Mesh。

但是当系统变成：

```
API Gateway
    ↓
User
Order
Payment
Inventory
Product
Notification
Recommendation
Search
...
```

服务之间通信越来越复杂：

```
Order
 ├── User
 ├── Payment
 ├── Inventory
 └── Notification

Payment
 ├── User
 └── Fraud

Inventory
 └── Product
```

这时需要统一管理服务通信。

------

### 33.2.2 如果全部写在业务代码里

例如 .NET：

```
try
{
    // HTTP request
}
catch
{
    // Retry
}
```

Java：

```
@Retry(...)
@Timeout(...)
```

Node.js：

```
// retry / timeout / tracing
```

问题是：

```
不同语言
不同框架
不同实现
不同配置
```

Service Mesh 可以把其中一部分通信能力放到：

```
Proxy
```

中。

于是：

```
Application
     │
     │ localhost
     ▼
Sidecar Proxy
     │
     │ Mesh
     ▼
Remote Sidecar
     │
     ▼
Application
```

业务代码不需要直接实现所有通信治理能力。

------

### 33.2.3 统一治理

例如要求：

```
所有 Service-to-Service 流量
必须使用 mTLS
```

传统方式：

```
每个服务自己实现
```

Service Mesh：

```
Mesh Policy
      ↓
所有 Proxy
      ↓
统一执行
```

这就是 Service Mesh 的一个重要价值：

> **把跨服务的网络策略从应用代码中统一抽离。**

------

## 33.3 Sidecar

### 33.3.1 什么是 Sidecar

Sidecar 是 Service Mesh 最重要的概念之一。

它的意思是：

> **在业务 Pod 中运行一个辅助容器，与业务容器共同组成一个 Pod。**

例如：

```
apiVersion: v1
kind: Pod

metadata:
  name: myapi

spec:
  containers:

    - name: myapi
      image: myapi:1.0

    - name: proxy
      image: proxy:1.0
```

一个 Pod：

```
Pod
├── myapi
└── proxy
```

------

### 33.3.2 为什么 Sidecar 可以拦截流量

因为同一个 Pod 中的容器共享网络 Namespace。

可以理解成：

```
Pod Network Namespace
        │
        ├── myapi
        │
        └── proxy
```

Service Mesh 可以通过：

```
iptables
eBPF
其他流量重定向机制
```

把应用流量导向 Proxy。

于是：

```
Application
     │
     ▼
Proxy
     │
     ▼
Network
     │
     ▼
Remote Proxy
     │
     ▼
Remote Application
```

业务应用并不需要知道 Mesh 的存在。

------

### 33.3.3 Sidecar 的价值

例如业务代码发送：

```
HTTP request
```

实际可能变成：

```
Application
   ↓
Sidecar Proxy
   ↓
Retry / Timeout / mTLS / Metrics
   ↓
Remote Proxy
   ↓
Remote Application
```

因此 Proxy 可以在不修改业务代码的情况下增加网络治理能力。

------

### 33.3.4 Sidecar 的问题

Sidecar 并不是免费能力。

如果：

```
100 Pods
```

每个 Pod：

```
Application
+
Proxy
```

就意味着：

```
100 个 Application Containers
+
100 个 Proxy Containers
```

Proxy 会产生：

```
CPU
Memory
Network
```

消耗。

同时故障排查复杂度也会上升：

```
Application
     ↓
Proxy
     ↓
Service
     ↓
Proxy
     ↓
Application
```

排查路径比直接：

```
Application
 ↓
Service
 ↓
Application
```

更复杂。

------

## 33.4 Traffic Management

### 33.4.1 什么是 Traffic Management

Traffic Management 就是：

> **控制服务之间的流量应该如何走。**

例如：

```
api-v1
api-v2
```

我们希望：

```
90% → v1
10% → v2
```

Service Mesh 可以根据规则实现。

------

### 33.4.2 Canary Release

例如：

```
               ┌──► v1 90%
Client → Proxy
               └──► v2 10%
```

逐渐调整：

```
90 / 10
 ↓
70 / 30
 ↓
50 / 50
 ↓
0 / 100
```

这就是 Canary Release 的典型思路。

------

### 33.4.3 Header 路由

例如：

```
Header:
x-user-type: beta
```

让 Beta 用户进入新版本：

```
普通用户
   ↓
v1

Beta 用户
   ↓
v2
```

这种基于请求属性的路由，在复杂微服务系统中非常有价值。

------

### 33.4.4 Retry

例如：

```
Service A
   ↓
Service B
   X
   ↓
Retry
   ↓
Service B
   ✓
```

但是 Retry 必须谨慎。

例如一个支付请求：

```
POST /payment
```

如果第一次实际上已经成功，只是响应丢失：

```
Payment succeeded
       ↓
Response lost
       ↓
Proxy Retry
       ↓
Payment again
```

可能产生：

```
重复扣款
```

所以生产环境必须考虑：

```
Idempotency
Retry Policy
Timeout
```

不能简单地：

> “所有请求失败都 Retry。”

------

### 33.4.5 Timeout

例如：

```
Service A
   ↓
Service B
```

如果 B 长时间没有响应：

```
2s
5s
10s
30s
```

请求可能大量堆积。

Mesh 可以设置：

```
timeout = 2s
```

达到：

```
2 秒
 ↓
取消请求
```

避免故障服务拖垮调用方。

------

### 33.4.6 Circuit Breaking

Circuit Breaking（熔断）用于防止故障扩散。

例如：

```
Service B
   ↓
持续故障
```

如果 A 不断请求 B：

```
A
 ↓↓↓↓↓↓↓↓↓
B
```

会导致：

```
连接堆积
线程堆积
请求堆积
CPU 上升
```

熔断后：

```
A
 ↓
Circuit Breaker
 ↓
直接失败
```

减少对 B 的压力。

------

## 33.5 mTLS

### 33.5.1 什么是 mTLS

mTLS：

> **Mutual TLS，双向 TLS。**

普通 HTTPS：

```
Client ──TLS──► Server
```

主要验证：

```
Server
```

mTLS：

```
Client ◄──TLS──► Server
```

双方都需要证明身份。

------

### 33.5.2 Service Mesh 中的 mTLS

假设：

```
Service A
    ↓
Service B
```

Mesh 可以让：

```
Proxy A
    ⇄
Proxy B
```

建立 mTLS。

于是：

```
Application A
      ↓
Proxy A
      ║
      ║ encrypted + authenticated
      ║
Proxy B
      ↓
Application B
```

业务应用本身不需要自己处理：

```
Certificate
Private Key
TLS Handshake
Certificate Rotation
```

这些通常由 Mesh 控制面和数据面共同处理。

------

### 33.5.3 mTLS 解决什么问题

主要解决：

```
身份认证
通信加密
防止中间人攻击
服务身份验证
```

例如：

```
payment-service
```

可以确认：

> “连接我的确实是经过授权的 order-service。”

这比仅仅依靠：

```
IP
```

更加可靠。

------

### 33.5.4 mTLS 不等于业务授权

非常重要：

```
mTLS
```

解决：

> “你是谁？”

Authorization 解决：

> “你能做什么？”

例如：

```
order-service
```

身份可信：

```
✓
```

但它是否允许：

```
DELETE /users/123
```

是另一个问题。

所以：

```
Authentication
+
Authorization
```

需要分别设计。

------

## 33.6 Observability

### 33.6.1 Service Mesh 提供什么可观测性

Service Mesh 可以从 Proxy 层获取：

```
Request Count
Latency
Error Rate
Status Code
Traffic Volume
Connection Information
```

例如：

```
Service A → Service B

Requests: 10000
Success: 9800
5xx: 200
P99: 850ms
```

这些数据不一定需要修改业务代码才能获得。

------

### 33.6.2 Metrics

可以获得：

```
HTTP request rate
HTTP error rate
Request latency
TCP connection
```

然后进入：

```
Prometheus
 ↓
Grafana
```

形成：

```
Service Mesh
     ↓
Metrics
     ↓
Prometheus
     ↓
Grafana
```

------

### 33.6.3 Distributed Tracing

复杂请求：

```
User
 ↓
Gateway
 ↓
Order
 ↓
Payment
 ↓
Inventory
```

如果出现：

```
响应时间 = 5s
```

仅看应用日志很难判断：

```
到底是谁慢？
```

Tracing 可以形成：

```
Request
 ├── Gateway: 50ms
 ├── Order: 100ms
 ├── Payment: 4.5s
 └── Inventory: 80ms
```

于是很快发现：

```
Payment
```

是主要瓶颈。

------

### 33.6.4 Service Graph

Mesh 还可以帮助生成：

```
Gateway
 ├── User
 ├── Order
 │    ├── Payment
 │    └── Inventory
 └── Product
```

这样的服务依赖图。

对于大型微服务系统非常有价值。

------

## 33.7 Istio

### 33.7.1 什么是 Istio

Istio 是目前 Kubernetes 生态中非常成熟的 Service Mesh 方案之一。

它提供：

```
Traffic Management
Security
Observability
```

并支持：

```
HTTP
HTTPS
gRPC
TCP
```

等通信场景。

------

### 33.7.2 Istio 的基本架构

现代 Istio 可以理解成：

```
                Istio Control Plane
                       │
              ┌────────┼────────┐
              │        │        │
              ▼        ▼        ▼
           Config   Security  Telemetry
              │
              ▼
        Data Plane
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
   Proxy A  Proxy B  Proxy C
      │       │       │
      ▼       ▼       ▼
   Service A Service B Service C
```

其中：

```
Control Plane
```

负责：

```
配置
证书
策略
服务信息
```

而：

```
Data Plane
```

负责：

```
真正处理业务流量
```

------

### 33.7.3 Istio Ambient 与 Sidecar

Istio 生态还提供 Ambient 模式，目的是减少传统：

```
每个 Pod 一个 Sidecar
```

带来的资源和运维成本。

概念上可以理解为：

```
传统 Sidecar：

Pod A → Proxy
Pod B → Proxy
Pod C → Proxy
```

而 Ambient 更倾向于：

```
Pod A ─┐
Pod B ─┼→ Node/Shared Mesh Components
Pod C ─┘
```

具体架构和能力会随 Istio 版本持续演进，因此生产环境部署时应以所使用版本的官方文档为准。

重要的是理解：

> **Sidecar 不是 Service Mesh 唯一的数据面实现方式。**

------

## 33.8 Linkerd

### 33.8.1 什么是 Linkerd

Linkerd 同样是 Kubernetes Service Mesh。

它的设计重点之一是：

> **尽量简单、轻量地提供 Service Mesh 能力。**

主要提供：

```
Traffic Management
mTLS
Observability
Service Reliability
```

------

### 33.8.2 Linkerd 的特点

可以粗略理解：

```
Istio
 ↓
功能非常丰富
 ↓
能力范围广
```

而：

```
Linkerd
 ↓
更加专注
 ↓
强调简单和轻量
```

这并不是简单的：

```
Istio 好
Linkerd 差
```

而是：

> **不同规模、不同需求下的工程取舍。**

------

### 33.8.3 如何选择

如果团队需要：

```
复杂流量治理
丰富安全策略
成熟生态
大量高级能力
```

通常会认真评估：

```
Istio
```

如果更关注：

```
简单
轻量
低运维复杂度
核心 Mesh 能力
```

可以评估：

```
Linkerd
```

实际选择还需要结合 Kubernetes 版本、团队能力和具体业务需求。

------

## 33.9 Service Mesh 的代价

这是生产环境非常重要的一部分。

Service Mesh **不是微服务系统的免费升级包**。

### 33.9.1 CPU / Memory 成本

传统 Sidecar：

```
100 Pods
+
100 Proxy
```

Proxy 本身消耗：

```
CPU
Memory
```

如果集群规模扩大：

```
1000 Pods
```

Sidecar 数量也会非常大。

------

### 33.9.2 网络复杂度增加

没有 Mesh：

```
A → B
```

有 Mesh：

```
A
 ↓
Proxy A
 ↓
Network
 ↓
Proxy B
 ↓
B
```

排查：

```
timeout
502
503
connection reset
```

时必须考虑：

```
Application
Proxy
Service
Network
Policy
Mesh Configuration
```

------

### 33.9.3 控制面复杂度

引入 Mesh 后：

```
Kubernetes
+
CNI
+
Ingress
+
Service Mesh
+
Observability
```

系统复杂度明显提高。

如果团队没有能力维护：

```
Control Plane
Certificates
Policies
Proxy
Upgrades
```

Service Mesh 反而可能增加风险。

------

### 33.9.4 调试难度

例如：

```
curl http://service-a
```

失败。

可能原因包括：

```
Application 没启动
Service 错误
Endpoint 错误
NetworkPolicy
DNS
Proxy
mTLS
Authorization Policy
Timeout
Route
```

因此排障复杂度明显增加。

------

### 33.9.5 性能开销

请求可能经过：

```
Application
 ↓
Proxy
 ↓
Proxy
 ↓
Application
```

相比直接：

```
Application
 ↓
Application
```

多了一些：

```
CPU
Memory
Network
Latency
```

虽然现代 Proxy 已经进行了大量优化，但在高吞吐场景仍然需要进行实际压测。

------

### 33.9.6 学习成本

团队需要理解：

```
Kubernetes
+
Networking
+
TLS
+
Service Mesh
+
Proxy
+
Observability
```

因此 Service Mesh 更适合已经具备一定 Kubernetes 和微服务运维能力的团队。

------

## 33.10 什么情况下不要使用 Service Mesh

这是生产环境最值得记住的一节。

### 33.10.1 微服务数量很少

例如：

```
Frontend
Backend
Database
Redis
```

这种系统通常：

```
Ingress
+
Service
+
普通 Kubernetes Networking
```

已经足够。

没有必要为了：

```
mTLS
Retry
Metrics
```

引入整个 Service Mesh。

------

### 33.10.2 单体应用

如果系统：

```
Client
  ↓
Monolith
  ↓
Database
```

Service Mesh 基本没有明显价值。

因为根本没有大量：

```
Service-to-Service
```

通信。

------

### 33.10.3 团队缺乏运维能力

如果团队连：

```
Kubernetes
Service
Ingress
DNS
NetworkPolicy
TLS
```

都还没有掌握：

不建议马上引入：

```
Service Mesh
```

否则容易变成：

```
原来的问题
+
Mesh 的问题
```

------

### 33.10.4 主要需求已经可以通过应用或基础设施解决

例如你只是需要：

```
HTTPS
```

Ingress 已经可以解决。

如果只是：

```
Metrics
```

Prometheus + Application Metrics 可以解决。

如果只是：

```
日志
```

日志系统可以解决。

如果只是：

```
简单重试
```

应用客户端或 SDK 可能已经足够。

因此不要因为：

> “Service Mesh 功能很多”

就把它全部引进来。

------

### 33.10.5 业务规模不足以抵消复杂度

一个非常实用的判断方式：

```
Service Mesh 带来的收益
        VS
Service Mesh 带来的复杂度
```

如果：

```
收益 < 复杂度
```

就不要使用。

------

## Service Mesh 在生产环境中的合理位置

可以把整个 Kubernetes 网络体系理解成：

```
                     Internet
                         │
                         ▼
                    LoadBalancer
                         │
                         ▼
                    Ingress
                         │
                         ▼
                     Service
                         │
                         ▼
                       Pod
                         │
                 ┌───────┴───────┐
                 │               │
            Application       Proxy
                 │               │
                 └───────┬───────┘
                         │
                  Service Mesh
                         │
                         ▼
                       Proxy
                         │
                         ▼
                    Application
```

其中：

```
Ingress
```

主要解决：

```
外部 → 集群
```

而：

```
Service Mesh
```

主要解决：

```
服务 → 服务
```

所以不要把：

```
Ingress
Service
Service Mesh
```

混为一谈。

------

## 本章核心知识总结

Service Mesh 最核心的思想可以浓缩成一句话：

> **把微服务之间复杂的网络通信治理能力，从业务代码中抽离出来，交给基础设施层统一处理。**

核心架构：

```
Application
     │
     ▼
Proxy
     │
     ▼
 Service Mesh
     │
     ▼
Proxy
     │
     ▼
Application
```

主要能力：

```
Service Mesh
│
├── Traffic Management
│   ├── Routing
│   ├── Retry
│   ├── Timeout
│   ├── Load Balancing
│   └── Canary
│
├── Security
│   ├── mTLS
│   ├── Identity
│   └── Authorization
│
└── Observability
    ├── Metrics
    ├── Tracing
    └── Service Graph
```

几个核心概念一定要区分：

```
Kubernetes Service
    ↓
服务发现 + 稳定访问

Ingress
    ↓
外部流量 → Kubernetes

Service Mesh
    ↓
Service → Service 通信治理

Sidecar
    ↓
Mesh 数据面的传统实现方式之一

mTLS
    ↓
服务身份认证 + 通信加密

Istio / Linkerd
    ↓
Service Mesh 实现
```

最后，生产环境不要形成这样的思维：

```
Kubernetes
 ↓
Service Mesh
 ↓
系统就更高级
```

正确的工程思维应该是：

```
业务需求
   ↓
实际问题
   ↓
是否需要 Mesh？
   ↓
收益是否超过复杂度？
   ↓
再决定是否引入
```

**Service Mesh 是解决复杂微服务通信问题的工具，而不是 Kubernetes 生产环境的必选组件。**