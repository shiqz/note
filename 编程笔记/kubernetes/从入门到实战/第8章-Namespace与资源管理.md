# 第8章：Namespace 与资源管理

## 学习目标

完成本章学习后，你将能够：

1. 理解 Namespace 的作用与使用场景
2. 区分命名空间级资源与集群级资源
3. 掌握 ResourceQuota 的配置与使用
4. 掌握 LimitRange 的配置与使用
5. 理解标签（Label）与注解（Annotation）的区别
6. 熟练使用标签选择器过滤和管理资源
7. 掌握资源清理策略与级联删除
8. 通过实战掌握多环境资源隔离的完整配置

---

## 8.1 Namespace 概述

### 8.1.1 Namespace 的作用

Namespace 是 K8s 中用于**资源隔离和分组**的逻辑分区，它不是物理隔离，而是一种逻辑边界。

```
┌─────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                  │
│                                                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│  │  Namespace  │ │  Namespace  │ │  Namespace  │       │
│  │   dev       │ │  staging    │ │ production  │       │
│  │             │ │             │ │             │       │
│  │ • Pods      │ │ • Pods      │ │ • Pods      │       │
│  │ • Services  │ │ • Services  │ │ • Services  │       │
│  │ • ConfigMap │ │ • ConfigMap │ │ • ConfigMap │       │
│  │ • Secrets   │ │ • Secrets   │ │ • Secrets   │       │
│  │ • Deployments│ │ •Deployments│ │ •Deployments│       │
│  │ • PVCs      │ │ • PVCs      │ │ • PVCs      │       │
│  └─────────────┘ └─────────────┘ └─────────────┘       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Cluster-Scoped Resources            │    │
│  │  • Namespaces  • Nodes  • PersistentVolumes     │    │
│  │  • ClusterRoles • ClusterRoleBindings           │    │
│  │  • StorageClasses • IngressClasses              │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 8.1.2 Namespace 的核心场景

| 场景 | 说明 | 示例 |
|------|------|------|
| 多租户隔离 | 不同团队/项目使用不同 Namespace | team-a、team-b |
| 多环境管理 | 开发/测试/生产环境隔离 | dev、staging、production |
| 资源分组 | 按业务模块组织资源 | order-service、payment-service |
| 权限控制 | 基于 Namespace 的 RBAC | 开发人员只能操作 dev |
| 资源配额 | 限制 Namespace 内的资源使用 | 限制 CPU/内存使用量 |
| 生命周期管理 | 创建/删除 Namespace 级联删除所有资源 | 删除 dev Namespace 清理所有开发资源 |

### 8.1.3 系统内置 Namespace

| Namespace | 说明 |
|-----------|------|
| default | 未指定 Namespace 时的默认命名空间 |
| kube-system | K8s 系统组件运行的命名空间 |
| kube-public | 可访问的公共命名空间（存放集群信息） |
| kube-node-lease | 节点心跳信息 |

```bash
# 查看所有 Namespace
kubectl get namespaces

# 查看 Namespace 详情
kubectl describe namespace default
```

---

## 8.2 命名空间级资源与集群级资源

### 8.2.1 资源分类

```
K8s 资源分类：
├── Namespace-Scoped Resources（命名空间级资源）
│   ├── Workload 资源: Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
│   ├── 网络资源: Services, Ingresses, NetworkPolicies
│   ├── 存储资源: PersistentVolumeClaims
│   ├── 配置资源: ConfigMaps, Secrets
│   ├── 权限资源: Roles, RoleBindings, ServiceAccounts
│   └── 其他: ResourceQuotas, LimitRanges, HorizontalPodAutoscalers
│
└── Cluster-Scoped Resources（集群级资源）
    ├── 节点资源: Nodes
    ├── 存储资源: PersistentVolumes, StorageClasses
    ├── 网络资源: ClusterRole, ClusterRoleBindings
    ├── 存储类: StorageClasses
    ├── 网络策略: NetworkPolicy（但在 Namespace 内）
    └── 其他: Namespaces, PodSecurityPolicies（已废弃）
```

### 8.2.2 命名空间级资源详解

命名空间级资源的特点：
- 创建时必须指定 Namespace
- 相同名称可在不同 Namespace 中存在
- 删除 Namespace 时级联删除

```bash
# 查看所有命名空间级资源
kubectl api-resources --namespaced=true

# 常见命名空间级资源
# Pods, Services, Deployments, ConfigMaps, Secrets, PVCs, Roles, ...
```

### 8.2.3 集群级资源详解

集群级资源的特点：
- 全集群唯一名称
- 不能在 Namespace 中查找
- 不随 Namespace 删除

```bash
# 查看所有集群级资源
kubectl api-resources --namespaced=false

# 常见集群级资源
# Nodes, PersistentVolumes, StorageClasses, Namespaces, ClusterRoles, ...
```

### 8.2.4 查询资源所在 Namespace

```bash
# 查询资源时指定 Namespace
kubectl get pods -n dev
kubectl get services -n production

# 查询所有 Namespace 的资源
kubectl get pods --all-namespaces
kubectl get pods -A                     # 简写

# 使用 namespace 字段
kubectl get pods --field-selector=metadata.namespace=dev
```

---

## 8.3 Namespace 的创建与管理

### 8.3.1 创建 Namespace

**方式一：命令行创建**

```bash
# 直接创建
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# 带标签创建
kubectl create namespace dev --labels=env=development,team=backend
```

**方式二：YAML 文件创建**

```yaml
# namespace-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    env: development
    team: backend
    cost-center: cc-12345
  annotations:
    owner: dev-team@example.com
    description: 开发环境
```

```bash
kubectl apply -f namespace-dev.yaml
```

### 8.3.2 Namespace 的状态

```
Active: Namespace 正常运行中
Terminating: Namespace 正在被删除
```

```bash
# 查看 Namespace 状态
kubectl get namespace dev

# 删除 Namespace（级联删除所有资源）
kubectl delete namespace dev
# 这会删除 dev 命名空间下的所有 Pod、Service、Deployment 等资源
```

### 8.3.3 在 Namespace 中创建资源

```yaml
# deployment-in-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: dev           # 指定 Namespace
  labels:
    app: web-app
    env: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
# 创建到指定 Namespace
kubectl apply -f deployment-in-dev.yaml

# 或使用 -n 覆盖
kubectl apply -f deployment-in-dev.yaml -n dev

# 查看指定 Namespace 的资源
kubectl get pods -n dev
kubectl get all -n dev
```

---

## 8.4 ResourceQuota：资源配额

### 8.4.1 ResourceQuota 概述

ResourceQuota 用于限制**命名空间内所有资源的总使用量**，防止资源滥用。

```
┌─────────────────────────────────────────────────────────┐
│                    Namespace: dev                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              ResourceQuota                       │    │
│  │  ├── requests.cpu: 20                            │    │
│  │  ├── requests.memory: 40Gi                       │    │
│  │  ├── limits.cpu: 40                              │    │
│  │  ├── limits.memory: 80Gi                         │    │
│  │  ├── persistentvolumeclaims: 10                 │    │
│  │  └── pods: 50                                    │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  资源使用：                                             │
│  ├── Pod A: requests=100m, 256Mi                       │    │
│  ├── Pod B: requests=200m, 512Mi                       │    │
│  ├── Pod C: ...                                       │    │
│  └── 总计：不能超过配额限制                             │    │
└─────────────────────────────────────────────────────────┘
```

### 8.4.2 可配额的资源类型

| 资源类型 | 说明 | 示例 |
|---------|------|------|
| `requests.cpu` | 总 CPU requests | 20, 20000m |
| `requests.memory` | 总 Memory requests | 40Gi, 4096Mi |
| `limits.cpu` | 总 CPU limits | 40, 40000m |
| `limits.memory` | 总 Memory limits | 80Gi |
| `persistentvolumeclaims` | PVC 数量 | 10 |
| `pods` | Pod 数量 | 50 |
| `services` | Service 数量 | 10 |
| `services.loadbalancers` | LoadBalancer Service 数量 | 2 |
| `configmaps` | ConfigMap 数量 | 20 |
| `secrets` | Secret 数量 | 20 |
| `count/deployments.apps` | Deployment 数量 | 10 |
| `count/jobs.batch` | Job 数量 | 5 |

### 8.4.3 创建 ResourceQuota

```yaml
# resource-quota-dev.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    pods: "50"
    services: "10"
    services.loadbalancers: "2"
    configmaps: "30"
    secrets: "30"
    count/deployments.apps: "20"
    count/jobs.batch: "10"
```

```bash
kubectl apply -f resource-quota-dev.yaml

# 查看配额
kubectl get resourcequota -n dev
kubectl describe resourcequota dev-quota -n dev

# 查看配额使用情况
kubectl get resourcequota dev-quota -n dev -o yaml
# 输出中会显示 used 和 hard 的对比
```

### 8.4.4 查看配额使用情况

```bash
# 使用 jsonpath 格式化输出
kubectl get resourcequota dev-quota -n dev -o jsonpath='{.status}' | python3 -m json.tool

# 查看特定配额的使用情况
kubectl get resourcequota dev-quota -n dev -o jsonpath='{.status.used.requests.cpu}'
kubectl get resourcequota dev-quota -n dev -o jsonpath='{.status.hard.requests.cpu}'

# 对比 used 和 hard
kubectl get resourcequota dev-quota -n dev \
  -o jsonpath='{range .status.hard}{.}{"\t"}{end}'
kubectl get resourcequota dev-quota -n dev \
  -o jsonpath='{range .status.used}{.}{"\t"}{end}'
```

### 8.4.5 更新 ResourceQuota

```bash
# 使用 kubectl edit 编辑
kubectl edit resourcequota dev-quota -n dev

# 使用 kubectl patch 更新
kubectl patch resourcequota dev-quota -n dev -p '{
  "spec": {
    "hard": {
      "requests.cpu": "30",
      "limits.cpu": "60",
      "pods": "100"
    }
  }
}'

# 验证新配额
kubectl get resourcequota dev-quota -n dev
```

### 8.4.6 ResourceQuota 的限制逻辑

- ResourceQuota 只检查**总量**，不限制单个 Pod
- 如果超出配额，新资源创建会被拒绝
- 已存在的资源不受影响
- 配额检查在 Pod 创建时进行，已运行的 Pod 不会被强制删除

```bash
# 示例：尝试创建超出配额的 Pod
kubectl run over-quota --image=nginx:1.25 --requests='cpu=200' --replicas=100 -n dev
# 如果超过 pods 配额，会报错：
# Error from server (Forbidden): pods "over-quota-..." is forbidden: 
# exceeded quota: dev-quota, requested: count, used: count, limited: count
```

---

## 8.5 LimitRange：默认资源限制

### 8.5.1 LimitRange 概述

LimitRange 为**命名空间内的 Pod/Container 设置默认的资源限制**，解决未指定 resources 的问题。

```
┌─────────────────────────────────────────────────────────┐
│                    Namespace: dev                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              LimitRange                          │    │
│  │  ├── 默认 requests: cpu=100m, memory=128Mi       │    │
│  │  ├── 默认 limits: cpu=200m, memory=256Mi         │    │
│  │  ├── 最大 limits: cpu=2, memory=4Gi              │    │
│  │  └── 最小 requests: cpu=50m, memory=64Mi         │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  Pod A (无 resources) → 自动应用默认值                    │
│  Pod B (有 requests) → 保留 requests，limit 用默认值       │
│  Pod C (超限) → 创建失败                                │
└─────────────────────────────────────────────────────────┘
```

### 8.5.2 LimitRange 的作用类型

| 类型 | 说明 |
|------|------|
| Container | 限制容器级别的 resources |
| Pod | 限制 Pod 级别的 resources |
| PersistentVolumeClaim | 限制 PVC 的容量 |

### 8.5.3 创建 LimitRange

```yaml
# limitrange-dev.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:           # 默认 limit（未指定时使用）
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:    # 默认 request
        cpu: "100m"
        memory: "128Mi"
      max:               # 最大允许值
        cpu: "2"
        memory: "4Gi"
      min:               # 最小允许值
        cpu: "50m"
        memory: "64Mi"
      maxLimitRequestRatio:   # limit/request 最大比例
        cpu: "4"
        memory: "4"
```

```bash
kubectl apply -f limitrange-dev.yaml

# 查看 LimitRange
kubectl get limitrange -n dev
kubectl describe limitrange dev-limits -n dev
```

### 8.5.4 LimitRange 的应用逻辑

```
容器创建时的资源计算：

1. 如果指定了 resources:
   - 使用指定的 requests 和 limits
   - 检查是否在 [min, max] 范围内
   - 检查 limit/request 比例

2. 如果只指定了 requests:
   - limits 使用 default 值
   - 检查是否在 [min, max] 范围内

3. 如果只指定了 limits:
   - requests 使用 defaultRequest 值
   - 检查是否在 [min, max] 范围内

4. 如果完全没指定:
   - requests 使用 defaultRequest 值
   - limits 使用 default 值
```

### 8.5.5 验证 LimitRange 效果

```bash
# 创建一个不指定 resources 的 Pod
cat > no-resources-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: no-resources-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF

kubectl apply -f no-resources-pod.yaml

# 查看实际应用的资源限制
kubectl get pod no-resources-pod -n dev -o jsonpath='{.spec.containers[0].resources}'
# 应该显示默认值：
# {"limits":{"cpu":"200m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}

# 创建一个超过最大限制的 Pod
cat > over-limit-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: over-limit-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        limits:
          cpu: "4"           # 超过 max: 2
          memory: "8Gi"      # 超过 max: 4Gi
EOF

kubectl apply -f over-limit-pod.yaml
# 应该报错：
# Error from server: ... maximum usage per Pod is cpu: 2, memory: 4Gi
```

### 8.5.6 ResourceQuota 与 LimitRange 的区别

| 特性 | ResourceQuota | LimitRange |
|------|--------------|------------|
| 作用范围 | 命名空间**总量** | 单个 Pod/Container |
| 限制对象 | 资源总和 | 每个资源的大小 |
| 未指定资源 | 不限制（可能超卖） | 提供默认值 |
| 超出时 | 新资源创建被拒 | 新资源创建被拒 |
| 配合使用 | 建议配合使用 | 建议配合使用 |

**最佳实践**：同时使用 ResourceQuota 和 LimitRange，两者互补。

---

## 8.6 标签与注解

### 8.6.1 标签（Label）

标签是附加在 K8s 对象上的**键值对**，用于分类和组织资源。

#### 标签的特点

- 可以被选择器（Selector）查询
- 用于资源分组、筛选
- 键名必须符合 DNS 子域名格式
- 值可以为空字符串

#### 标签格式

```
Key: [prefix/]name
  - prefix: 可选，DNS 子域名格式（如 app.kubernetes.io）
  - name: 必须以字母或数字开头和结尾，中间可以包含 -, _, ., /

示例：
  - env: prod
  - app: order-service
  - team: backend
  - version: v1.0.0
  - app.kubernetes.io/name: myapp
  - app.kubernetes.io/version: 2.0.0
```

#### 标签的使用场景

| 场景 | 示例 |
|------|------|
| 环境标识 | `env: dev/staging/prod` |
| 应用标识 | `app: order-service` |
| 版本标识 | `version: v1.0.0` |
| 团队标识 | `team: backend` |
| 组件标识 | `component: database` |
| 分层标识 | `tier: frontend/backend` |

#### 设置与修改标签

```yaml
# 创建时设置标签
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: web-app
    env: dev
    version: "1.0.0"
    team: frontend
spec:
  containers:
    - name: app
      image: nginx:1.25
```

```bash
# 使用 kubectl label 修改标签
kubectl label pod labeled-pod env=staging --overwrite
kubectl label pod labeled-pod version=2.0.0
kubectl label pod labeled-pod team-    # 删除标签

# 批量修改（使用选择器）
kubectl label pods -l app=web-app env=production --overwrite
```

### 8.6.2 注解（Annotation）

注解是附加在 K8s 对象上的**非标识性元数据**，用于存储附加信息。

#### 注解的特点

- 不能被选择器查询
- 用于存储非分类信息
- 可存储任意格式的大段数据
- 常用于记录工具状态、变更记录等

#### 注解的使用场景

| 场景 | 示例 |
|------|------|
| 工具/框架状态 | `ingress.kubernetes.io/rewrite-target: /` |
| 变更记录 | `kubernetes.io/change-cause: "升级到 v2"` |
| 运维信息 | `owner: dev-team`, `contact: ops@example.com` |
| Ingress 配置 | `nginx.ingress.kubernetes.io/...` |
| CI/CD 信息 | `ci-pipeline: build-123`, `commit-hash: abc123` |

#### 设置与修改注解

```yaml
# 创建时设置注解
apiVersion: v1
kind: Deployment
metadata:
  name: annotated-deployment
  annotations:
    owner: dev-team@example.com
    contact: ops-team@example.com
    last-deployed: "2024-01-15"
    change-cause: "升级到 v2.0.0"
    kubernetes.io/change-cause: "升级到 v2.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

```bash
# 使用 kubectl annotate 修改注解
kubectl annotate deployment annotated-deployment owner=team-alpha
kubectl annotate deployment annotated-deployment contact-    # 删除注解

# 查看注解
kubectl describe deployment annotated-deployment
kubectl get deployment annotated-deployment -o jsonpath='{.metadata.annotations}'
```

### 8.6.3 标签与注解的区别

| 特性 | 标签（Label） | 注解（Annotation） |
|------|--------------|-------------------|
| 能否查询 | ✅ 可以（选择器） | ❌ 不能 |
| 用途 | 分类、筛选 | 存储附加信息 |
| 数据类型 | 简单键值对 | 任意格式数据 |
| 大小限制 | 有限 | 较大（可存 JSON） |
| 典型用户 | K8s 系统、开发者 | 运维工具、CI/CD |
| 示例 | `env: prod` | `owner: dev-team` |

---

## 8.7 标签选择器的使用

### 8.7.1 kubectl -l 过滤资源

```bash
# 按单个标签过滤
kubectl get pods -l env=prod
kubectl get pods -l app=order-service

# 按多个标签过滤（AND 关系）
kubectl get pods -l env=prod,app=order-service

# 按标签值存在（不关心具体值）
kubectl get pods -l env

# 按标签值不存在
kubectl get pods -l '!env'

# 按标签值集合（IN、NOT IN）
kubectl get pods -l 'env in (dev, staging)'
kubectl get pods -l 'env notin (dev, staging)'

# 按标签值比较
kubectl get pods -l 'version > 1'
kubectl get pods -l 'version >= 2'
```

### 8.7.2 在 YAML 中使用标签选择器

```yaml
# Deployment 使用标签选择器
apiVersion: apps/v1
kind: Deployment
metadata:
  name: label-selector-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app        # 必须与 template.labels 匹配
      tier: frontend
  template:
    metadata:
      labels:
        app: web-app
        tier: frontend
        version: "2.0"
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

```yaml
# Service 使用标签选择器
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app
    tier: frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```yaml
# NetworkPolicy 使用标签选择器
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-app
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 8080
```

### 8.7.3 复杂标签选择器示例

```yaml
# 使用 matchExpressions 实现复杂逻辑
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-selector-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
    matchExpressions:
      - key: env
        operator: In
        values:
          - prod
          - staging
      - key: version
        operator: NotIn
        values:
          - "1.0"
          - "1.5"
  template:
    metadata:
      labels:
        app: web-app
        env: prod
        version: "2.0"
    spec:
      containers:
        - name: app
          image: nginx:1.25
```

### 8.7.4 跨 Namespace 操作

```bash
# 在特定 Namespace 中按标签过滤
kubectl get pods -n dev -l env=dev
kubectl get services -n production -l app=order-service

# 跨 Namespace 查看（所有 Namespace）
kubectl get pods -A -l app=web-app
kubectl get deployments -A -l team=backend

# 在所有 Namespace 中删除指定标签的资源
kubectl delete pods -A -l env=dev
# 谨慎使用！这会删除所有 Namespace 中带有 env=dev 标签的 Pod
```

---

## 8.8 资源清理策略

### 8.8.1 级联删除

```
删除 Namespace 时的级联关系：

Namespace  ──删除──▶ 所有命名空间级资源
  │                    ├── Pods
  │                    ├── Services
  │                    ├── Deployments
  │                    ├── ConfigMaps
  │                    ├── Secrets
  │                    ├── PVCs
  │                    ├── Roles/RoleBindings
  │                    ├── ResourceQuotas
  │                    └── LimitRanges
  │
  ├── 集群级资源不受影响
  │     ├── Nodes
  │     ├── PVs
  │     ├── StorageClasses
  │     └── ClusterRoles
```

### 8.8.2 删除 Namespace

```bash
# 删除 Namespace（级联删除所有资源）
kubectl delete namespace dev

# 等待 Namespace 删除完成
kubectl get namespace dev
# 输出 "NotFound" 表示删除成功

# 如果 Namespace 卡在 Terminating 状态
# 检查 finalizers
kubectl get namespace dev -o jsonpath='{.metadata.finalizers}'

# 强制删除（从 API Server 移除，不清理资源）
kubectl delete namespace dev --grace-period=0 --force
# 注意：这只是移除 Namespace 对象，内部资源可能还存在
```

### 8.8.3 按标签批量清理

```bash
# 删除特定标签的所有 Pod
kubectl delete pods -l env=dev -n dev

# 删除特定标签的所有 Deployment
kubectl delete deployments -l env=dev -n dev

# 删除特定标签的所有 Service
kubectl delete services -l env=dev -n dev

# 一次性删除所有资源
kubectl delete all -l env=dev -n dev

# 清理配置和密钥
kubectl delete configmaps,secrets -l env=dev -n dev
```

### 8.8.4 清理策略对比

| 策略 | 命令 | 说明 |
|------|------|------|
| 保留 Namespace | 删除内部资源 | `kubectl delete all -n dev` |
| 删除 Namespace | 级联删除全部 | `kubectl delete namespace dev` |
| 按标签清理 | 精确控制 | `kubectl delete pods -l app=xxx` |
| 强制删除 | 跳过优雅删除 | `--grace-period=0 --force` |

### 8.8.5 清理前检查

```bash
# 删除前检查将被清理的资源
kubectl get all -n dev
kubectl get configmaps,secrets -n dev
kubectl get persistentvolumeclaims -n dev

# 跨 Namespace 检查
kubectl get all -A -l env=dev

# 使用 dry-run 预览
kubectl delete all -n dev --dry-run=client
kubectl delete namespace dev --dry-run=client
```

---

## 8.9 实战演练

### 实战1：创建 dev/staging/production 三个 Namespace

**步骤 1**：创建 Namespace 配置文件

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    env: development
    team: backend
    owner: dev-team
  annotations:
    description: 开发环境
    contact: dev-team@example.com
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
    team: backend
    owner: qa-team
  annotations:
    description: 预发环境
    contact: qa-team@example.com
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: backend
    owner: ops-team
  annotations:
    description: 生产环境
    contact: ops-team@example.com
```

**步骤 2**：部署 Namespace

```bash
kubectl apply -f namespaces.yaml

# 验证创建
kubectl get namespaces -l team=backend
kubectl describe namespace dev
```

**步骤 3**：在不同环境中创建应用

```yaml
# apps-dev.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: dev
  labels:
    app: order-service
    env: dev
    team: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
      env: dev
  template:
    metadata:
      labels:
        app: order-service
        env: dev
    spec:
      containers:
        - name: order
          image: nginx:1.25
          ports:
            - containerPort: 8080
          env:
            - name: ENV
              value: "dev"
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: dev
  labels:
    app: order-service
    env: dev
spec:
  selector:
    app: order-service
    env: dev
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

```bash
kubectl apply -f apps-dev.yaml

# 为 staging 创建副本（修改 namespace 和 env 标签）
kubectl apply -f apps-dev.yaml -n staging
kubectl label deployment order-service env=staging --overwrite -n staging

# 为 production 创建
kubectl apply -f apps-dev.yaml -n production
kubectl label deployment order-service env=production --overwrite -n production
kubectl scale deployment order-service --replicas=3 -n production
```

**步骤 4**：验证多环境隔离

```bash
# 查看各环境的资源
kubectl get all -n dev
kubectl get all -n staging
kubectl get all -n production

# 按环境查看所有 Pod
kubectl get pods -A -l app=order-service
```

### 实战2：为 dev 命名空间设置 ResourceQuota

**步骤 1**：创建 ResourceQuota

```yaml
# quota-dev.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
  labels:
    app: quota
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    pods: "50"
    services: "10"
    configmaps: "30"
    secrets: "30"
    count/deployments.apps: "20"
```

```bash
kubectl apply -f quota-dev.yaml
kubectl get resourcequota -n dev
```

**步骤 2**：验证配额效果

```bash
# 查看配额使用情况
kubectl get resourcequota dev-quota -n dev -o yaml

# 创建高资源 Pod
kubectl run high-resource-pod --image=nginx:1.25 \
  --requests='cpu=500m,memory=1Gi' \
  --limits='cpu=1,memory=2Gi' \
  -n dev

# 尝试创建超出配额的 Pod
kubectl run over-quota-pod --image=nginx:1.25 \
  --requests='cpu=500m,memory=2Gi' \
  --replicas=100 \
  -n dev
# 应该报错

# 查看当前使用量
kubectl get resourcequota dev-quota -n dev \
  -o jsonpath='{range .status.hard}{@}{"\t"}{end}'
kubectl get resourcequota dev-quota -n dev \
  -o jsonpath='{range .status.used}{@}{"\t"}{end}'
```

**步骤 3**：更新配额

```bash
# 为 dev 环境扩容
kubectl patch resourcequota dev-quota -n dev -p '{
  "spec": {
    "hard": {
      "requests.cpu": "50",
      "limits.cpu": "100",
      "requests.memory": "100Gi",
      "limits.memory": "200Gi",
      "pods": "200"
    }
  }
}'

# 验证更新
kubectl get resourcequota dev-quota -n dev
```

**步骤 4**：为 staging 和 production 创建配额

```yaml
# quota-staging.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "30"
    requests.memory: "60Gi"
    limits.cpu: "60"
    limits.memory: "120Gi"
    pods: "100"
    persistentvolumeclaims: "20"
---
# quota-production.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "500"
    persistentvolumeclaims: "50"
    services.loadbalancers: "5"
```

```bash
kubectl apply -f quota-staging.yaml
kubectl apply -f quota-production.yaml
```

### 实战3：为命名空间配置 LimitRange（默认 256Mi 内存限制）

**步骤 1**：创建 LimitRange

```yaml
# limitrange-default.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
  labels:
    app: limitrange
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "100m"
        memory: "64Mi"
      maxLimitRequestRatio:
        cpu: "4"
        memory: "4"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
```

```bash
kubectl apply -f limitrange-default.yaml
kubectl get limitrange -n dev
```

**步骤 2**：验证默认限制

```bash
# 创建不指定 resources 的 Pod
kubectl run default-resource-pod --image=nginx:1.25 -n dev

# 查看自动应用的资源限制
kubectl get pod default-resource-pod -n dev \
  -o jsonpath='{.spec.containers[0].resources}'
# 应该显示默认值：
# {"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"250m","memory":"128Mi"}}
```

**步骤 3**：验证超限拒绝

```bash
# 创建超过最大限制的 Pod
cat > over-limit-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: over-limit-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        limits:
          cpu: "8"
          memory: "16Gi"
        requests:
          cpu: "200m"
          memory: "256Mi"
EOF

kubectl apply -f over-limit-pod.yaml
# 应该报错：超过 max 限制
```

**步骤 4**：验证最小限制

```bash
# 创建低于最小限制的 Pod
cat > under-limit-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: under-limit-pod
  namespace: dev
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        limits:
          cpu: "50m"        # 低于 min: 100m
          memory: "32Mi"    # 低于 min: 64Mi
        requests:
          cpu: "10m"       # 低于 min: 100m
          memory: "16Mi"    # 低于 min: 64Mi
EOF

kubectl apply -f under-limit-pod.yaml
# 应该报错：低于 min 限制
```

**步骤 5**：为 staging 和 production 创建 LimitRange

```yaml
# limitrange-staging.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limits
  namespace: staging
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "250m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "100m"
        memory: "64Mi"
---
# limitrange-production.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "1000m"
        memory: "512Mi"
      defaultRequest:
        cpu: "500m"
        memory: "256Mi"
      max:
        cpu: "8"
        memory: "16Gi"
      min:
        cpu: "200m"
        memory: "128Mi"
```

```bash
kubectl apply -f limitrange-staging.yaml
kubectl apply -f limitrange-production.yaml
```

### 实战4：使用 Label 对资源进行分类管理

**步骤 1**：创建带完整标签的应用

```yaml
# label-managed-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  labels:
    app: api-server
    env: production
    team: backend
    tier: api
    version: "2.1.0"
    managed-by: kubernetes
    cost-center: "cc-12345"
  annotations:
    owner: backend-team@example.com
    contact: oncall@example.com
    last-updated: "2024-01-15"
    change-cause: "升级到 v2.1.0，修复登录问题"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
      env: production
  template:
    metadata:
      labels:
        app: api-server
        env: production
        team: backend
        tier: api
        version: "2.1.0"
    spec:
      containers:
        - name: api
          image: nginx:1.25
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: api-server
  namespace: production
  labels:
    app: api-server
    env: production
    team: backend
    tier: api
spec:
  selector:
    app: api-server
    env: production
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

```bash
kubectl apply -f label-managed-app.yaml
```

**步骤 2**：按标签过滤和管理资源

```bash
# 按应用名查看
kubectl get all -l app=api-server -n production

# 按团队查看
kubectl get all -l team=backend -A

# 按环境和层级查看
kubectl get pods -l 'env=production,tier=api' -n production

# 按版本查看
kubectl get pods -l 'version=2.1.0' -A

# 查看所有生产环境资源
kubectl get all -l env=production -A

# 查看团队资源（跨环境）
kubectl get deployments -l team=backend -A
```

**步骤 3**：批量修改标签

```bash
# 升级版本标签
kubectl label deployments -l app=api-server version=2.2.0 --overwrite -A
kubectl label pods -l app=api-server version=2.2.0 --overwrite -A

# 修改团队归属
kubectl label deployments -l team=backend team=frontend --overwrite -n production

# 添加新标签
kubectl label deployments -l app=api-server deprecated=false -A
```

**步骤 4**：使用注解添加元数据

```bash
# 添加维护窗口注解
kubectl annotate deployment api-server \
  maintenance-window="2024-01-20T02:00:00Z" \
  -n production

# 添加变更原因
kubectl annotate deployment api-server \
  kubernetes.io/change-cause="紧急修复：安全补丁" \
  -n production

# 查看注解
kubectl get deployment api-server -n production \
  -o jsonpath='{.metadata.annotations}' | python3 -m json.tool
```

**步骤 5**：基于标签的滚动更新

```bash
# 只更新指定标签的 Deployment
kubectl set image deployment -l app=api-server api=nginx:1.25-alpine -n production

# 只更新指定环境的 Deployment
kubectl set image deployment -l 'app=api-server,env=production' api=nginx:1.25-alpine -A

# 按版本回滚
kubectl rollout undo deployment -l app=api-server -n production
```

### 实战5：使用 kubectl 按 Namespace 和 Label 隔离操作

**步骤 1**：创建多环境资源

```yaml
# multi-env-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
  labels:
    app: frontend
    env: dev
    tier: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        env: dev
    spec:
      containers:
        - name: web
          image: nginx:1.25
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: staging
  labels:
    app: frontend
    env: staging
    tier: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        env: staging
    spec:
      containers:
        - name: web
          image: nginx:1.25
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: production
  labels:
    app: frontend
    env: production
    tier: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        env: production
    spec:
      containers:
        - name: web
          image: nginx:1.25
```

```bash
kubectl apply -f multi-env-apps.yaml
```

**步骤 2**：按 Namespace 隔离操作

```bash
# 只查看 dev 环境
kubectl get all -n dev

# 只查看 staging 环境
kubectl get all -n staging

# 只查看 production 环境
kubectl get all -n production

# 在 dev 中扩缩容
kubectl scale deployment frontend --replicas=3 -n dev

# 在 production 中扩缩容
kubectl scale deployment frontend --replicas=5 -n production
```

**步骤 3**：按 Label 隔离操作

```bash
# 跨 Namespace 查看所有 frontend
kubectl get pods -l app=frontend -A

# 查看不同环境的 frontend
kubectl get pods -l 'app=frontend,env=dev' -A
kubectl get pods -l 'app=frontend,env=staging' -A
kubectl get pods -l 'app=frontend,env=production' -A

# 只重启 dev 环境的 frontend
kubectl rollout restart deployment frontend -l 'env=dev' -A

# 只重启 production 环境的 frontend
kubectl rollout restart deployment frontend -n production
```

**步骤 4**：组合 Namespace 和 Label 操作

```bash
# 在 dev 中按标签过滤
kubectl get pods -n dev -l tier=web

# 在 production 中按标签过滤
kubectl get deployments -n production -l 'tier=web,app=frontend'

# 批量删除指定 Namespace 中指定标签的资源
kubectl delete pods -n dev -l tier=web

# 批量更新指定 Namespace 中指定标签的镜像
kubectl set image deployment -l app=frontend web=nginx:1.25-alpine -n dev
```

**步骤 5**：使用 kubectl 上下文快速切换

```bash
# 设置默认 Namespace
kubectl config set-context --current --namespace=dev

# 查看当前上下文
kubectl config view | grep namespace

# 切换到其他 Namespace
kubectl config set-context --current --namespace=production

# 清除默认 Namespace
kubectl config set-context --current --namespace=default
```

**步骤 6**：批量清理指定环境

```bash
# 清理 dev 环境的所有应用（保留 Namespace）
kubectl delete all -n dev
kubectl delete configmaps,secrets -n dev
kubectl delete rolebindings,roles -n dev
kubectl delete resourcequota,limitrange -n dev

# 彻底删除 dev Namespace（级联删除所有资源）
kubectl delete namespace dev

# 验证清理
kubectl get namespace dev
# 输出: Error from server (NotFound)
```

---

## 8.10 常见问题排查

### 8.10.1 ResourceQuota 不生效

**问题**：ResourceQuota 已创建，但 Pod 仍能超出配额创建。

**排查步骤**：

```bash
# 1. 检查 ResourceQuota 是否存在
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <name> -n <namespace>

# 2. 检查配额是否正确设置
kubectl get resourcequota <name> -n <namespace> -o yaml
# 确认 spec.hard 中的值

# 3. 检查是否超出配额
kubectl get resourcequota <name> -n <namespace> -o jsonpath='{.status}'

# 4. 注意：ResourceQuota 只检查 Pod 的 spec 中明确声明的 requests/limits
# 如果 Pod 没有声明 resources，不计入配额！
# 建议配合 LimitRange 使用，为所有 Pod 设置默认值

# 5. 检查 ResourceQuota 的作用范围
# ResourceQuota 只在命名空间级别生效
# 集群级资源不受 ResourceQuota 限制
```

### 8.10.2 LimitRange 不生效

**问题**：LimitRange 已创建，但 Pod 没有应用默认限制。

**排查步骤**：

```bash
# 1. 检查 LimitRange 是否存在
kubectl get limitrange -n <namespace>
kubectl describe limitrange <name> -n <namespace>

# 2. 检查 Pod 是否在正确的 Namespace
kubectl get pod <name> -o jsonpath='{.metadata.namespace}'

# 3. 检查 Pod 的 resources 是否已显式设置
# 如果 Pod 已设置 resources，LimitRange 不会覆盖
kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources}'

# 4. 检查 LimitRange 的 default 设置
kubectl get limitrange <name> -n <namespace> -o jsonpath='{.spec.limits[0].default}'

# 5. 检查是否有多个 LimitRange 冲突
kubectl get limitrange -n <namespace>
# 如果有多个 LimitRange，确保配置一致
```

### 8.10.3 Namespace 删除卡住

**问题**：Namespace 删除后一直处于 Terminating 状态。

**排查步骤**：

```bash
# 1. 检查 Namespace 的 finalizers
kubectl get namespace <name> -o jsonpath='{.metadata.finalizers}'

# 2. 强制删除（跳过 finalizers）
kubectl delete namespace <name> --grace-period=0 --force

# 3. 手动移除 finalizers
kubectl patch namespace <name> -p '{
  "metadata": {
    "finalizers": []
  }
}'

# 4. 检查是否有资源阻塞删除
kubectl get all -n <namespace>
kubectl get configmaps,secrets -n <namespace>

# 5. 检查是否有 finalizer 阻塞的资源
kubectl get apiservices
# 某些 API service 可能导致阻塞
```

### 8.10.4 标签选择器不匹配

**问题**：使用标签选择器找不到预期的资源。

**排查步骤**：

```bash
# 1. 检查标签拼写
kubectl get pod <name> -o jsonpath='{.metadata.labels}'
# 确保标签键和值完全匹配（区分大小写）

# 2. 检查 YAML 中的 matchLabels 是否与 template.labels 一致
kubectl get deployment <name> -o jsonpath='{.spec.selector.matchLabels}'
kubectl get deployment <name> -o jsonpath='{.spec.template.metadata.labels}'
# 两者必须一致

# 3. 检查是否在正确的 Namespace
kubectl get resources -n <namespace> -l <selector>
kubectl get resources -A -l <selector>

# 4. 使用 --show-labels 查看所有标签
kubectl get pods --show-labels
kubectl get deployments --show-labels
```

### 8.10.5 ResourceQuota 与 LimitRange 冲突

**问题**：同时设置 ResourceQuota 和 LimitRange 时，Pod 创建被拒绝。

**排查步骤**：

```bash
# 1. 确认 ResourceQuota 限制
kubectl get resourcequota -n <namespace> -o yaml

# 2. 确认 LimitRange 限制
kubectl get limitrange -n <namespace> -o yaml

# 3. 确认 Pod 的 resources 设置
kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources}'

# 4. 常见冲突场景：
#    - LimitRange 的 max 小于 ResourceQuota 中单个 Pod 的合理值
#    - ResourceQuota 总量太小，无法容纳多个 Pod
#    - LimitRange 的默认值导致总量超出 ResourceQuota

# 5. 调整建议：
#    - LimitRange 控制单 Pod 的资源范围
#    - ResourceQuota 控制命名空间的总量
#    - 两者协调：LimitRange.max <= ResourceQuota.hard 的合理分配
```

---

## 8.11 章节小结

### 核心知识点回顾

1. **Namespace 基础**：
   - 多租户隔离、资源分组、生命周期管理
   - 命名空间级资源 vs 集群级资源
   - 默认 Namespace：default、kube-system、kube-public、kube-node-lease

2. **ResourceQuota**：
   - 命名空间级总资源限制
   - 限制 CPU、内存、Pod 数量、PVC 数量等
   - 只检查 Pod 声明的 resources，需配合 LimitRange 使用

3. **LimitRange**：
   - 为 Pod/Container 设置默认资源限制
   - 设置最小/最大允许值
   - 设置 limit/request 的最大比例

4. **标签与注解**：
   - 标签：可查询、用于分类
   - 注解：不可查询、用于存储附加信息
   - 两者格式和使用场景不同

5. **资源清理**：
   - 删除 Namespace 级联删除所有资源
   - 按标签精确清理
   - 清理前使用 `--dry-run=client` 预览

### 常用命令速查

```bash
# Namespace 管理
kubectl create namespace <name>
kubectl get namespaces
kubectl get namespaces -l <label>
kubectl delete namespace <name>
kubectl delete namespace <name> --grace-period=0 --force

# ResourceQuota 管理
kubectl get resourcequota -n <namespace>
kubectl get resourcequota <name> -n <namespace> -o yaml
kubectl apply -f quota.yaml
kubectl delete resourcequota <name> -n <namespace>
kubectl patch resourcequota <name> -n <namespace> -p '{...}'

# LimitRange 管理
kubectl get limitrange -n <namespace>
kubectl get limitrange <name> -n <namespace> -o yaml
kubectl apply -f limitrange.yaml
kubectl delete limitrange <name> -n <namespace>

# 标签操作
kubectl label <resource> <name> <key>=<value>
kubectl label <resource> <name> <key>-     # 删除标签
kubectl label <resource> -l <selector> <key>=<value>  # 批量修改
kubectl get <resource> --show-labels

# 注解操作
kubectl annotate <resource> <name> <key>=<value>
kubectl annotate <resource> <name> <key>-  # 删除注解
kubectl get <resource> <name> -o jsonpath='{.metadata.annotations}'

# 标签选择器
kubectl get pods -l app=myapp
kubectl get pods -l 'env in (dev, staging)'
kubectl get pods -l 'app=myapp,tier=api' -n <namespace>
kubectl get pods -A -l <selector>           # 跨 Namespace
```

### 最佳实践总结

| 场景 | 推荐方案 |
|------|---------|
| 多环境隔离 | 不同 Namespace + 环境标签 |
| 多团队隔离 | 不同 Namespace + RBAC |
| 资源配额 | ResourceQuota + LimitRange 配合 |
| 资源分类 | 使用 Labels + Annotations |
| 批量操作 | 使用标签选择器 `-l` |
| 清理资源 | 优先删除 Namespace，或按标签精确清理 |
| 生产环境 | 严格的 ResourceQuota + LimitRange + RBAC |

### Namespace 规划建议

```
推荐的 Namespace 组织方式：

方式一：按环境（简单）
├── dev          # 开发环境
├── staging      # 预发环境
├── production   # 生产环境
└── kube-system  # 系统

方式二：按团队 + 环境（推荐）
├── team-a-dev
├── team-a-staging
├── team-a-production
├── team-b-dev
├── team-b-staging
└── team-b-production

方式三：按业务模块
├── order-service
├── payment-service
├── user-service
├── notification-service
└── kube-system
```
