# 第4章 Deployment 与应用部署

## 学习目标

- 理解 Deployment 的核心功能：声明式更新、滚动升级、版本回滚、扩缩容
- 掌握 Deployment 与 ReplicaSet、Pod 的层级关系
- 能够独立编写 Deployment YAML 配置，理解每个字段的含义
- 熟练配置滚动更新策略（RollingUpdate / Recreate），理解 maxUnavailable 和 maxSurge 的含义
- 掌握版本回滚操作：`kubectl rollout history`、`kubectl rollout undo`
- 熟练进行手动扩缩容操作，了解自动扩缩容基础
- 理解 Pod 模板变更如何触发滚动更新
- 通过 5 个实战练习，熟练掌握 Deployment 的部署、更新、回滚和扩缩容操作
- 实现蓝绿部署和金丝雀发布的实战演练

---

## 4.1 Deployment 核心功能概述

### 4.1.1 什么是 Deployment

Deployment 是 Kubernetes 中用于管理**无状态应用**的控制器，它提供了声明式更新、滚动升级、版本回滚和扩缩容等核心能力。

### 4.1.2 Deployment 的核心功能

| 功能 | 说明 | 关键命令/参数 |
|------|------|--------------|
| **声明式更新** | 只需描述期望状态，Deployment 自动完成操作 | `kubectl apply -f` |
| **滚动升级** | 逐步替换旧版本 Pod，实现零停机发布 | `strategy: RollingUpdate` |
| **版本回滚** | 快速恢复到历史版本 | `kubectl rollout undo` |
| **扩缩容** | 动态调整副本数量 | `kubectl scale` |
| **副本管理** | 确保始终维持指定数量的 Pod 副本 | `replicas` |
| **历史版本** | 保留历史版本信息，方便回滚 | `revisionHistoryLimit` |

### 4.1.3 Deployment 的典型应用场景

- **Web 服务部署**：Nginx、Apache 等无状态服务
- **API 服务部署**：RESTful API、微服务
- **批量处理服务**：需要水平扩展的无状态应用
- **灰度发布**：通过调整副本比例实现金丝雀发布

---

## 4.2 Deployment 与 ReplicaSet 的关系

### 4.2.1 层级关系

```
┌─────────────────────────────────────────────────────────────┐
│                      Deployment                             │
│  (管理 ReplicaSet，维护期望状态)                              │
│                                                              │
│  期望副本数: 3                                                │
│  滚动策略: RollingUpdate                                     │
│  Pod 模板: { image: nginx:1.25 }                             │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  ReplicaSet (rs-v1)                  │    │
│  │  (由 Deployment 创建和管理)                           │    │
│  │  匹配标签: app=nginx, version=v1                     │    │
│  │  副本数: 3                                           │    │
│  │                                                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│  │  │ Pod-1    │  │ Pod-2    │  │ Pod-3    │           │    │
│  │  │ v1       │  │ v1       │  │ v1       │           │    │
│  │  └──────────┘  └──────────┘  └──────────┘           │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  版本 v2 发布时：                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  ReplicaSet (rs-v2)    │             │    │
│  │  (由 Deployment 创建，逐步替换 rs-v1)    │            │    │
│  │  匹配标签: app=nginx, version=v2                     │    │
│  │  副本数: 3 (逐步增加)                                │    │
│  │                                                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │    │
│  │  │ Pod-4    │  │ Pod-5    │  │ Pod-6    │           │    │
│  │  │ v2       │  │ v2       │  │ v2       │           │    │
│  │  └──────────┘  └──────────┘  └──────────┘           │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 4.2.2 三者职责

| 层级 | 职责 | 创建者 |
|------|------|--------|
| **Deployment** | 声明式管理副本数和 Pod 模板，控制滚动更新和回滚 | 用户 |
| **ReplicaSet** | 确保指定数量的 Pod 副本持续运行 | Deployment Controller |
| **Pod** | 运行应用容器的最小单元 | ReplicaSet Controller |

### 4.2.3 为什么需要 ReplicaSet

- **解耦设计**：ReplicaSet 专注于维持副本数，Deployment 专注于更新策略
- **版本管理**：每次 Pod 模板变更，Deployment 创建新的 ReplicaSet
- **平滑过渡**：旧 ReplicaSet 逐步缩减，新 ReplicaSet 逐步增加
- **历史追溯**：每个 ReplicaSet 代表一个版本，方便回滚

### 4.2.4 ReplicaSet 自动清理

- Deployment 会保留最近的 `revisionHistoryLimit`（默认 10）个 ReplicaSet
- 超出限制的旧 ReplicaSet 会被自动清理
- 标记为当前版本的 ReplicaSet 不会被清理

---

## 4.3 Deployment YAML 结构详解

### 4.3.1 完整 YAML 配置示例

```yaml
apiVersion: apps/v1              # API 版本
kind: Deployment                 # 资源类型
metadata:
  name: nginx-deployment         # Deployment 名称
  namespace: default             # 命名空间
  labels:                        # 标签
    app: nginx
    tier: frontend
  annotations:                   # 注解
    description: "Nginx Web 服务部署"

spec:
  # 副本数量
  replicas: 3                    # Pod 副本数量

  # 标签选择器（必填，不可修改）
  selector:
    matchLabels:
      app: nginx                 # 必须与 template.metadata.labels 匹配
      tier: frontend

  # 历史版本保留数量
  revisionHistoryLimit: 10       # 保留最近 10 个历史版本

  # 更新策略
  strategy:
    type: RollingUpdate          # 更新类型：RollingUpdate / Recreate
    rollingUpdate:
      maxUnavailable: 25%        # 最大不可用副本数（绝对值或百分比）
      maxSurge: 25%              # 最大额外副本数（绝对值或百分比）

  # 暂停更新
  paused: false                  # 是否暂停部署（暂停时变更不会触发更新）

  # 进度超时（Kubernetes 1.19+）
  progressDeadlineSeconds: 600   # 滚动更新超时时间（秒），超时回滚

  # Pod 模板
  template:
    metadata:
      labels:                    # Pod 标签（必须与 selector 匹配）
        app: nginx
        tier: frontend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9113"

    spec:
      # 调度配置
      nodeSelector: {}
      affinity: {}
      tolerations: []
      priorityClassName: ""

      # 容器定义
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          imagePullPolicy: IfNotPresent

          # 端口声明
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

          # 环境变量
          env:
            - name: NGINX_HOST
              value: "example.com"
            - name: NGINX_PORT
              value: "80"

          # 资源限制
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

          # 健康检查
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3

          # 卷挂载
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
              readOnly: true
            - name: nginx-logs
              mountPath: /var/log/nginx

          # 安全上下文
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true

      # 卷定义
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
        - name: nginx-logs
          emptyDir: {}

      # Pod 重启策略
      restartPolicy: Always

      # 服务账户
      serviceAccountName: default

      # DNS 配置
      dnsPolicy: ClusterFirst

  # 最小就绪时间（Kubernetes 1.25+）
  minReadySeconds: 10            # Pod 就绪后需要等待的最小时间
```

### 4.3.2 YAML 字段速查表

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `spec.replicas` | integer | 1 | Pod 副本数量 |
| `spec.selector.matchLabels` | map | - | 标签选择器，不可修改 |
| `spec.strategy.type` | string | RollingUpdate | 更新类型 |
| `spec.strategy.rollingUpdate.maxUnavailable` | string | 25% | 最大不可用副本 |
| `spec.strategy.rollingUpdate.maxSurge` | string | 25% | 最大额外副本 |
| `spec.revisionHistoryLimit` | integer | 10 | 历史版本保留数 |
| `spec.paused` | boolean | false | 是否暂停部署 |
| `spec.progressDeadlineSeconds` | integer | 600 | 滚动更新超时 |
| `spec.minReadySeconds` | integer | 0 | Pod 最小就绪时间 |
| `spec.template` | object | - | Pod 模板 |

---

## 4.4 滚动更新策略详解

### 4.4.1 RollingUpdate 策略

RollingUpdate 是默认的更新策略，逐步替换旧版本 Pod。

```
滚动更新过程示例（replicas=3, maxUnavailable=1, maxSurge=1）：

初始状态：
  RS-v1: [Pod-1][Pod-2][Pod-3]  (3 个副本)

步骤 1: 创建 RS-v2，先启动 1 个新 Pod
  RS-v1: [Pod-1][Pod-2][Pod-3]  (3 个副本)
  RS-v2: [Pod-4]                 (1 个副本)
  总计: 4 (maxSurge=1, 正常)

步骤 2: 缩减旧 RS，增加新 RS
  RS-v1: [Pod-1][Pod-2]          (2 个副本)
  RS-v2: [Pod-4][Pod-5]          (2 个副本)
  总计: 4

步骤 3: 继续缩减和增加
  RS-v1: [Pod-1]                 (1 个副本)
  RS-v2: [Pod-4][Pod-5][Pod-6]   (3 个副本)
  总计: 4

步骤 4: 最终状态
  RS-v1: []                      (0 个副本)
  RS-v2: [Pod-4][Pod-5][Pod-6]   (3 个副本)
  总计: 3
```

#### maxUnavailable 参数

```
maxUnavailable: 25%
├── 计算方式：replicas × 25% = 3 × 0.25 = 0.75 → 向上取整为 1
├── 含义：更新过程中，最多允许 1 个 Pod 不可用
├── 百分比：相对于 replicas 的比例
└── 绝对值：直接指定具体数量
```

#### maxSurge 参数

```
maxSurge: 25%
├── 计算方式：replicas × 25% = 3 × 0.25 = 0.75 → 向上取整为 1
├── 含义：更新过程中，最多允许比期望副本数多出 1 个 Pod
├── 百分比：相对于 replicas 的比例
└── 绝对值：直接指定具体数量
```

#### 滚动更新数学关系

```
设 replicas = N, maxUnavailable = U, maxSurge = S

更新过程中满足：
  不可用 Pod 数 ≤ U
  总 Pod 数 ≤ N + S

关键约束：
  U + S ≥ 1 （否则无法完成滚动更新）

示例（N=3, U=25%, S=25%）：
  U = ceil(3 × 0.25) = 1
  S = ceil(3 × 0.25) = 1
  最小可用副本数 = N - U = 3 - 1 = 2
  最大总副本数 = N + S = 3 + 1 = 4
```

### 4.4.2 Recreate 策略

Recreate 策略会先销毁所有旧 Pod，再创建新 Pod。

```
Recreate 更新过程：

初始状态：
  RS-v1: [Pod-1][Pod-2][Pod-3]

步骤 1: 销毁所有旧 Pod
  RS-v1: []  (Pod 全部终止)

步骤 2: 创建新 Pod
  RS-v2: [Pod-4][Pod-5][Pod-6]

特点：
  - 优点：简单，不会同时运行两个版本
  - 缺点：更新期间服务完全中断
  - 适用：有状态应用、需要避免并发版本的场景
```

### 4.4.3 策略配置示例

```yaml
# 示例一：默认 RollingUpdate 配置
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%

# 示例二：高可用配置（零停机）
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0           # 不允许不可用
      maxSurge: 100%              # 允许临时翻倍

# 示例三：快速滚动配置
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1           # 最多 1 个不可用
      maxSurge: 1                 # 最多 1 个额外副本

# 示例四：Recreate 配置
spec:
  strategy:
    type: Recreate

# 示例五：自定义滚动策略
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 33%
      maxSurge: 33%
  minReadySeconds: 30             # Pod 就绪后 30 秒才视为可用
  progressDeadlineSeconds: 300   # 5 分钟内必须完成滚动
```

### 4.4.4 滚动更新状态

```bash
# 查看滚动更新状态
kubectl rollout status deployment/nginx-deployment

# 输出示例：
# deployment "nginx-deployment" successfully rolled out
# deployment "nginx-deployment" has been progressing: 3 out of 3 new pods have been updated

# 查看部署状态
kubectl get deployment nginx-deployment

# 输出列：
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           5m
#
# READY: 就绪的 Pod 数 / 期望副本数
# UP-TO-DATE: 最新模板的 Pod 数
# AVAILABLE: 可用的 Pod 数
```

---

## 4.5 回滚操作

### 4.5.1 查看历史版本

```bash
# 查看 Deployment 历史版本
kubectl rollout history deployment/nginx-deployment

# 输出示例：
# deployment.apps/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         kubectl create deployment nginx-deployment --image=nginx:1.25
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.26
# 3         kubectl set image deployment/nginx-deployment nginx=nginx:1.27

# 查看指定版本的详情
kubectl rollout history deployment/nginx-deployment --revision=2

# 输出示例：
# deployment.apps/nginx-deployment with revision #2
# Pod Template:
#   Labels:	app=nginx
#           pod-template-hash=5d4b6c7f8
#   Containers:
#    nginx:
#     Image:	nginx:1.26
#     Port:	80/TCP
#   ...
```

### 4.5.2 回滚到指定版本

```bash
# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# 回滚后的历史版本示例：
# REVISION  CHANGE-CAUSE
# 4         kubectl set image deployment/nginx-deployment nginx=nginx:1.27
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.26  (当前版本)
# 3         <none>  (被跳过的版本)
# 1         kubectl create deployment nginx-deployment --image=nginx:1.25

# 注意：回滚后版本号会增加，但 CHANGE-CAUSE 对应原来的版本
```

### 4.5.3 暂停与恢复部署

```bash
# 暂停部署（Pod 模板变更不会触发更新）
kubectl rollout pause deployment/nginx-deployment

# 修改配置（不会立即生效）
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
kubectl scale deployment nginx-deployment --replicas=5

# 恢复部署（合并所有变更后统一执行）
kubectl rollout resume deployment/nginx-deployment
```

### 4.5.4 取消进行中的滚动更新

```bash
# 取消正在进行的滚动更新
kubectl rollout undo deployment/nginx-deployment

# 注意：如果新 Pod 已经创建，它们不会被立即销毁
# 需要等待回滚操作完成
```

---

## 4.6 扩缩容操作

### 4.6.1 手动扩缩容

```bash
# 方式一：使用 kubectl scale
kubectl scale deployment nginx-deployment --replicas=5

# 查看扩缩容状态
kubectl scale deployment nginx-deployment --replicas=5 --current-replicas=3
# deployment.apps/nginx-deployment scaled

# 方式二：使用 kubectl edit
kubectl edit deployment nginx-deployment
# 修改 replicas 字段

# 方式三：使用 kubectl patch
kubectl patch deployment nginx-deployment -p '{"spec":{"replicas":5}}'

# 验证扩缩容结果
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx
```

### 4.6.2 自动扩缩容基础（HPA）

Horizontal Pod Autoscaler（HPA）根据 CPU/内存利用率自动调整副本数。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

```bash
# 创建 HPA
kubectl apply -f nginx-hpa.yaml

# 查看 HPA 状态
kubectl get hpa
kubectl describe hpa nginx-hpa

# 查看 HPA 历史
kubectl get hpa nginx-hpa -o yaml
```

### 4.6.3 扩缩容策略对比

| 方式 | 触发条件 | 适用场景 |
|------|----------|----------|
| **手动扩缩容** | 人工操作 | 流量预测明确、临时调整 |
| **HPA（CPU/内存）** | CPU/内存使用率 | 通用场景、基于资源指标 |
| **HPA（自定义指标）** | 自定义指标 | 基于 QPS、队列深度等业务指标 |
| **VPA（垂直扩缩容）** | Pod 资源建议 | 需要调整单个 Pod 的资源 |

---

## 4.7 Pod 模板变更如何触发滚动更新

### 4.7.1 触发更新的变更

| 变更类型 | 是否触发滚动更新 |
|----------|-----------------|
| 修改 `spec.template.spec.containers[].image` | 是 |
| 修改 `spec.template.spec.containers[].env` | 是 |
| 修改 `spec.template.spec.containers[].resources` | 是 |
| 修改 `spec.template.spec.containers[].args` | 是 |
| 修改 `spec.template.spec.volumes` | 是 |
| 修改 `spec.template.metadata.labels` | 是 |
| 修改 `spec.replicas` | 否（仅扩缩容） |
| 修改 `spec.strategy` | 否（仅更新策略） |
| 修改 `spec.selector` | 否（不允许修改） |

### 4.7.2 镜像更新触发滚动

```bash
# 更新镜像（触发滚动更新）
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# 输出示例：
# deployment.apps/nginx-deployment image updated

# 查看更新状态
kubectl rollout status deployment/nginx-deployment

# 查看新旧 ReplicaSet
kubectl get rs -l app=nginx
```

### 4.7.3 配置变更触发滚动

```bash
# 修改环境变量
kubectl set env deployment/nginx-deployment NGINX_PORT=8080

# 修改资源限制
kubectl set resources deployment/nginx-deployment \
  containers=nginx \
  limits=cpu:1,memory:512Mi \
  requests=cpu:200m,memory:256Mi

# 修改注解（添加变更追踪信息）
kubectl annotate deployment nginx-deployment \
  kubernetes.io/change-cause="升级到 nginx 1.26，修复安全漏洞"

# 验证变更
kubectl describe deployment nginx-deployment
# 查看 CHANGE-CAUSE 字段
```

### 4.7.4 使用 kubectl apply 触发更新

```bash
# 直接 apply 新的 YAML 文件
cat > /tmp/nginx-v2.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.26-alpine  # 变更镜像版本
          ports:
            - containerPort: 80
EOF

# 应用变更
kubectl apply -f /tmp/nginx-v2.yaml
kubectl rollout status deployment/nginx-deployment
```

### 4.7.5 变更追踪

```bash
# 查看每个版本的变更原因
kubectl rollout history deployment/nginx-deployment

# 设置变更原因
kubectl set image deployment/nginx-deployment nginx=nginx:1.27 \
  --record=true

# 或者使用 annotate 设置 CHANGE-CAUSE
kubectl annotate deployment nginx-deployment \
  deployment.kubernetes.io/revision-history="v1.27: 性能优化"
```

---

## 4.8 实战练习

### 实战 1：部署 Nginx Deployment（3 副本）

**目标：** 创建一个标准的 Nginx Deployment，包含 3 个副本。

```bash
# 第 1 步：创建 Deployment YAML
cat > /tmp/nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx
    tier: frontend
  annotations:
    description: "生产环境 Nginx 部署"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  template:
    metadata:
      labels:
        app: nginx
        tier: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
          volumeMounts:
            - name: nginx-tmp
              mountPath: /tmp
            - name: nginx-cache
              mountPath: /var/cache/nginx
      volumes:
        - name: nginx-tmp
          emptyDir: {}
        - name: nginx-cache
          emptyDir: {}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
EOF

# 第 2 步：创建 Deployment
kubectl apply -f /tmp/nginx-deployment.yaml

# 第 3 步：查看 Deployment 状态
kubectl get deployment nginx-deployment

# 第 4 步：查看创建的 Pod
kubectl get pods -l app=nginx -o wide

# 第 5 步：查看 ReplicaSet
kubectl get rs -l app=nginx

# 第 6 步：等待所有 Pod 就绪
kubectl wait --for=condition=Available deployment/nginx-deployment --timeout=120s

# 第 7 步：查看 Deployment 详情
kubectl describe deployment nginx-deployment

# 第 8 步：验证 Pod 分布（观察调度器行为）
kubectl get pods -l app=nginx -o wide
# 注意观察 Pod 是否分布在不同节点上

# 第 9 步：创建 Service 暴露 Deployment
cat > /tmp/nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
EOF

kubectl apply -f /tmp/nginx-service.yaml

# 第 10 步：验证 Service 负载均衡
kubectl get endpoints nginx-service
kubectl run curl-test --image=curlimages/curl --rm -it -- sh -c \
  'for i in $(seq 1 10); do curl -s http://nginx-service/ | head -1; done'

# 第 11 步：查看 Deployment 的 YAML
kubectl get deployment nginx-deployment -o yaml | head -50

# 第 12 步：清理
kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
```

### 实战 2：手动扩缩容副本数

**目标：** 手动调整 Deployment 的副本数，观察 Pod 变化。

```bash
# 第 1 步：重新创建 Deployment（如果已删除）
kubectl apply -f /tmp/nginx-deployment.yaml
kubectl wait --for=condition=Available deployment/nginx-deployment --timeout=120s

# 第 2 步：查看当前副本数
kubectl get deployment nginx-deployment
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           2m

# 第 3 步：扩容到 5 副本
kubectl scale deployment nginx-deployment --replicas=5

# 第 4 步：观察 Pod 增加过程
echo "等待扩容完成..."
for i in $(seq 1 10); do
  READY=$(kubectl get deployment nginx-deployment -o jsonpath='{.status.readyReplicas}')
  DESIRED=$(kubectl get deployment nginx-deployment -o jsonpath='{.status.replicas}')
  echo "Time ${i}: Ready=${READY}, Desired=${DESIRED}"
  if [ "$READY" = "$DESIRED" ]; then
    echo "扩容完成！"
    break
  fi
  sleep 3
done

# 第 5 步：查看扩容后的 Pod
kubectl get pods -l app=nginx

# 第 6 步：缩容到 2 副本
kubectl scale deployment nginx-deployment --replicas=2

# 第 7 步：观察 Pod 缩减过程
echo "等待缩容完成..."
for i in $(seq 1 10); do
  READY=$(kubectl get deployment nginx-deployment -o jsonpath='{.status.readyReplicas}')
  DESIRED=$(kubectl get deployment nginx-deployment -o jsonpath='{.status.replicas}')
  echo "Time ${i}: Ready=${READY}, Desired=${DESIRED}"
  if [ "$READY" = "$DESIRED" ]; then
    echo "缩容完成！"
    break
  fi
  sleep 3
done

# 第 8 步：查看缩容后的 Pod
kubectl get pods -l app=nginx

# 第 9 步：使用 patch 方式扩缩容
kubectl patch deployment nginx-deployment -p '{"spec":{"replicas":4}}'

# 第 10 步：使用 edit 方式扩缩容
# kubectl edit deployment nginx-deployment
# 修改 replicas: 4 → 6

# 第 11 步：验证扩缩容结果
kubectl get deployment nginx-deployment
kubectl get pods -l app=nginx --no-headers | wc -l

# 第 12 步：恢复到 3 副本
kubectl scale deployment nginx-deployment --replicas=3

# 第 13 步：清理
kubectl delete deployment nginx-deployment
```

### 实战 3：配置滚动更新策略并触发版本更新

**目标：** 配置自定义滚动更新策略，触发版本更新并观察滚动过程。

```bash
# 第 1 步：创建带自定义滚动策略的 Deployment
cat > /tmp/rolling-update-demo.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-demo
  labels:
    app: rolling-demo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: rolling-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  minReadySeconds: 5
  progressDeadlineSeconds: 120
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
            failureThreshold: 2
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
EOF

# 第 2 步：创建 Deployment
kubectl apply -f /tmp/rolling-update-demo.yaml

# 第 3 步：等待部署完成
kubectl wait --for=condition=Available deployment/rolling-update-demo --timeout=120s

# 第 4 步：查看初始状态
echo "=== 初始状态 ==="
kubectl get deployment rolling-update-demo
kubectl get rs -l app=rolling-demo
kubectl get pods -l app=rolling-demo

# 第 5 步：触发滚动更新（更新镜像版本）
kubectl set image deployment/rolling-update-demo nginx=nginx:1.26-alpine

# 第 6 步：实时观察滚动过程
echo "=== 观察滚动更新过程 ==="
for i in $(seq 1 30); do
  DEPLOY=$(kubectl get deployment rolling-update-demo -o jsonpath='{.status.readyReplicas}')
  RS_NEW=$(kubectl get rs -l app=rolling-demo -o jsonpath='{.items[1].status.replicas}' 2>/dev/null || echo "0")
  RS_OLD=$(kubectl get rs -l app=rolling-demo -o jsonpath='{.items[0].status.replicas}' 2>/dev/null || echo "0")
  echo "Time ${i}: Ready=${DEPLOY}, OldRS=${RS_OLD}, NewRS=${RS_NEW}"

  # 检查是否有 Pod 在更新
  PODS=$(kubectl get pods -l app=rolling-demo --no-headers | wc -l)
  echo "  Total Pods: ${PODS}, maxUnavailable=1, maxSurge=1"

  if [ "$RS_NEW" = "5" ]; then
    echo "滚动更新完成！"
    break
  fi
  sleep 2
done

# 第 7 步：查看更新后的状态
echo "=== 更新完成状态 ==="
kubectl get deployment rolling-update-demo
kubectl get rs -l app=rolling-demo

# 第 8 步：查看更新历史
kubectl rollout history deployment/rolling-update-demo

# 第 9 步：验证新版本镜像
kubectl get pods -l app=rolling-demo -o jsonpath='{.items[*].spec.containers[0].image}'
echo ""

# 第 10 步：触发多次更新
# 更新 1
kubectl set image deployment/rolling-update-demo nginx=nginx:1.26-alpine \
  --record
# 更新 2
kubectl set image deployment/rolling-update-demo nginx=nginx:1.27-alpine \
  --record
# 更新 3
kubectl set image deployment/rolling-update-demo nginx=nginx:1.28-alpine \
  --record

# 第 11 步：查看所有历史版本
kubectl rollout history deployment/rolling-update-demo

# 第 12 步：配置零停机滚动策略
kubectl patch deployment rolling-update-demo -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":"100%"}}}}'

# 第 13 步：触发零停机更新
kubectl set image deployment/rolling-update-demo nginx=nginx:1.29-alpine

# 第 14 步：验证零停机
# 检查服务是否在更新期间保持可用
kubectl run load-test --image=curlimages/curl --rm -it -- sh -c \
  'for i in $(seq 1 20); do 
     STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://rolling-update-demo);
     echo "Request ${i}: HTTP ${STATUS}";
     sleep 1;
   done'

# 第 15 步：清理
kubectl delete deployment rolling-update-demo
```

### 实战 4：执行回滚操作恢复到上一个版本

**目标：** 触发版本更新后执行回滚操作，恢复到历史版本。

```bash
# 第 1 步：创建 Deployment
cat > /tmp/rollback-demo.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollback-demo
  labels:
    app: rollback-demo
  annotations:
    description: "回滚演示"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rollback-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 5
  template:
    metadata:
      labels:
        app: rollback-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
EOF

# 创建并等待就绪
kubectl apply -f /tmp/rollback-demo.yaml
kubectl wait --for=condition=Available deployment/rollback-demo --timeout=120s

# 第 2 步：记录初始版本
echo "=== 初始版本 (Revision 1) ==="
kubectl rollout history deployment/rollback-demo
kubectl get pods -l app=rollback-demo -o jsonpath='{.items[*].spec.containers[0].image}'
echo ""

# 第 3 步：执行第一次更新
kubectl set image deployment/rollback-demo nginx=nginx:1.26-alpine --record
kubectl wait --for=condition=Available deployment/rollback-demo --timeout=120s

echo "=== 第一次更新后 (Revision 2) ==="
kubectl rollout history deployment/rollback-demo

# 第 4 步：执行第二次更新
kubectl set image deployment/rollback-demo nginx=nginx:1.27-alpine --record
kubectl wait --for=condition=Available deployment/rollback-demo --timeout=120s

echo "=== 第二次更新后 (Revision 3) ==="
kubectl rollout history deployment/rollback-demo

# 第 5 步：执行第三次更新
kubectl set image deployment/rollback-demo nginx=nginx:1.28-alpine --record
kubectl wait --for=condition=Available deployment/rollback-demo --timeout=120s

echo "=== 第三次更新后 (Revision 4) ==="
kubectl rollout history deployment/rollback-demo

# 第 6 步：回滚到上一个版本
kubectl rollout undo deployment/rollback-demo

echo "=== 回滚到 Revision 3 ==="
kubectl rollout history deployment/rollback-demo

# 第 7 步：回滚到指定版本
kubectl rollout undo deployment/rollback-demo --to-revision=2

echo "=== 回滚到 Revision 2 ==="
kubectl rollout history deployment/rollback-demo

# 第 8 步：验证回滚结果
kubectl get pods -l app=rollback-demo -o jsonpath='{.items[*].spec.containers[0].image}'
echo ""

# 第 9 步：查看指定版本详情
kubectl rollout history deployment/rollback-demo --revision=1
kubectl rollout history deployment/rollback-demo --revision=4

# 第 10 步：暂停更新、批量变更、统一执行
echo "=== 暂停部署 ==="
kubectl rollout pause deployment/rollback-demo

# 在暂停期间多次变更
kubectl set image deployment/rollback-demo nginx=nginx:1.29-alpine
kubectl scale deployment rollback-demo --replicas=5

# 变更尚未生效
kubectl get deployment rollback-demo

# 恢复部署，统一执行变更
kubectl rollout resume deployment/rollback-demo

# 等待生效
kubectl wait --for=condition=Available deployment/rollback-demo --timeout=120s

# 第 11 步：清理
kubectl delete deployment rollback-demo
```

### 实战 5：蓝绿部署与金丝雀发布

**目标：** 使用两个 Deployment + Service 切换实现蓝绿部署和金丝雀发布。

```bash
# ==================== 蓝绿部署 ====================

# 场景：准备新版本时，先部署到绿色环境，验证通过后再切换流量

# 第 1 步：创建蓝色环境（v1）
cat > /tmp/blue-green-blue.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
  labels:
    app: web
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: blue
  template:
    metadata:
      labels:
        app: web
        version: blue
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          env:
            - name: VERSION
              value: "blue-v1"
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web
    version: blue    # 初始指向蓝色
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF

kubectl apply -f /tmp/blue-green-blue.yaml
kubectl wait --for=condition=Available deployment/web-blue --timeout=120s

# 第 2 步：创建绿色环境（v2）
cat > /tmp/blue-green-green.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
  labels:
    app: web
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
        - name: nginx
          image: nginx:1.26-alpine  # 新版本
          ports:
            - containerPort: 80
          env:
            - name: VERSION
              value: "green-v2"
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
EOF

kubectl apply -f /tmp/blue-green-green.yaml
kubectl wait --for=condition=Available deployment/web-green --timeout=120s

# 第 3 步：验证绿色环境
kubectl get pods -l version=green
kubectl run test-green --image=curlimages/curl --rm -it -- sh -c \
  'curl -s http://web-green.default.svc.cluster.local/ && echo " - Green OK"'

# 第 4 步：切换流量到绿色
kubectl patch service web-service -p '{"spec":{"selector":{"app":"web","version":"green"}}}'

# 第 5 步：验证流量已切换
kubectl run test-service --image=curlimages/curl --rm -it -- sh -c \
  'for i in $(seq 1 5); do 
     RESP=$(curl -s http://web-service/)
     echo "Request ${i}: ${RESP}";
   done'

# 第 6 步：确认绿色稳定后，删除蓝色
# kubectl delete deployment web-blue

# ==================== 金丝雀发布 ====================

# 场景：新版本只让少量用户访问，观察无误后全量发布

# 第 7 步：创建主版本 Deployment（90% 流量）
cat > /tmp/canary-main.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-main
  labels:
    app: canary
    track: stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: canary
      track: stable
  template:
    metadata:
      labels:
        app: canary
        track: stable
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          env:
            - name: VERSION
              value: "v1-stable"
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
EOF

kubectl apply -f /tmp/canary-main.yaml
kubectl wait --for=condition=Available deployment/canary-main --timeout=120s

# 第 8 步：创建金丝雀版本 Deployment（10% 流量）
cat > /tmp/canary-canary.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-canary
  labels:
    app: canary
    track: canary
spec:
  replicas: 1          # 10% 流量（1 / (9+1) = 10%）
  selector:
    matchLabels:
      app: canary
      track: canary
  template:
    metadata:
      labels:
        app: canary
        track: canary
    spec:
      containers:
        - name: nginx
          image: nginx:1.26-alpine
          ports:
            - containerPort: 80
          env:
            - name: VERSION
              value: "v2-canary"
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 3
EOF

kubectl apply -f /tmp/canary-canary.yaml
kubectl wait --for=condition=Available deployment/canary-canary --timeout=120s

# 第 9 步：创建覆盖 Service
cat > /tmp/canary-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: canary-service
  labels:
    app: canary
spec:
  type: ClusterIP
  selector:
    app: canary          # 同时匹配 stable 和 canary
  ports:
    - name: http
      port: 80
      targetPort: 80
EOF

kubectl apply -f /tmp/canary-service.yaml

# 第 10 步：验证流量分布
echo "=== 金丝雀流量分布测试 ==="
kubectl run canary-test --image=curlimages/curl --rm -it -- sh -c \
  'stable=0; canary=0; 
   for i in $(seq 1 100); do
     VERSION=$(curl -s http://canary-service/ | grep -o "v[0-9]" || echo "unknown")
     if echo "$VERSION" | grep -q "v1"; then stable=$((stable+1)); 
     elif echo "$VERSION" | grep -q "v2"; then canary=$((canary+1)); 
     fi
   done
   echo "稳定版本请求: ${stable}%"
   echo "金丝雀版本请求: ${canary}%"'

# 第 11 步：逐步扩大金丝雀比例
# 调整副本数：50% canary
kubectl scale deployment canary-main --replicas=5
kubectl scale deployment canary-canary --replicas=5
kubectl wait --for=condition=Available deployment/canary-main --timeout=120s
kubectl wait --for=condition=Available deployment/canary-canary --timeout=120s

# 第 12 步：全量发布
kubectl scale deployment canary-main --replicas=0
kubectl scale deployment canary-canary --replicas=10
kubectl wait --for=condition=Available deployment/canary-canary --timeout=120s

# 或直接删除主版本
# kubectl delete deployment canary-main

# 第 13 步：如果金丝雀出现问题，快速回滚
# 立即将 canary 副本数减为 0
kubectl scale deployment canary-canary --replicas=0
kubectl scale deployment canary-main --replicas=10

# 第 14 步：验证回滚成功
kubectl run rollback-test --image=curlimages/curl --rm -it -- sh -c \
  'for i in $(seq 1 10); do 
     VERSION=$(curl -s http://canary-service/ | grep -o "v[0-9]" || echo "unknown")
     echo "Request ${i}: ${VERSION}"
   done'

# 第 15 步：清理所有资源
kubectl delete deployment web-blue web-green canary-main canary-canary
kubectl delete service web-service canary-service
```

---

## 4.9 常见问题排查

### Q1：Deployment 更新卡在进行中

```bash
# 查看 Deployment 状态
kubectl describe deployment <name>

# 查看滚动更新进度
kubectl rollout status deployment/<name>

# 常见原因：
# 1. 新 Pod 无法调度（资源不足、节点选择器不匹配）
# 2. 健康检查失败（readinessProbe 不通过）
# 3. 镜像拉取失败（镜像不存在或拉取权限不足）
# 4. 资源限制导致 OOMKilled

# 排查步骤
kubectl get rs -l app=<label>
kubectl get pods -l app=<label>
kubectl describe pod <new-pod>    # 查看新 Pod 事件
kubectl logs <new-pod>             # 查看新 Pod 日志

# 检查镜像拉取
kubectl describe pod <new-pod> | grep -A 5 "Events"
# 常见错误：ImagePullBackOff, ErrImagePull

# 检查调度
kubectl describe pod <new-pod> | grep "Events" -A 20
# 常见错误：Insufficient CPU, nodeSelector 不匹配

# 强制回滚
kubectl rollout undo deployment/<name>
```

### Q2：回滚后新版本丢失

```bash
# 查看所有历史版本
kubectl rollout history deployment/<name>

# revisionHistoryLimit 限制了保留的版本数
# 超过限制的旧版本会被删除

# 增加历史版本保留数
kubectl patch deployment <name> -p '{"spec":{"revisionHistoryLimit":20}}'

# 建议在创建时设置足够大的历史版本
```

### Q3：Pod 标签冲突导致更新异常

```bash
# 检查 selector 和 template.labels 是否匹配
kubectl get deployment <name> -o yaml | grep -A 5 selector
kubectl get deployment <name> -o yaml | grep -A 10 template

# 注意：spec.selector 一旦创建不可修改
# 如果标签不匹配，需要删除重建 Deployment

# 正确做法：
# spec.selector.matchLabels = spec.template.metadata.labels 的子集
```

### Q4：Recreate 策略导致服务中断

```bash
# Recreate 策略会先销毁所有 Pod 再创建新的
# 更新期间服务完全不可用

# 解决方案：
# 1. 使用 RollingUpdate 策略（推荐）
# 2. 调整 maxUnavailable=0 实现零停机
# 3. 使用蓝绿部署

# 修改策略为 RollingUpdate
kubectl patch deployment <name> \
  -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxUnavailable":0,"maxSurge":"100%"}}}}'
```

### Q5：HPA 与 Deployment 冲突

```bash
# HPA 会自动调整 replicas，可能与手动调整冲突
# 避免同时使用手动扩缩容和 HPA

# 查看 HPA 状态
kubectl get hpa
kubectl describe hpa

# 暂停 HPA（暂时手动管理）
kubectl delete hpa <name>

# 或者修改 HPA 配置
kubectl edit hpa <name>
```

### Q6：金丝雀部署流量不均

```bash
# 金丝雀流量基于 Pod 数量分配
# Service 使用轮询（RoundRobin）策略

# 确保 Pod 数量比例正确
kubectl get pods -l app=<label> -l track=stable | wc -l
kubectl get pods -l app=<label> -l track=canary | wc -l

# 如果需要精确的流量比例：
# 1. 使用 Istio 等 Service Mesh（基于权重路由）
# 2. 使用 Nginx Ingress 的 canary 功能
# 3. 实现自定义的流量分配逻辑
```

### Q7：滚动更新期间资源不足

```bash
# 滚动更新需要额外的资源（maxSurge）
# 如果资源不足，更新会卡住

# 解决方案：
# 1. 降低 maxSurge
kubectl patch deployment <name> \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1}}}}'

# 2. 降低 maxUnavailable 减少滚动压力
# 3. 先缩容再更新
kubectl scale deployment <name> --replicas=2
kubectl set image deployment/<name> <container>=<image>
kubectl scale deployment <name> --replicas=5
```

---

## 4.10 章节小结

### 核心概念

1. **Deployment 是无状态应用的管理控制器**，提供声明式更新、滚动升级、版本回滚和扩缩容
2. **三层架构**：Deployment → ReplicaSet → Pod，各司其职
3. **滚动更新**：通过 maxUnavailable 和 maxSurge 参数控制更新节奏
4. **版本回滚**：保留历史版本，支持一键回滚到任意版本
5. **扩缩容**：支持手动扩缩容和自动扩缩容（HPA）
6. **发布策略**：蓝绿部署（双环境切换）、金丝雀发布（渐进式灰度）

### 实战命令速查

```bash
# 创建 Deployment
kubectl apply -f deployment.yaml

# 查看 Deployment 状态
kubectl get deployment
kubectl describe deployment <name>

# 滚动更新
kubectl set image deployment/<name> <container>=<image>
kubectl rollout status deployment/<name>

# 回滚
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=N
kubectl rollout history deployment/<name>

# 扩缩容
kubectl scale deployment <name> --replicas=N

# 暂停/恢复
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>

# HPA
kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10

# 删除
kubectl delete deployment <name>
```

### 最佳实践

- **标签规范**：使用 `app`、`version`、`track` 等标签体系，方便蓝绿/金丝雀部署
- **健康检查**：配置合理的 readinessProbe，确保只有就绪的 Pod 才接收流量
- **滚动策略**：生产环境推荐 `maxUnavailable=0` + `maxSurge=100%` 实现零停机
- **历史版本**：设置足够大的 `revisionHistoryLimit`（至少 10），方便回滚
- **变更追踪**：使用 `--record` 参数或 `annotate` 记录变更原因
- **HPA 配置**：合理设置 minReplicas 和 maxReplicas，避免资源浪费
- **发布流程**：先在预发布环境验证，再在生产环境执行金丝雀发布
- **回滚预案**：每次发布前确认回滚命令可用，确保能快速回滚