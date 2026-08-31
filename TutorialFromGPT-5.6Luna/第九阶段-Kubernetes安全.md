# 第 24 章：Kubernetes 权限模型

Kubernetes 安全中，**权限模型是最核心的基础之一**。

在生产环境里，不能让所有人都拥有：

```
cluster-admin
```

否则任何一个账号泄露、Token 泄露，甚至一个错误的 `kubectl` 命令，都可能影响整个集群。

Kubernetes 权限模型主要解决两个问题：

```
谁可以访问 Kubernetes？
        ↓
Authentication

访问之后可以做什么？
        ↓
Authorization
        ↓
RBAC
```

本章重点掌握：

```
User
 ↓
Authentication
 ↓
Authorization
 ↓
RBAC
 ↓
Role / ClusterRole
 ↓
RoleBinding / ClusterRoleBinding
```

------

## 24.1 Authentication

### 24.1.1 Authentication 是什么

Authentication，简称 **AuthN**，中文通常叫：

> **身份认证**

它回答的问题是：

> **“你是谁？”**

例如：

```
Alice
Bob
admin
CI/CD
Pod
ServiceAccount
```

当执行：

```
kubectl get pods
```

Kubernetes API Server 首先需要知道：

> 这个请求是谁发出来的？

------

### 24.1.2 Kubernetes Authentication 的位置

典型请求：

```
kubectl
   │
   │ HTTPS
   ▼
API Server
   │
   ▼
Authentication
   │
   ▼
确认身份
```

认证成功后，才进入 Authorization。

------

### 24.1.3 常见身份来源

Kubernetes 支持多种认证方式，例如：

```
X.509 Client Certificate
Bearer Token
ServiceAccount Token
OIDC
Webhook
```

生产环境通常不会自己维护大量 Kubernetes 本地用户，而会结合企业身份系统，例如：

```
Microsoft Entra ID
Okta
LDAP
OIDC Identity Provider
```

核心思想是：

```
企业身份系统
      ↓
Authentication
      ↓
Kubernetes API Server
```

------

### 24.1.4 Authentication 不决定权限

这是初学者最容易混淆的地方。

假设：

```
Alice
```

已经成功认证。

这只能证明：

```
Alice 确实是 Alice
```

并不代表：

```
Alice 可以删除 Pod
Alice 可以修改 Deployment
Alice 可以删除 Namespace
```

这些属于：

> **Authorization**

所以：

```
Authentication
= 你是谁？

Authorization
= 你能做什么？
```

------

## 24.2 Authorization

### 24.2.1 Authorization 是什么

Authorization，简称 **AuthZ**，中文叫：

> **授权**

它回答：

> **“这个身份有没有权限执行这个操作？”**

例如 Alice 执行：

```
kubectl get pods -n shop
```

API Server 可能判断：

```
User: Alice
Verb: get
Resource: pods
Namespace: shop
```

然后检查：

```
Alice 有没有这个权限？
```

如果有：

```
允许
```

否则：

```
Forbidden
```

------

### 24.2.2 一个完整请求

例如：

```
kubectl delete pod api-xxx -n shop
```

可以抽象成：

```
kubectl
   │
   ▼
Authentication
   │
   │ Alice
   ▼
Authorization
   │
   │
   ├── Verb: delete
   ├── Resource: pods
   └── Namespace: shop
   │
   ▼
RBAC
   │
   ▼
Allow / Deny
```

------

### 24.2.3 常见错误

没有权限时：

```
Error from server (Forbidden)
```

例如：

```
User "alice" cannot delete resource "pods"
in API group "" in the namespace "shop"
```

这个错误非常有价值。

它已经告诉你：

```
User
Verb
Resource
API Group
Namespace
```

可以根据这些信息反查 RBAC 配置。

------

## 24.3 RBAC

### 24.3.1 RBAC 是什么

RBAC：

> **Role-Based Access Control**

中文：

> **基于角色的访问控制**

Kubernetes 使用 RBAC 来管理权限。

它的核心思想不是：

```
Alice 可以 get Pod
Alice 可以 delete Pod
Alice 可以 update Deployment
```

而是：

```
Role
 ↓
定义一组权限

Binding
 ↓
把权限授予某个身份
```

最终形成：

```
User / Group / ServiceAccount
              │
              ▼
           Binding
              │
              ▼
       Role / ClusterRole
              │
              ▼
           Rules
```

------

### 24.3.2 RBAC Rule

一个权限 Rule 通常由几个核心部分组成：

```
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

这里：

```
apiGroups
resources
verbs
```

分别描述：

```
操作哪个 API Group
操作什么资源
允许执行什么操作
```

------

### 24.3.3 常见 verbs

最常见：

```
get
list
watch
create
update
patch
delete
deletecollection
```

例如：

```
verbs:
  - get
  - list
  - watch
```

表示：

```
可以读取
不能创建
不能修改
不能删除
```

------

### 24.3.4 RBAC 默认是拒绝

这是生产环境非常重要的原则。

如果用户没有匹配到允许的 Rule：

```
没有权限
```

而不是：

```
默认拥有权限
```

因此 RBAC 的设计通常是：

> **只授予需要的权限。**

------

## 24.4 Role

### 24.4.1 Role 是什么

Role 用来定义：

> **某一个 Namespace 范围内的权限。**

例如：

```
Namespace: shop
```

创建：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: shop
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

这个 Role 表示：

```
shop Namespace
    ↓
pods
    ↓
get/list/watch
```

------

### 24.4.2 Role 不等于用户

Role 只是：

> **权限定义。**

它本身没有把权限授予任何人。

所以：

```
Role
```

还需要：

```
RoleBinding
```

才能真正产生授权效果。

------

### 24.4.3 查看 Role

```
kubectl get role -n shop
```

查看详细内容：

```
kubectl describe role pod-reader -n shop
```

------

## 24.5 ClusterRole

### 24.5.1 ClusterRole 是什么

ClusterRole 与 Role 最大的区别是：

> **ClusterRole 不局限于单个 Namespace。**

它可以描述：

```
Namespace 资源权限
```

也可以描述：

```
Cluster-scoped 资源权限
```

例如：

```
nodes
persistentvolumes
namespaces
clusterroles
```

这些资源本身就不是普通 Namespace 级别资源。

------

### 24.5.2 示例

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
```

表示允许：

```
get/list/watch Nodes
```

------

### 24.5.3 Role 与 ClusterRole 的关键区别

| 项目           | Role         | ClusterRole |
| -------------- | ------------ | ----------- |
| Scope          | Namespace    | Cluster     |
| Namespace 资源 | 可以         | 可以        |
| Cluster 资源   | 不可以       | 可以        |
| 常见用途       | 应用团队权限 | 集群级权限  |

但需要特别注意：

> ClusterRole 本身只是权限定义，最终能不能访问某个 Namespace，还取决于它怎么 Binding。

------

## 24.6 RoleBinding

### 24.6.1 RoleBinding 是什么

RoleBinding 的作用是：

> **把 Role 或 ClusterRole 授予某个 User、Group 或 ServiceAccount。**

例如前面有：

```
Role
pod-reader
```

现在授予：

```
User: alice
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: shop
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

最终：

```
Alice
  ↓
RoleBinding
  ↓
Role: pod-reader
  ↓
shop Namespace
  ↓
pods get/list/watch
```

------

### 24.6.2 一个非常重要的特点

RoleBinding 本身属于 Namespace。

例如：

```
metadata:
  namespace: shop
```

所以它授予的权限也作用于：

```
shop
```

------

### 24.6.3 RoleBinding 绑定 ClusterRole

RoleBinding 也可以引用 ClusterRole：

```
roleRef:
  kind: ClusterRole
  name: pod-reader
```

此时：

> **ClusterRole 中定义的权限，通过这个 RoleBinding，只在 RoleBinding 所在 Namespace 生效。**

例如：

```
ClusterRole: pod-reader
        ↓
RoleBinding: shop
        ↓
只能访问 shop
```

而不是自动获得整个集群范围的权限。

这是实际使用 ClusterRole 时非常重要的一点。

------

## 24.7 ClusterRoleBinding

### 24.7.1 ClusterRoleBinding 是什么

ClusterRoleBinding 用来：

> **把 ClusterRole 授予 User、Group 或 ServiceAccount，并让权限在整个 Cluster 范围生效。**

例如：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alice-node-reader
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

结果：

```
Alice
 ↓
ClusterRoleBinding
 ↓
ClusterRole node-reader
 ↓
整个 Cluster
 ↓
Nodes get/list/watch
```

------

### 24.7.2 RoleBinding 与 ClusterRoleBinding

最容易记忆的方法：

```
RoleBinding
    ↓
Namespace 范围

ClusterRoleBinding
    ↓
Cluster 范围
```

例如：

```
RoleBinding
shop
 └── Pod Reader

ClusterRoleBinding
Cluster
 └── Node Reader
```

------

## 24.8 ServiceAccount

### 24.8.1 ServiceAccount 是什么

ServiceAccount，简称：

```
SA
```

可以理解为：

> **给 Pod / 工作负载使用的 Kubernetes 身份。**

之前的：

```
Alice
Bob
```

更偏向：

```
人
```

而：

```
ServiceAccount
```

主要用于：

```
Pod
Application
Controller
Automation
```

------

### 24.8.2 创建 ServiceAccount

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shop-api
  namespace: shop
```

或者：

```
kubectl create serviceaccount shop-api -n shop
```

查看：

```
kubectl get serviceaccount -n shop
```

------

### 24.8.3 Pod 使用 ServiceAccount

```
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: shop
spec:
  serviceAccountName: shop-api
  containers:
    - name: api
      image: nginx:1.27
```

这样：

```
Pod
 ↓
ServiceAccount: shop-api
```

------

### 24.8.4 ServiceAccount + RBAC

例如：

```
Pod
 ↓
ServiceAccount: shop-api
 ↓
RoleBinding
 ↓
Role
 ↓
允许 get ConfigMap
```

Role：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: shop
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
      - list
```

Binding：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: shop-api-config-reader
  namespace: shop
subjects:
  - kind: ServiceAccount
    name: shop-api
    namespace: shop
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

最终：

```
shop-api Pod
     ↓
ServiceAccount
     ↓
RoleBinding
     ↓
Role
     ↓
ConfigMap get/list
```

------

### 24.8.5 不要让所有 Pod 使用高权限 ServiceAccount

生产环境推荐：

```
Pod A
 ↓
SA-A
 ↓
权限 A

Pod B
 ↓
SA-B
 ↓
权限 B
```

而不是：

```
所有 Pod
   ↓
default
   ↓
超级权限
```

因为如果某个应用被攻击：

```
攻击者
 ↓
获得 Pod 权限
 ↓
使用 ServiceAccount 身份访问 API Server
```

那么 ServiceAccount 权限越大，潜在影响范围越大。

------

### 24.8.6 禁止 Pod 使用 Kubernetes API

如果应用根本不需要访问 Kubernetes API，可以考虑：

```
automountServiceAccountToken: false
```

例如：

```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: shop
spec:
  automountServiceAccountToken: false
  containers:
    - name: frontend
      image: nginx:1.27
```

这属于很有价值的生产安全实践。

------

## 24.9 最小权限原则

最小权限原则：

> **一个身份只应该拥有完成工作所必需的最少权限。**

例如应用只需要：

```
读取 ConfigMap
```

不要给：

```
pods/*
deployments/*
secrets/*
nodes/*
```

更不要：

```
cluster-admin
```

------

### 24.9.1 错误做法

例如：

```
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

这基本等价于：

> **什么都可以操作。**

生产环境应该尽量避免。

------

### 24.9.2 正确做法

如果应用只需要读取 ConfigMap：

```
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
```

如果需要监听变化：

```
verbs:
  - get
  - list
  - watch
```

权限越具体越好。

------

### 24.9.3 权限设计原则

生产环境推荐：

```
权限
 ↓
尽可能小

Namespace
 ↓
尽可能明确

Resource
 ↓
只允许必要资源

Verb
 ↓
只允许必要操作
```

例如：

```
shop-api
 ↓
shop Namespace
 ↓
ConfigMap
 ↓
get
```

而不是：

```
shop-api
 ↓
整个 Cluster
 ↓
所有资源
 ↓
所有操作
```

------

## 24.10 如何限制用户只能操作某个 Namespace

这是生产环境非常常见的需求。

例如公司有：

```
Namespace:
shop
payment
monitoring
```

现在：

```
Alice
```

是：

```
shop
```

团队成员。

要求：

```
Alice
 ↓
可以操作 shop
 ↓
不能操作 payment
 ↓
不能操作 monitoring
 ↓
不能操作其他 Namespace
```

------

### 24.10.1 创建 Namespace

```
kubectl create namespace shop
```

------

### 24.10.2 创建 Role

例如允许 Alice 管理 Pod：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: shop-developer
  namespace: shop
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
      - watch
  - apiGroups: [""]
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
      - delete
```

保存为：

```
shop-developer-role.yaml
```

执行：

```
kubectl apply -f shop-developer-role.yaml
```

------

### 24.10.3 创建 RoleBinding

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-shop-developer
  namespace: shop
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: shop-developer
  apiGroup: rbac.authorization.k8s.io
```

执行：

```
kubectl apply -f alice-shop-binding.yaml
```

现在：

```
Alice
 ↓
RoleBinding
 ↓
Role
 ↓
shop
```

------

### 24.10.4 验证权限

Kubernetes 提供：

```
kubectl auth can-i
```

例如：

```
kubectl auth can-i get pods \
  --as=alice \
  -n shop
```

应该：

```
yes
```

测试：

```
kubectl auth can-i delete pods \
  --as=alice \
  -n shop
```

如果 Role 没有授予：

```
no
```

再测试其他 Namespace：

```
kubectl auth can-i get pods \
  --as=alice \
  -n payment
```

应该：

```
no
```

这就是：

> **Namespace 级权限隔离。**

------

### 24.10.5 一个非常重要的注意点

如果你给 Alice 创建：

```
ClusterRoleBinding
```

并绑定一个权限很大的 ClusterRole，那么她就可能获得整个 Cluster 的权限。

因此：

```
只允许操作 shop
```

通常应该优先考虑：

```
Role
+
RoleBinding
```

而不是：

```
ClusterRoleBinding
```

------

## 24.11 如何限制 Pod 权限

这里的“限制 Pod 权限”主要是指：

> **限制 Pod 使用 Kubernetes API 的权限，以及限制 Pod 内进程对 Kubernetes 资源的操作能力。**

需要区分两个概念：

```
Pod 对 Kubernetes API 的权限
```

和：

```
Container 在 Linux 中的权限
```

它们不是一回事。

------

### 24.11.1 Kubernetes API 权限

例如：

```
Pod
 ↓
ServiceAccount
 ↓
RoleBinding
 ↓
Role
 ↓
API Server
```

假设应用只需要读取 ConfigMap：

```
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
```

那么这个 Pod：

```
可以：
get ConfigMap

不能：
delete Pod
create Deployment
读取 Secret
修改 Node
```

------

### 24.11.2 每个应用使用独立 ServiceAccount

推荐：

```
frontend
 ↓
frontend-sa

api
 ↓
api-sa

worker
 ↓
worker-sa
```

而不是：

```
所有应用
    ↓
default
```

例如：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shop-api
  namespace: shop
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
  namespace: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shop-api
  template:
    metadata:
      labels:
        app: shop-api
    spec:
      serviceAccountName: shop-api
      containers:
        - name: api
          image: nginx:1.27
```

------

### 24.11.3 应用不需要 Kubernetes API 时

推荐：

```
spec:
  automountServiceAccountToken: false
```

这样可以减少：

```
Pod
 ↓
ServiceAccount Token
 ↓
Kubernetes API
```

被滥用的风险。

对于不需要 Kubernetes API 的普通：

```
.NET API
Vue
Nginx
Redis
```

通常不应该因为“默认存在”就给它 Kubernetes API 权限。

------

### 24.11.4 不要把 Secret 权限随便授予 Pod

这是生产环境尤其需要注意的。

例如：

```
resources:
  - secrets
verbs:
  - get
```

意味着这个身份可以读取 Secret。

如果 Pod 被攻击：

```
攻击者
 ↓
进入 Pod
 ↓
使用 ServiceAccount
 ↓
调用 Kubernetes API
 ↓
读取 Secrets
```

那么可能进一步造成：

```
数据库密码泄露
Redis 密码泄露
TLS 私钥泄露
Registry 凭据泄露
```

所以：

> **Secret 的 RBAC 权限必须严格控制。**

------

### 24.11.5 ServiceAccount 权限只是第一层

真正的 Pod 安全还包括：

```
ServiceAccount
RBAC
SecurityContext
Linux User
Capabilities
Filesystem
NetworkPolicy
Pod Security
```

例如：

```
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
```

这属于：

> **Container / Linux 层面的安全控制**

而不是 RBAC 本身。

因此：

```
RBAC
```

主要解决：

> **“Pod 能不能调用 Kubernetes API 做某件事情？”**

而：

```
SecurityContext
```

主要解决：

> **“Container 内的进程具有什么 Linux 权限？”**

这两个概念不要混淆。

------

## 本章核心知识总结

Kubernetes 权限模型可以浓缩成：

```
                    请求
                     │
                     ▼
              Authentication
                     │
                你是谁？
                     │
                     ▼
              Authorization
                     │
                你能做什么？
                     │
                     ▼
                    RBAC
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
        Role               ClusterRole
          │                     │
          ▼                     ▼
    RoleBinding        ClusterRoleBinding
          │                     │
          └──────────┬──────────┘
                     ▼
              User / Group /
              ServiceAccount
```

最需要牢牢记住的是：

| 概念                 | 解决什么问题                           |
| -------------------- | -------------------------------------- |
| Authentication       | 你是谁                                 |
| Authorization        | 你能做什么                             |
| RBAC                 | 如何定义和授予权限                     |
| Role                 | Namespace 范围的权限定义               |
| ClusterRole          | Cluster 范围/可复用的权限定义          |
| RoleBinding          | 在某个 Namespace 授予 Role/ClusterRole |
| ClusterRoleBinding   | 在 Cluster 范围授予 ClusterRole        |
| ServiceAccount       | 给 Pod/工作负载使用的身份              |
| `kubectl auth can-i` | 检查某个身份是否有权限                 |

生产环境最重要的原则则是：

```
默认拒绝
    ↓
最小权限
    ↓
Namespace 隔离
    ↓
独立 ServiceAccount
    ↓
不需要 API 就关闭 Token
    ↓
严格限制 Secret 权限
```

其中最值得形成肌肉记忆的两个判断是：

```
kubectl auth can-i get pods -n shop
```

以及：

```
kubectl auth can-i delete pods \
  --as=alice \
  -n shop
```

前者回答：

> **当前身份能不能做这件事？**

后者回答：

> **指定身份能不能做这件事？**

在生产环境排查 RBAC 问题时，这两个命令非常实用。

# 第 25 章：Pod 与 Cluster 安全

上一章解决的是：

> **“谁可以访问 Kubernetes API，以及可以做什么？”**

本章进一步解决：

> **“Pod 和 Container 本身具有什么权限？Pod 之间能不能互相访问？镜像是否可信？Namespace 和 Secret 是否得到正确保护？”**

生产环境中的 Kubernetes 安全不是单一功能，而是多个层次共同组成：

```
                    Kubernetes Security
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   Identity/RBAC      Workload Security   Network Security
        │                  │                  │
     User/SA         SecurityContext      NetworkPolicy
     Role/RBAC       Capabilities
                     NonRoot
                     ReadOnly FS
                     Privileged
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                  Supply Chain Security
                           │
                    Image / Registry
                    SBOM / Signing
                    Vulnerability
```

------

## 25.1 SecurityContext

### 25.1.1 SecurityContext 是什么

`SecurityContext` 用于控制：

> **Pod 或 Container 运行时的安全属性。**

例如可以控制：

- 以哪个 Linux UID 运行
- 是否必须使用非 root 用户
- 是否允许权限提升
- Linux Capabilities
- Root Filesystem 是否只读
- SELinux 等安全机制

简单理解：

```
Pod
 │
 └── SecurityContext
       │
       ├── User
       ├── Group
       ├── Capabilities
       ├── Privilege
       └── Filesystem
```

------

### 25.1.2 Pod 级别与 Container 级别

SecurityContext 可以配置在：

```
spec:
  securityContext:
```

也可以配置在：

```
containers:
  - securityContext:
```

例如：

```
apiVersion: v1
kind: Pod
metadata:
  name: security-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000

  containers:
    - name: app
      image: nginx:1.27
```

Pod 级别的配置可以作为默认安全配置。

Container 级别可以针对某个 Container 单独设置。

------

### 25.1.3 生产环境基本安全模板

一个普通应用可以从类似配置开始：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000

      containers:
        - name: api
          image: example/api:1.0.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
```

不过需要注意：

> **不能机械地把这份配置复制到所有应用。**

例如某些镜像必须以 root 启动，或者需要写入特定目录，就需要调整镜像和应用设计。

------

## 25.2 runAsUser

### 25.2.1 runAsUser 是什么

`runAsUser` 指定 Container 内进程使用的：

> **Linux UID**

例如：

```
securityContext:
  runAsUser: 1000
```

意味着应用进程以：

```
UID = 1000
```

运行，而不是：

```
UID = 0
```

其中 UID `0` 就是：

```
root
```

------

### 25.2.2 为什么不推荐 Root

假设应用存在漏洞：

```
攻击者
   ↓
利用应用漏洞
   ↓
进入 Container
   ↓
获得应用进程权限
```

如果应用是：

```
root
```

攻击者获得的权限通常更大。

如果应用是：

```
UID 1000
```

攻击者受到 Linux 权限限制。

所以：

> **应用没有必要使用 root 时，不应该使用 root。**

------

### 25.2.3 示例

```
spec:
  securityContext:
    runAsUser: 1000
```

查看 Container 中实际用户：

```
kubectl exec -it security-demo -- id
```

可能看到：

```
uid=1000 gid=1000
```

------

### 25.2.4 常见问题

最常见的问题是：

```
Permission denied
```

例如应用需要写：

```
/app/data
```

但目录属于 root：

```
root:root
```

而应用运行：

```
1000:1000
```

就可能无法写入。

解决方法通常不是简单恢复 root，而是：

- 修改镜像中的目录权限
- 使用正确的 `fsGroup`
- 使用合适的 Volume
- 修改应用写入目录

------

## 25.3 runAsNonRoot

### 25.3.1 runAsNonRoot 是什么

```
securityContext:
  runAsNonRoot: true
```

表示：

> **禁止 Container 以 root 身份运行。**

它和：

```
runAsUser: 1000
```

不是完全一样。

例如：

```
runAsUser: 1000
```

是：

> 指定具体 UID。

而：

```
runAsNonRoot: true
```

是：

> 要求最终运行用户不能是 UID 0。

------

### 25.3.2 推荐组合

生产环境常见：

```
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
```

这样表达更加明确：

```
必须非 root
    +
明确使用 UID 1000
```

------

### 25.3.3 常见问题

某些镜像默认：

```
root
```

并且应用没有正确配置非 root 用户。

此时：

```
runAsNonRoot: true
```

可能导致 Container 无法正常启动。

因此真正的生产实践应该是：

```
Dockerfile
   ↓
创建应用用户
   ↓
调整文件权限
   ↓
应用使用非 root
   ↓
Kubernetes runAsNonRoot
```

也就是说：

> **镜像本身应该为非 root 运行做好准备。**

------

## 25.4 Linux Capabilities

### 25.4.1 Capabilities 是什么

Linux 原本把 root 的大量权限集中在：

```
UID 0
```

现代 Linux 可以把部分特权拆分成不同的：

> **Capabilities**

例如：

```
NET_ADMIN
NET_RAW
SYS_ADMIN
CHOWN
SETUID
SETGID
```

这样可以实现：

```
不要给 Container 完整 root 权限
        ↓
只给它真正需要的一个能力
```

------

### 25.4.2 最常见的生产实践

对于普通 Web API：

```
securityContext:
  capabilities:
    drop:
      - ALL
```

意思是：

> **删除所有额外 Linux Capabilities。**

然后如果某个应用确实需要特定能力，再单独添加。

例如：

```
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

------

### 25.4.3 为什么需要 NET_BIND_SERVICE

Linux 中普通用户通常不能直接绑定：

```
TCP 1-1023
```

这样的低端口。

例如应用希望直接监听：

```
80
```

可以使用：

```
NET_BIND_SERVICE
```

但生产环境更推荐考虑：

```
Application
 ↓
8080
 ↓
Service
 ↓
80
```

这样往往无需给应用额外 Capability。

------

### 25.4.4 SYS_ADMIN 特别危险

例如：

```
SYS_ADMIN
```

属于权限非常广的 Capability。

不要看到“程序启动失败”就随便增加：

```
capabilities:
  add:
    - SYS_ADMIN
```

应该先确定：

> **应用为什么需要这个 Capability？**

------

## 25.5 Privileged Container

### 25.5.1 Privileged 是什么

```
securityContext:
  privileged: true
```

表示：

> **让 Container 获得非常强的主机级权限。**

它与普通 Container 的隔离程度明显不同。

------

### 25.5.2 为什么危险

正常情况下：

```
Container
   │
   │ 隔离
   ▼
Host
```

而 Privileged Container：

```
Container
   │
   │ 大量特权
   ▼
Host
```

如果攻击者控制了 Privileged Container，攻击 Kubernetes Node 的风险会显著提高。

因此：

> **生产环境应该默认禁止 Privileged Container。**

------

### 25.5.3 哪些场景可能需要

少数基础设施程序可能需要，例如：

- 某些网络组件
- 某些存储组件
- 特殊硬件访问
- Node-level agent

但即使这些场景，也应该：

> **只给必要权限，而不是无条件使用 `privileged: true`。**

------

## 25.6 ReadOnlyRootFilesystem

### 25.6.1 是什么

```
securityContext:
  readOnlyRootFilesystem: true
```

表示：

> **Container 的 Root Filesystem 设置为只读。**

例如：

```
/
├── app
├── bin
├── etc
└── tmp
```

默认情况下应用可能可以修改这些路径。

开启后：

```
/
├── app       READ ONLY
├── bin       READ ONLY
├── etc       READ ONLY
└── tmp       需要单独提供可写 Volume
```

------

### 25.6.2 为什么需要

假设攻击者利用应用漏洞：

```
漏洞
 ↓
写入恶意脚本
 ↓
修改 Container 文件
 ↓
建立持久化
```

如果 Root Filesystem 是只读：

```
漏洞
 ↓
尝试写入 /
 ↓
失败
```

可以降低攻击面的持久化能力。

------

### 25.6.3 配合 emptyDir

很多应用仍然需要：

```
/tmp
```

可以提供：

```
spec:
  containers:
    - name: app
      image: example/api:1.0.0
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp

  volumes:
    - name: tmp
      emptyDir: {}
```

这样：

```
Root FS
   ↓
只读

/tmp
   ↓
可写
```

这是很常见的生产设计。

------

## 25.7 Pod Security Standards

### 25.7.1 是什么

Pod Security Standards，简称：

> **PSS**

Kubernetes 定义了三种安全级别：

```
Privileged
Baseline
Restricted
```

------

### 25.7.2 Privileged

```
Privileged
```

基本不限制 Pod。

适合：

```
特殊基础设施组件
```

不适合作为普通业务 Namespace 的默认安全级别。

------

### 25.7.3 Baseline

```
Baseline
```

主要防止一些明显危险的配置。

例如限制：

- Privileged Container
- 某些危险 Host Namespace
- 某些危险 HostPath 等

可以理解为：

> **基本安全基线。**

------

### 25.7.4 Restricted

```
Restricted
```

是更严格的安全级别。

通常要求工作负载：

```
非 root
限制权限提升
合理使用 Capabilities
使用安全的 SecurityContext
```

生产业务应用应该尽量向：

```
Restricted
```

靠拢。

------

### 25.7.5 Namespace 启用 Pod Security

例如：

```
kubectl label namespace shop \
  pod-security.kubernetes.io/enforce=restricted
```

查看：

```
kubectl get namespace shop --show-labels
```

也可以设置审计和告警：

```
kubectl label namespace shop \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

实际生产环境建议先使用：

```
warn
audit
```

观察现有工作负载，再逐步启用：

```
enforce
```

否则可能一次性导致现有 Pod 无法创建或更新。

------

## 25.8 NetworkPolicy

### 25.8.1 NetworkPolicy 是什么

NetworkPolicy 用来控制：

> **Pod 与 Pod、Namespace、外部网络之间允许什么网络流量。**

默认情况下，如果集群网络插件支持 NetworkPolicy，Pod 之间可能可以直接通信。

例如：

```
frontend ───────► api
   │                │
   │                ▼
   └────────────► database
```

但真实生产环境通常应该限制成：

```
frontend
   │
   ▼
api
   │
   ▼
database
```

而不是：

```
frontend ─► database
api ──────► frontend
database ─► frontend
```

------

### 25.8.2 为什么需要

假设 API Pod 被攻击：

```
攻击者
  ↓
API Pod
  ↓
尝试扫描整个集群
  ↓
Redis
PostgreSQL
其他 API
内部管理服务
```

如果有 NetworkPolicy：

```
API Pod
 │
 ├──► PostgreSQL   ALLOW
 ├──► Redis        ALLOW
 └──► 其他服务     DENY
```

即使应用被攻破，攻击范围也可以被限制。

这叫：

> **网络层面的最小权限原则。**

------

### 25.8.3 一个基本示例

假设：

```
Namespace: shop
```

有：

```
frontend
api
postgres
```

要求：

```
frontend → api
api → postgres
```

例如限制 PostgreSQL 只接受 API：

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-ingress
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 5432
```

最终：

```
api → postgres:5432
```

允许。

而：

```
frontend → postgres:5432
```

没有匹配规则时则被拒绝。

------

### 25.8.4 NetworkPolicy 的重要前提

NetworkPolicy 是否真正生效，取决于：

> **集群使用的 CNI 是否支持 NetworkPolicy。**

例如：

```
Calico
Cilium
```

都支持强大的网络策略能力。

所以不要只创建 YAML 就认为：

```
网络已经安全
```

必须验证实际 CNI 行为。

------

### 25.8.5 常见问题

NetworkPolicy 最容易出现：

```
应用突然无法连接
```

例如：

```
API → PostgreSQL
```

原本正常。

添加：

```
NetworkPolicy
```

以后：

```
connection timeout
```

这时应该检查：

```
kubectl get networkpolicy -A
```

并检查：

```
kubectl describe networkpolicy postgres-ingress -n shop
```

重点确认：

```
podSelector
namespaceSelector
ports
Ingress / Egress
```

------

## 25.9 Namespace 隔离

### 25.9.1 Namespace 是什么

Namespace 本身不是完整的安全边界。

它主要提供：

> **资源逻辑隔离和权限作用域。**

例如：

```
shop
payment
monitoring
```

可以把不同团队、应用、环境分开。

------

### 25.9.2 生产环境常见设计

例如：

```
production
 ├── shop
 ├── payment
 └── order

staging
 ├── shop
 ├── payment
 └── order
```

或者：

```
prod-shop
prod-payment
staging-shop
staging-payment
```

具体怎么划分应该根据组织和权限模型设计。

------

### 25.9.3 Namespace 不是完整隔离

例如：

```
Namespace A
Namespace B
```

并不意味着：

```
A 完全无法访问 B
```

网络访问需要：

```
NetworkPolicy
```

API 权限需要：

```
RBAC
```

因此：

```
Namespace
+
RBAC
+
NetworkPolicy
+
Pod Security
```

才能形成比较完整的隔离体系。

------

### 25.9.4 生产环境隔离思路

可以形成：

```
用户
 ↓
RBAC
 ↓
Namespace

Pod
 ↓
NetworkPolicy
 ↓
其他 Pod

Pod
 ↓
SecurityContext
 ↓
Linux 权限

Namespace
 ↓
Pod Security Standards
 ↓
工作负载安全
```

------

## 25.10 Secret 安全

上一章已经学习过 Secret。

本章重点是：

> **Secret 虽然叫 Secret，但默认并不等于“加密保险箱”。**

------

### 25.10.1 Base64 不是加密

例如：

```
echo -n 'mypassword' | base64
```

得到：

```
bXlwYXNzd29yZA==
```

这只是：

```
Encoding
```

不是：

```
Encryption
```

任何人都可以解码：

```
echo 'bXlwYXNzd29yZA==' | base64 -d
```

------

### 25.10.2 Secret 不应该直接提交 Git

错误：

```
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
data:
  password: bXlwYXNzd29yZA==
```

然后：

```
git add .
git commit
git push
```

即使是 Base64：

> **密码仍然已经进入 Git 历史。**

即使之后删除文件，也可能仍然存在于：

```
Git history
CI logs
Backup
Artifact
```

------

### 25.10.3 Kubernetes Secret 的权限控制

如果应用不需要读取 Secret：

```
不要授予 secrets/get
```

例如 RBAC：

```
rules:
  - apiGroups: [""]
    resources:
      - secrets
    verbs:
      - get
```

这个权限应该非常谨慎。

因为：

```
Secret
 ↓
数据库密码
API Key
TLS 私钥
Registry 凭据
```

都可能存放在里面。

------

### 25.10.4 生产环境应该考虑 Secret 加密

Kubernetes 可以配置：

> **Encryption at Rest**

让存储在 Kubernetes 后端的数据进行静态加密。

但这仍然不是完整的 Secret 管理方案。

生产环境更常见的高级方案包括：

```
Vault
External Secrets
Cloud Secret Manager
KMS
```

核心目标：

```
应用需要 Secret
      ↓
安全的 Secret Store
      ↓
动态/受控同步
      ↓
Kubernetes
```

------

## 25.11 Container 安全

Container 安全不能只看 Kubernetes YAML。

完整的 Container 安全应该从：

```
源代码
 ↓
依赖
 ↓
Dockerfile
 ↓
Base Image
 ↓
Build
 ↓
Registry
 ↓
Kubernetes
 ↓
Runtime
```

全部考虑。

------

### 25.11.1 镜像不要使用 root

Dockerfile 可以创建专用用户：

```
FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

COPY . .

USER 1000

ENTRYPOINT ["dotnet", "MyApi.dll"]
```

实际生产环境需要根据基础镜像和应用文件权限进行调整。

------

### 25.11.2 不要把 Secret 烧进镜像

错误：

```
ENV DB_PASSWORD=SuperSecretPassword
```

或者：

```
COPY .env .
```

因为镜像可能进入：

```
Registry
CI/CD
Backup
开发机器
生产机器
```

应该使用：

```
Kubernetes Secret
External Secret
Vault
Cloud Secret Manager
```

在运行时注入。

------

### 25.11.3 最小化镜像

镜像越复杂：

```
软件越多
 ↓
潜在漏洞越多
```

例如普通 API 不需要：

```
gcc
git
curl
vim
完整编译工具链
```

可以使用：

```
Multi-stage Build
```

让最终 Runtime Image 尽可能小。

------

### 25.11.4 不要随便安装调试工具

生产 Container 中不要为了方便排障长期保留：

```
ssh
telnet
gcc
python
curl
vim
```

应该通过：

- 临时 Debug Container
- Ephemeral Container
- 专用诊断工具

进行排查，而不是让业务镜像越来越臃肿。

------

## 25.12 Supply Chain Security

### 25.12.1 什么是 Supply Chain Security

Supply Chain Security：

> **软件供应链安全。**

也就是说：

> **从代码到最终运行的 Container，中间任何一个环节都可能被攻击。**

完整链路：

```
Developer
   ↓
Source Code
   ↓
Dependencies
   ↓
Build
   ↓
Docker Image
   ↓
Registry
   ↓
Kubernetes
   ↓
Production
```

------

### 25.12.2 常见攻击点

例如：

```
恶意第三方依赖
        ↓
代码被篡改
        ↓
CI/CD 被攻击
        ↓
Docker Image 被植入恶意程序
        ↓
Registry Image 被替换
        ↓
生产 Kubernetes 拉取恶意 Image
```

所以：

> **Kubernetes 安全不能只检查 Pod YAML。**

------

### 25.12.3 镜像漏洞扫描

生产环境应该对 Image 进行漏洞扫描。

常见工具包括：

```
Trivy
Grype
Clair
```

例如使用 Trivy 扫描本地镜像：

```
trivy image example/api:1.0.0
```

重点关注：

```
CRITICAL
HIGH
```

但也不能简单理解为：

> “有一个 HIGH 就绝对不能上线”。

应该结合：

```
漏洞是否可利用
是否运行到相关代码
是否存在 Exploit
是否可以升级
是否有业务影响
```

建立自己的安全门禁策略。

------

### 25.12.4 Image Digest

不要只依赖：

```
example/api:1.0.0
```

更强的方式是使用 Digest：

```
example/api@sha256:xxxxxxxx...
```

因为 Tag 可以被重新指向其他镜像。

Digest 则对应：

> **具体的镜像内容。**

生产环境可以采用：

```
开发
 ↓
Build Image
 ↓
Scan
 ↓
Sign
 ↓
Push Registry
 ↓
固定 Digest
 ↓
Deploy
```

------

### 25.12.5 Image Signing

进一步可以对镜像进行签名：

```
Build
 ↓
Image
 ↓
Sign
 ↓
Registry
 ↓
Kubernetes
 ↓
验证签名
 ↓
允许运行
```

常见生态工具：

```
Cosign
Sigstore
```

目标是确保：

> **生产集群运行的是经过授权、可信来源构建的镜像。**

------

### 25.12.6 SBOM

SBOM：

> **Software Bill of Materials**

可以理解为：

> **这个镜像里面到底包含哪些软件和依赖？**

例如：

```
example/api
 ├── .NET Runtime
 ├── Newtonsoft.Json
 ├── OpenSSL
 ├── libc
 └── 其他依赖
```

当某个组件出现重大漏洞时：

```
CVE
 ↓
查询 SBOM
 ↓
哪些 Image 使用了这个组件？
 ↓
哪些 Pod 正在运行？
 ↓
升级 Image
```

这对生产环境漏洞响应非常重要。

------

### 25.12.7 生产供应链安全建议

一个比较完整的流程：

```
             Source Code
                  │
                  ▼
             Dependency
               Scan
                  │
                  ▼
                Build
                  │
                  ▼
             Docker Image
                  │
          ┌───────┴────────┐
          ▼                ▼
     Vulnerability        SBOM
        Scan               │
          │                │
          └───────┬────────┘
                  ▼
              Image Sign
                  │
                  ▼
               Registry
                  │
                  ▼
          Kubernetes Deploy
                  │
                  ▼
          Verify / Admission
                  │
                  ▼
             Production
```

这才是比较完整的：

> **Supply Chain Security。**

------

## 本章核心知识总结

本章最重要的是建立**纵深防御**思维。

不要认为：

```
RBAC
```

解决了 Kubernetes 安全。

实际上应该是：

```
                    Kubernetes
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
      RBAC          Pod Security    NetworkPolicy
       │               │                │
    API 权限        Container 权限     网络访问
       │               │                │
       └───────────────┼────────────────┘
                       ▼
                 Secret Security
                       │
                       ▼
              Supply Chain Security
```

生产环境中，一个普通 API Pod 可以尽量做到：

```
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000

  fsGroup: 1000

containers:
  - name: api
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
```

再配合：

```
Namespace
    +
RBAC
    +
Pod Security Standards
    +
NetworkPolicy
    +
Secret 管理
    +
Image Scan
    +
Image Digest
    +
Image Signing
    +
SBOM
```

形成生产环境的多层安全体系。

最值得记住的一句话是：

> **Kubernetes 安全不是“给 Pod 加一个 SecurityContext”就完成了，而是身份、权限、Container、网络、Secret、Namespace 和软件供应链共同构成的纵深防御体系。**