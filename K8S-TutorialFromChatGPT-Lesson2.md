## 第二阶段 第一章：Helm 实战（一）—— 从零开始编写 Helm Chart

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入真正的**生产环境实战**。

如果说前面的内容都是在学习 Kubernetes 的"语法"。

那么从现在开始，我们学习的是：

> **企业到底是怎么部署项目的。**

很多新手学习 Kubernetes 时，会有一种误区：

> **学完 Deployment、Service、Ingress 就可以上线了。**

实际上。

真正的企业部署流程，几乎都是：

```
Git
   │
   ▼
CI（GitHub Actions / GitLab CI / Azure DevOps）
   │
   ▼
Docker Image
   │
   ▼
Helm Chart
   │
   ▼
Dev
   │
   ▼
Test
   │
   ▼
Staging
   │
   ▼
Production
```

这里面最重要的一环。

就是：

> **Helm Chart**

上一章我们已经知道 Helm 是什么。

这一章。

我们要亲手写一个真正可以部署的 Helm Chart。

而且。

不是 Hello World。

而是：

> **ASP.NET Core Web API + Vben Admin(Vue3) + Redis**

这也是很多公司最典型的一套架构。

#### 本章学习目标

学完本章，你应该能够回答：

- 一个 Helm Chart 到底长什么样？
- Chart.yaml 是干什么的？
- values.yaml 为什么这么重要？
- templates 目录里面应该放什么？
- 一个企业级 Helm Chart 应该如何组织？
- 如何做到一套模板，多套环境？

------

### 第一节：为什么企业几乎都用 Helm？

假设。

你有一个项目。

```
Order.Api
```

开发环境：

镜像：

```
order-api:v1.0.0-dev
```

副本：

```
1
```

数据库：

```
Dev SQL Server
```

------

测试环境：

镜像：

```
order-api:v1.0.0-test
```

副本：

```
2
```

数据库：

```
Test SQL Server
```

------

生产环境：

镜像：

```
order-api:v1.0.0
```

副本：

```
6
```

数据库：

```
Production SQL Server
```

如果不用 Helm。

你可能会有：

```
deployment-dev.yaml

deployment-test.yaml

deployment-prod.yaml
```

Service：

又三份。

Ingress：

又三份。

HPA：

又三份。

很快。

整个仓库：

变成：

```
deployment-dev.yaml
deployment-test.yaml
deployment-prod.yaml

service-dev.yaml
service-test.yaml
service-prod.yaml

ingress-dev.yaml
ingress-test.yaml
ingress-prod.yaml

...
```

修改一次端口。

可能要改九个文件。

这就是 Helm 要解决的问题。

------

### 第二节：Helm Chart 的目录结构

我们先不要急着写模板。

先认识目录。

假设：

我们的项目叫：

```
order-api
```

执行：

```
helm create order-api
```

Helm 会自动生成：

```
order-api/

├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── hpa.yaml
├── serviceaccount.yaml
├── NOTES.txt
│
└── _helpers.tpl
```

第一次看到。

很多人都会懵。

其实。

真正重要的只有几个文件。

------

### 第三节：Chart.yaml

这是 Helm Chart 的身份证。

例如：

```
apiVersion: v2

name: order-api

description: ASP.NET Core Order API

type: application

version: 1.0.0

appVersion: "1.0.0"
```

下面解释每个字段。

------

#### apiVersion

不是 Kubernetes API。

而是：

Helm Chart 格式。

目前一般都是：

```
apiVersion: v2
```

基本不用改。

------

#### name

Chart 名称。

例如：

```
name: order-api
```

以后：

安装：

就是：

```
order-api
```

------

#### description

项目描述。

例如：

```
description: Order API
```

只是说明文字。

不会影响部署。

------

#### version

注意。

很多新人：

都会搞错。

这里：

不是：

应用版本。

而是：

> **Chart 自己的版本。**

例如：

模板：

修改了一次。

Chart：

版本：

可以：

```
1.0.0

↓

1.0.1
```

即使：

你的程序：

还是：

```
v8.0
```

Chart：

也可以：

升级。

------

#### appVersion

真正：

程序：

版本。

例如：

```
appVersion: "2.1.5"
```

它一般对应：

Docker Image：

```
order-api:2.1.5
```

注意：

很多团队会把镜像 Tag 放在 `values.yaml` 中，而 `appVersion` 主要用于描述和展示，不一定直接参与部署。

------

### 第四节：values.yaml

这是：

整个 Helm：

最重要：

的文件。

可以理解成：

> **所有可以修改的配置，都放这里。**

例如：

```
replicaCount: 2
```

部署：

模板：

不用：

写：

```
replicas: 2
```

而是：

写：

```
replicas: {{ .Values.replicaCount }}
```

以后。

开发：

```
replicaCount: 1
```

生产：

```
replicaCount: 6
```

模板：

完全：

不用：

修改。

------

### 第五节：一个企业 values.yaml 应该长什么样？

下面是一份比较典型的结构：

```
replicaCount: 2

image:
  repository: mycompany/order-api
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: api.example.com

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

是不是：

几乎：

所有：

可变：

内容。

都：

集中：

在这里。

------

### 第六节：为什么 values.yaml 要这样设计？

假设。

镜像：

变成：

```
1.0.1
```

以前。

改：

Deployment。

现在。

只需要：

```
image:

  tag: "1.0.1"
```

即可。

例如：

CI/CD：

最后一步：

甚至：

直接：

修改：

这个：

Tag。

然后：

执行：

```
helm upgrade
```

完成：

部署。

很多公司的自动发布流水线就是这样工作的。

------

### 第七节：templates 目录

这里。

是真正：

放 Kubernetes YAML 模板的地方。

例如：

```
templates/

deployment.yaml

service.yaml

ingress.yaml

hpa.yaml
```

这些：

其实：

就是：

普通：

Kubernetes YAML。

唯一：

区别：

就是：

里面：

可以：

使用：

模板变量。

例如：

普通：

Deployment：

```
replicas: 2
```

Helm：

Deployment：

```
replicas: {{ .Values.replicaCount }}
```

是不是：

只有：

这一点：

不同？

------

### 第八节：_helpers.tpl 是什么？

这是很多新人最害怕的文件。

其实。

它不是 Kubernetes。

而是 Helm 模板里的**公共函数**。

例如。

多个文件都需要生成统一的资源名称：

```
order-api

order-api-service

order-api-config
```

如果：

每个文件：

都自己拼。

以后：

改名字。

全部：

都要：

修改。

于是：

Helm：

允许：

把：

公共：

逻辑。

放：

这里。

例如：

（这里只理解概念，不要求记模板语法。）

```
生成统一名称

生成统一 Label

生成统一 Selector
```

以后。

Deployment。

Service。

Ingress。

全部：

调用：

同一个：

公共模板。

这和你熟悉的 C# 方法、公共类有点类似：**避免重复代码**。

------

### 第九节：企业 Helm Chart 推荐目录

随着项目变大，通常会增加更多模板：

```
templates/

deployment.yaml

service.yaml

ingress.yaml

configmap.yaml

secret.yaml

hpa.yaml

pdb.yaml

networkpolicy.yaml

serviceaccount.yaml
```

这里先简单介绍几个新文件：

- **PodDisruptionBudget（PDB）**：保证节点维护、升级时，不会一下子把所有 Pod 都驱逐掉。
- **NetworkPolicy**：限制哪些 Pod 可以访问哪些 Pod，实现网络隔离。
- **ServiceAccount**：Pod 在 Kubernetes 中使用的身份，后续学习 RBAC 时会详细介绍。

这些都是生产环境非常常见的资源。

------

### 第十节：一套模板，多套环境

这是 Helm 最大的价值。

例如：

目录：

```
values.yaml

values-dev.yaml

values-test.yaml

values-prod.yaml
```

开发：

```
replicaCount: 1
```

测试：

```
replicaCount: 2
```

生产：

```
replicaCount: 8
```

部署：

开发：

```
helm upgrade --install order-api . -f values-dev.yaml
```

部署：

测试：

```
helm upgrade --install order-api . -f values-test.yaml
```

部署：

生产：

```
helm upgrade --install order-api . -f values-prod.yaml
```

**模板完全一样，只是配置不同。**

这就是 Helm 在企业里最常见的使用方式。

------

### 本章总结（建议牢记）

请记住 Helm Chart 最重要的几点：

1. **Chart.yaml 是 Chart 的元信息，不是业务配置。**
2. **values.yaml 存放所有可变配置，是不同环境差异的核心。**
3. **templates 目录存放 Kubernetes YAML 模板。**
4. **模板通过 `{{ .Values.xxx }}` 读取配置。**
5. **推荐采用"一套模板 + 多个 values 文件"管理开发、测试和生产环境。**
6. **随着项目成熟，可以逐步加入 HPA、PDB、NetworkPolicy 等生产资源。**

------

### 下一章预告：Helm 实战（二）——编写第一个 Deployment 模板

下一章，我们将真正开始写 Helm 模板，而不是只讲概念。

我们会一步一步完成一个可以直接部署的 **ASP.NET Core Web API Helm Chart**，包括：

- 将 Deployment 改造成 Helm 模板
- 使用 `{{ .Values.xxx }}` 替换固定值
- 引用镜像、端口、副本数等配置
- 编写统一的 Labels 和 Selectors
- 使用 `helm template` 查看最终渲染出来的 Kubernetes YAML
- 理解 Helm 渲染过程与 Kubernetes 部署过程的关系

从这一章开始，你会逐渐具备**独立编写企业级 Helm Chart** 的能力，而不仅仅是会使用别人写好的 Chart。

## 第二阶段 第二章：Helm 实战（二）—— 编写第一个 Deployment 模板

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们正式进入 **Helm 模板开发**。

如果说上一章只是认识 Helm。

这一章，我们要真正开始写模板。

而且不是简单地讲语法。

而是按照**企业真实项目**来写。

整个系列我们都会围绕下面这个项目：

```
Order.Api（ASP.NET Core Web API）
```

部署以后，最终会得到：

```
Deployment
        │
        ▼
Pod
        │
        ▼
Service
        │
        ▼
Ingress
```

不过。

所有固定配置都会变成：

```
{{ .Values.xxx }}
```

这就是 Helm。

#### 本章学习目标

学习完本章，你应该能够回答：

- Helm Template 到底是怎么工作的？
- `{{ }}` 是什么意思？
- `.Values` 是什么？
- 如何把普通 Deployment 改造成 Helm 模板？
- 为什么企业很少直接写固定值？
- 什么是 `_helpers.tpl`？
- Labels 为什么要抽出来？

------

### 第一节：先回忆普通 Deployment

假设。

以前。

我们写：

Deployment：

```
apiVersion: apps/v1

kind: Deployment

metadata:
  name: order-api

spec:

  replicas: 2

  selector:
    matchLabels:
      app: order-api

  template:

    metadata:
      labels:
        app: order-api

    spec:

      containers:

      - name: order-api

        image: mycompany/order-api:v1.0.0

        ports:

        - containerPort: 8080
```

是不是：

全部：

都是：

固定值。

例如：

```
order-api

2

8080

v1.0.0
```

如果：

生产：

变成：

```
8

v2.0.0
```

全部：

要改。

于是。

Helm：

出现。

------

### 第二节：Helm Template 到底是什么？

一句话：

> **Template（模板）就是一份带变量的 Kubernetes YAML。**

例如。

以前：

写：

```
replicas: 2
```

Helm：

改成：

```
replicas: {{ .Values.replicaCount }}
```

什么意思？

假设：

values.yaml：

```
replicaCount: 3
```

Helm：

运行：

以后。

生成：

```
replicas: 3
```

然后。

再：

发送：

给：

API Server。

注意。

Kubernetes：

**完全不知道 Helm 的存在。**

它最终收到的依然是普通 YAML。

这一点非常重要。

------

### 第三节：理解 {{ }}

很多新人：

第一次：

看到：

```
{{ }}
```

都会：

害怕。

其实。

如果你是后端开发者。

可以把它理解成：

C#：

字符串模板。

例如：

C#：

```
var name = "Andy";

Console.WriteLine($"Hello {name}");
```

输出：

```
Hello Andy
```

Helm：

也是：

一样。

例如：

```
image:

  tag: {{ .Values.image.tag }}
```

假设：

```
image:

  tag: "2.0.0"
```

最终：

生成：

```
image:

  tag: 2.0.0
```

所以。

可以理解成：

> **{{ }} = 把变量放进 YAML。**

------

### 第四节：什么是 .Values？

这是 Helm 用得最多的对象。

```
.Values
```

表示：

> **读取 values.yaml。**

例如：

values.yaml：

```
replicaCount: 2

image:

  repository: mycompany/order-api

  tag: "1.0.0"

service:

  port: 8080
```

Deployment：

模板：

可以：

写：

```
replicas: {{ .Values.replicaCount }}
```

镜像：

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

端口：

```
containerPort: {{ .Values.service.port }}
```

最终：

Helm：

自动：

替换。

------

### 第五节：一步一步改造 Deployment

现在。

开始。

真正：

模板化。

原来：

```
replicas: 2
```

改成：

```
replicas: {{ .Values.replicaCount }}
```

------

原来：

```
image: mycompany/order-api:v1.0.0
```

改成：

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

------

原来：

```
containerPort: 8080
```

改成：

```
containerPort: {{ .Values.service.port }}
```

是不是。

其实：

没有：

那么：

复杂？

------

### 第六节：metadata.name 为什么也要模板化？

很多新人：

喜欢：

写：

```
metadata:

  name: order-api
```

其实。

企业：

一般：

不会。

而是：

写：

```
metadata:

  name: {{ include "order-api.fullname" . }}
```

第一次：

看到：

是不是：

懵？

不用：

急。

这里只需要理解：

它不是 Kubernetes。

而是：

Helm 的一个**公共函数调用**。

作用：

就是：

统一：

生成：

资源名称。

例如：

以后：

Release：

名字：

叫：

```
prod
```

生成：

可能：

就是：

```
prod-order-api
```

以后：

测试：

环境：

```
test-order-api
```

这样：

不同环境部署到同一个集群，也不会因为资源重名而冲突。

------

### 第七节：为什么 Label 不直接写？

以前：

写：

```
labels:

  app: order-api
```

企业：

通常：

写：

```
labels:

  {{- include "order-api.labels" . | nindent 4 }}
```

是不是：

更：

看不懂？

其实。

原因：

只有：

一个。

避免：

重复。

例如：

Deployment：

需要：

Labels。

Service：

需要：

Labels。

Pod：

需要：

Labels。

Ingress：

可能：

也：

需要。

如果：

全部：

自己：

写。

以后：

改：

Label。

改：

十几处。

于是。

统一：

抽：

出来。

------

### 第八节：_helpers.tpl 到底做什么？

例如：

里面：

可能：

有：

一个：

公共模板：

```
生成应用名称

生成 Labels

生成 Selector Labels
```

Deployment：

调用。

Service：

调用。

Ingress：

调用。

以后：

全部：

保持：

一致。

你可以把 `_helpers.tpl` 理解为：

```
C#

↓

Helper.cs
```

里面：

放：

公共：

方法。

而：

不是：

业务：

代码。

------

### 第九节：完整的 Deployment 模板（简化版）

下面是一份简化后的 Helm Deployment 模板。

请重点关注 **哪些地方变成了变量**。

```
apiVersion: apps/v1

kind: Deployment

metadata:

  name: {{ include "order-api.fullname" . }}

spec:

  replicas: {{ .Values.replicaCount }}

  selector:

    matchLabels:

      app: {{ include "order-api.name" . }}

  template:

    metadata:

      labels:

        app: {{ include "order-api.name" . }}

    spec:

      containers:

      - name: order-api

        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

        ports:

        - containerPort: {{ .Values.service.port }}
```

是不是。

除了：

变量。

剩下：

全部：

还是：

普通：

Deployment。

所以。

学习 Helm。

真正：

学习：

只有：

模板：

语法。

Kubernetes：

知识：

还是：

原来的。

------

### 第十节：Helm 如何渲染？

假设：

values.yaml：

```
replicaCount: 3

image:

  repository: mycompany/order-api

  tag: "2.0.0"

service:

  port: 8080
```

执行：

```
helm template order-api .
```

Helm：

输出：

```
replicas: 3

image:

  mycompany/order-api:2.0.0

containerPort: 8080
```

然后：

这些：

普通：

YAML。

才：

真正：

发送：

到：

Kubernetes。

因此：

一个非常重要的调试技巧就是：

> **先用 `helm template` 看生成的 YAML，再部署到集群。**

很多模板错误，在这一步就能发现，而不用等到 Kubernetes 创建失败。

------

### 第十一节：企业里的开发流程

真实项目中，通常不会直接修改线上 Chart，而是：

```
修改 values.yaml
        │
        ▼
helm template（检查渲染结果）
        │
        ▼
helm lint（检查 Chart 是否规范）
        │
        ▼
helm upgrade --install
        │
        ▼
Deployment 更新
        │
        ▼
Rolling Update
```

这里出现了一个新命令：

```
helm lint
```

作用：

就是：

检查：

Chart：

有没有：

明显：

错误。

它类似于：

```
C#

↓

编译器检查
```

虽然：

不能：

保证：

业务：

正确。

但是：

可以：

发现：

很多：

模板：

问题。

------

### 第十二节：模板化时容易踩的坑

很多初学者都会遇到下面这些问题：

##### ① 忘记给字符串加引号

例如：

```
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

有时会因为特殊字符导致 YAML 解析问题。

更稳妥的写法是：

```
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

------

##### ② values.yaml 和模板字段不一致

例如：

模板：

```
{{ .Values.images.tag }}
```

而：

values.yaml：

却是：

```
image:

  tag: "1.0.0"
```

由于：

```
images
```

不存在。

渲染：

结果：

就是：

错误。

因此：

保持字段命名一致非常重要。

------

##### ③ 过度模板化

不是所有内容都必须放进 `values.yaml`。

例如：

```
apiVersion: apps/v1

kind: Deployment
```

通常：

都是：

固定：

值。

没有：

必要：

模板化。

经验原则：

> **只有会因环境或版本变化而改变的内容，才放进 values.yaml。**

------

### 本章总结（建议牢记）

请记住 Helm 模板最重要的几点：

1. **Helm Template 本质上还是 Kubernetes YAML，只是加入了模板变量。**
2. **`{{ }}` 表示模板表达式，用于把变量或函数结果插入 YAML。**
3. **`.Values` 用于读取 `values.yaml` 中的配置。**
4. **`include` 常用于调用 `_helpers.tpl` 中定义的公共模板，减少重复。**
5. **先使用 `helm template` 和 `helm lint`，再部署，是企业最佳实践。**
6. **不要为了模板化而模板化，把真正会变化的配置放到 `values.yaml` 即可。**

------

### 下一章预告：Helm 实战（三）——Service、Ingress、ConfigMap 模板化

下一章，我们会继续完成整个 Helm Chart，包括：

- 编写 `Service` 模板
- 编写 `Ingress` 模板
- 编写 `ConfigMap` 模板
- 使用 `Secret` 管理数据库连接字符串
- 使用 `if`、`with`、`range` 等 Helm 常用模板语法
- 支持开发、测试、生产三套配置共用一套模板

到那时，你将拥有一个**完整可用于 ASP.NET Core 项目**的 Helm Chart，而不仅仅是一个 Deployment 模板。

## 第二阶段 第三章：Helm 实战（三）—— Service、Ingress、ConfigMap 与 Helm 模板语法

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始。

我们要把整个 Helm Chart 写完整。

上一章。

我们已经完成了：

```
Deployment
```

但是。

一个真正的 ASP.NET Core 项目，仅仅有 Deployment 是远远不够的。

至少还需要：

```
Service
Ingress
ConfigMap
Secret
```

除此之外。

我们还要学习 Helm 最常用的模板语法：

```
if

with

range
```

因为几乎所有企业 Helm Chart 都会用到它们。

#### 本章学习目标

学习完本章，你应该能够回答：

- 如何把 Service 改造成 Helm 模板？
- 如何让 Ingress 可以开启或关闭？
- ConfigMap 如何读取 values.yaml？
- Secret 为什么不能直接写进 values.yaml？
- Helm 的 if、with、range 分别有什么作用？
- 企业 Helm Chart 为什么大量使用条件判断？

------

### 第一节：继续完成我们的项目

目前。

我们的 Chart 已经有：

```
templates/

deployment.yaml
```

现在。

继续增加：

```
templates/

deployment.yaml

service.yaml

ingress.yaml

configmap.yaml

secret.yaml
```

以后。

执行：

```
helm install
```

这些资源。

会一起创建。

------

### 第二节：Service 模板化

先看普通 Service：

```
apiVersion: v1

kind: Service

metadata:

  name: order-api

spec:

  type: ClusterIP

  selector:

    app: order-api

  ports:

  - port: 80

    targetPort: 8080
```

里面。

哪些：

应该：

放到：

values？

一般：

有：

这些：

```
Service 类型

Port

TargetPort
```

于是。

改成：

```
spec:

  type: {{ .Values.service.type }}

  ports:

  - port: {{ .Values.service.port }}

    targetPort: {{ .Values.service.targetPort }}
```

对应：

values.yaml

```
service:

  type: ClusterIP

  port: 80

  targetPort: 8080
```

以后。

如果：

改成：

NodePort。

只需要：

```
service:

  type: NodePort
```

模板：

不用：

修改。

------

### 第三节：Ingress 为什么要支持开关？

不是：

所有：

环境：

都需要：

Ingress。

例如：

开发：

环境：

可能：

直接：

```
kubectl port-forward
```

或者：

```
NodePort
```

所以。

Ingress：

应该：

可以：

关闭。

Helm：

通常：

这样：

写：

```
ingress:

  enabled: true
```

然后。

模板：

写：

```
{{- if .Values.ingress.enabled }}

apiVersion: networking.k8s.io/v1

kind: Ingress

...

{{- end }}
```

什么意思？

如果：

```
enabled: true
```

生成：

Ingress。

如果：

```
enabled: false
```

整个：

Ingress.yaml：

不会：

生成。

------

### 一个生活例子

可以把：

```
if
```

理解成：

C#：

```
if(enable)
{
    创建Ingress();
}
```

是不是：

一模一样？

Helm：

只是：

换了一种：

写法。

------

### 第四节：Ingress 如何读取 Host？

例如：

以前：

写：

```
host: api.example.com
```

现在：

改成：

```
host: {{ .Values.ingress.host }}
```

values：

```
ingress:

  enabled: true

  host: api.example.com
```

开发：

环境：

```
host: api-dev.example.com
```

生产：

环境：

```
host: api.example.com
```

是不是：

完全：

不用：

修改：

模板？

------

### 第五节：ConfigMap 模板

假设。

ASP.NET Core：

有：

配置：

```
{
    "LogLevel":"Information",

    "CacheTime":300
}
```

以前：

可能：

写死。

现在。

可以：

放：

values：

```
config:

  logLevel: Information

  cacheTime: 300
```

ConfigMap：

模板：

```
apiVersion: v1

kind: ConfigMap

metadata:

  name: {{ include "order-api.fullname" . }}

data:

  LogLevel: "{{ .Values.config.logLevel }}"

  CacheTime: "{{ .Values.config.cacheTime }}"
```

是不是：

所有：

配置：

都：

来自：

values？

------

### 第六节：为什么 Secret 不建议直接放 values.yaml？

很多新人：

第一次：

都会：

这样：

写：

```
database:

  password: 123456
```

这是：

非常：

危险：

的。

为什么？

因为：

values.yaml：

一般：

都会：

进入：

Git。

于是。

密码：

上传：

GitHub。

生产：

事故。

------

企业：

一般：

有：

几种：

方案：

```
方案一：

CI/CD 注入

────────────

方案二：

External Secret

────────────

方案三：

云平台 Secret Manager

────────────

方案四：

独立 values-prod.yaml（不提交 Git）
```

如果只是学习，可以把 Secret 放在独立的 `values-local.yaml` 中，并确保它不会提交到代码仓库。

真正生产环境，更推荐使用专门的密钥管理方案。

------

### 第七节：Helm 的 with

这是：

第二个：

最常见：

语法。

例如：

以前：

一直：

写：

```
{{ .Values.image.repository }}

{{ .Values.image.tag }}

{{ .Values.image.pullPolicy }}
```

是不是：

很长？

可以：

写：

```
{{- with .Values.image }}

repository: {{ .repository }}

tag: {{ .tag }}

pullPolicy: {{ .pullPolicy }}

{{- end }}
```

这里：

```
with
```

相当于：

进入：

一个：

对象。

以后：

不用：

一直：

写：

```
.Values.image
```

了。

------

一个 C# 类比

例如：

原来：

```
Console.WriteLine(user.Name);

Console.WriteLine(user.Age);

Console.WriteLine(user.City);
```

如果：

进入：

对象。

是不是：

就：

不用：

一直：

写：

user？

Helm：

就是：

这个：

意思。

------

### 第八节：Helm 的 range

这是：

第三个：

最重要：

语法。

例如。

Ingress：

可能：

有：

多个：

Host。

values：

```
hosts:

- api.example.com

- admin.example.com

- test.example.com
```

模板：

可以：

写：

```
{{- range .Values.hosts }}

- host: {{ . }}

{{- end }}
```

生成：

```
- host: api.example.com

- host: admin.example.com

- host: test.example.com
```

所以。

> **range = 循环。**

是不是：

很像：

C#：

```
foreach(...)
{
}
```

------

### 第九节：企业 Chart 为什么大量使用 if？

例如：

开发：

环境：

不要：

HPA。

```
autoscaling:

  enabled: false
```

模板：

```
{{- if .Values.autoscaling.enabled }}

HPA

{{- end }}
```

生产：

环境：

```
enabled: true
```

自动：

生成：

HPA。

同样的思路还常用于：

- 是否创建 Ingress
- 是否创建 ServiceAccount
- 是否挂载额外 Volume
- 是否开启 PodDisruptionBudget（PDB）

这样一套模板可以适配多种部署场景。

------

### 第十节：values-dev、values-prod

企业：

最常见：

目录：

```
values.yaml

values-dev.yaml

values-test.yaml

values-prod.yaml
```

例如。

开发：

```
replicaCount: 1

ingress:

  enabled: false

autoscaling:

  enabled: false
```

生产：

```
replicaCount: 6

ingress:

  enabled: true

autoscaling:

  enabled: true
```

部署：

开发：

```
helm install order-api . -f values-dev.yaml
```

生产：

```
helm install order-api . -f values-prod.yaml
```

模板：

只有：

一份。

------

### 第十一节：真实企业项目通常还会有哪些 values？

很多公司都会把下面这些内容放进 values.yaml：

```
image:

resources:

nodeSelector:

tolerations:

affinity:

env:

volumeMounts:

volumes:

probes:

autoscaling:
```

为什么？

因为：

这些：

都是：

可能：

因环境而变化的配置。

例如：

测试环境：

不用：

HPA。

生产：

需要：

HPA。

GPU：

节点：

需要：

NodeSelector。

普通：

节点：

不用。

------

### 第十二节：一个完整的渲染流程

最终。

整个：

Helm：

流程：

其实：

只有：

四步。

```
values-prod.yaml

        │

        ▼

templates/

Deployment

Service

Ingress

ConfigMap

Secret

        │

        ▼

Helm Template

        │

        ▼

普通 YAML

        │

        ▼

Kubernetes API Server
```

请记住：

> **Helm 负责"生成"，Kubernetes 负责"运行"。**

两者职责非常明确。

------

### 第十三节：Helm 常见模板语法总结

| 语法                | 作用           | 类比                        |
| ------------------- | -------------- | --------------------------- |
| `{{ .Values.xxx }}` | 读取配置       | `config.Value`              |
| `if`                | 条件判断       | `if (...) {}`               |
| `with`              | 进入对象作用域 | `var image = config.Image;` |
| `range`             | 遍历列表       | `foreach`                   |
| `include`           | 调用公共模板   | 调用公共方法                |

如果你有 C# 开发经验，这些语法会很容易理解。

------

### 本章总结（建议牢记）

请记住 Helm 模板开发最重要的几点：

1. **Service、Ingress、ConfigMap 都应该尽量通过 `values.yaml` 配置可变参数。**
2. **`if` 用于按需创建资源，是企业 Helm Chart 中最常见的模板语法。**
3. **`with` 可以减少重复路径，提高模板可读性。**
4. **`range` 用于遍历列表，非常适合生成多个 Host、环境变量或端口。**
5. **不要把生产密码直接写进会提交到 Git 的 `values.yaml`。**
6. **推荐采用 `values-dev.yaml`、`values-test.yaml`、`values-prod.yaml` 管理不同环境。**

------

### 到这里，你已经能够编写一个完整的 Helm Chart

```
Chart.yaml
        │
        ▼
values-prod.yaml
        │
        ▼
templates/
    │
    ├── Deployment
    ├── Service
    ├── Ingress
    ├── ConfigMap
    └── Secret
        │
        ▼
helm template
        │
        ▼
Kubernetes YAML
        │
        ▼
API Server
```

这已经接近企业中 Helm Chart 的基本形态。

------

### 下一章预告：Helm 实战（四）——部署 ASP.NET Core + Vben Admin 完整项目

下一章，我们将把前几章学到的内容整合起来，完成一个接近生产环境的完整案例，包括：

- 一个 ASP.NET Core Web API
- 一个 Vben Admin（Vue3）前端
- Redis
- ConfigMap 与 Secret
- Ingress 路由配置
- HPA 自动扩缩容
- 多环境（Dev/Test/Prod）部署
- 从 `helm install` 到应用成功访问的完整流程

这一章结束后，你将拥有一个可以作为实际项目模板的 Helm Chart，也会理解企业是如何组织和交付 Kubernetes 应用的。

## 第二阶段 第四章：Helm 实战（四）——部署一个 ASP.NET Core + Vben Admin 完整项目

提示词：

```
开始下一章
```

正文：

非常好。

从这一章开始，我们不再单独学习某个 Kubernetes 对象，而是把前面所有知识串联起来。

这是整个教程中**第一个真正接近企业生产环境的完整案例**。

很多人在学习 Kubernetes 时有一个问题：

> 「我知道 Deployment、Service、Ingress、ConfigMap、Secret、HPA，但不知道它们怎么组合起来。」

这一章，就是回答这个问题。

> **本章是一个完整案例。建议反复阅读，并画出自己的架构图。**

------

### 本章学习目标

学完本章，你应该能够回答：

- 一个完整的 Kubernetes 项目包含哪些资源？
- 前端、后端、Redis 是如何协作的？
- 为什么前后端通常会拆成两个 Deployment？
- ConfigMap 和 Secret 应该如何组织？
- Helm 如何统一管理整个项目？
- 一次部署到底创建了哪些 Kubernetes 对象？

------

### 第一节：先认识整个系统

假设，我们要部署一个企业常见的系统：

```
Vben Admin（Vue3）
        │
        ▼
ASP.NET Core API
        │
        ▼
Redis
```

对应到 Kubernetes 中，大致会变成：

```
                Internet
                    │
                    ▼
              Ingress Controller
                    │
      ┌─────────────┴─────────────┐
      ▼                           ▼
 Frontend Service            Backend Service
      │                           │
      ▼                           ▼
Vue3 Deployment          ASP.NET Core Deployment
                                  │
                                  ▼
                            Redis Service
                                  │
                                  ▼
                           Redis StatefulSet
```

> **这是企业最常见的三层结构之一。**

------

### 第二节：为什么前后端要分开部署？

很多新人第一次接触 Kubernetes，会问：

> 为什么不把 Vue 和 ASP.NET Core 放进一个 Pod？

答案是：**职责不同，生命周期也不同。**

例如：

前端：

```
v1.0.0
```

后端：

```
v1.0.3
```

今天：

后端修复了一个 Bug。

需要升级。

如果：

放在一个 Pod。

意味着：

```
Vue
+
API
```

一起：

重新部署。

实际上：

前端根本没有变化。

因此：

企业一般采用：

```
Frontend Deployment

Backend Deployment
```

分别管理。

这样：

两边可以独立扩容、独立升级、独立回滚。

------

### 第三节：整个项目会有哪些 Kubernetes 资源？

一个典型项目可能包含：

| 资源                   | 数量 | 作用                   |
| ---------------------- | ---- | ---------------------- |
| Namespace              | 1    | 隔离整个项目           |
| Deployment（Frontend） | 1    | Vue 前端               |
| Deployment（Backend）  | 1    | ASP.NET Core API       |
| StatefulSet（Redis）   | 1    | Redis 服务             |
| Service                | 3    | 对外或内部访问         |
| Ingress                | 1    | HTTP 路由              |
| ConfigMap              | 2    | 非敏感配置             |
| Secret                 | 2    | 数据库密码、JWT 密钥等 |
| HPA                    | 2    | 前后端自动扩缩容       |
| PVC                    | 1    | Redis 持久化           |

所以：

一次：

```
helm install
```

实际上：

可能：

创建：

十几个：

Kubernetes 资源。

------

### 第四节：为什么 Redis 用 StatefulSet？

回忆我们前面学过的内容。

Redis：

需要：

```
数据
```

不能：

随便：

删除。

因此：

不能：

用：

```
Deployment
```

而是：

```
StatefulSet
```

因为：

Redis：

需要：

```
固定名称

固定存储

PVC
```

如果只是开发环境，也可以使用 Deployment 部署 Redis，但生产环境通常推荐 StatefulSet。

------

### 第五节：ASP.NET Core Deployment

它主要负责：

```
接收 HTTP 请求

执行业务逻辑

访问数据库

访问 Redis

返回 JSON
```

通常包含：

```
Deployment
        │
        ▼
Pod
        │
        ▼
Container
```

Container 内部运行：

```
dotnet Order.Api.dll
```

镜像：

例如：

```
mycompany/order-api:1.0.0
```

------

### 第六节：Vue Deployment

Vue 项目通常已经编译完成。

镜像里面一般运行的是：

```
Nginx
```

里面放着：

```
dist/
```

例如：

```
/usr/share/nginx/html
```

所以：

Vue Deployment：

实际上：

不是运行 Node.js。

而是：

运行：

```
Nginx
```

提供：

静态文件。

这是绝大多数 Vue、React、Angular 项目的生产部署方式。

------

### 第七节：ConfigMap 应该放什么？

例如：

ASP.NET Core：

```
Logging

CacheTime

FeatureFlags

ApiUrl
```

Vue：

```
API_BASE_URL

APP_NAME

Theme
```

这些：

都：

适合：

ConfigMap。

因为：

它们：

不是：

秘密。

------

### 第八节：Secret 应该放什么？

企业里面：

一般：

放：

```
SQL Password

Redis Password

JWT Secret

OAuth Client Secret

SMTP Password

第三方 API Key
```

一句话：

> **任何泄露后可能造成安全问题的内容，都应该放入 Secret 或更专业的密钥管理系统。**

------

### 第九节：Ingress 如何路由？

假设：

域名：

```
app.company.com
```

访问：

首页：

```
/
```

进入：

Vue。

API：

```
/api
```

进入：

ASP.NET Core。

整个：

路由：

如下：

```
                app.company.com

                       │

          ┌────────────┴────────────┐

          ▼                         ▼

          /                      /api

          │                         │

          ▼                         ▼

Frontend Service         Backend Service
```

> 也就是说，用户看到的是一个域名，但 Ingress 会根据路径把请求分发到不同的 Service。

------

### 第十节：整个 Helm Chart 应该如何组织？

企业里面，通常会有两种组织方式。

#### 方案一：一个总 Chart（推荐中小型项目）

```
my-system/

Chart.yaml

values.yaml

templates/

    frontend-deployment.yaml

    backend-deployment.yaml

    redis-statefulset.yaml

    ingress.yaml

    services.yaml
```

适合：

中小项目。

一次：

部署：

整个系统。

------

#### 方案二：多个 Chart（推荐大型微服务）

```
frontend-chart/

backend-chart/

redis-chart/

gateway-chart/
```

每个：

独立：

发布。

大型公司：

基本：

都是：

这种。

原因：

每个：

微服务：

都有：

自己的：

生命周期。

------

### 第十一节：一次 helm install 到底发生了什么？

假设：

执行：

```
helm upgrade --install order-system .
```

实际上：

Helm：

做了：

下面：

这些：

事情：

```
读取 values.yaml

        │

        ▼

渲染 Deployment

渲染 Service

渲染 ConfigMap

渲染 Secret

渲染 Ingress

渲染 HPA

渲染 StatefulSet

        │

        ▼

生成普通 YAML

        │

        ▼

提交 API Server

        │

        ▼

Scheduler 调度

        │

        ▼

Pod 创建

        │

        ▼

Service 建立

        │

        ▼

Ingress 生效

        │

        ▼

用户访问成功
```

你会发现：

Helm 只是**开始**。

真正执行资源创建和调度的，依然是 Kubernetes。

------

### 第十二节：部署后的检查顺序（企业运维经验）

很多新手部署成功后，只会执行：

```
kubectl get pods
```

其实，生产环境更推荐按照下面的顺序检查：

1. **Pod 是否正常运行**

```
kubectl get pods
```

确认状态为 `Running`，没有频繁重启（`RESTARTS` 不断增加）。

------

1. **Deployment 是否全部就绪**

```
kubectl get deployment
```

检查 `READY` 是否达到期望副本数，例如 `3/3`。

------

1. **Service 是否创建成功**

```
kubectl get svc
```

确认 ClusterIP、NodePort 或 LoadBalancer 信息是否符合预期。

------

1. **Ingress 是否获取地址**

```
kubectl get ingress
```

如果使用云平台，还需要等待外部 IP 或域名生效。

------

1. **查看事件**

```
kubectl get events --sort-by=.metadata.creationTimestamp
```

很多调度失败、镜像拉取失败、PVC 挂载失败，都能在这里看到线索。

------

1. **查看日志**

```
kubectl logs <pod-name>
```

如果 Pod 正常启动但应用异常，日志通常是第一手信息。

> 后面的"生产故障排查"章节，我们会系统学习这些命令。

------

### 第十三节：为什么企业喜欢这种部署方式？

因为它带来了几个非常重要的能力：

- **标准化**：所有项目都遵循相同的部署结构。
- **可重复**：开发、测试、生产只需要切换 `values` 文件。
- **可回滚**：Helm 保留版本历史，升级失败可快速恢复。
- **可扩展**：后续增加 Prometheus、Grafana、Kafka 等组件，只需新增 Chart 或模板。

这也是 Helm 成为 Kubernetes 主流交付方式的重要原因。

------

### 本章总结（建议牢记）

请记住一个完整 Kubernetes 项目的核心组成：

1. **前端（Vue/Nginx）和后端（ASP.NET Core）通常分别部署为不同的 Deployment。**
2. **Redis、MySQL 等有状态服务优先使用 StatefulSet。**
3. **ConfigMap 保存普通配置，Secret 保存敏感信息。**
4. **Ingress 负责把一个域名的不同路径路由到不同 Service。**
5. **一次 `helm install` 往往会创建十几个 Kubernetes 资源，而不是只有一个 Deployment。**
6. **部署完成后，应按 Pod → Deployment → Service → Ingress → Events → Logs 的顺序检查。**

------

### 到这里，你已经具备了部署完整业务系统的知识

你现在已经能够理解一个典型企业项目从 Helm 到 Kubernetes 的完整流程，也理解了各类资源之间的协作关系。

不过，一个项目部署成功，并不意味着工作结束。

真正的生产环境里，运维工程师每天做得最多的事情其实是：

- 服务变慢了怎么办？
- Pod 为什么不断重启？
- 用户反馈接口报错，如何定位？
- 如何知道 CPU、内存是否达到瓶颈？
- 日志应该去哪里看？

这些问题都离不开**可观测性（Observability）**。

------

### 下一章预告：生产运维第一课——日志体系（Logging）

下一章，我们将正式进入生产运维三大支柱之一：**日志（Logging）**。

我们会详细讲解：

- 为什么 `kubectl logs` 不够用？
- Kubernetes 日志是如何产生的？
- 容器日志为什么不能直接写到文件？
- 什么是 stdout / stderr？
- 什么是节点日志？
- 如何收集整个集群的日志？
- 为什么生产环境几乎都会使用 **Loki + Grafana** 或 **EFK（Elasticsearch + Fluent Bit + Kibana）**？
- 如何为 ASP.NET Core 配置结构化日志（如 Serilog），让日志更容易排查问题？

这一章结束后，你将开始真正掌握**生产环境运维**所需的第一项核心能力。

## 第二阶段 第五章：生产运维第一课——Kubernetes 日志体系（Logging）

提示词：

```
开始下一章
```

正文：

很好。

从这一章开始，我们进入 **Kubernetes 生产运维三大支柱**。

整个现代运维（包括 Kubernetes、Docker、云原生）都有一个非常重要的概念：

> **Observability（可观测性）**

可观测性主要由三部分组成：

```
Observability（可观测性）

├── Logging（日志）
├── Metrics（指标）
└── Tracing（链路追踪）
```

以后你无论去腾讯、阿里、字节、美团，还是国外公司，这三个概念都会反复出现。

今天，我们先学习第一部分：

> **Logging（日志）**

#### 本章学习目标

学习完本章，你应该能够回答：

- Kubernetes 日志到底存在哪里？
- 为什么容器不能像以前一样直接写日志文件？
- stdout、stderr 到底是什么？
- kubectl logs 的原理是什么？
- 为什么 kubectl logs 不适合生产环境？
- 企业如何收集整个集群日志？
- ASP.NET Core 如何正确输出日志？
- 什么是结构化日志？

------

### 第一节：先回忆一下传统服务器

以前。

没有 Docker。

没有 Kubernetes。

部署 ASP.NET Core：

一般都是：

```
Windows IIS

或者

Linux + Nginx
```

日志通常写到：

例如：

```
D:\Logs\api.log
```

或者：

```
/var/log/order-api.log
```

出了问题。

SSH：

进入服务器：

```
tail -f api.log
```

结束。

是不是很简单？

------

### 第二节：为什么 Kubernetes 不建议写日志文件？

现在。

你的项目运行在：

Pod。

例如：

```
Node1

┌────────────────────┐

Pod

Order.API

└────────────────────┘
```

突然。

Pod：

挂了。

Deployment：

重新创建：

```
Node2

┌────────────────────┐

新的 Pod

Order.API

└────────────────────┘
```

请问。

旧 Pod：

里面：

日志文件：

还在吗？

答案：

> **没了。**

因为：

Pod：

本身：

就是：

临时的。

这就是云原生中非常重要的一条原则：

> **容器应该是无状态（Stateless）的。**

因此：

日志不能依赖容器本地磁盘。

------

### 第三节：那日志写到哪里？

答案：

很多新人第一次知道都会觉得很奇怪：

> **直接输出到控制台（Console）。**

也就是：

```
Console.WriteLine(...)
```

或者：

ASP.NET Core：

```
logger.LogInformation(...)
```

最终：

都会输出：

```
stdout
```

或者：

```
stderr
```

------

### 第四节：什么是 stdout 和 stderr？

这是 Linux 非常基础，但又非常重要的概念。

每个进程启动时，默认都有三个标准流：

| 名称   | 作用         |
| ------ | ------------ |
| stdin  | 标准输入     |
| stdout | 标准输出     |
| stderr | 标准错误输出 |

例如：

C#：

```
Console.WriteLine("Hello");
```

其实：

就是：

输出到：

```
stdout
```

例如：

抛异常：

```
NullReferenceException
```

通常：

输出：

```
stderr
```

Docker：

就是收集：

这两个：

输出。

------

#### 一个生活中的例子

想象你在办公室工作：

- **stdout**：你正常汇报工作。
- **stderr**：你举手报告："出问题了！"

领导（Docker）会把两种信息都记录下来。

------

### 第五节：Docker 如何保存日志？

很多人以为：

Docker：

不会：

保存。

其实：

会。

流程：

如下：

```
ASP.NET Core

        │

Console.WriteLine()

        │

stdout

        │

Docker Engine

        │

json log

        │

磁盘
```

例如：

Docker：

默认：

会保存：

JSON：

日志。

所以：

执行：

```
docker logs 容器ID
```

其实：

就是：

读取：

这些：

日志。

------

### 第六节：kubectl logs 到底做了什么？

现在。

Docker：

变成：

Kubernetes。

流程：

变成：

```
ASP.NET Core

        │

stdout

        │

Container Runtime（containerd 等）

        │

Node

        │

kubectl logs
```

所以：

执行：

```
kubectl logs order-api-xxxxx
```

其实：

就是：

读取：

容器：

stdout。

而：

不是：

读取：

某个：

日志文件。

这一点：

很多新人：

都会：

误解。

------

### 第七节：kubectl logs 常用命令

查看：

日志：

```
kubectl logs pod-name
```

持续：

查看：

```
kubectl logs -f pod-name
```

查看：

上一轮崩溃前的日志：

```
kubectl logs --previous pod-name
```

查看：

Deployment：

```
kubectl logs deployment/order-api
```

如果：

Deployment：

有：

多个：

Pod。

默认：

读取：

其中：

一个。

如果要查看指定 Pod，建议先：

```
kubectl get pods
```

再查看对应 Pod。

------

### 第八节：为什么 kubectl logs 不适合生产环境？

假设：

你的：

API：

有：

```
20

Pod
```

分布：

```
Node1

Node2

Node3

Node4
```

用户：

报错。

你：

怎么办？

难道：

一个：

一个：

执行：

```
kubectl logs
```

二十次？

显然：

不现实。

更大的问题是：

Pod 删除后，本地日志最终也会消失，因此很难做长期分析。

------

### 第九节：企业如何收集日志？

生产环境。

都会：

部署：

日志：

采集器。

例如：

```
Node

────────────

Pod A

Pod B

Pod C

────────────

Fluent Bit
```

Fluent Bit：

负责：

读取：

Node：

所有：

Container：

日志。

然后：

发送：

到：

日志平台。

------

整个：

流程：

如下：

```
ASP.NET Core

        │

stdout

        │

Container Runtime

        │

Fluent Bit

        │

Loki / Elasticsearch

        │

Grafana / Kibana
```

以后。

查看：

日志。

不用：

SSH。

不用：

kubectl。

直接：

打开：

网页。

搜索。

即可。

------

### 第十节：目前企业主流日志方案

目前最常见的有两种。

------

#### 方案一：Loki + Grafana（越来越流行）

```
ASP.NET Core

        │

stdout

        │

Fluent Bit

        │

Loki

        │

Grafana
```

优点：

- 资源占用较低
- 与 Grafana 集成非常好
- 运维成本相对较低
- 特别适合 Kubernetes

近年来，越来越多的新项目会优先考虑这一方案。

------

#### 方案二：EFK

也就是：

```
Elasticsearch

Fluent Bit（或 Fluentd）

Kibana
```

流程：

```
stdout

↓

Fluent Bit

↓

Elasticsearch

↓

Kibana
```

优点：

- 搜索能力强
- 生态成熟
- 很多老项目仍在使用

缺点：

- Elasticsearch 占用资源较高
- 运维复杂度相对更高

------

### 第十一节：ASP.NET Core 应该如何写日志？

很多新人：

喜欢：

```
Console.WriteLine("开始执行");
```

可以。

但是。

企业：

一般：

不用。

而是：

使用：

```
ILogger<OrderService>
```

例如：

```
_logger.LogInformation("Order {OrderId} created successfully.", orderId);
```

为什么？

因为：

日志：

会：

自动：

包含：

```
时间

日志级别

类别

消息
```

更重要的是，它支持**结构化日志**。

------

### 第十二节：什么是结构化日志？

很多新手会写：

```
_logger.LogInformation("订单创建成功：" + orderId);
```

虽然能工作。

但：

更推荐：

```
_logger.LogInformation(
    "订单创建成功，订单号：{OrderId}",
    orderId);
```

为什么？

因为：

日志平台：

能够：

识别：

```
OrderId
```

以后。

Grafana。

Kibana。

可以：

直接：

搜索：

```
OrderId=10001
```

这就是：

结构化日志。

> **不要把所有信息拼成字符串，而是把关键字段作为独立属性记录。**

这是现代日志系统最重要的理念之一。

------

### 第十三节：日志级别

ASP.NET Core 常见日志级别如下：

| 级别        | 说明         | 是否常用于生产 |
| ----------- | ------------ | -------------- |
| Trace       | 最详细       | 很少开启       |
| Debug       | 调试信息     | 开发环境常用   |
| Information | 正常业务流程 | ✅ 最常用       |
| Warning     | 潜在问题     | ✅ 常用         |
| Error       | 错误         | ✅ 必须记录     |
| Critical    | 严重故障     | ✅ 必须记录     |

一般建议：

- 开发环境：`Debug` 或 `Information`
- 测试环境：`Information`
- 生产环境：通常以 `Information` 为主，必要时临时提高日志级别排查问题。

------

### 第十四节：日志最佳实践

结合 Kubernetes 和 ASP.NET Core，推荐遵循以下原则：

1. **不要把业务日志写入容器内固定文件。**
2. **统一输出到 stdout / stderr。**
3. **使用 `ILogger<T>` 而不是 `Console.WriteLine()`。**
4. **使用结构化日志，不要简单拼接字符串。**
5. **通过 Fluent Bit 收集日志，再集中存储到 Loki 或 Elasticsearch。**
6. **使用 Grafana 或 Kibana 查询日志，而不是频繁登录节点。**
7. **日志不要记录密码、Token、身份证号等敏感信息。**

------

### 本章总结（建议牢记）

请记住 Kubernetes 日志体系最重要的几点：

1. **Pod 是临时资源，因此不要依赖容器内日志文件。**
2. **Docker/Kubernetes 默认收集 stdout 和 stderr。**
3. **`kubectl logs` 实际上查看的是容器标准输出，而不是某个日志文件。**
4. **生产环境需要集中式日志平台，而不是逐个 Pod 查看日志。**
5. **推荐使用 `ILogger<T>` 和结构化日志，提高检索和分析能力。**
6. **Loki + Grafana 和 EFK 都是主流日志方案，各有适用场景。**

------

### 到这里，你已经掌握了生产环境日志体系

你已经知道：

```
ASP.NET Core
        │
ILogger<T>
        │
stdout / stderr
        │
Container Runtime
        │
Fluent Bit
        │
Loki / Elasticsearch
        │
Grafana / Kibana
```

这条链路，就是现代 Kubernetes 日志系统的核心工作流程。

------

### 下一章预告：生产运维第二课——监控体系（Prometheus + Grafana）

日志告诉我们：

> **发生了什么。**

下一章学习的**监控（Metrics）**则回答另一个问题：

> **系统现在运行得怎么样？**

我们将深入学习：

- 什么是 Metrics（指标）？
- 为什么仅有日志还不够？
- Prometheus 是如何采集 Kubernetes 指标的？
- 什么是 Exporter？
- 什么是 Alertmanager？
- Grafana 为什么几乎成为监控平台的标准？
- 如何监控 ASP.NET Core API、Redis、Node、Pod、Ingress？
- 如何配置 CPU、内存、响应时间、错误率等告警？

这一章结束后，你将具备搭建和理解**企业级 Kubernetes 监控体系**的能力。

## 第二阶段 第六章：生产运维第二课——Prometheus + Grafana 监控体系

提示词：

```
开始下一章
```

正文：

非常好。

这一章，是整个 Kubernetes 运维里面**最重要的一章之一**。

很多新人都会认为：

> 有日志，不就够了吗？

实际上。

真正的生产环境。

**日志（Logging）和监控（Metrics）解决的是两个完全不同的问题。**

例如：

凌晨 3 点。

你的手机收到一条告警：

```
Order API CPU 持续超过 95%
```

这时候。

你还没有看日志。

就已经知道：

系统出问题了。

这就是：

> **Metrics（监控指标）**

它负责：

**提前发现问题。**

而日志负责：

**定位问题原因。**

> **这是 Kubernetes 运维中必须掌握的内容。**

------

### 本章学习目标

学习完本章，你应该能够回答：

- 什么是 Metrics？
- Prometheus 为什么会成为 Kubernetes 的事实标准？
- Prometheus 是如何采集数据的？
- Exporter 是什么？
- Grafana 为什么几乎人人都在用？
- Alertmanager 有什么作用？
- 如何监控 ASP.NET Core？
- 企业一般监控哪些指标？
- 什么是 RED、USE 监控方法？

------

### 第一节：什么是 Metrics（指标）？

先来看一个例子。

假设：

你的 API：

目前运行情况如下：

```
CPU：28%

Memory：412MB

QPS：560

Response Time：35ms

Error Rate：0.03%
```

这些数字。

就是：

Metrics。

一句话：

> **Metrics 是用数字描述系统运行状态。**

它最大的特点是：

- 可以画曲线
- 可以做统计
- 可以设置告警
- 可以观察趋势

例如：

CPU：

```
10%

20%

30%

95%

100%
```

是不是：

一眼就能发现问题？

------

### 第二节：为什么日志不够？

假设：

凌晨。

系统：

突然：

很慢。

如果：

只有：

日志。

你需要：

```
打开日志

↓

搜索 Error

↓

搜索 Exception

↓

分析原因
```

但是。

如果：

先看：

Grafana：

你可能：

一分钟：

就发现：

```
CPU：

99%
```

或者：

```
Memory：

95%
```

或者：

```
Redis：

连接数爆满
```

是不是：

快很多？

所以。

一句话：

> **Metrics 告诉你哪里出了问题，Logs 告诉你为什么出了问题。**

------

### 第三节：Prometheus 是什么？

一句话：

> **Prometheus 是一个时序数据库（Time Series Database）+ 监控系统。**

它专门保存：

```
时间

+

指标
```

例如：

```
09:00 CPU 25%

09:01 CPU 28%

09:02 CPU 30%

09:03 CPU 95%
```

这些数据。

都会保存。

以后：

Grafana：

画：

曲线。

就是：

读取：

这些：

历史数据。

------

### 第四节：Prometheus 如何采集数据？

很多新人：

以为：

Prometheus：

主动进入 Pod。

其实：

不是。

绝大多数情况下，它采用的是：

> **Pull（主动拉取）模式。**

流程：

```
Prometheus

        │

HTTP GET /metrics

        │

        ▼

ASP.NET Core

返回指标
```

例如：

每：

15 秒。

Prometheus：

访问：

```
http://pod-ip:8080/metrics
```

然后：

保存：

结果。

------

#### 一个生活例子

想象老师点名：

- **Pull**：老师主动叫每个同学回答。
- **Push**：每个同学自己跑去告诉老师。

Prometheus 默认采用前者。

------

### 第五节：Exporter 是什么？

很多程序。

不会：

直接：

输出：

监控数据。

怎么办？

于是：

出现：

Exporter。

一句话：

> **Exporter 是把各种系统的数据转换成 Prometheus 能理解的格式。**

例如：

```
Node

↓

Node Exporter
```

采集：

```
CPU

Memory

Disk

Network
```

------

Redis：

```
Redis

↓

Redis Exporter
```

采集：

```
Memory

Hit Rate

Connected Clients

Commands
```

------

MySQL：

```
MySQL

↓

MySQL Exporter
```

采集：

```
Slow Query

Connections

TPS
```

所以：

Exporter：

可以理解成：

> **翻译官。**

------

### 第六节：ASP.NET Core 如何提供指标？

ASP.NET Core：

通常：

通过：

OpenTelemetry 或 Prometheus 相关库暴露：

```
/metrics
```

例如：

访问：

```
http://localhost:8080/metrics
```

可能：

返回：

```
http_requests_total 18231

http_request_duration_seconds 0.032

process_cpu_seconds_total 5.2
```

Prometheus：

就是：

读取：

这些：

数据。

> 近年来，越来越多团队会采用 **OpenTelemetry** 统一输出 Metrics、Logs 和 Traces，再由 Prometheus 等系统采集。

------

### 第七节：Prometheus 整个工作流程

这是必须记住的一张图。

```
ASP.NET Core

        │

/metrics

        │

        ▼

Prometheus

        │

保存指标

        │

        ▼

Grafana

        │

画图

        │

        ▼

Alertmanager

        │

企业微信

邮件

Slack

Teams
```

是不是：

其实：

很简单？

------

### 第八节：Grafana 是什么？

一句话：

> **Grafana 是监控数据可视化平台。**

Prometheus：

保存：

数字。

Grafana：

负责：

展示。

例如：

CPU：

```
██████████
```

内存：

```
██████
```

QPS：

```
────────╮
        │
────────╯
```

Grafana：

可以：

做：

Dashboard。

例如：

```
Order API Dashboard

CPU

Memory

QPS

Response Time

Error Rate
```

企业里面。

几乎：

每天：

都会：

打开。

------

### 第九节：Alertmanager 是什么？

Prometheus：

负责：

采集。

Grafana：

负责：

展示。

那么：

谁：

负责：

报警？

答案：

Alertmanager。

例如：

规则：

```
CPU > 90%

持续：

5 分钟
```

Alertmanager：

发送：

```
企业微信

邮件

Slack

Microsoft Teams
```

所以。

凌晨：

手机：

响。

就是：

它：

发的。

------

### 第十节：企业一般监控哪些内容？

一个典型的 ASP.NET Core + Kubernetes 项目，通常会监控四个层面：

| 层面       | 典型指标                                  |
| ---------- | ----------------------------------------- |
| Node       | CPU、内存、磁盘、网络                     |
| Kubernetes | Pod 数量、重启次数、Pending Pod、HPA 状态 |
| 应用       | QPS、响应时间、错误率、并发数             |
| 中间件     | Redis、MySQL、RabbitMQ、Kafka 等          |

这些组合起来，基本能覆盖绝大多数生产问题。

------

### 第十一节：RED 方法（应用监控）

Google 提出了很多监控理念。

其中。

最经典：

就是：

RED。

```
R

Rate

请求数量

────────────

E

Errors

错误率

────────────

D

Duration

响应时间
```

例如：

你的 API：

应该：

至少：

监控：

```
QPS

500

────────

Error Rate

0.02%

────────

Latency

35ms
```

如果：

突然：

```
Latency

35ms

↓

1200ms
```

不用：

看：

日志。

已经：

知道：

有：

问题。

------

### 第十二节：USE 方法（基础设施监控）

除了应用。

Node。

也：

需要：

监控。

这里：

最经典：

的是：

USE。

```
Utilization

利用率

────────────

Saturation

饱和度

────────────

Errors

错误
```

例如：

CPU：

```
95%
```

利用率：

很高。

磁盘：

IO：

排队：

很多。

说明：

饱和。

网卡：

丢包。

说明：

错误。

这套方法非常适合分析服务器、Kubernetes 节点和存储性能。

------

### 第十三节：ASP.NET Core 最值得监控的指标

如果你主要开发 ASP.NET Core，建议重点关注：

- HTTP 请求数（Requests / QPS）
- HTTP 响应时间（Latency）
- HTTP 错误率（4xx / 5xx）
- CPU 使用率
- 内存使用量
- GC 次数与暂停时间
- 数据库连接池使用情况
- Redis 命中率
- 后台任务队列长度

这些指标足以帮助你发现大部分性能问题。

------

### 第十四节：监控最佳实践

生产环境建议：

1. **所有应用都暴露 Metrics 接口（推荐使用 OpenTelemetry）。**
2. **使用 Prometheus 定时采集。**
3. **Grafana 建立统一 Dashboard。**
4. **Alertmanager 配置关键告警。**
5. **不要只监控基础设施，也要监控业务指标。**
6. **告警不要过多，避免"告警疲劳"。真正影响业务的问题才应立即通知。**

------

### 第十五节：Logging、Metrics、Tracing 的关系

最后。

一定要理解：

这三者：

不是：

替代。

而是：

互补。

```
Logging

↓

为什么报错？

────────────

Metrics

↓

什么时候开始变慢？

────────────

Tracing

↓

到底是哪一个服务慢？
```

三者：

合起来。

才是：

真正：

完整：

的：

可观测性。

------

### 本章总结（建议牢记）

请记住监控体系最重要的几点：

1. **Metrics 用数字描述系统状态，适合画图、统计和告警。**
2. **Prometheus 是 Kubernetes 最主流的监控系统，默认采用 Pull 模式采集数据。**
3. **Exporter 用于把各种系统的数据转换为 Prometheus 可识别的格式。**
4. **Grafana 负责展示监控面板，Alertmanager 负责发送告警。**
5. **应用建议采用 RED 方法监控，基础设施建议采用 USE 方法监控。**
6. **日志、指标和链路追踪共同组成现代可观测性体系。**

------

### 到这里，你已经理解了企业级监控架构

```
ASP.NET Core
        │
   /metrics
        │
        ▼
Prometheus
        │
        ├── 存储时序数据
        │
        ├── Alertmanager（告警）
        │
        ▼
Grafana（可视化）
```

这也是目前 Kubernetes 社区和绝大多数企业采用的标准方案。

------

### 下一章预告：生产运维第三课——链路追踪（Tracing）

到目前为止：

- **日志（Logs）**告诉我们：发生了什么。
- **指标（Metrics）**告诉我们：什么时候开始变慢。

但是，还有一个问题没有解决：

> **到底是哪一个服务、哪一个数据库查询、哪一次 RPC 调用导致了请求变慢？**

下一章，我们将学习可观测性的最后一块拼图——**分布式链路追踪（Tracing）**，包括：

- 什么是 Trace、Span？
- OpenTelemetry 是什么？
- Jaeger、Tempo 的作用
- 如何追踪一次请求经过的所有微服务
- 如何为 ASP.NET Core 接入 OpenTelemetry
- 如何快速定位微服务性能瓶颈

学完这一章，你就会完整掌握现代 Kubernetes 生产环境的三大可观测性能力：**Logging、Metrics、Tracing**。

很好。

这一章，是现代 Kubernetes 和微服务体系中最容易让新人觉得抽象，但一旦理解后又会觉得非常自然的一章。

前两章，我们已经学习了：

```
Logs（日志）

回答：

为什么出错？
Metrics（监控）

回答：

什么时候开始变慢？
```

但是。

还有一个问题。

假设你的系统已经拆成了几十个微服务。

例如：

```
用户下单
      │
      ▼
Order API
      │
      ▼
Inventory API
      │
      ▼
Payment API
      │
      ▼
Redis
      │
      ▼
MySQL
```

用户告诉你：

> 下单接口响应了 **8 秒**。

请问：

到底是哪一个服务慢？

Redis？

MySQL？

库存？

支付？

还是网络？

Logs 很难告诉你。

Metrics 也只能告诉你：

> **确实变慢了。**

但是：

到底是谁慢？

不知道。

于是。

第三块拼图出现了：

> **Tracing（链路追踪）**

## 第二阶段 第七章：生产运维第三课——分布式链路追踪（Tracing）

> **这是现代微服务排障的核心能力。**

------

提示词：

```
开始下一章
```

正文：

很好。

这一章，是现代 Kubernetes 和微服务体系中最容易让新人觉得抽象，但一旦理解后又会觉得非常自然的一章。

前两章，我们已经学习了：

```
Logs（日志）

回答：

为什么出错？
Metrics（监控）

回答：

什么时候开始变慢？
```

但是。

还有一个问题。

假设你的系统已经拆成了几十个微服务。

例如：

```
用户下单
      │
      ▼
Order API
      │
      ▼
Inventory API
      │
      ▼
Payment API
      │
      ▼
Redis
      │
      ▼
MySQL
```

用户告诉你：

> 下单接口响应了 **8 秒**。

请问：

到底是哪一个服务慢？

Redis？

MySQL？

库存？

支付？

还是网络？

Logs 很难告诉你。

Metrics 也只能告诉你：

> **确实变慢了。**

但是：

到底是谁慢？

不知道。

于是。

第三块拼图出现了：

> **Tracing（链路追踪）**

### 本章学习目标

学习完本章，你应该能够回答：

- 什么是 Trace？
- 什么是 Span？
- Trace ID 有什么作用？
- OpenTelemetry 为什么越来越重要？
- Jaeger、Tempo 是什么？
- 如何追踪一次 ASP.NET Core 请求？
- 企业为什么一定会做链路追踪？
- Logging、Metrics、Tracing 如何协同工作？

------

### 第一节：为什么需要链路追踪？

先看一个单体应用。

```
Browser

    │

    ▼

ASP.NET Core

    │

    ▼

MySQL
```

这里只有：

一个 API。

定位问题：

很简单。

但是。

如果系统变成：

```
Browser

    │

    ▼

Gateway

    │

    ▼

Order API

    │

    ├───────────────┐

    ▼               ▼

Inventory API   User API

    │               │

    ▼               ▼

Redis          MySQL
```

现在。

一次请求。

可能经过：

五六个服务。

如果：

用户说：

```
下单用了：

6 秒
```

请问：

哪一步：

最慢？

不知道。

------

### 第二节：什么是 Trace？

一句话：

> **Trace 就是一次完整请求的生命周期。**

例如：

用户点击：

```
立即下单
```

整个过程：

```
浏览器

    │

    ▼

Gateway

    │

    ▼

Order API

    │

    ▼

Inventory API

    │

    ▼

Redis

    │

    ▼

MySQL
```

这一整条路径。

就叫：

```
Trace
```

所以。

可以理解为：

> **Trace = 一条完整的请求路线。**

------

### 第三节：什么是 Span？

如果说：

Trace：

是一趟旅行。

那么：

Span：

就是：

旅程中的每一站。

例如：

```
Trace

──────────────

Gateway

30ms

──────────────

Order API

120ms

──────────────

Inventory API

800ms

──────────────

Redis

5ms

──────────────

MySQL

1200ms
```

这里：

每一个：

矩形。

都是：

```
Span
```

一句话：

> **Span 是一次具体操作的耗时记录。**

例如：

一个 Span 可以表示：

- 一次 HTTP 请求
- 一次数据库查询
- 一次 Redis 操作
- 一次消息队列发送
- 一次调用第三方接口

------

#### 一个生活中的例子

假设你从台北去高雄：

```
台北

↓

台中

↓

嘉义

↓

台南

↓

高雄
```

整趟旅行：

就是：

```
Trace
```

每一段高铁：

就是：

```
Span
```

如果：

嘉义到台南：

堵车：

3 小时。

是不是：

马上知道：

哪里：

慢？

------

### 第四节：什么是 Trace ID？

假设：

同一时间：

有：

10000 个用户。

大家：

都：

在：

下单。

系统：

怎么知道：

哪些日志。

属于：

同一次：

请求？

答案：

每一次请求都会生成：

```
Trace ID
```

例如：

```
Trace ID

4fdc7a98...
```

以后：

所有：

服务：

都会：

携带：

这个：

ID。

例如：

```
Gateway

TraceID=A123

↓

Order API

TraceID=A123

↓

Inventory API

TraceID=A123

↓

Redis

TraceID=A123
```

于是。

日志。

监控。

链路。

全部：

关联：

起来。

------

### 第五节：OpenTelemetry 是什么？

以前。

每家公司。

都有：

自己的：

监控 SDK。

后来。

行业发现：

太乱了。

于是。

诞生了：

> **OpenTelemetry（OTel）**

一句话：

> **OpenTelemetry 是可观测性的统一标准。**

它统一了：

```
Logs

Metrics

Tracing
```

现在。

ASP.NET Core。

Java。

Go。

Python。

Node.js。

都：

支持：

OpenTelemetry。

所以。

越来越多企业：

都会：

直接：

接入：

OTel。

------

### 第六节：ASP.NET Core 如何接入 OpenTelemetry？

ASP.NET Core 本身已经很好地支持 OpenTelemetry。

典型流程如下：

```
HTTP 请求

      │

      ▼

OpenTelemetry SDK

      │

生成：

Trace

Span

Metrics

Logs

      │

      ▼

OTLP Exporter

      │

      ▼

Jaeger

Tempo

Prometheus

Loki
```

你会发现：

OpenTelemetry 更像一个"采集标准"，真正负责存储和展示的是后面的系统。

------

### 第七节：Jaeger 是什么？

Jaeger：

是：

最经典：

的：

Trace：

平台。

例如：

打开：

Jaeger：

可能：

看到：

```
Order API

2.8s

──────────────

Gateway

20ms

──────────────

Inventory

300ms

──────────────

Redis

2ms

──────────────

MySQL

2400ms
```

是不是：

一眼：

就知道：

数据库：

最慢？

------

### 第八节：Tempo 又是什么？

近年来。

Grafana 推出了：

```
Tempo
```

作用：

也是：

保存：

Trace。

区别：

大致如下：

| 平台   | 特点                                                  |
| ------ | ----------------------------------------------------- |
| Jaeger | 成熟、功能丰富、学习资料多                            |
| Tempo  | 与 Grafana、Loki、Prometheus 集成更紧密，资源占用较低 |

很多新建 Kubernetes 平台，会采用：

```
Grafana

+

Prometheus

+

Loki

+

Tempo
```

形成统一的可观测性平台。

------

### 第九节：一次请求是如何被追踪的？

假设：

浏览器：

访问：

```
POST /api/order
```

整个过程：

```
Browser

        │

        ▼

Gateway

        │

        ▼

Order API

        │

        ▼

Inventory API

        │

        ▼

Redis

        │

        ▼

MySQL
```

每一步：

都会：

记录：

```
开始时间

结束时间

耗时

状态

Trace ID

Span ID
```

最后：

Jaeger：

画成：

时间线。

例如：

```
Gateway

████

Order API

████████████

Inventory

██████████████████████

Redis

█

MySQL

██████████████████████████
```

是不是：

马上：

知道：

MySQL：

慢？

------

### 第十节：Logging、Metrics、Tracing 如何协同？

这是整个可观测性最重要的一张图。

假设：

用户：

投诉：

```
订单：

很慢
```

你的排查步骤：

第一步。

Grafana：

看：

Metrics。

发现：

```
Latency

40ms

↓

1800ms
```

说明：

真的：

变慢。

------

第二步。

Jaeger：

查看：

Trace。

发现：

```
Inventory API

耗时：

1300ms
```

已经：

定位：

到：

服务。

------

第三步。

Loki：

搜索：

```
TraceID=A123
```

马上：

找到：

所有：

日志。

例如：

```
SQL Timeout

Redis Retry

Network Timeout
```

最终。

定位：

问题。

整个过程：

```
Metrics

↓

发现问题

↓

Tracing

↓

定位服务

↓

Logging

↓

定位代码
```

这就是企业排障的标准流程。

------

### 第十一节：为什么要把 Trace ID 写进日志？

如果日志里没有 Trace ID：

```
Error

Timeout

NullReferenceException
```

你根本不知道：

它属于哪个请求。

因此。

企业一般都会让日志自动带上：

```
TraceId

SpanId

RequestId

UserId（按需）
```

这样：

当你在日志平台搜索某个 Trace ID 时，就能看到整个请求相关的所有日志。

这也是结构化日志和链路追踪结合的重要价值。

------

### 第十二节：ASP.NET Core 最佳实践

对于 ASP.NET Core 项目，推荐逐步采用以下方案：

- 使用 `ILogger<T>` 输出结构化日志。
- 使用 OpenTelemetry 自动采集 HTTP、数据库等 Trace。
- 使用 OTLP Exporter 将数据发送到可观测性平台。
- 使用 Prometheus 采集 Metrics。
- 使用 Loki 收集日志。
- 使用 Grafana 展示 Metrics、Logs、Traces。
- 为关键业务（如下单、支付、登录）建立专门的 Dashboard。

------

### 第十三节：现代企业的可观测性架构

一个典型架构如下：

```
                Browser
                    │
                    ▼
             ASP.NET Core API
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     Logs        Metrics      Traces
        │           │           │
        ▼           ▼           ▼
      Loki     Prometheus     Tempo
           └──────┬────────────┘
                  ▼
              Grafana
```

你会发现：

Grafana 已经不仅仅是一个监控工具，而是很多企业统一的可观测性入口。

------

### 第十四节：生产环境最佳实践

建议遵循以下原则：

1. **所有服务都开启 OpenTelemetry。**
2. **所有日志都携带 Trace ID。**
3. **优先使用自动埋点，再针对核心业务增加手动埋点。**
4. **不要采集所有细节，避免产生过多 Trace 数据。**
5. **关注慢请求（Slow Trace），而不是每一个普通请求。**
6. **结合 Metrics、Tracing、Logging，而不是单独依赖其中一种。**

------

### 本章总结（建议牢记）

请记住链路追踪最重要的几点：

1. **Trace 表示一次完整请求，Span 表示请求中的一个具体步骤。**
2. **Trace ID 是关联日志、指标和链路的关键。**
3. **OpenTelemetry 已成为现代可观测性的统一标准。**
4. **Jaeger 和 Tempo 都是主流的 Trace 存储与查询系统。**
5. **链路追踪最适合定位微服务之间的性能瓶颈。**
6. **企业排障通常遵循：Metrics 发现问题 → Tracing 定位服务 → Logging 定位代码。**

------

### 到这里，你已经掌握了现代 Kubernetes 可观测性的完整体系

```
                 Application
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
     Logging       Metrics       Tracing
        │             │             │
      Loki      Prometheus       Tempo
        └─────────────┬─────────────┘
                      ▼
                  Grafana
```

这是目前云原生领域最主流、也是最值得掌握的一套可观测性架构。

------

### 下一章预告：Kubernetes 生产故障排查（Troubleshooting）

到目前为止，你已经会：

- 部署应用
- 编写 Helm Chart
- 收集日志
- 查看监控
- 分析链路

下一章，我们将进入真正的**生产运维实战**。

你将学习：

- Pod 一直是 `Pending` 怎么办？
- Pod 一直 `CrashLoopBackOff` 怎么办？
- `ImagePullBackOff` 如何排查？
- Service 无法访问怎么办？
- Ingress 不生效怎么办？
- 为什么探针（Liveness/Readiness）会导致应用不断重启？
- 如何建立一套系统化的 Kubernetes 故障排查思路？

这一章开始，你将从"会部署"真正迈向"会运维生产环境"。