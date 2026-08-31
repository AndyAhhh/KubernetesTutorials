# 第 26 章：Helm

前面的章节中，我们一直使用 Kubernetes 原生 YAML 管理应用。

例如一个生产 API 服务可能需要：

```
Deployment
Service
Ingress
ConfigMap
Secret
ServiceAccount
HPA
NetworkPolicy
```

手动维护这些 YAML 在学习阶段完全没问题，但到了生产环境，通常会遇到：

```
同一个应用
   │
   ├── Development
   ├── Testing
   └── Production
```

它们大部分 Kubernetes 配置相同，只是：

```
Image
Replicas
CPU / Memory
Domain
Database
Environment Variables
Ingress
```

存在差异。

如果复制三套甚至几十套 YAML：

```
dev/
test/
staging/
prod/
```

维护成本会迅速增加。

**Helm 的核心价值，就是把 Kubernetes 应用变成一个可以安装、升级、回滚、配置和版本管理的软件包。**

------

## 26.1 Helm 是什么

### 26.1.1 Helm 的定义

Helm 是 Kubernetes 的：

> **Package Manager（包管理器）**

可以类比：

```
Linux
 └── apt / yum

Node.js
 └── npm

Python
 └── pip

Kubernetes
 └── Helm
```

Helm 管理的 Kubernetes 应用包叫：

> **Chart**

例如：

```
my-api
 ├── Deployment
 ├── Service
 ├── ConfigMap
 └── Ingress
```

可以打包成：

```
my-api-1.0.0.tgz
```

然后通过 Helm 安装：

```
helm install my-api ./my-api
```

------

### 26.1.2 Helm 解决什么问题

Helm 主要解决：

```
YAML 管理
应用打包
配置管理
版本管理
安装
升级
回滚
卸载
```

最终可以把：

```
很多 Kubernetes YAML
```

变成：

```
一个 Chart
+
不同 values
```

------

## 26.2 为什么需要 Helm

假设一个 API 有：

```
Deployment
Service
Ingress
ConfigMap
HPA
ServiceAccount
NetworkPolicy
```

开发环境：

```
replicas: 1
image: api:1.0.0
domain: api-dev.example.com
```

生产环境：

```
replicas: 5
image: api:3.2.1
domain: api.example.com
```

如果不用 Helm，可能需要维护：

```
deployment-dev.yaml
deployment-prod.yaml

service-dev.yaml
service-prod.yaml

ingress-dev.yaml
ingress-prod.yaml
```

大量重复。

Helm 可以变成：

```
Chart
 │
 ├── templates/
 │
 └── values.yaml
```

然后：

```
values-dev.yaml
values-prod.yaml
```

------

### 26.2.1 Helm 的核心模型

可以先记住：

```
Chart
  +
Values
  +
Templates
  │
  ▼
Helm Render
  │
  ▼
Kubernetes YAML
  │
  ▼
Kubernetes API Server
```

也就是说：

> **Helm 并没有替代 Kubernetes。**

它主要负责：

> **生成和管理 Kubernetes 应用资源。**

------

## 26.3 Chart

### 26.3.1 Chart 是什么

Chart 是 Helm 的：

> **应用打包格式**

一个 Chart 可以包含：

```
Deployment
Service
Ingress
ConfigMap
Secret
RBAC
HPA
PVC
```

例如：

```
my-api/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

------

### 26.3.2 Chart 可以理解成应用模板

例如：

```
my-api Chart
```

不是：

```
只代表某一个生产环境
```

而是：

```
应用模板
```

可以通过不同 Values 生成：

```
Development
Testing
Production
```

不同的 Kubernetes Manifest。

------

## 26.4 Release

### 26.4.1 Release 是什么

这是 Helm 非常重要的概念。

如果：

```
helm install my-api ./my-api
```

那么：

```
my-api
```

就是一个：

> **Release Name**

可以理解为：

> **Chart 的一次安装实例。**

例如：

```
Chart:
my-api
```

可以安装多次：

```
Release 1:
my-api-dev

Release 2:
my-api-test

Release 3:
my-api-prod
```

它们都来自：

```
my-api Chart
```

但是是不同 Release。

------

### 26.4.2 Chart 和 Release 的区别

非常重要：

```
Chart
= 应用模板 / 安装包

Release
= Chart 的一次实际安装
```

可以类比：

```
Docker Image
    ↓
Container
```

虽然不是完全等价，但有助于初学者理解：

```
Chart
    ↓
Release
```

------

### 26.4.3 查看 Release

```
helm list
```

指定 Namespace：

```
helm list -n shop
```

查看所有 Namespace：

```
helm list -A
```

------

## 26.5 Repository

### 26.5.1 Repository 是什么

Helm Repository 是：

> **存放 Chart 的仓库。**

类似：

```
Docker
 └── Registry
       └── Image

Helm
 └── Repository
       └── Chart
```

例如：

```
Repository
   │
   ├── nginx
   ├── redis
   ├── prometheus
   └── grafana
```

------

### 26.5.2 添加 Repository

例如使用 Bitnami Chart：

[Bitnami Helm Charts](https://github.com/bitnami/charts?utm_source=chatgpt.com)

添加：

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

更新索引：

```
helm repo update
```

查看：

```
helm search repo bitnami
```

搜索 Redis：

```
helm search repo bitnami/redis
```

------

### 26.5.3 生产环境 Repository

生产环境不一定直接使用公共 Repository。

企业通常会使用：

```
开发
 ↓
内部 Chart
 ↓
CI/CD
 ↓
私有 Helm Repository
 ↓
生产 Kubernetes
```

也可以使用支持 OCI 的 Registry 存储 Helm Chart。

核心目标是：

> **生产部署使用经过审核和版本控制的 Chart。**

------

## 26.6 Chart 目录结构

### 26.6.1 创建 Chart

Helm 可以直接生成一个 Chart：

```
helm create my-api
```

查看：

```
tree my-api
```

典型结构：

```
my-api/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl
│   └── tests/
└── .helmignore
```

------

### 26.6.2 Chart.yaml

例如：

```
apiVersion: v2
name: my-api
description: My API Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
```

几个重要字段：

```
name
    Chart 名称

version
    Chart 自身版本

appVersion
    应用版本
```

注意：

> **Chart version 和应用 version 是两个不同概念。**

------

### 26.6.3 templates

这里放：

> **Kubernetes Manifest 模板**

例如：

```
templates/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

最终 Helm 会把这些 Template 渲染成 Kubernetes YAML。

------

### 26.6.4 values.yaml

用于定义：

> **默认配置。**

例如：

```
replicaCount: 2

image:
  repository: example.com/my-api
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80
```

Template 再读取这些值。

------

## 26.7 values.yaml

### 26.7.1 values.yaml 是什么

`values.yaml` 是 Helm Chart 的：

> **默认配置文件。**

例如：

```
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"

service:
  type: ClusterIP
  port: 80
```

Template：

```
spec:
  replicas: {{ .Values.replicaCount }}
```

Helm 渲染后：

```
spec:
  replicas: 2
```

------

### 26.7.2 为什么需要 Values

没有 Values：

```
每个环境
 ↓
修改 Template
```

有 Values：

```
Template
   +
values-dev.yaml
```

或者：

```
Template
   +
values-prod.yaml
```

Template 本身不用修改。

这就是 Helm 最重要的能力之一：

> **模板与环境配置分离。**

------

## 26.8 Template

### 26.8.1 Template 是什么

Helm Template 使用：

> **Go Template**

例如：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
```

其中：

```
.Release.Name
```

是 Helm Release 信息。

```
.Values.replicaCount
```

来自：

```
values.yaml
```

------

### 26.8.2 一个完整例子

`values.yaml`：

```
replicaCount: 3

image:
  repository: nginx
  tag: "1.27"
```

`templates/deployment.yaml`：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      app: {{ .Release.Name }}

  template:
    metadata:
      labels:
        app: {{ .Release.Name }}

    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

执行：

```
helm template my-api ./my-api
```

可以直接看到最终 YAML。

------

### 26.8.3 强烈推荐先使用 helm template

在真正安装之前：

```
helm template my-api ./my-api
```

先检查：

```
最终 YAML 是否正确
```

这对 Helm 排错非常重要。

------

## 26.9 Helm Functions

### 26.9.1 Functions 是什么

Helm Template 提供大量函数，可以对数据进行：

```
字符串处理
默认值
转换
编码
判断
格式化
```

例如：

```
name: {{ .Values.name | quote }}
```

------

### 26.9.2 default

例如：

```
replicas: {{ .Values.replicaCount | default 1 }}
```

如果：

```
replicaCount
```

没有设置，则使用：

```
1
```

------

### 26.9.3 quote

```
value: {{ .Values.appName | quote }}
```

可以生成：

```
value: "my-api"
```

------

### 26.9.4 upper / lower

例如：

```
{{ .Values.name | upper }}
```

可以把字符串转换成大写。

------

### 26.9.5 required

生产 Chart 中非常有用：

```
{{ required "image.repository is required" .Values.image.repository }}
```

如果没有设置：

```
image.repository
```

Helm 会直接报错，而不是生成错误的 Kubernetes YAML。

------

### 26.9.6 常见 Pipeline

Helm 经常看到：

```
{{ .Values.image.tag | quote }}
```

可以理解成：

```
Values
  ↓
image.tag
  ↓
quote
  ↓
最终字符串
```

------

## 26.10 Helm Variables

### 26.10.1 Variable 是什么

Helm 可以定义 Template 变量。

例如：

```
{{- $name := .Release.Name }}
```

之后可以使用：

```
{{ $name }}
```

------

### 26.10.2 为什么需要变量

当一个值需要多次使用时：

```
{{- $fullName := printf "%s-%s" .Release.Name .Chart.Name }}
```

之后：

```
metadata:
  name: {{ $fullName }}
```

以及：

```
app: {{ $fullName }}
```

这样可以减少重复。

------

### 26.10.3 with

例如：

```
{{- with .Values.image }}
image: "{{ .repository }}:{{ .tag }}"
{{- end }}
```

在 `with` 内部：

```
.
```

会指向：

```
.Values.image
```

所以：

```
.repository
.tag
```

可以直接访问。

------

### 26.10.4 range

`range` 常用于遍历列表。

Values：

```
env:
  - name: ASPNETCORE_ENVIRONMENT
    value: Production
  - name: LOG_LEVEL
    value: Information
```

Template：

```
env:
{{- range .Values.env }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```

最终：

```
env:
  - name: ASPNETCORE_ENVIRONMENT
    value: "Production"
  - name: LOG_LEVEL
    value: "Information"
```

------

## 26.11 Helm Install

### 26.11.1 Install 是什么

安装 Chart：

```
helm install my-api ./my-api
```

其中：

```
my-api
```

是 Release Name。

```
./my-api
```

是 Chart 路径。

------

### 26.11.2 指定 Namespace

如果 Namespace 已存在：

```
helm install my-api ./my-api \
  -n shop
```

如果不存在，可以：

```
helm install my-api ./my-api \
  -n shop \
  --create-namespace
```

------

### 26.11.3 指定 Values

```
helm install my-api ./my-api \
  -n shop \
  -f values-prod.yaml
```

也可以直接覆盖某个值：

```
helm install my-api ./my-api \
  -n shop \
  --set replicaCount=5
```

------

### 26.11.4 推荐生产流程

不要直接：

```
helm install
```

然后发现问题。

推荐：

```
helm lint ./my-api
```

然后：

```
helm template my-api ./my-api \
  -f values-prod.yaml
```

确认生成结果后：

```
helm install my-api ./my-api \
  -n shop \
  -f values-prod.yaml
```

------

### 26.11.5 查看 Release

```
helm status my-api -n shop
```

查看历史：

```
helm history my-api -n shop
```

------

## 26.12 Helm Upgrade

### 26.12.1 Upgrade 是什么

应用已经安装：

```
Release: my-api
```

现在修改：

```
Image
Replicas
Config
Ingress
```

可以：

```
helm upgrade my-api ./my-api \
  -n shop
```

------

### 26.12.2 指定生产 Values

```
helm upgrade my-api ./my-api \
  -n shop \
  -f values-prod.yaml
```

------

### 26.12.3 Install + Upgrade 合并

CI/CD 中经常使用：

```
helm upgrade --install my-api ./my-api \
  -n shop \
  --create-namespace \
  -f values-prod.yaml
```

含义：

```
Release 不存在
    ↓
Install

Release 已存在
    ↓
Upgrade
```

这是自动化部署中非常常见的写法。

------

### 26.12.4 Upgrade 前检查

推荐：

```
helm lint ./my-api
```

然后：

```
helm template my-api ./my-api \
  -n shop \
  -f values-prod.yaml
```

再执行：

```
helm upgrade my-api ./my-api \
  -n shop \
  -f values-prod.yaml
```

------

## 26.13 Helm Rollback

### 26.13.1 Rollback 是什么

假设：

```
Revision 1
    ↓
正常

Revision 2
    ↓
正常

Revision 3
    ↓
上线 Bug
```

可以回滚：

```
helm rollback my-api 2 -n shop
```

即：

```
Revision 3
    ↓
Rollback
    ↓
Revision 2
```

------

### 26.13.2 查看历史

```
helm history my-api -n shop
```

可能看到：

```
REVISION  STATUS
1         superseded
2         superseded
3         deployed
```

确定目标 Revision：

```
helm rollback my-api 2 -n shop
```

------

### 26.13.3 回滚并不等于解决问题

这是生产环境非常重要的一点。

Rollback 只解决：

> **快速恢复之前的 Release 状态。**

如果问题来自：

```
Database Migration
External Service
PVC
ConfigMap
Secret
数据结构变化
```

单纯 Helm Rollback 可能无法恢复整个系统。

所以生产发布必须考虑：

```
Application Version
Database Version
Config Version
Infrastructure Version
```

之间的兼容关系。

------

## 26.14 Helm Uninstall

### 26.14.1 Uninstall 是什么

删除 Release：

```
helm uninstall my-api -n shop
```

Helm 会删除该 Release 管理的 Kubernetes 资源。

------

### 26.14.2 查看

```
helm list -n shop
```

执行：

```
helm uninstall my-api -n shop
```

再次：

```
helm list -n shop
```

Release 就不再处于当前安装状态。

------

### 26.14.3 PVC 要特别注意

生产环境尤其需要注意：

> **不要把 Helm Uninstall 简单理解成“所有数据都一定消失”。**

PVC、StorageClass、底层存储的生命周期需要单独考虑。

因此生产环境执行：

```
helm uninstall
```

之前必须确认：

```
是否有 PVC？
PVC 是否需要保留？
数据库数据在哪里？
Storage 的 reclaimPolicy 是什么？
```

不要把生产数据库当作普通 Deployment 一样随便删除。

------

## 26.15 环境配置

Helm 真正进入生产环境后，一个重要设计原则是：

> **Template 尽量稳定，环境差异放到 Values。**

例如：

```
Chart
 │
 ├── templates/
 │
 └── values.yaml
```

环境：

```
values-dev.yaml
values-test.yaml
values-prod.yaml
```

------

### 26.15.1 基础 Values

```
replicaCount: 2

image:
  repository: example.com/shop-api
  tag: "1.0.0"

service:
  port: 80
```

------

### 26.15.2 环境差异

Development：

```
replicaCount: 1

image:
  tag: "dev"

ingress:
  host: api-dev.example.com
```

Production：

```
replicaCount: 5

image:
  tag: "3.2.1"

ingress:
  host: api.example.com
```

Template 不需要复制。

------

## 26.16 开发 / 测试 / 生产 values

推荐目录：

```
my-api/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-test.yaml
├── values-prod.yaml
└── templates/
```

------

### 26.16.1 values.yaml

放：

> **所有环境的合理默认值。**

例如：

```
replicaCount: 1

image:
  repository: example.com/shop-api
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80
```

------

### 26.16.2 values-dev.yaml

```
replicaCount: 1

image:
  tag: "dev"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

------

### 26.16.3 values-prod.yaml

```
replicaCount: 5

image:
  tag: "3.2.1"

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 2Gi
```

------

### 26.16.4 部署不同环境

Development：

```
helm upgrade --install my-api ./my-api \
  -n shop-dev \
  --create-namespace \
  -f values-dev.yaml
```

Production：

```
helm upgrade --install my-api ./my-api \
  -n shop-prod \
  --create-namespace \
  -f values-prod.yaml
```

同一个 Chart：

```
my-api
```

不同：

```
values-dev.yaml
values-prod.yaml
```

生成不同环境。

------

### 26.16.5 多个 Values 文件

Helm 可以指定多个：

```
helm upgrade --install my-api ./my-api \
  -n shop-prod \
  -f values.yaml \
  -f values-prod.yaml
```

后面的 Values 通常用于覆盖前面的值。

因此可以形成：

```
values.yaml
    ↓
公共默认配置

values-prod.yaml
    ↓
生产环境覆盖
```

这比维护完整复制的生产 YAML 更容易管理。

------

## 26.17 自己制作 Helm Chart

现在开始真正制作一个简单 Chart。

------

### 26.17.1 创建 Chart

```
helm create my-api
```

为了方便学习，可以删除默认模板：

```
rm -rf my-api/templates/*
```

然后创建：

```
my-api/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

------

### 26.17.2 Chart.yaml

```
apiVersion: v2
name: my-api
description: A Helm chart for my API
type: application
version: 1.0.0
appVersion: "1.0.0"
```

------

### 26.17.3 values.yaml

```
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

------

### 26.17.4 Deployment Template

`templates/deployment.yaml`：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      app: {{ .Release.Name }}

  template:
    metadata:
      labels:
        app: {{ .Release.Name }}

    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:
            - containerPort: 80
```

------

### 26.17.5 Service Template

`templates/service.yaml`：

```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}

  selector:
    app: {{ .Release.Name }}

  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
```

------

### 26.17.6 检查 Chart

```
helm lint ./my-api
```

如果正常：

```
1 chart(s) linted, 0 chart(s) failed
```

------

### 26.17.7 渲染 Template

```
helm template my-api ./my-api
```

这一步非常重要。

你应该检查最终生成的：

```
Deployment
Service
```

是否符合预期。

------

### 26.17.8 安装

```
helm install my-api ./my-api \
  -n shop \
  --create-namespace
```

查看：

```
helm list -n shop
```

查看 Pod：

```
kubectl get pods -n shop
```

查看 Service：

```
kubectl get svc -n shop
```

------

### 26.17.9 修改配置

例如：

```
helm upgrade my-api ./my-api \
  -n shop \
  --set replicaCount=3
```

查看：

```
kubectl get deployment -n shop
```

应该看到：

```
READY
3/3
```

------

### 26.17.10 查看 Release 历史

```
helm history my-api -n shop
```

然后测试回滚：

```
helm rollback my-api 1 -n shop
```

再次查看：

```
helm history my-api -n shop
```

这样就完整体验了：

```
Chart
 ↓
Install
 ↓
Release
 ↓
Upgrade
 ↓
Revision
 ↓
Rollback
```

------

### 26.17.11 打包 Chart

Chart 完成后可以打包：

```
helm package ./my-api
```

生成：

```
my-api-1.0.0.tgz
```

这个文件就可以上传到：

```
Helm Repository
```

或者企业内部的 OCI Registry。

------

### 26.17.12 生产 Chart 的基本结构

一个真正的生产 Chart 通常会比学习示例复杂：

```
my-api/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-test.yaml
├── values-prod.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── networkpolicy.yaml
│   ├── pdb.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl
│   └── tests/
└── .helmignore
```

但不要为了“看起来专业”而把所有资源都塞进 Chart。

应该根据应用实际需要决定。

------

### 26.17.13 Helm Chart 的生产设计原则

生产环境建议遵循：

```
1. Template 与环境配置分离
2. values.yaml 提供合理默认值
3. 生产值通过独立 values 管理
4. 不把密码直接写入 Git
5. 使用 helm lint
6. 部署前使用 helm template
7. CI/CD 中使用 helm upgrade --install
8. 保留 Release History
9. 谨慎执行 rollback
10. Chart 与应用版本分别管理
```

尤其是：

> **不要把 Secret 明文放进 `values-prod.yaml`。**

例如不要：

```
database:
  password: "SuperSecretPassword"
```

更合理的生产方案是让 Helm 与：

```
External Secrets
Vault
Cloud Secret Manager
```

等 Secret 管理系统配合。

------

## 本章核心知识总结

Helm 最核心的几个概念一定要区分：

```
Chart
= Kubernetes 应用模板 / 软件包

Values
= 配置参数

Template
= Kubernetes YAML 模板

Release
= Chart 的一次安装实例

Repository
= 存放 Chart 的仓库
```

完整工作流程：

```
                Chart
                  │
          ┌───────┴────────┐
          │                │
      Template          Values
          │                │
          └───────┬────────┘
                  ▼
             Helm Render
                  │
                  ▼
          Kubernetes YAML
                  │
                  ▼
            helm install
                  │
                  ▼
              Release
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Upgrade   Rollback  Uninstall
```

生产环境的典型目录：

```
my-api/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-test.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    └── ...
```

典型生产部署：

```
helm lint ./my-api

helm template my-api ./my-api \
  -f values-prod.yaml

helm upgrade --install my-api ./my-api \
  -n shop-prod \
  --create-namespace \
  -f values-prod.yaml
```

出现问题：

```
helm history my-api -n shop-prod
```

确认历史 Revision 后：

```
helm rollback my-api <REVISION> -n shop-prod
```

最终应该形成这样的认知：

> **Kubernetes YAML 解决“资源怎么定义”，Helm 解决“这些资源如何作为一个应用进行模板化、配置化、版本化和生命周期管理”。**