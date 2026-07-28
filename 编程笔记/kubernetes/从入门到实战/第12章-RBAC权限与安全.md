# 第12章 RBAC 权限与安全

## 学习目标

- 理解 RBAC 的核心概念：Role、ClusterRole、RoleBinding、ClusterRoleBinding、ServiceAccount
- 掌握 RBAC 的作用域划分：Namespace 级与 Cluster 级
- 理解权限描述结构（apiGroups / resources / verbs）并能独立编写
- 掌握常见权限场景：只读权限、命名空间管理员、集群管理员
- 理解最小权限原则并能在实际中应用
- 熟练使用 `kubectl auth can-i` 验证权限
- 通过 5 个实战练习，掌握 RBAC 的创建、绑定和验证

---

## 12.1 RBAC 核心概念

### 12.1.1 什么是 RBAC

RBAC（Role-Based Access Control，基于角色的访问控制）是 Kubernetes 中管理集群访问权限的核心机制。它通过**角色**定义权限，通过**绑定**将角色赋予用户或 Pod。

```
┌─────────────────────────────────────────────────────────────┐
│                    RBAC 权限模型                               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    主体（Subject）                        │ │
│  │                                                         │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────────────────┐    │ │
│  │  │  User   │  │  Group  │  │ ServiceAccount     │    │ │
│  │  │  集群用户│  │  用户组 │  │ Pod 使用的身份       │    │ │
│  │  └────┬────┘  └────┬────┘  └────────┬────────────┘    │ │
│  │       │            │                │                    │ │
│  └───────┼────────────┼────────────────┼────────────────────┘ │
│          │            │                │                       │
│          ▼            ▼                ▼                       │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    绑定（Binding）                       │ │
│  │                                                         │ │
│  │  ┌─────────────────┐    ┌─────────────────────────┐   │ │
│  │  │  RoleBinding    │    │  ClusterRoleBinding      │   │ │
│  │  │  命名空间级绑定  │    │  集群级绑定              │   │ │
│  │  └────────┬────────┘    └────────────┬─────────────┘   │ │
│  └───────────┼──────────────────────────┼───────────────────┘ │
│              │                          │                     │
│              ▼                          ▼                     │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    角色（Role）                          │ │
│  │                                                         │ │
│  │  ┌─────────────────┐    ┌─────────────────────────┐   │ │
│  │  │  Role           │    │  ClusterRole             │   │ │
│  │  │  命名空间级角色  │    │  集群级角色              │   │ │
│  │  └────────┬────────┘    └────────────┬─────────────┘   │ │
│  └───────────┼──────────────────────────┼───────────────────┘ │
│              │                          │                     │
│              ▼                          ▼                     │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    权限（Rules）                         │ │
│  │                                                         │ │
│  │  apiGroups: ["apps"]                                    │ │
│  │  resources: ["deployments"]                             │ │
│  │  verbs: ["get", "list", "watch"]                        │ │
│  │                                                         │ │
│  │  → 允许对 apps/deployments 执行 get/list/watch 操作     │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 12.1.2 RBAC 四大核心资源

| 资源 | 作用 | 作用域 |
|------|------|--------|
| **Role** | 定义在某个命名空间内的权限规则 | Namespace |
| **ClusterRole** | 定义集群范围内的权限规则 | Cluster |
| **RoleBinding** | 将 Role 绑定到主体（用户/组/ServiceAccount） | Namespace |
| **ClusterRoleBinding** | 将 ClusterRole 绑定到主体 | Cluster |

### 12.1.3 Role 与 ClusterRole

```
Role（命名空间级）:
┌─────────────────────────────────────┐
│ Namespace: production                │
│ ├── 允许读取 production 命名空间的    │
│ │   deployments, pods, services      │
│ └── 不影响其他命名空间                │
└─────────────────────────────────────┘

ClusterRole（集群级）:
┌─────────────────────────────────────┐
│ Cluster: all                        │
│ ├── 允许读取所有命名空间的            │
│ │   deployments, pods, services      │
│ ├── 允许查看集群级资源（nodes, ns）    │
│ └── 跨命名空间生效                    │
└─────────────────────────────────────┘
```

### 12.1.4 ServiceAccount

ServiceAccount 是 Pod 在集群中的身份标识，Pod 通过 ServiceAccount 访问集群 API 时会根据绑定的 RBAC 规则进行权限校验。

```
┌─────────────────────────────────────────────────────────┐
│              ServiceAccount 工作流程                      │
│                                                           │
│  Pod (ServiceAccount: app-sa)                             │
│    │                                                      │
│    │  1. Pod 自动挂载 ServiceAccount Token                 │
│    │     /var/run/secrets/kubernetes.io/serviceaccount/   │
│    │     ├── token                                        │
│    │     ├── ca.crt                                       │
│    │     └── namespace                                    │
│    │                                                      │
│    ▼                                                      │
│  Kubernetes API Server                                    │
│    │                                                      │
│    │  2. 验证 Token，提取 ServiceAccount 身份              │
│    │                                                      │
│    ▼                                                      │
│  RBAC 权限检查                                            │
│    │                                                      │
│    │  3. 检查 RoleBinding / ClusterRoleBinding             │
│    │     → 确定该 SA 拥有哪些权限                         │
│    │                                                      │
│    ▼                                                      │
│  允许 / 拒绝操作                                           │
└─────────────────────────────────────────────────────────┘
```

---

## 12.2 RBAC 作用域详解

### 12.2.1 两种作用域对比

| 维度 | Namespace 级 | Cluster 级 |
|------|-------------|------------|
| **资源** | Role + RoleBinding | ClusterRole + ClusterRoleBinding |
| **生效范围** | 单个命名空间 | 所有命名空间 |
| **可管理资源** | 命名空间级资源 | 集群级 + 所有命名空间级资源 |
| **复用性** | 每个命名空间需单独创建 | 创建一次，全局生效 |
| **适用场景** | 多租户隔离 | 全局管理 |

### 12.2.2 命名空间级资源 vs 集群级资源

#### 命名空间级资源（受 Role 控制）

```bash
# 以下资源在命名空间内生效
kubectl api-resources --namespaced=true
# NAME                  SHORTNAMES   APIVERSION
# configmaps            cm           v1
# pods                  po           v1
# secrets               sec          v1
# serviceaccounts       sa           v1
# services              svc          v1
# deployments           deploy       apps/v1
# statefulsets          sts          apps/v1
# daemonsets            ds           apps/v1
# replicasets           rs           apps/v1
# jobs                  job          batch/v1
# cronjobs              cj           batch/v1
# ingresses             ing          networking.k8s.io/v1
# ...
```

#### 集群级资源（仅受 ClusterRole 控制）

```bash
# 以下资源是集群级的，不受 Role 控制
kubectl api-resources --namespaced=false
# NAME                  SHORTNAMES   APIVERSION
# nodes                 no           v1
# namespaces            ns           v1
# persistentvolumes     pv           v1
# clusterroles          rbac.authorization.k8s.io/v1
# clusterrolebindings   rb           rbac.authorization.k8s.io/v1
# roles                 r            rbac.authorization.k8s.io/v1
# rolebindings          rb           rbac.authorization.k8s.io/v1
# storageclasses        sc           storage.k8s.io/v1
# customresourcedefinitions   crd    apiextensions.k8s.io/v1
# ...
```

### 12.2.3 ClusterRole 的双重用途

ClusterRole 既可以在集群范围内使用，也可以通过 RoleBinding 在特定命名空间内引用，实现权限复用。

```yaml
# 方式一：ClusterRoleBinding（集群级）
# 适用于需要跨命名空间或访问集群级资源的场景
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

# 方式二：RoleBinding 引用 ClusterRole（命名空间级）
# 适用于多个命名空间需要相同权限配置的场景
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development      # 仅在 development 命名空间生效
subjects:
  - kind: User
    name: developer
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-permissions  # 引用已存在的 ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

---

## 12.3 权限描述结构

### 12.3.1 Rules 结构详解

每条权限规则（Rule）由三个核心字段组成：

```yaml
rules:
  - apiGroups: ["apps"]          # API 组
    resources: ["deployments"]  # 资源类型
    verbs: ["get", "list", "watch"]  # 操作动词

  # 含义：允许对 apps/deployments 资源执行 get/list/watch 操作
```

#### apiGroups

| 值 | 说明 | 示例 |
|------|------|------|
| `[""]` | 核心 API 组（空字符串） | pods, services, configmaps, secrets |
| `["apps"]` | apps API 组 | deployments, statefulsets, daemonsets |
| `["batch"]` | batch API 组 | jobs, cronjobs |
| `["networking.k8s.io"]` | 网络 API 组 | ingresses, networkpolicies |
| `["rbac.authorization.k8s.io"]` | RBAC API 组 | roles, rolebindings, clusterroles |
| `["*"]` | 所有 API 组 | 全部 |

#### resources

| 值 | 说明 |
|------|------|
| `["pods"]` | Pod 资源 |
| `["services"]` | Service 资源 |
| `["deployments"]` | Deployment 资源 |
| `["*"]` | 所有资源 |

#### verbs

| 值 | 说明 | HTTP 方法 |
|------|------|-----------|
| `get` | 获取单个资源 | GET |
| `list` | 列出资源集合 | GET（集合） |
| `watch` | 监听资源变更 | WSS |
| `create` | 创建资源 | POST |
| `update` | 更新资源（全量） | PUT |
| `patch` | 更新资源（部分） | PATCH |
| `delete` | 删除资源 | DELETE |
| `deletecollection` | 删除资源集合 | DELETE（集合） |
| `*` | 所有操作 | - |

#### 特殊资源与子资源

```yaml
# 子资源（Subresource）权限
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "list", "create"]

  # 注意：pods/log, pods/exec 是 Pod 的子资源
  # 需要单独授权才能使用 kubectl logs, kubectl exec
```

### 12.3.2 权限组合示例

#### 示例 1：只读权限

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch"]
```

#### 示例 2：完整管理权限（特定命名空间）

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

#### 示例 3：限制删除权限

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  # 不包含 delete 和 deletecollection
```

#### 示例 4：Pod 日志访问权限

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
```

---

## 12.4 常见权限场景

### 12.4.1 只读权限（ReadOnly）

适用于：监控系统、审计系统、只读运维

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: reader
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/status"]
    verbs: ["get", "list"]
```

### 12.4.2 命名空间管理员权限（Namespace Admin）

适用于：项目负责人、开发团队

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: development
rules:
  # 管理应用资源
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理 Pod 和日志
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "create"]

  # 管理服务和网络
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理配置
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理存储
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理 CronJob 和 Job
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理 HPA
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理资源配额和限制
  - apiGroups: [""]
    resources: ["resourcequotas", "limitranges"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### 12.4.3 集群管理员权限（Cluster Admin）

**警告：** 集群管理员拥有完全控制权，应谨慎使用

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  - nonResourceURLs: ["*"]
    verbs: ["*"]
```

#### 内置 ClusterRole

| 名称 | 说明 |
|------|------|
| `cluster-admin` | 完全控制权 |
| `admin` | 命名空间级管理员（不含集群级操作） |
| `edit` | 命名空间级编辑权限 |
| `view` | 命名空间级只读权限 |

### 12.4.4 权限场景汇总表

| 场景 | 角色 | 作用域 | 风险等级 |
|------|------|--------|----------|
| 监控系统 | ClusterRole: reader | 全局 | 低 |
| 开发团队 | Role: namespace-admin | 命名空间 | 中 |
| 运维团队 | ClusterRole: admin | 全局 | 高 |
| CI/CD 系统 | 自定义 Role | 命名空间 | 中 |
| 审计系统 | ClusterRole: reader + logs | 全局 | 低 |
| 服务 Mesh | ClusterRole | 全局 | 高 |
| 日志收集 | ClusterRole + 读 Pod 日志 | 全局 | 中 |

---

## 12.5 最小权限原则

### 12.5.1 什么是最小权限原则

最小权限原则（Principle of Least Privilege）是指授予主体完成其任务所需的**最少权限**，不多也不少。

```
┌─────────────────────────────────────────────────────────┐
│              最小权限原则示意                              │
│                                                           │
│  需求：日志收集 Pod 需要读取 Pod 日志                     │
│                                                           │
│  ❌ 过度授权：                                            │
│     verbs: ["*"]  resources: ["*"]                       │
│     → 可以删除所有资源                                    │
│                                                           │
│  ✅ 最小权限：                                            │
│     verbs: ["get", "list"]                               │
│     resources: ["pods/log"]                              │
│     → 只能读取 Pod 日志                                   │
│                                                           │
│  关键问题：                                                │
│  1. 任务需要哪些权限？                                    │
│  2. 能否限制只在特定命名空间生效？                        │
│  3. 能否限制只访问特定资源类型？                          │
│  4. 能否限制只执行特定操作？                              │
└─────────────────────────────────────────────────────────┘
```

### 12.5.2 实施最小权限的步骤

1. **明确需求**：列出主体需要执行的所有操作
2. **分析 API 调用**：确定每个操作对应的 API 资源和动词
3. **选择作用域**：尽量使用 Role 而非 ClusterRole
4. **绑定主体**：使用 RoleBinding 将权限绑定到最小范围
5. **验证权限**：使用 `kubectl auth can-i` 验证
6. **持续审计**：定期审查权限配置

### 12.5.3 常见过度授权案例

| 案例 | 问题 | 改进 |
|------|------|------|
| Pod 读日志却给了 `*` | 权限过大 | 只授权 `pods/log` 的 `get,list` |
| 监控 Pod 有写权限 | 违反只读原则 | 只授权 `get,list,watch` |
| 单租户使用 ClusterRole | 影响范围过大 | 改用 Role + RoleBinding |
| CI/CD 有删除权限 | 误删风险 | 限制删除特定资源类型 |

### 12.5.4 权限风险评估

```
风险等级评估：

低风险：只读权限，无修改操作
  → view, reader 角色

中风险：创建/修改权限，无删除操作
  → edit, developer 角色

高风险：删除权限，可能影响业务
  → admin, namespace-admin 角色

极高风险：集群级完全控制
  → cluster-admin 角色
  → 仅限极少数管理员使用
```

---

## 12.6 kubectl auth can-i 验证权限

### 12.6.1 基本用法

```bash
# 检查当前用户是否有权执行操作
kubectl auth can-i <verb> <resource>
kubectl auth can-i list pods
kubectl auth can-i get deployments
kubectl auth can-i create pods
kubectl auth can-i delete services

# 检查指定命名空间
kubectl auth can-i get pods --namespace=production
kubectl auth can-i list deployments -n staging

# 检查指定用户（使用 --as 参数模拟）
kubectl auth can-i get pods --as=developer
kubectl auth can-i create deployments --as=ci-user

# 检查指定 ServiceAccount
kubectl auth can-i list pods --as=system:serviceaccount:default:app-sa

# 检查当前上下文的用户
kubectl config view --minify | grep user
```

### 12.6.2 检查 RBAC 配置

```bash
# 查看当前用户拥有的所有权限
kubectl auth can-i --list

# 查看指定用户的权限
kubectl auth can-i --list --as=developer

# 查看指定 ServiceAccount 的权限
kubectl auth can-i --list --as=system:serviceaccount:default:app-sa

# 查看所有 ClusterRole
kubectl get clusterrole

# 查看 ClusterRole 详情
kubectl describe clusterrole <name>

# 查看所有 ClusterRoleBinding
kubectl get clusterrolebinding

# 查看 ClusterRoleBinding 详情
kubectl describe clusterrolebinding <name>

# 查看所有 Role（指定命名空间）
kubectl get role -n <namespace>

# 查看所有 RoleBinding（指定命名空间）
kubectl get rolebinding -n <namespace>
```

### 12.6.3 排查 RBAC 问题

```bash
# 场景：Pod 中 ServiceAccount 无法访问 API

# 步骤 1：检查 ServiceAccount 是否存在
kubectl get serviceaccount <sa-name> -n <namespace>

# 步骤 2：检查 RoleBinding
kubectl get rolebinding -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>

# 步骤 3：检查 Role
kubectl get role -n <namespace>
kubectl describe role <role-name> -n <namespace>

# 步骤 4：模拟 SA 身份验证权限
kubectl auth can-i get pods \
  --as=system:serviceaccount:<namespace>:<sa-name>

# 步骤 5：查看 SA 可以执行的所有操作
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name>

# 步骤 6：检查 Pod 中 SA Token
kubectl exec -it <pod-name> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# 步骤 7：查看 RBAC 事件
kubectl get events -n <namespace> | grep -i rbac
```

---

## 12.7 实战练习

### 实战 1：创建只读 Role 和 RoleBinding

**目标：** 创建一个只读 Role，允许用户查看指定命名空间的所有资源。

```bash
# 第 1 步：创建开发命名空间
kubectl create namespace demo-rbac

# 第 2 步：创建只读 Role
cat > /tmp/reader-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: reader
  namespace: demo-rbac
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/status"]
    verbs: ["get", "list"]
EOF

kubectl apply -f /tmp/reader-role.yaml

# 第 3 步：创建 RoleBinding
cat > /tmp/reader-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: reader-binding
  namespace: demo-rbac
subjects:
  - kind: User
    name: readonly-user
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: readonly-group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/reader-binding.yaml

# 第 4 步：部署测试应用
kubectl create deployment nginx -n demo-rbac --image=nginx:alpine
kubectl expose deployment nginx -n demo-rbac --port=80

# 第 5 步：验证只读权限
# 可以读取
kubectl auth can-i get pods -n demo-rbac --as=readonly-user
kubectl auth can-i list pods -n demo-rbac --as=readonly-user
kubectl auth can-i get deployments -n demo-rbac --as=readonly-user

# 不能创建
kubectl auth can-i create pods -n demo-rbac --as=readonly-user
# no

# 不能删除
kubectl auth can-i delete pods -n demo-rbac --as=readonly-user
# no

# 不能修改
kubectl auth can-i update deployments -n demo-rbac --as=readonly-user
# no

# 第 6 步：查看权限详情
kubectl get role reader -n demo-rbac -o yaml
kubectl get rolebinding reader-binding -n demo-rbac -o yaml

# 第 7 步：查看 SA 拥有的所有权限
kubectl auth can-i --list --as=readonly-user -n demo-rbac

# 第 8 步：清理
kubectl delete role reader -n demo-rbac
kubectl delete rolebinding reader-binding -n demo-rbac
kubectl delete namespace demo-rbac
```

### 实战 2：创建命名空间管理员权限

**目标：** 为开发团队创建命名空间管理员权限，允许其管理开发命名空间的所有应用资源。

```bash
# 第 1 步：创建开发命名空间
kubectl create namespace dev-team

# 第 2 步：创建命名空间管理员 Role
cat > /tmp/namespace-admin-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: dev-team
rules:
  # 管理工作负载
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理 Pod 及相关操作
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "create"]

  # 管理网络资源
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "networkpolicies"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理配置和密钥
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理存储
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理 ServiceAccount
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get", "list", "watch", "create"]

  # 管理批处理任务
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理扩缩容
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

kubectl apply -f /tmp/namespace-admin-role.yaml

# 第 3 步：创建 RoleBinding
cat > /tmp/namespace-admin-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: namespace-admin-binding
  namespace: dev-team
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
  - kind: User
    name: lead-developer
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/namespace-admin-binding.yaml

# 第 4 步：部署测试应用
kubectl create deployment web-app -n dev-team --image=nginx:alpine
kubectl expose deployment web-app -n dev-team --port=80

# 第 5 步：验证管理员权限
echo "=== 验证开发团队权限 ==="

# 可以部署应用
kubectl auth can-i create deployments -n dev-team --as=lead-developer
# yes

# 可以删除应用
kubectl auth can-i delete deployments -n dev-team --as=lead-developer
# yes

# 可以查看日志
kubectl auth can-i get pods/log -n dev-team --as=lead-developer
# yes

# 可以 exec 进入 Pod
kubectl auth can-i create pods/exec -n dev-team --as=lead-developer
# yes

# 不能修改 RBAC
kubectl auth can-i create roles -n dev-team --as=lead-developer
# no

# 不能访问其他命名空间
kubectl auth can-i get pods -n default --as=lead-developer
# no

# 不能管理节点
kubectl auth can-i get nodes --as=lead-developer
# no

# 第 6 步：查看完整权限列表
kubectl auth can-i --list --as=lead-developer -n dev-team

# 第 7 步：对比内置 admin 角色
kubectl create rolebinding admin-binding -n dev-team \
  --clusterrole=admin \
  --user=admin-user

kubectl auth can-i --list --as=admin-user -n dev-team

# 第 8 步：清理
kubectl delete role namespace-admin -n dev-team
kubectl delete rolebinding namespace-admin-binding -n dev-team
kubectl delete rolebinding admin-binding -n dev-team
kubectl delete namespace dev-team
```

### 实战 3：创建 ServiceAccount 并绑定权限，在 Pod 中使用

**目标：** 创建一个 ServiceAccount，绑定指定权限，并在 Pod 中使用它访问 Kubernetes API。

```bash
# 第 1 步：创建测试命名空间
kubectl create namespace sa-demo

# 第 2 步：创建 ServiceAccount
cat > /tmp/app-sa.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: sa-demo
  labels:
    app: app-sa
    component: api-accessor
  annotations:
    description: "用于应用访问 K8s API 的 ServiceAccount"
EOF

kubectl apply -f /tmp/app-sa.yaml

# 第 3 步：创建 Role（只允许读取 Pod 信息）
cat > /tmp/app-role.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: sa-demo
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list"]
EOF

kubectl apply -f /tmp/app-role.yaml

# 第 4 步：创建 RoleBinding
cat > /tmp/app-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-binding
  namespace: sa-demo
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: sa-demo
roleRef:
  kind: Role
  name: app-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/app-binding.yaml

# 第 5 步：创建使用该 SA 的 Pod
cat > /tmp/api-client-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: api-client
  namespace: sa-demo
  labels:
    app: api-client
spec:
  serviceAccountName: app-sa
  serviceAccount: app-sa
  containers:
    - name: client
      image: bitnami/kubectl:latest
      command:
        - sh
        - -c
        - |
          echo "=== ServiceAccount 信息 ==="
          echo "Namespace: $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)"
          echo ""
          
          echo "=== 列出当前命名空间的 Pod ==="
          kubectl get pods -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
          echo ""
          
          echo "=== 列出 Deployment ==="
          kubectl get deployments -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
          echo ""
          
          echo "=== 列出 Services ==="
          kubectl get services -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
          echo ""
          
          echo "=== 测试受限操作（应该被拒绝）==="
          kubectl delete pod --help | head -1
          echo "如果没有权限，删除操作会失败"
          echo ""
          
          echo "=== 持续运行以保留 Pod ==="
          while true; do
            kubectl get pods -n $(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
            sleep 30
          done
      env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
  restartPolicy: Never
EOF

kubectl apply -f /tmp/api-client-pod.yaml

# 第 6 步：创建测试 Deployment（验证 SA 能读取）
kubectl create deployment test-app -n sa-demo --image=nginx:alpine
kubectl expose deployment test-app -n sa-demo --port=80

# 第 7 步：查看 Pod 日志
kubectl wait --for=condition=Ready pod/api-client -n sa-demo --timeout=60s
kubectl logs -f api-client -n sa-demo

# 第 8 步：验证 SA 权限
echo "=== SA 权限验证 ==="

# 从 Pod 内验证权限
kubectl exec -it api-client -n sa-demo -- kubectl auth can-i list pods
# yes

kubectl exec -it api-client -n sa-demo -- kubectl auth can-i list deployments
# yes

kubectl exec -it api-client -n sa-demo -- kubectl auth can-i delete pods
# no

# 从外部模拟 SA 验证
kubectl auth can-i list pods -n sa-demo \
  --as=system:serviceaccount:sa-demo:app-sa
# yes

kubectl auth can-i delete pods -n sa-demo \
  --as=system:serviceaccount:sa-demo:app-sa
# no

# 第 9 步：查看 SA 完整权限
kubectl auth can-i --list --as=system:serviceaccount:sa-demo:app-sa -n sa-demo

# 第 10 步：禁用 SA 的自动挂载 Token
# 安全加固：如果 Pod 不需要访问 API，可以禁用
cat > /tmp/no-token-sa.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-sa
  namespace: sa-demo
automountServiceAccountToken: false
EOF

kubectl apply -f /tmp/no-token-sa.yaml

# 第 11 步：清理
kubectl delete pod api-client -n sa-demo
kubectl delete deployment test-app -n sa-demo
kubectl delete service test-app -n sa-demo
kubectl delete role app-role -n sa-demo
kubectl delete rolebinding app-binding -n sa-demo
kubectl delete serviceaccount app-sa no-api-sa -n sa-demo
kubectl delete namespace sa-demo
```

### 实战 4：使用 kubectl auth can-i 验证权限

**目标：** 通过各种场景演示 `kubectl auth can-i` 的使用，排查权限问题。

```bash
# 第 1 步：创建测试命名空间和 RBAC 配置
kubectl create namespace auth-test

cat > /tmp/auth-test-setup.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: limited-role
  namespace: auth-test
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["*"]
  - apiGroups: ["batch"]
    resources: ["cronjobs"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: limited-binding
  namespace: auth-test
subjects:
  - kind: User
    name: limited-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: limited-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/auth-test-setup.yaml

# 第 2 步：基本权限检查
echo "=== 基本权限检查 ==="

# 检查 get pods 权限
kubectl auth can-i get pods --as=limited-user -n auth-test
# yes

# 检查 list pods 权限
kubectl auth can-i list pods --as=limited-user -n auth-test
# yes

# 检查 create pods 权限（应该没有）
kubectl auth can-i create pods --as=limited-user -n auth-test
# no

# 检查 delete pods 权限（应该没有）
kubectl auth can-i delete pods --as=limited-user -n auth-test
# no

# 第 3 步：检查不同资源类型
echo "=== 资源类型检查 ==="

# Deployments（有 get/list/create）
kubectl auth can-i get deployments --as=limited-user -n auth-test
# yes
kubectl auth can-i create deployments --as=limited-user -n auth-test
# yes
kubectl auth can-i delete deployments --as=limited-user -n auth-test
# no

# Services（有 * 权限）
kubectl auth can-i get services --as=limited-user -n auth-test
# yes
kubectl auth can-i create services --as=limited-user -n auth-test
# yes
kubectl auth can-i delete services --as=limited-user -n auth-test
# yes
kubectl auth can-i update services --as=limited-user -n auth-test
# yes

# CronJobs（只有 get/list/watch）
kubectl auth can-i get cronjobs --as=limited-user -n auth-test
# yes
kubectl auth can-i create cronjobs --as=limited-user -n auth-test
# no

# 其他资源（没有配置权限）
kubectl auth can-i get secrets --as=limited-user -n auth-test
# no
kubectl auth can-i get configmaps --as=limited-user -n auth-test
# no

# 第 4 步：跨命名空间检查
echo "=== 跨命名空间检查 ==="

kubectl auth can-i get pods --as=limited-user -n default
# no

kubectl auth can-i get pods --as=limited-user -n kube-system
# no

# 第 5 步：查看完整权限列表
echo "=== 完整权限列表 ==="
kubectl auth can-i --list --as=limited-user -n auth-test

# 第 6 步：非资源 URL 检查
echo "=== 非资源 URL 检查 ==="

# /healthz, /readyz, /livez 等端点
kubectl auth can-i get /healthz
kubectl auth can-i get /readyz
kubectl auth can-i get /api/v1/namespaces

# 第 7 步：模拟 Pod 中的 ServiceAccount
echo "=== ServiceAccount 权限检查 ==="

# 创建 ServiceAccount 和绑定
kubectl create serviceaccount test-sa -n auth-test
kubectl create rolebinding test-sa-binding -n auth-test \
  --role=limited-role \
  --serviceaccount=auth-test:test-sa

# 验证 SA 权限
kubectl auth can-i list pods -n auth-test \
  --as=system:serviceaccount:auth-test:test-sa
# yes

kubectl auth can-i delete pods -n auth-test \
  --as=system:serviceaccount:auth-test:test-sa
# no

# 查看 SA 权限列表
kubectl auth can-i --list \
  --as=system:serviceaccount:auth-test:test-sa \
  -n auth-test

# 第 8 步：排查常见 RBAC 错误
echo "=== 常见错误排查 ==="

# 错误 1：Pod 无法访问 API（403 Forbidden）
# 检查 ServiceAccount 是否存在
kubectl get serviceaccount -n auth-test
# 检查 RoleBinding 是否创建
kubectl get rolebinding -n auth-test
# 检查 Role 是否有对应权限
kubectl get role limited-role -n auth-test -o yaml

# 错误 2：权限已配置但仍被拒绝
# 检查是否拼写错误
kubectl auth can-i get po --as=limited-user -n auth-test
# 注意：资源名称必须用复数形式（pods，不是 po）

# 错误 3：跨命名空间访问被拒绝
# Role 只在当前命名空间生效
# 需要在目标命名空间单独创建 RoleBinding
kubectl create rolebinding test-sa-binding-2 -n default \
  --clusterrole=view \
  --serviceaccount=auth-test:test-sa

# 第 9 步：使用 kubectl explain 查看 RBAC 资源
kubectl explain role
kubectl explain rolebindings
kubectl explain clusterrole
kubectl explain clusterrolebinding

# 第 10 步：清理
kubectl delete role limited-role -n auth-test
kubectl delete rolebinding limited-binding test-sa-binding -n auth-test
kubectl delete serviceaccount test-sa -n auth-test
kubectl delete rolebinding test-sa-binding-2 -n default
kubectl delete namespace auth-test
```

### 实战 5：创建跨命名空间的 ClusterRole

**目标：** 创建一个 ClusterRole，允许运维团队跨命名空间查看日志和管理应用。

```bash
# 第 1 步：创建集群级 ClusterRole（日志查看）
cat > /tmp/cluster-log-reader.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-log-reader
  labels:
    app: ops-monitoring
    environment: production
rules:
  # 读取所有命名空间的 Pod 信息
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

  # 读取 Pod 日志
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]

  # 查看 Pod 状态
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["get", "list", "watch"]

  # 查看部署状态
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]

  # 查看服务状态
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch"]
EOF

kubectl apply -f /tmp/cluster-log-reader.yaml

# 第 2 步：创建 ClusterRoleBinding
cat > /tmp/cluster-log-reader-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-log-reader-binding
  labels:
    app: ops-monitoring
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
  - kind: User
    name: senior-operator
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: cluster-log-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/cluster-log-reader-binding.yaml

# 第 3 步：创建运维管理 ClusterRole（更多权限）
cat > /tmp/cluster-ops-admin.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-ops-admin
  labels:
    app: ops-management
rules:
  # 管理所有命名空间的应用
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 管理 Pod
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "create"]

  # 管理服务和网络
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # 查看集群信息
  - apiGroups: [""]
    resources: ["nodes", "namespaces"]
    verbs: ["get", "list", "watch"]

  # 查看存储信息
  - apiGroups: [""]
    resources: ["persistentvolumes", "storageclasses"]
    verbs: ["get", "list", "watch"]
EOF

kubectl apply -f /tmp/cluster-ops-admin.yaml

# 第 4 步：创建对应的 ClusterRoleBinding
cat > /tmp/cluster-ops-admin-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-ops-admin-binding
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-ops-admin
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/cluster-ops-admin-binding.yaml

# 第 5 步：验证跨命名空间权限
echo "=== 验证跨命名空间权限 ==="

# 创建测试资源
kubectl create namespace ns-a
kubectl create namespace ns-b
kubectl create deployment app-a -n ns-a --image=nginx:alpine
kubectl create deployment app-b -n ns-b --image=nginx:alpine

# 检查跨命名空间日志查看
kubectl auth can-i get pods/log -n ns-a --as=senior-operator
# yes
kubectl auth can-i get pods/log -n ns-b --as=senior-operator
# yes

# 检查跨命名空间部署查看
kubectl auth can-i list deployments -n ns-a --as=senior-operator
# yes
kubectl auth can-i list deployments -n ns-b --as=senior-operator
# yes

# 检查集群级信息
kubectl auth can-i list nodes --as=senior-operator
# yes
kubectl auth can-i list namespaces --as=senior-operator
# yes

# 检查删除权限（log-reader 没有删除权限）
kubectl auth can-i delete pods -n ns-a --as=senior-operator
# no

# 使用 ops-admin 的用户验证
kubectl auth can-i delete pods -n ns-a --as=ops-team 2>/dev/null || echo "需要用 --as= 模拟用户"
kubectl auth can-i delete deployments -n ns-a --as=ops-team 2>/dev/null || true

# 第 6 步：将 ClusterRole 绑定到多个命名空间（替代方案）
# 如果不想用 ClusterRoleBinding，可以在每个命名空间用 RoleBinding 引用 ClusterRole

# 在 ns-a 中绑定
cat > /tmp/ns-a-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-log-reader-ns-a
  namespace: ns-a
subjects:
  - kind: Group
    name: ns-a-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-log-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/ns-a-binding.yaml

# 在 ns-b 中绑定
cat > /tmp/ns-b-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-log-reader-ns-b
  namespace: ns-b
subjects:
  - kind: Group
    name: ns-b-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-log-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f /tmp/ns-b-binding.yaml

# 第 7 步：查看所有 RBAC 资源
echo "=== RBAC 资源总览 ==="

echo "--- ClusterRole ---"
kubectl get clusterrole | grep -E "(log|ops)"

echo "--- ClusterRoleBinding ---"
kubectl get clusterrolebinding | grep -E "(log|ops)"

echo "--- RoleBinding ---"
kubectl get rolebinding -A | grep -E "(log|ops)"

# 第 8 步：模拟 Group 权限验证
echo "=== Group 权限验证 ==="

# 创建测试用户的 kubeconfig（使用 --as= 模拟即可）
kubectl auth can-i list pods -n ns-a --as=system:group:ops-team
kubectl auth can-i list pods -n ns-a --as=system:group:ns-a-team
kubectl auth can-i list pods -n ns-b --as=system:group:ns-a-team
# 注意：ns-a-team 在 ns-b 没有权限

# 第 9 步：验证 ServiceAccount 权限
echo "=== ServiceAccount 权限验证 ==="

kubectl auth can-i list pods -n default \
  --as=system:serviceaccount:monitoring:monitoring-sa
kubectl auth can-i list nodes \
  --as=system:serviceaccount:monitoring:monitoring-sa

# 第 10 步：清理
kubectl delete clusterrole cluster-log-reader cluster-ops-admin
kubectl delete clusterrolebinding cluster-log-reader-binding cluster-ops-admin-binding
kubectl delete rolebinding -n ns-a cluster-log-reader-ns-a
kubectl delete rolebinding -n ns-b cluster-log-reader-ns-b
kubectl delete namespace ns-a ns-b
```

---

## 12.8 常见问题排查

### Q1：Pod 中 ServiceAccount 无法访问 API（403 Forbidden）

```bash
# 问题：Pod 内的应用调用 K8s API 返回 403

# 排查步骤：

# 1. 确认 Pod 使用的 ServiceAccount
kubectl get pod <pod-name> -n <namespace> -o yaml | grep serviceAccount

# 2. 确认 ServiceAccount 存在
kubectl get serviceaccount <sa-name> -n <namespace>

# 3. 确认 RoleBinding 存在
kubectl get rolebinding -n <namespace> | grep <sa-name>
kubectl describe rolebinding <binding-name> -n <namespace>

# 4. 确认 Role 有对应权限
kubectl get role <role-name> -n <namespace> -o yaml

# 5. 模拟 SA 验证权限
kubectl auth can-i <verb> <resource> -n <namespace> \
  --as=system:serviceaccount:<namespace>:<sa-name>

# 6. 查看 SA 完整权限
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name>

# 7. 检查是否是跨命名空间访问
# Role 只在当前命名空间生效，跨命名空间需要 ClusterRole
```

### Q2：RoleBinding 创建后仍然无权限

```bash
# 问题：已经创建了 Role 和 RoleBinding，但用户仍然无法操作

# 常见原因：

# 1. RoleBinding 绑定的主体不匹配
# 检查 subjects 中的 name 是否与实际身份一致
kubectl get rolebinding <name> -o yaml

# 2. RoleBinding 绑定的 Role 不存在或权限不足
kubectl get role <name> -o yaml
# 检查 roleRef.name 和 roleRef.apiGroup

# 3. 使用了 ClusterRole 但只绑定了 RoleBinding
# ClusterRole 需要 ClusterRoleBinding 才能跨命名空间
# 如果只在特定命名空间使用，RoleBinding 引用 ClusterRole 也可以

# 4. 资源名称错误
# 使用复数形式：pods（不是 pod），deployments（不是 deployment）

# 5. API 组错误
# 核心资源用 ""，apps 组用 "apps"，batch 组用 "batch"

# 快速验证方法：
kubectl auth can-i <verb> <resource> \
  --as=<user-or-group> -n <namespace> \
  -v=6  # 显示更详细的调试信息
```

### Q3：ClusterRole 和 Role 权限冲突

```bash
# 问题：同一主体同时绑定了 ClusterRole 和 Role

# 权限是累加的，主体的有效权限 = ClusterRole 权限 + Role 权限
# ClusterRole 绑定全局生效，Role 仅在特定命名空间生效

# 查看主体的完整权限
kubectl auth can-i --list --as=<user>

# 建议：
# 1. 避免重复授权
# 2. 优先使用 Role（命名空间级），减少 ClusterRole 使用
# 3. 定期审查权限配置
```

### Q4：如何查看当前用户的所有权限

```bash
# 查看当前上下文用户的权限
kubectl auth can-i --list

# 查看指定用户的权限
kubectl auth can-i --list --as=<username>

# 查看指定 ServiceAccount 的权限
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name>

# 查看指定命名空间的权限
kubectl auth can-i --list --as=<user> -n <namespace>

# 注意：
# --list 输出的是权限的简化形式
# 实际权限可能更复杂（通配符等）
# 使用 -v=6 查看详细的权限检查过程
```

### Q5：如何临时禁用某个用户的权限

```bash
# 方法 1：删除 RoleBinding / ClusterRoleBinding
kubectl delete rolebinding <name> -n <namespace>
kubectl delete clusterrolebinding <name>

# 方法 2：修改 Role 权限（移除关键 verbs）
kubectl patch role <name> -p '{"rules":null}'
# 或直接编辑
kubectl edit role <name>

# 方法 3：创建拒绝规则（需要 admission webhook）
# Kubernetes RBAC 本身不支持 Deny 规则
# 可以使用 OPA/Gatekeeper 等策略引擎实现
```

### Q6：ServiceAccount Token 过期了怎么办

```bash
# Kubernetes 1.21+ 默认使用 Bound Service Account Token
# Token 有过期时间（默认 1 小时），会自动刷新

# 检查 Token 过期时间
kubectl exec -it <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d. -f2 | base64 -d 2>/dev/null | \
  python3 -m json.tool 2>/dev/null || \
  echo "Token payload"

# 注意：如果 Pod 中应用直接使用 Token 访问 API
# 应该使用 client-go 等官方库，它会自动处理 Token 刷新
# 避免将 Token 硬编码到应用配置中
```

### Q7：如何查看 Pod 中 ServiceAccount 的 Token

```bash
# 查看 Pod 挂载的 Token
kubectl exec -it <pod-name> -n <namespace> -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token

# 解码 Token（获取 Payload 部分）
TOKEN=$(kubectl exec -it <pod> -n <namespace> -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/token)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null

# 输出包含：
# - sub: ServiceAccount 身份
# - iss: 签发者
# - exp: 过期时间
# - namespace: 所在命名空间
```

---

## 12.9 章节小结

### 核心概念总结

1. **RBAC 四大资源**：Role / ClusterRole（定义权限）+ RoleBinding / ClusterRoleBinding（绑定主体）
2. **两种作用域**：Namespace 级（Role + RoleBinding）和 Cluster 级（ClusterRole + ClusterRoleBinding）
3. **权限三要素**：apiGroups（API 组）、resources（资源）、verbs（操作动词）
4. **ServiceAccount**：Pod 在集群中的身份，通过 Token 访问 API
5. **最小权限原则**：授予完成任务所需的最少权限

### RBAC 决策流程图

```
需要控制权限？
│
├── 集群级资源（Node、Namespace、PV）？
│   └── 是 → ClusterRole + ClusterRoleBinding
│
├── 需要跨命名空间？
│   └── 是 → ClusterRole + ClusterRoleBinding
│
├── Pod 需要访问 API？
│   └── 是 → ServiceAccount + RoleBinding
│
└── 单命名空间权限？
    └── 是 → Role + RoleBinding
```

### 实战命令速查

```bash
# 创建 RBAC
kubectl create role <name> --verb=<verb> --resource=<resource> -n <ns>
kubectl create clusterrole <name> --verb=<verb> --resource=<resource>
kubectl create rolebinding <name> --role=<role> --user=<user> -n <ns>
kubectl create clusterrolebinding <name> --clusterrole=<role> --user=<user>

# 查看 RBAC
kubectl get role -n <ns>
kubectl get rolebinding -n <ns>
kubectl get clusterrole
kubectl get clusterrolebinding
kubectl describe role <name> -n <ns>
kubectl describe clusterrolebinding <name>

# 验证权限
kubectl auth can-i <verb> <resource> --as=<user> -n <ns>
kubectl auth can-i --list --as=<user>
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>

# ServiceAccount
kubectl get serviceaccount -n <ns>
kubectl create serviceaccount <name> -n <ns>
kubectl describe serviceaccount <name> -n <ns>

# 删除 RBAC
kubectl delete role <name> -n <ns>
kubectl delete rolebinding <name> -n <ns>
kubectl delete clusterrole <name>
kubectl delete clusterrolebinding <name>
```

### 最佳实践

- **最小权限**：遵循最小权限原则，只授予必要的权限
- **命名空间隔离**：优先使用 Role + RoleBinding，减少 ClusterRole 使用
- **合理分组**：使用 Group 而非单个 User 管理权限，便于维护
- **ServiceAccount 安全**：为 Pod 使用专用 ServiceAccount，不使用 default
- **定期审计**：定期审查 RBAC 配置，移除不必要的权限
- **避免通配符**：生产环境避免使用 `verbs: ["*"]` 和 `resources: ["*"]`
- **Token 安全**：不要将 ServiceAccount Token 暴露到集群外部
- **变更记录**：记录 RBAC 变更原因，便于追溯
- **使用 OPA**：复杂权限场景可使用 OPA/Gatekeeper 等策略引擎