# 第 29 章：CI/CD

CI/CD 是把“开发人员提交代码”变成“生产环境稳定发布”的自动化流程。

在前面的章节中，我们已经分别学习了：

```
Git
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
```

本章把这些能力连接起来，形成完整的自动发布链：

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

生产环境的核心目标不是：

> “让部署自动执行。”

而是：

> **让软件能够重复、可追踪、可验证、可回滚地发布。**

------

## 29.1 CI/CD 基础

### 29.1.1 什么是 CI

CI 是 **Continuous Integration，持续集成**。

开发人员提交代码：

```
Developer
    ↓
git push
    ↓
Git Repository
    ↓
CI Pipeline
```

CI 自动执行：

```
Checkout
 ↓
Restore Dependencies
 ↓
Build
 ↓
Unit Test
 ↓
Lint / Static Analysis
 ↓
Security Scan
```

如果测试失败：

```
Test Failed
    ↓
Pipeline Failed
    ↓
禁止进入发布阶段
```

------

### 29.1.2 什么是 CD

CD 通常指：

- Continuous Delivery：持续交付
- Continuous Deployment：持续部署

持续交付：

```
Code
 ↓
Build
 ↓
Test
 ↓
Image
 ↓
Ready for Deployment
```

但生产环境可能需要人工批准：

```
Production
    ↑
Manual Approval
```

持续部署则进一步自动化：

```
Code
 ↓
Build
 ↓
Test
 ↓
Image
 ↓
Deploy Production
```

------

### 29.1.3 CI/CD 与 Kubernetes 的关系

Kubernetes 本身不是 CI/CD 系统。

它主要负责：

```
Deploy
Run
Scale
Restart
Service Discovery
Health Management
```

CI/CD 系统负责：

```
Build
Test
Package
Release
Deploy
```

因此：

```
Git
 ↓
CI/CD
 ↓
Kubernetes
```

是非常常见的生产架构。

------

### 29.1.4 一个完整 Pipeline

例如 .NET API：

```
Developer
   │
   ▼
Git Push
   │
   ▼
CI
   │
   ├── dotnet restore
   ├── dotnet build
   ├── dotnet test
   └── Security Scan
          │
          ▼
    Docker Build
          │
          ▼
    Docker Image
          │
          ▼
       Registry
          │
          ▼
   Kubernetes Deploy
          │
          ▼
    Rollout Status
          │
          ▼
    Health Check
          │
          ▼
      Release
```

------

## 29.2 GitHub Actions

### 29.2.1 什么是 GitHub Actions

GitHub Actions 是 GitHub 提供的 CI/CD 平台。

Pipeline 通常定义在：

```
.github/workflows/
```

例如：

```
.github/
└── workflows/
    └── ci.yml
```

------

### 29.2.2 最简单的 CI

例如一个 .NET 项目：

```
name: CI

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Test
        run: dotnet test --no-build
```

流程：

```
push
 ↓
GitHub Actions
 ↓
Checkout
 ↓
Restore
 ↓
Build
 ↓
Test
```

------

### 29.2.3 为什么 CI 必须先测试

错误方式：

```
Git
 ↓
Docker Build
 ↓
Production
```

如果代码本身已经有 Bug：

```
Buggy Code
 ↓
Docker Image
 ↓
Production
```

就只是：

> **更快地把错误发布到生产环境。**

正确方式：

```
Git
 ↓
Build
 ↓
Test
 ↓
Docker Build
 ↓
Registry
```

------

### 29.2.4 GitHub Actions 的 Secret

生产环境不能这样：

```
env:
  PASSWORD: "my-password"
```

应该使用：

```
GitHub Secrets
```

例如：

```
env:
  REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
  REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

但要注意：

> CI/CD Secret 和 Kubernetes Secret 是两个不同层面的 Secret。

例如：

```
GitHub Secret
    ↓
CI/CD 身份凭证

Kubernetes Secret
    ↓
运行时应用凭证
```

不要混淆。

------

## 29.3 GitLab CI

### 29.3.1 什么是 GitLab CI

GitLab CI 是 GitLab 自带的 CI/CD 能力。

配置文件通常：

```
.gitlab-ci.yml
```

例如：

```
stages:
  - build
  - test

build:
  stage: build
  image: mcr.microsoft.com/dotnet/sdk:8.0
  script:
    - dotnet restore
    - dotnet build --no-restore

test:
  stage: test
  image: mcr.microsoft.com/dotnet/sdk:8.0
  script:
    - dotnet test
```

执行：

```
Git Push
   ↓
GitLab Runner
   ↓
build
   ↓
test
```

------

### 29.3.2 GitLab Runner

GitLab CI 的任务不是 GitLab Server 自己直接执行。

通常由：

```
GitLab
   ↓
GitLab Runner
   ↓
Job
```

执行。

Runner 可以部署在：

```
VM
Container
Kubernetes
```

如果使用 Kubernetes：

```
GitLab
 ↓
Runner
 ↓
Kubernetes
 ↓
Temporary CI Pod
```

这样每个 Job 都可以运行在临时 Pod 中。

------

### 29.3.3 GitHub Actions 与 GitLab CI 怎么选

核心能力都可以完成：

```
Build
Test
Docker
Registry
Kubernetes Deploy
```

选择通常取决于：

```
代码托管平台
团队习惯
企业内部基础设施
权限体系
Runner 管理方式
```

没有必要为了学习 Kubernetes 而同时在生产环境使用多个 CI 平台。

------

## 29.4 Jenkins

### 29.4.1 什么是 Jenkins

Jenkins 是经典 CI/CD 自动化平台。

典型结构：

```
Git
 ↓
Jenkins
 ↓
Pipeline
 ↓
Build / Test
 ↓
Docker
 ↓
Registry
 ↓
Kubernetes
```

Jenkins Pipeline 通常使用：

```
Jenkinsfile
```

例如：

```
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'dotnet build'
            }
        }

        stage('Test') {
            steps {
                sh 'dotnet test'
            }
        }
    }
}
```

------

### 29.4.2 Jenkins 的优势

Jenkins 最大的优势之一是：

> **高度可扩展。**

大量 Plugin 可以集成：

```
Git
Docker
Kubernetes
Cloud
通知系统
安全扫描
制品仓库
```

------

### 29.4.3 Jenkins 的问题

Jenkins 灵活，但维护成本也比较高。

需要考虑：

```
Jenkins Controller
Jenkins Agent
Plugin
Plugin Compatibility
Credential
Backup
Upgrade
Security
```

因此生产环境选择 Jenkins 时，不能只考虑：

> “它能不能跑 Pipeline？”

还需要考虑：

> “谁负责长期维护 Jenkins 本身？”

------

## 29.5 Docker Build

CI 通过测试后，就需要创建 Image。

例如：

```
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY . .

RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Pipeline：

```
Source Code
    ↓
Docker Build
    ↓
Image
```

------

### 29.5.1 Image Tag 不应该使用 latest

不推荐：

```
docker build -t myapi:latest .
```

生产环境推荐：

```
docker build -t registry.example.com/myapi:1.2.3 .
```

更进一步可以使用：

```
Git Commit SHA
```

例如：

```
docker build \
  -t registry.example.com/myapi:a8f31c2 \
  .
```

这样可以建立：

```
Git Commit
     ↓
Image Tag
     ↓
Kubernetes Deployment
```

之间的明确对应关系。

------

### 29.5.2 推荐的 Image 标识

例如：

```
myapi:1.2.3
```

同时记录：

```
Git Commit:
a8f31c2
```

最终：

```
Git
a8f31c2
   ↓
Image
myapi:1.2.3
   ↓
Digest
sha256:xxxx
```

这样出现问题时可以准确知道：

> 生产环境运行的是哪一次代码。

------

## 29.6 Registry

### 29.6.1 CI 为什么需要 Registry

CI 创建 Image 后：

```
CI Runner
    ↓
Docker Image
```

Kubernetes Worker Node 不一定能直接访问 CI Runner 上的 Image。

所以：

```
CI
 ↓
Registry
 ↓
Kubernetes
```

例如：

```
docker push registry.example.com/myapi:1.2.3
```

Kubernetes：

```
image: registry.example.com/myapi:1.2.3
```

------

### 29.6.2 私有 Registry

生产环境通常使用：

```
Harbor
ECR
ACR
GCR / Artifact Registry
```

或者其他企业级 Registry。

重点不是 Registry 的名字，而是它应该具备：

```
Authentication
Authorization
TLS
Image Retention
Vulnerability Scanning
Audit
High Availability
Backup
```

------

### 29.6.3 ImagePullSecrets

如果 Registry 是私有的，Kubernetes 需要凭证。

例如：

```
kubectl create secret docker-registry registry-secret \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword' \
  -n production
```

然后：

```
spec:
  template:
    spec:
      imagePullSecrets:
        - name: registry-secret
```

生产环境不要把 Registry 密码直接写进 Git。

------

## 29.7 Kubernetes Deployment

### 29.7.1 CI 如何部署 Kubernetes

最简单的方法是：

```
CI
 ↓
kubectl
 ↓
Kubernetes API Server
```

例如：

```
kubectl -n production set image deployment/myapi \
  myapi=registry.example.com/myapi:1.2.3
```

然后：

```
kubectl -n production rollout status deployment/myapi
```

如果成功：

```
Deployment
    ↓
ReplicaSet
    ↓
Pods
    ↓
New Image
```

------

### 29.7.2 为什么不推荐 CI 大量执行 kubectl create

例如：

```
kubectl create deployment ...
kubectl expose ...
kubectl create configmap ...
```

这种方式适合学习和简单操作，但复杂生产环境会变得难以维护。

更推荐：

```
Git
 ↓
Kubernetes Manifest / Helm
 ↓
Review
 ↓
CI/CD
 ↓
Kubernetes
```

这样：

```
Infrastructure as Code
```

可以被版本控制。

------

### 29.7.3 使用 YAML

例如：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi
  namespace: production

spec:
  replicas: 3

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
          image: registry.example.com/myapi:1.2.3

          ports:
            - containerPort: 8080

          readinessProbe:
            httpGet:
              path: /health
              port: 8080

          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1
              memory: 512Mi
```

然后：

```
kubectl apply -f deployment.yaml
```

------

## 29.8 自动更新 Image

### 29.8.1 最简单的自动更新

Pipeline：

```
Git Push
 ↓
Build
 ↓
Test
 ↓
Docker Build
 ↓
Push Registry
 ↓
kubectl set image
 ↓
Deployment
```

例如：

```
IMAGE_TAG="${GITHUB_SHA}"

docker build \
  -t registry.example.com/myapi:${IMAGE_TAG} \
  .

docker push \
  registry.example.com/myapi:${IMAGE_TAG}

kubectl -n production set image deployment/myapi \
  myapi=registry.example.com/myapi:${IMAGE_TAG}
```

------

### 29.8.2 为什么使用 Commit SHA 很有价值

假设：

```
Production
 ↓
myapi:a8f31c2
```

发生故障。

可以立即找到：

```
Git Commit
    ↓
Source Code
    ↓
CI Job
    ↓
Docker Image
    ↓
Kubernetes Release
```

形成完整的：

> **Traceability，可追溯性。**

------

### 29.8.3 GitOps 模式

生产环境还可以采用 GitOps。

架构：

```
Application Repository
        │
        ▼
       CI
        │
        ▼
     Build Image
        │
        ▼
     Registry
        │
        ▼
Manifest Repository
        │
        ▼
       Git
        │
        ▼
Argo CD / Flux
        │
        ▼
   Kubernetes
```

这里有一个重要区别：

传统模式：

```
CI → Kubernetes
```

GitOps：

```
CI → Git
        ↓
GitOps Controller → Kubernetes
```

这样 Kubernetes 的期望状态由 Git 管理。

------

## 29.9 环境管理

生产环境通常至少有：

```
Development
Testing
Staging
Production
```

不应该让所有环境直接使用：

```
同一个 Config
同一个 Secret
同一个 Database
同一个 Image Tag
```

------

### 29.9.1 环境配置

例如：

```
dev
 ├── replicas: 1
 ├── database: dev-db
 └── log level: debug

staging
 ├── replicas: 2
 ├── database: staging-db
 └── log level: info

production
 ├── replicas: 5
 ├── database: production-db
 └── log level: warn
```

------

### 29.9.2 Helm 管理环境

前面已经学习 Helm，可以使用：

```
values-dev.yaml
values-staging.yaml
values-prod.yaml
```

例如：

```
# values-prod.yaml

replicaCount: 5

image:
  repository: registry.example.com/myapi
  tag: "1.2.3"

resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

部署：

```
helm upgrade --install myapi ./chart \
  -n production \
  --create-namespace \
  -f values-prod.yaml
```

这样：

```
同一个 Chart
       │
 ┌─────┼────────┐
 ▼     ▼        ▼
 dev  staging  production
```

通过不同 Values 管理环境差异。

------

### 29.9.3 Secret 不应该进入普通 Values

不要：

```
database:
  password: "SuperSecret123"
```

直接提交到 Git。

应该使用：

```
External Secrets
Vault
Cloud Secret Manager
Sealed Secrets
```

等方案。

核心原则：

> **配置可以版本化，敏感凭证应该采用专门的 Secret 管理机制。**

------

## 29.10 发布失败处理

这是生产 CI/CD 最重要的部分之一。

完整 Pipeline 不能只有：

```
Deploy
 ↓
Done
```

而应该：

```
Deploy
 ↓
Rollout Status
 ↓
Health Check
 ↓
Success?
 ├── Yes → Release
 └── No
       ↓
    Rollback
```

------

### 29.10.1 Kubernetes Rollout 检查

部署：

```
kubectl -n production set image deployment/myapi \
  myapi=registry.example.com/myapi:1.2.3
```

检查：

```
kubectl -n production rollout status deployment/myapi \
  --timeout=5m
```

如果失败：

```
kubectl -n production rollout history deployment/myapi
```

查看：

```
kubectl -n production get pods
```

进一步：

```
kubectl -n production describe deployment myapi
```

以及：

```
kubectl -n production logs deployment/myapi
```

------

### 29.10.2 自动 Rollback

例如：

```
if ! kubectl -n production rollout status deployment/myapi --timeout=5m; then
  kubectl -n production rollout undo deployment/myapi
  exit 1
fi
```

流程：

```
Deploy v1.2.3
      ↓
Rollout
      ↓
Health Check
      ↓
失败
      ↓
Rollback
      ↓
v1.2.2
```

但需要注意：

> **Rollback Deployment ≠ 数据库自动回滚。**

例如：

```
Application v2
 ↓
Database Migration
 ↓
Schema changed
```

此时简单：

```
kubectl rollout undo
```

不一定能够恢复业务。

因此生产发布必须特别考虑：

```
Database Migration
Backward Compatibility
API Compatibility
Data Migration
```

------

### 29.10.3 更可靠的发布策略

对于重要生产系统，不建议：

```
直接替换全部 Pod
```

可以采用：

```
Rolling Update
```

或者进一步：

```
Blue-Green
Canary
Progressive Delivery
```

例如 Canary：

```
                     ┌── 95% → v1
User → Ingress ──────┤
                     └── 5%  → v2
```

观察：

```
Error Rate
Latency
CPU
Memory
Business Metrics
```

如果正常：

```
5%
 ↓
25%
 ↓
50%
 ↓
100%
```

如果异常：

```
v2
 ↓
停止发布
 ↓
流量回到 v1
```

这才是真正面向生产的发布体系。

------

## 完整 CI/CD Pipeline

把本章所有知识串起来：

```
                    Developer
                        │
                        ▼
                       Git
                        │
                        ▼
                 CI/CD Platform
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
           Build                  Test
             │                     │
             └──────────┬──────────┘
                        ▼
                  Docker Build
                        │
                        ▼
                     Image
                        │
                        ▼
                    Registry
                        │
                        ▼
              Kubernetes Deployment
                        │
                        ▼
                  Rolling Update
                        │
                        ▼
                 Readiness Probe
                        │
                        ▼
                  Health Check
                        │
                 ┌──────┴──────┐
                 ▼             ▼
               PASS          FAIL
                 │             │
                 ▼             ▼
              Release       Rollback
```

生产环境进一步可以演进为：

```
Developer
   ↓
Git
   ↓
CI
   ├── Build
   ├── Unit Test
   ├── Security Scan
   └── Docker Build
          ↓
      Registry
          ↓
    Image Version
          ↓
    GitOps Repository
          ↓
   Argo CD / Flux
          ↓
     Kubernetes
          ↓
 Rolling / Canary
          ↓
 Health Check
          ↓
 Metrics + Logs
          ↓
   Release / Rollback
```

------

## 本章核心实践原则

生产 CI/CD 最终应该做到以下几点：

| 能力         | 目标                   |
| ------------ | ---------------------- |
| Build        | 代码可以稳定构建       |
| Test         | 测试失败不能进入发布   |
| Image        | 每个版本都有唯一标识   |
| Registry     | Image 可追踪、可获取   |
| Deploy       | 发布过程自动化         |
| Health Check | 发布后自动验证         |
| Rollback     | 发布失败能够快速恢复   |
| Environment  | Dev / Test / Prod 隔离 |
| Secret       | 凭证不进入 Git         |
| Audit        | 能追踪谁发布了什么     |
| GitOps       | 生产期望状态可版本化   |

最终，一个成熟的 Kubernetes 发布流程应该回答清楚 **5 个问题**：

```
1. 我发布了什么？
        ↓
   Image / Commit

2. 谁发布的？
        ↓
   CI/CD / Git Audit

3. 发布到哪里？
        ↓
   Environment / Cluster / Namespace

4. 发布成功了吗？
        ↓
   Rollout / Probe / Metrics

5. 失败怎么办？
        ↓
   Rollback / Recovery
```

做到这一步，CI/CD 才不只是“自动执行 kubectl”，而是真正成为 Kubernetes 生产运维体系的一部分。

# 第 30 章：GitOps 与 Argo CD

在上一章中，我们建立了传统 CI/CD 发布链：

```
Git
 ↓
CI
 ↓
Build
 ↓
Test
 ↓
Docker Image
 ↓
Registry
 ↓
kubectl / Helm
 ↓
Kubernetes
```

这种方式可以正常工作，但存在一个核心问题：

> **CI/CD 系统直接修改 Kubernetes 集群状态。**

例如：

```
kubectl set image deployment/myapi \
  myapi=registry.example.com/myapi:a8f31c2
```

那么 Kubernetes 当前到底应该运行什么版本，需要去查看：

```
Kubernetes Cluster
```

而不是 Git。

GitOps 就是解决这个问题的另一种生产级思路：

```
Application Code
       ↓
      CI
       ↓
 Docker Image
       ↓
   Registry

        │
        ▼

 GitOps Repository
        │
        ▼
      Argo CD
        │
        ▼
   Kubernetes
```

核心思想是：

> **Git 保存 Kubernetes 的期望状态，Argo CD 持续让集群实际状态向 Git 中的期望状态靠拢。**

------

## 30.1 GitOps 是什么

### 30.1.1 什么是 GitOps

GitOps 可以理解成：

> **使用 Git 作为系统声明式配置和部署状态的主要事实来源（Source of Truth），由自动化控制器把 Git 中定义的状态同步到 Kubernetes。**

传统 CI/CD：

```
CI
 │
 ├── Build
 ├── Test
 └── Deploy
        │
        ▼
   Kubernetes
```

GitOps：

```
CI
 │
 ├── Build
 ├── Test
 └── Push Image
        │
        ▼
    Registry

GitOps Repository
        │
        ▼
      Argo CD
        │
        ▼
   Kubernetes
```

最大的区别是：

```
传统：
CI → 修改 Kubernetes

GitOps：
Git → Argo CD → Kubernetes
```

------

### 30.1.2 GitOps 的核心原则

生产环境中可以把 GitOps 理解为几个关键原则：

```
Declarative
      ↓
声明式配置

Versioned
      ↓
Git 版本控制

Auditable
      ↓
可以审计

Automated
      ↓
自动同步

Reconciliation
      ↓
持续纠正实际状态
```

因此 GitOps 并不是：

> “把 YAML 放进 Git。”

而是：

> **让 Git 中的声明式状态成为 Kubernetes 持续部署和状态管理的依据。**

------

### 30.1.3 GitOps 与 CI/CD 的关系

GitOps **不是 CI/CD 的替代品**。

更准确的关系是：

```
CI
│
├── Build
├── Test
├── Security Scan
└── Docker Image
       ↓
    Registry

CD / GitOps
       ↓
GitOps Repository
       ↓
    Argo CD
       ↓
 Kubernetes
```

CI 主要解决：

> **代码能不能变成一个可靠的软件制品？**

GitOps/CD 主要解决：

> **这个制品应该部署到什么环境？集群当前状态是否符合期望？**

------

## 30.2 为什么需要 GitOps

### 30.2.1 传统 CI/CD 的问题

假设生产环境有：

```
production
 ├── API
 ├── Frontend
 ├── Redis
 └── PostgreSQL
```

CI 直接执行：

```
kubectl set image ...
kubectl apply ...
helm upgrade ...
```

久而久之可能出现：

```
Git
 ↓
不知道生产到底是什么状态

CI
 ↓
某次 Pipeline 修改了集群

Kubernetes
 ↓
又有人手工 kubectl 修改
```

最终：

```
Git 状态 ≠ Kubernetes 状态
```

这会造成很严重的问题。

------

### 30.2.2 GitOps 解决状态来源问题

GitOps：

```
Git
 │
 │ Desired State
 ▼
Argo CD
 │
 │ Reconciliation
 ▼
Kubernetes
 │
 │ Actual State
 ▼
比较
```

如果：

```
Desired State ≠ Actual State
```

Argo CD 就会发现：

```
OutOfSync
```

然后根据配置执行同步。

------

### 30.2.3 GitOps 带来的好处

#### 可追踪

例如：

```
Git Commit:
a8f31c2
```

修改：

```
image:
  tag: "1.2.3"
```

可以知道：

```
谁修改
什么时候修改
修改了什么
为什么修改
```

------

#### 可审计

生产环境不再主要依赖：

```
kubectl edit
```

而是：

```
Pull Request
 ↓
Code Review
 ↓
Merge
 ↓
Argo CD
 ↓
Production
```

这非常适合企业生产环境。

------

#### 自动纠偏

例如某人手动执行：

```
kubectl scale deployment myapi --replicas=1
```

但 Git 中定义：

```
replicas: 3
```

Argo CD 可以发现：

```
Git:
replicas = 3

Cluster:
replicas = 1
```

形成：

```
OutOfSync
```

根据 Sync Policy，可以自动恢复到：

```
replicas = 3
```

这就是 GitOps 最重要的能力之一：

> **Reconciliation，持续协调。**

------

## 30.3 Desired State

### 30.3.1 什么是 Desired State

Kubernetes 本身就是声明式系统。

例如：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapi

spec:
  replicas: 3
```

这表示：

> 我希望 `myapi` 有 3 个副本。

这就是：

```
Desired State
期望状态
```

而实际集群可能是：

```
Actual State
实际状态
```

例如：

```
Desired:
replicas = 3

Actual:
replicas = 2
```

Kubernetes Controller 会持续尝试：

```
Actual → Desired
```

------

### 30.3.2 GitOps 的 Desired State

GitOps 把这个概念进一步扩展：

```
Git
 ↓
Desired State
```

例如 Git 中：

```
replicas: 3

image:
  repository: registry.example.com/myapi
  tag: "1.2.3"
```

表示：

```
生产环境应该：

3 replicas
Image = 1.2.3
```

Argo CD 的任务就是：

```
Git Desired State
        ↓
      Argo CD
        ↓
Kubernetes Actual State
```

------

### 30.3.3 Desired State 与 Actual State

可以把 Argo CD 理解为一个持续运行的对比器：

```
              Git
               │
               ▼
        Desired State
               │
               │
               ▼
           Argo CD
               │
               │ compare
               ▼
        Actual State
               │
               ▼
        Kubernetes
```

如果：

```
Desired == Actual
```

则：

```
Synced
```

如果：

```
Desired != Actual
```

则：

```
OutOfSync
```

------

## 30.4 Argo CD

### 30.4.1 Argo CD 是什么

Argo CD 是一个面向 Kubernetes 的 GitOps Continuous Delivery 工具。

它的核心职责：

```
Git Repository
      ↓
   Argo CD
      ↓
Kubernetes
```

Argo CD 会读取 Git 中的 Kubernetes 配置，然后将其应用到 Kubernetes 集群。

------

### 30.4.2 Argo CD 的核心架构

可以简化成：

```
                  Git Repository
                        │
                        ▼
                 ┌─────────────┐
                 │   Argo CD   │
                 └─────────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Dev Cluster  Staging Cluster  Prod Cluster
```

Argo CD 本身运行在 Kubernetes 中：

```
Kubernetes
└── argocd namespace
    ├── API Server
    ├── Repo Server
    ├── Application Controller
    └── Redis
```

不同版本的 Argo CD 组件可能有所变化，因此生产环境应以实际版本的官方架构文档为准。

------

### 30.4.3 Argo CD Application Controller

其中最重要的是：

```
Application Controller
```

它负责：

```
读取 Application
        ↓
读取 Git
        ↓
生成 Desired State
        ↓
比较 Kubernetes
        ↓
发现 Drift
        ↓
Sync
```

可以把它理解成：

> **GitOps 世界中的 Kubernetes Controller。**

------

## 30.5 Git Repository

### 30.5.1 Git Repository 保存什么

GitOps Repository 通常保存：

```
Kubernetes YAML
Helm Chart
Kustomize
Environment Configuration
```

例如：

```
gitops/
├── apps/
│   └── myapi/
│       ├── deployment.yaml
│       └── service.yaml
│
└── environments/
    ├── dev/
    ├── staging/
    └── production/
```

------

### 30.5.2 Application Code 与 GitOps Repository

生产环境推荐将：

```
Application Source
```

和：

```
Kubernetes Deployment Configuration
```

逻辑分离。

例如：

```
myapi/
├── src/
├── tests/
├── Dockerfile
└── ...

gitops/
├── dev/
├── staging/
└── production/
```

流程：

```
Application Repository
        ↓
       CI
        ↓
Docker Image
        ↓
Registry
        ↓
GitOps Repository
        ↓
     Argo CD
        ↓
   Kubernetes
```

这样职责更加清晰。

------

### 30.5.3 GitOps Repository 中不要保存明文 Secret

不要：

```
stringData:
  password: "production-password"
```

直接提交 Git。

即使 Repository 是 Private，也不应该把生产凭证当普通配置管理。

可以使用：

```
External Secrets
Vault
Cloud Secret Manager
Sealed Secrets
SOPS
```

等方案。

------

## 30.6 Application

### 30.6.1 Argo CD Application 是什么

Argo CD 中最重要的资源之一：

```
Application
```

它描述：

> **某个 Git Repository 中的配置，应该部署到哪个 Kubernetes Cluster 的哪个 Namespace。**

可以理解为：

```
Application
=
Source
+
Destination
+
Sync Policy
```

------

### 30.6.2 Application 示例

例如：

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: myapi-production
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/example/gitops.git
    targetRevision: main
    path: environments/production/myapi

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

这里最重要的几个字段：

```
repoURL
```

Git Repository。

```
targetRevision
```

Git 分支、Tag 或 Commit。

```
path
```

配置所在目录。

```
destination
```

部署目标 Cluster。

```
namespace
```

目标 Namespace。

------

### 30.6.3 Application 的逻辑

例如：

```
Git:
https://github.com/example/gitops.git

Path:
environments/production/myapi

        ↓

Argo CD Application

        ↓

Cluster:
production-cluster

Namespace:
production
```

因此 Application 就建立了：

```
Git
 ↓
Kubernetes
```

之间的映射关系。

------

## 30.7 Sync

### 30.7.1 什么是 Sync

Sync 就是：

> **让 Kubernetes 实际状态与 Git Desired State 对齐。**

例如 Git：

```
replicas: 3
```

Cluster：

```
replicas: 1
```

Argo CD：

```
OutOfSync
```

执行 Sync：

```
Git
replicas=3
 ↓
Argo CD
 ↓
Kubernetes
replicas=3
```

最终：

```
Synced
```

------

### 30.7.2 手动 Sync

如果使用手动同步，可以通过 Argo CD CLI：

```
argocd app sync myapi-production
```

查看：

```
argocd app get myapi-production
```

或者使用 Web UI 查看 Application 状态。

------

### 30.7.3 Sync 的本质

Sync 并不是：

```
Git → 直接复制 YAML
```

而是：

```
Git
 ↓
Render
 ↓
Desired Kubernetes Resources
 ↓
Compare
 ↓
Apply
 ↓
Kubernetes
```

如果使用 Helm：

```
Git
 ↓
Helm Template
 ↓
Rendered YAML
 ↓
Argo CD
 ↓
Kubernetes
```

------

## 30.8 Auto Sync

### 30.8.1 什么是 Auto Sync

手动 Sync：

```
Git Change
    ↓
Argo CD
    ↓
OutOfSync
    ↓
管理员点击 Sync
    ↓
Kubernetes
```

Auto Sync：

```
Git Change
    ↓
Argo CD
    ↓
OutOfSync
    ↓
自动 Sync
    ↓
Kubernetes
```

生产环境非常常见。

------

### 30.8.2 开启 Auto Sync

例如：

```
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

其中：

```
automated
```

开启自动同步。

```
prune
```

允许删除 Git 中已经不存在的资源。

```
selfHeal
```

发现集群发生 Drift 时自动纠正。

------

### 30.8.3 Prune 的风险

例如 Git 原来：

```
Deployment
Service
ConfigMap
```

后来删除了：

```
ConfigMap
```

如果：

```
prune: true
```

Argo CD 可能删除 Kubernetes 中对应的 ConfigMap。

因此生产环境开启 Prune 前必须非常谨慎。

尤其需要注意：

```
数据库
PersistentVolume
PersistentVolumeClaim
CRD
```

等具有潜在破坏性的资源。

------

### 30.8.4 Self-Heal

假设 Git：

```
replicas: 3
```

有人手工：

```
kubectl scale deployment myapi --replicas=1
```

产生：

```
Git:
3

Kubernetes:
1
```

Argo CD 检测到：

```
Drift
```

开启：

```
selfHeal: true
```

后：

```
Kubernetes 1
     ↓
Argo CD
     ↓
发现 Drift
     ↓
恢复
     ↓
Kubernetes 3
```

这就是 GitOps 的自动纠偏。

------

## 30.9 Rollback

### 30.9.1 GitOps 中的 Rollback

传统 Kubernetes：

```
kubectl rollout undo deployment/myapi
```

但 GitOps 环境需要特别注意：

> **不要只修改 Kubernetes，而忽略 Git。**

假设：

```
Git:
v1.2.3

Cluster:
v1.2.2
```

如果你直接：

```
kubectl rollout undo
```

可能出现：

```
Git Desired State = v1.2.3
Cluster Actual State = v1.2.2
```

Argo CD 发现 Drift 后，可能再次把它恢复成：

```
v1.2.3
```

------

### 30.9.2 GitOps 正确 Rollback 思路

例如：

```
Git
v1.2.3
```

出现问题。

将 GitOps Repository 修改为：

```
v1.2.2
```

然后：

```
Git Commit
    ↓
Argo CD
    ↓
Sync
    ↓
Kubernetes v1.2.2
```

这样：

```
Git Desired State
        =
Kubernetes Actual State
```

------

### 30.9.3 为什么 Git Rollback 更可靠

因为：

```
Git
```

保存了：

```
v1.2.1
v1.2.2
v1.2.3
```

可以：

```
git log
```

查看历史。

然后：

```
Git Revert
```

恢复配置。

这样整个变更过程：

```
可追踪
可审计
可 Review
可复现
```

这比直接登录生产服务器执行：

```
kubectl edit
```

更加可靠。

------

## 30.10 多环境管理

生产环境通常有：

```
Development
Staging
Production
```

GitOps 可以采用：

```
Git Repository
│
├── dev
├── staging
└── production
```

例如：

```
gitops/
├── apps/
│   └── myapi/
│
└── environments/
    ├── dev/
    │   └── myapi/
    ├── staging/
    │   └── myapi/
    └── production/
        └── myapi/
```

------

### 30.10.1 多 Cluster

如果：

```
Dev Cluster
Staging Cluster
Production Cluster
```

可以：

```
                 Argo CD
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
      Dev        Staging       Prod
    Cluster      Cluster      Cluster
```

每个 Application 指向不同：

```
destination
```

------

### 30.10.2 Promotion

生产发布不要简单理解成：

```
开发完成
 ↓
直接 Production
```

更合理：

```
Dev
 ↓
Test
 ↓
Staging
 ↓
Validation
 ↓
Production
```

GitOps 中可以表现为：

```
dev:
1.2.3
 ↓
staging:
1.2.3
 ↓
production:
1.2.3
```

每一步都可以通过：

```
Pull Request
Code Review
Automated Test
Approval
```

控制。

------

## 30.11 Helm + Argo CD

### 30.11.1 为什么 Helm 与 Argo CD 可以一起使用

Helm 负责：

```
Package
Template
Parameterization
```

Argo CD 负责：

```
GitOps
Sync
Reconciliation
Deployment
```

两者职责不同：

```
Helm
 ↓
生成 Kubernetes Manifest

Argo CD
 ↓
管理这些 Manifest 的部署状态
```

------

### 30.11.2 Git Repository

例如：

```
gitops/
└── myapi/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        └── service.yaml
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
    path: myapi

    helm:
      valueFiles:
        - values-production.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

------

### 30.11.3 Helm + Argo CD 的工作过程

```
Git
 │
 │ Chart + Values
 ▼
Argo CD
 │
 │ Helm Template
 ▼
Kubernetes Manifest
 │
 ▼
Compare
 │
 ▼
Sync
 │
 ▼
Kubernetes
```

因此：

> **Argo CD 不等于 Helm。**

Helm 是应用打包和模板工具。

Argo CD 是 GitOps CD 控制器。

------

## 30.12 Kustomize + Argo CD

### 30.12.1 Kustomize 是什么

Kustomize 是 Kubernetes 原生生态中的配置定制工具。

它的特点之一是：

> **不需要使用模板语言，也可以针对不同环境修改 Kubernetes YAML。**

例如：

```
base/
├── deployment.yaml
├── service.yaml
└── kustomization.yaml

overlays/
├── dev/
│   └── kustomization.yaml
├── staging/
│   └── kustomization.yaml
└── production/
    └── kustomization.yaml
```

------

### 30.12.2 Base

例如：

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
          image: registry.example.com/myapi:1.2.3
```

`base/kustomization.yaml`：

```
resources:
  - deployment.yaml
  - service.yaml
```

------

### 30.12.3 Production Overlay

例如：

```
overlays/
└── production/
    ├── kustomization.yaml
    └── replica-patch.yaml
```

可以通过 Patch：

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
resources:
  - ../../base

patches:
  - path: replica-patch.yaml
```

最终：

```
Base:
replicas = 2

Production:
replicas = 5
```

------

### 30.12.4 Argo CD + Kustomize

Application：

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
    path: overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Argo CD 会处理：

```
Git
 ↓
Kustomize
 ↓
Rendered Manifest
 ↓
Argo CD
 ↓
Kubernetes
```

------

### 30.12.5 Helm 与 Kustomize 怎么选择

两者都能用于多环境配置，但思路不同。

| 项目              | Helm     | Kustomize       |
| ----------------- | -------- | --------------- |
| 核心思想          | Template | Overlay / Patch |
| 参数化            | 很强     | 相对简单        |
| Package           | 强       | 弱              |
| 第三方应用        | 很常见   | 也可以          |
| Kubernetes 原生感 | 较低     | 较高            |
| 学习成本          | 中       | 较低            |
| 适合复杂应用      | 很适合   | 适合            |
| Argo CD 支持      | 支持     | 支持            |

实际生产中也经常组合：

```
Helm
 +
Kustomize
 +
Argo CD
```

但不要为了“技术完整”而强行叠加工具。

------

## GitOps 完整生产架构

把第 29 章和本章连接起来：

```
                 Developer
                     │
                     ▼
                    Git
                     │
                     ▼
              ┌─────────────┐
              │     CI      │
              └─────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Build         Test      Security Scan
        │
        ▼
    Docker Image
        │
        ▼
     Registry
        │
        │
        ▼
 ┌─────────────────┐
 │ GitOps Repository│
 └─────────────────┘
        │
        ▼
    ┌────────┐
    │Argo CD │
    └────────┘
        │
        ▼
 ┌───────────────────┐
 │ Desired State     │
 │        ↓          │
 │ Compare / Sync    │
 │        ↓          │
 │ Actual State      │
 └───────────────────┘
        │
        ▼
   Kubernetes
        │
        ├── Deployment
        ├── Service
        ├── Ingress
        ├── ConfigMap
        └── Secret
```

最终形成：

```
Code
 ↓
CI
 ↓
Test
 ↓
Image
 ↓
Registry
 ↓
GitOps Repository
 ↓
Argo CD
 ↓
Kubernetes
 ↓
Health Check
 ↓
Metrics / Logs
```

这套模式的关键变化只有一个，但非常重要：

```
传统 CI/CD：

CI ───────────────→ Kubernetes


GitOps：

CI → GitOps Repository
              ↓
           Argo CD
              ↓
         Kubernetes
```

**CI 负责产生可靠的软件制品，GitOps Repository 负责描述应该部署什么，Argo CD 负责持续让 Kubernetes 达到这个状态。**

这样，生产 Kubernetes 集群就不再主要依赖人工 `kubectl` 操作，而是形成了一个**声明式、可审计、可回滚、可持续纠偏**的部署体系。