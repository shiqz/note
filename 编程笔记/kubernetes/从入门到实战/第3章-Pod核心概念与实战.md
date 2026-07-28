# 第3章 Pod 核心概念与实战

## 学习目标

- 理解 Pod 的定义与设计哲学，掌握 Kubernetes 选择 Pod 而非直接管理容器的原因
- 掌握 Pod 的生命周期状态（Pending / Running / Succeeded / Failed / Unknown）及其转换条件
- 能够独立编写 Pod YAML 配置，理解每个字段的含义
- 掌握 Sidecar 模式与 Init Container 模式的使用场景与配置方法
- 理解 requests/limits 的区别及 QoS 等级（Guaranteed / Burstable / BestEffort）的调度影响
- 熟练配置 livenessProbe、readinessProbe、startupProbe 三种健康检查探针
- 掌握 Pod 重启策略（Always / OnFailure / Never）的适用场景
- 理解 Pod 间通信机制：同 Pod 内共享 localhost，跨 Pod 通过 Service 通信
- 通过 5 个实战练习，熟练掌握 Pod 的创建、调试与运维操作

---

## 3.1 Pod 的定义与特点

### 3.1.1 什么是 Pod

Pod 是 Kubernetes 中**最小的部署单元**，它是一组紧密相关的容器的集合。Pod 将多个容器组合在一起，作为一个逻辑单元进行调度和管理。

### 3.1.2 为什么 K8s 使用 Pod 而不是直接管理容器

| 原因 | 说明 |
|------|------|
| **共享资源** | 同一 Pod 内的容器共享网络命名空间（IP、端口）和存储卷，可以直接通过 `localhost` 通信 |
| **联合调度** | Pod 作为调度原子，确保所有容器在同一节点上运行，避免跨节点通信开销 |
| **生命周期绑定** | Pod 内的容器同生共死，一个容器退出则整个 Pod 重启 |
| **简化管理** | 将相关容器打包为一个单元，简化部署和扩缩容操作 |
| **IP 唯一性** | 每个 Pod 拥有唯一的集群内 IP，无需为每个容器单独分配 |

### 3.1.3 Pod 的核心特性

- **短暂性（Ephemeral）**：Pod 可以随时被销毁和重建，IP 地址可能变化
- **确定性**：Pod 中的容器共享网络和存储
- **原子性**：Pod 作为最小调度和部署单元

---

## 3.2 Pod 的生命周期状态详解

### 3.2.1 Pod 状态概览

```
                    ┌──────────┐
                    │  Pending  │ 调度中，Pod 已创建但尚未被调度到节点
                    └────┬─────┘
                         │ 调度成功
                         ▼
                    ┌──────────┐
           ┌────────│ Running   │ Pod 已绑定到节点，所有容器已启动
           │        └────┬─────┘
           │             │
           │    ┌────────┴────────┐
           │    ▼                 ▼
           │  ┌──────────┐  ┌──────────┐
           │  │ Succeeded│  │  Failed   │
           │  └──────────┘  └──────────┘
           │  所有容器正常终止  至少一个容器异常终止
           │
           │  ┌──────────┐
           └─>│ Unknown  │ 节点失联，状态未知
              └──────────┘
```

### 3.2.2 各状态详解

| 状态 | 含义 | 转换条件 |
|------|------|----------|
| **Pending** | Pod 已创建但尚未被调度到任何节点 | Pod 创建后立即进入此状态，等待调度器分配节点 |
| **Running** | Pod 已绑定到节点，所有容器已创建并启动 | 调度成功 + 节点分配成功 + 至少一个容器在运行 |
| **Succeeded** | Pod 中所有容器均正常终止（退出码 0） | 所有容器执行完毕并以退出码 0 终止 |
| **Failed** | Pod 中至少有一个容器以非 0 退出码终止，且所有容器已终止 | 容器异常退出，且重启策略不允许重启 |
| **Unknown** | 无法获取 Pod 的状态 | 所在节点失联（网络故障或 kubelet 停止） |

### 3.2.3 Pod Condition（子状态）

Pod 的顶层 `status.phase` 之外，还有更详细的 `status.conditions`：

```yaml
status:
  phase: Running
  conditions:
    - type: PodScheduled      # Pod 是否已被调度
      status: "True"
    - type: Ready              # Pod 是否就绪（所有容器就绪）
      status: "True"
    - type: ContainersReady    # 所有容器是否就绪
      status: "True"
    - type: Initialized        # 所有 init 容器是否完成
      status: "True"
```

---

## 3.3 Pod YAML 结构详解

### 3.3.1 完整 YAML 配置示例

```yaml
apiVersion: v1                    # API 版本，Pod 使用 v1
kind: Pod                         # 资源类型：Pod
metadata:
  name: nginx-pod                 # Pod 名称（在 namespace 内唯一）
  namespace: default              # 命名空间，默认 default
  labels:                         # 标签，用于筛选和分组
    app: nginx
    environment: production
    version: "1.25"
  annotations:                    # 注解，存储非标识性元数据
    description: "生产环境 Nginx Pod"
    owner: "devops-team"
  # 如果需要在创建时指定节点，可使用 nodeSelector
  # annotations:
  #   kubernetes.io/hostname: "node-1"

spec:
  # 基础调度相关
  nodeName: ""                    # 指定调度到特定节点（通常由调度器决定）
  nodeSelector: {}                # 节点标签选择器，Pod 只能调度到匹配的节点
  affinity: {}                    # 亲和性/反亲和性，更灵活的调度控制
  tolerations: []                 # 容忍的污点，允许 Pod 调度到有污点的节点
  priorityClassName: ""           # 优先级类，高优先级 Pod 可抢占低优先级 Pod

  # 容器定义
  containers:
    - name: nginx                 # 容器名称（Pod 内唯一）
      image: nginx:1.25-alpine    # 容器镜像（仓库地址:标签）
      imagePullPolicy: IfNotPresent # 镜像拉取策略：Always / Never / IfNotPresent
      command: ["/bin/sh"]        # 容器入口命令（覆盖 ENTRYPOINT）
      args: ["-c", "echo hello"]  # 命令参数（覆盖 CMD）
      workingDir: "/etc/nginx"    # 容器工作目录

      # 环境变量配置
      env:
        - name: NGINX_PORT
          value: "80"
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: CONFIG_PATH
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: config-path

      # 资源限制
      resources:
        requests:                 # 调度时保证的最小资源
          cpu: "100m"             # 100 millicores = 0.1 CPU
          memory: "128Mi"         # 128 Mebibytes
        limits:                   # 最大允许使用的资源
          cpu: "500m"             # 最多使用 0.5 CPU
          memory: "256Mi"         # 最多使用 256 MiB 内存
          # ephemeral-storage: "1Gi"  # 临时存储限制

      # 端口声明（仅作文档用途，不影响实际监听）
      ports:
        - name: http
          containerPort: 80       # 容器内监听的端口
          protocol: TCP           # 协议：TCP / UDP / SCTP
          hostPort: 0             # 宿主机端口（通常不指定）

      # 卷挂载
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: tmp-volume
          mountPath: /tmp
        - name: secret-volume
          mountPath: /etc/secret
          readOnly: true

      # 存活探针
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 15   # 容器启动后等待 15 秒开始探测
        periodSeconds: 10         # 每 10 秒探测一次
        timeoutSeconds: 5         # 超时时间
        failureThreshold: 3       # 连续 3 次失败判定为不健康
        successThreshold: 1       # 连续 1 次成功判定为健康

      # 就绪探针
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
        successThreshold: 1

      # 启动探针
      startupProbe:
        httpGet:
          path: /startup
          port: 80
        initialDelaySeconds: 0
        periodSeconds: 5
        failureThreshold: 30       # 启动阶段最多等待 150 秒（30 × 5s）

      # 安全上下文
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop: ["ALL"]

      # 容器的生命周期回调
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo Container started"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]

  # init 容器（在主容器启动前执行）
  initContainers:
    - name: init-myservice
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "等待 MyService 启动..."
          until nslookup myservice.default.svc.cluster.local; do
            echo "等待 MyService..."
            sleep 2
          done
          echo "MyService 已就绪"
    - name: init-clone
      image: busybox:1.36
      command: ["git", "clone", "https://github.com/user/repo.git", "/workspace"]
      volumeMounts:
        - name: workspace-volume
          mountPath: /workspace

  # 卷定义
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
    - name: tmp-volume
      emptyDir: {}
    - name: secret-volume
      secret:
        secretName: my-secret
    - name: workspace-volume
      emptyDir: {}

  # Pod 重启策略
  restartPolicy: Always            # Always / OnFailure / Never

  # Pod 自动销毁时间
  terminationGracePeriodSeconds: 30  # 优雅终止等待时间

  # 服务账户
  serviceAccountName: default      # 使用的 ServiceAccount
  automountServiceAccountToken: true  # 自动挂载 SA Token

  # 网络配置
  hostNetwork: false               # 是否使用宿主机网络
  hostPID: false                   # 是否共享宿主机 PID 命名空间
  hostIPC: false                   # 是否共享宿主机 IPC 命名空间
  shareProcessNamespace: false     # 是否与 Pod 内其他容器共享 PID 命名空间

  # DNS 配置
  dnsPolicy: ClusterFirst          # DNS 策略：Default / ClusterFirst / ClusterFirstWithHostNet / None
  dnsConfig:                       # 自定义 DNS 配置
    nameservers: ["10.96.0.10"]
    searches: ["default.svc.cluster.local", "svc.cluster.local", "cluster.local"]
    options:
      - name: ndots
        value: "5"

  # 安全上下文（Pod 级别）
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
```

### 3.3.2 YAML 字段速查表

| 字段 | 类型 | 说明 |
|------|------|------|
| `apiVersion` | string | API 版本 |
| `kind` | string | 资源类型 |
| `metadata.name` | string | Pod 名称 |
| `metadata.labels` | map | 标签键值对 |
| `spec.containers` | array | 容器定义列表 |
| `spec.initContainers` | array | 初始化容器列表 |
| `spec.volumes` | array | 卷定义列表 |
| `spec.restartPolicy` | string | 重启策略 |
| `spec.nodeSelector` | map | 节点选择器 |
| `spec.affinity` | object | 亲和性配置 |
| `spec.tolerations` | array | 污点容忍 |
| `spec.priorityClassName` | string | 优先级类 |
| `spec.serviceAccountName` | string | 服务账户名 |
| `spec.hostNetwork` | bool | 宿主机网络 |
| `spec.dnsPolicy` | string | DNS 策略 |

---

## 3.4 Pod 中的容器模式

### 3.4.1 Sidecar 模式

Sidecar 模式是指在 Pod 中运行一个辅助容器（Sidecar），与主容器协同工作。

#### 场景一：日志收集

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-collector-pod
  labels:
    app: log-collector
spec:
  containers:
    - name: app                    # 主应用容器
      image: myapp:1.0
      volumeMounts:
        - name: log-directory
          mountPath: /var/log/app
    - name: log-agent              # Sidecar 日志收集容器
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: log-directory
          mountPath: /var/log/app
          readOnly: true
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
  volumes:
    - name: log-directory
      emptyDir: {}
```

#### 场景二：配置热更新

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-reloader-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config
          subPath: config.yaml
    - name: config-reloader         # 监听 ConfigMap 变化并通知主容器
      image: containerslabs/config-reloader:latest
      args:
        - -volume-dir=/etc/app/config
        - -watch
        - -on-change
        - "kill -HUP 1"
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config
      securityContext:
        runAsUser: 0
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

### 3.4.2 Init Container 模式

Init Container 在主容器启动前执行，用于完成初始化工作。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-pod
spec:
  initContainers:
    # 等待依赖服务就绪
    - name: wait-for-db
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "等待数据库就绪..."
          until nc -z my-db-service 5432; do
            echo "数据库未就绪，2秒后重试..."
            sleep 2
          done
          echo "数据库已就绪！"

    # 从远端下载配置文件
    - name: fetch-config
      image: curlimages/curl:latest
      command:
        - sh
        - -c
        - |
          echo "下载配置文件..."
          curl -s https://config-server.example.com/config > /shared/config.json
          echo "配置下载完成"

    # 初始化数据
    - name: init-data
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "初始化数据..."
          cat /shared/config.json
          echo '{"initialized": true}' > /shared/status.json
          echo "初始化完成"
      volumeMounts:
        - name: shared-data
          mountPath: /shared

  containers:
    - name: main-app
      image: myapp:1.0
      volumeMounts:
        - name: shared-data
          mountPath: /shared
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5

  volumes:
    - name: shared-data
      emptyDir: {}
```

**Init Container 的执行规则：**
- 按定义顺序依次执行
- 前一个成功完成后才执行下一个
- 所有 Init Container 成功后，主容器才启动
- 任一 Init Container 失败则 Pod 重启（重启策略为 Always/OnFailure 时）

---

## 3.5 Pod 的资源限制

### 3.5.1 Requests 与 Limits 的区别

| 维度 | Requests | Limits |
|------|----------|--------|
| **含义** | 调度时保证分配的最小资源 | 允许使用的最大资源上限 |
| **调度影响** | 调度器根据 requests 决定 Pod 调度到哪个节点 | 节点上 Pod 的总 limits 不能超过节点可分配资源 |
| **超限行为** | 不影响调度（仅作为参考） | 超过 CPU limits 会被限流（throttle），超过 memory limits 会被 OOM Kill |
| **未指定时** | 默认等于 limits | 默认等于节点可分配资源的默认值 |

```yaml
resources:
  requests:
    cpu: "100m"          # 调度时至少分配 0.1 CPU
    memory: "128Mi"      # 调度时至少分配 128 MiB 内存
    ephemeral-storage: "1Gi"  # 调度时至少分配 1Gi 临时存储
  limits:
    cpu: "500m"          # 最多使用 0.5 CPU（超出会被限流）
    memory: "256Mi"      # 最多使用 256 MiB 内存（超出会被 OOM Kill）
    ephemeral-storage: "2Gi"  # 最多使用 2Gi 临时存储
```

### 3.5.2 QoS 等级

Kubernetes 根据 Pod 的 `requests` 与 `limits` 配置，将 Pod 划分为三种 QoS（Quality of Service）等级：

```
┌─────────────────────────────────────────────────────────────────┐
│                        QoS 等级划分                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Guaranteed (最高优先级)                                         │
│  ├── 所有容器都设置了 requests 和 limits                        │
│  ├── requests == limits（CPU 和 Memory）                        │
│  └── 示例：requests: {cpu: 100m, memory: 128Mi}                 │
│            limits:  {cpu: 100m, memory: 128Mi}                   │
│                                                                 │
│  Burstable (中等优先级)                                          │
│  ├── 至少一个容器设置了 requests 或 limits                      │
│  ├── 且不满足 Guaranteed 条件                                    │
│  └── 示例：requests: {cpu: 100m, memory: 128Mi}                 │
│            limits:  {cpu: 200m, memory: 256Mi}                  │
│                                                                 │
│  BestEffort (最低优先级)                                         │
│  ├── 所有容器都没有设置 requests 和 limits                      │
│  └── 示例：不设置任何 resources 配置                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### QoS 示例

```yaml
# Guaranteed QoS
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "200m"
          memory: "256Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
---
# Burstable QoS
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
---
# BestEffort QoS
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
    - name: app
      image: nginx:1.25
```

**内存不足时的驱逐顺序：** BestEffort → Burstable → Guaranteed

---

## 3.6 Pod 的健康检查

### 3.6.1 三种探针对比

| 探针 | 作用 | 失败行为 | 使用场景 |
|------|------|----------|----------|
| **livenessProbe** | 检测容器是否存活 | 重启容器 | 检测应用死锁、无需重启即可恢复的问题 |
| **readinessProbe** | 检测容器是否就绪 | 从 Service 中摘除 | 应用需要时间启动、预热缓存、等待依赖 |
| **startupProbe** | 检测容器是否已启动 | 重启容器，期间其他探针暂停 | 启动慢的应用，避免启动阶段误判 |

### 3.6.2 探针配置详解

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-check-demo
spec:
  containers:
    - name: web-app
      image: nginx:1.25

      # 方式一：HTTP GET 探测
      livenessProbe:
        httpGet:
          path: /healthz          # 探测路径
          port: 80               # 探测端口
          httpHeaders:           # 自定义请求头
            - name: Custom-Header
              value: check-health
        initialDelaySeconds: 30  # 启动后等待 30 秒
        periodSeconds: 10        # 每 10 秒探测一次
        timeoutSeconds: 3        # 单次探测超时
        failureThreshold: 3      # 连续 3 次失败才算失败
        successThreshold: 1      # 连续 1 次成功算成功

      # 方式二：TCP Socket 探测
      readinessProbe:
        tcpSocket:
          port: 8080
          host: localhost
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3

      # 方式三：Exec 命令探测
      startupProbe:
        exec:
          command:
            - /bin/sh
            - -c
            - "curl -f http://localhost:8080/startup || exit 1"
        initialDelaySeconds: 0
        periodSeconds: 5
        failureThreshold: 30
```

### 3.6.3 三种探针的协作关系

```
时间线：
  t=0s    Pod 启动
  t=0s    startupProbe 开始探测（每 5s 一次，最多 30 次 = 150s）
  t=30s   startupProbe 成功
          livenessProbe 开始探测（initialDelaySeconds=30）
  t=30s   readinessProbe 开始探测（initialDelaySeconds=5，已等待 25s）
  t=30s+  readinessProbe 成功 → Pod Ready → 加入 Service Endpoints
  t=∞     livenessProbe 持续探测 → 失败则重启容器
          readinessProbe 持续探测 → 失败则从 Service 摘除
```

**关键规则：**
- `startupProbe` 存在时，`livenessProbe` 和 `readinessProbe` 会被暂停，直到 startupProbe 成功
- `startupProbe` 成功后，其他探针开始工作
- 推荐配置 startProbe，用于处理启动慢的应用，避免 startup 期间误触发 livenessProbe 重启

---

## 3.7 Pod 的重启策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **Always** | 容器终止后总是重启（默认值） | 长期运行的服务型应用（Web 服务、API 服务等） |
| **OnFailure** | 容器以非 0 退出码终止时重启 | 批处理任务、需要重试的 Job |
| **Never** | 容器终止后永不重启 | 一次性任务、调试用途 |

```yaml
# Always 策略
apiVersion: v1
kind: Pod
metadata:
  name: always-restart-pod
spec:
  restartPolicy: Always           # 默认值，可省略
  containers:
    - name: web
      image: nginx:1.25

# OnFailure 策略
apiVersion: v1
kind: Pod
metadata:
  name: onfailure-restart-pod
spec:
  restartPolicy: OnFailure
  containers:
    - name: batch
      image: busybox:1.36
      command: ["sh", "-c", "echo '处理数据'; exit 0"]

# Never 策略
apiVersion: v1
kind: Pod
metadata:
  name: never-restart-pod
spec:
  restartPolicy: Never
  containers:
    - name: one-shot
      image: busybox:1.36
      command: ["sh", "-c", "echo '一次性执行'; exit 0"]
```

**重启策略与 Pod 状态的关系：**

| 重启策略 | 容器退出码 | Pod 状态 |
|----------|-----------|----------|
| Always | 0 | Running（重启） |
| Always | 非 0 | Running（重启） |
| OnFailure | 0 | Succeeded |
| OnFailure | 非 0 | Running（重启） |
| Never | 0 | Succeeded |
| Never | 非 0 | Failed |

---

## 3.8 Pod 之间的通信

### 3.8.1 同一 Pod 内的容器通信

同一 Pod 内的容器共享**网络命名空间**，因此：
- 通过 `localhost` 或 `127.0.0.1` 直接通信
- 共享所有网络接口和 IP 地址
- 共享端口空间（不能重复）

```
Pod 内部：
┌─────────────────────────────────────────┐
│              Network Namespace          │
│  IP: 10.244.1.5 (Pod IP)               │
│  ┌──────────────┐    ┌──────────────┐   │
│  │  Container A │    │  Container B │   │
│  │  Port: 8080  │───▶│  Port: 9090  │   │
│  │  localhost:9090   │              │   │
│  └──────────────┘    └──────────────┘   │
└─────────────────────────────────────────┘
```

### 3.8.2 跨 Pod 通信

跨 Pod 通信通过 **Service** 实现：

```
                  ┌──────────────────────┐
                  │   Cluster DNS        │
                  │  kube-dns / CoreDNS  │
                  └──────────┬───────────┘
                             │ 名称解析
                             ▼
┌──────────────┐    Service    ┌──────────────┐
│   Pod A      │ ─────────────▶│   Pod B      │
│  10.244.1.5  │   Pod B IP    │  10.244.2.8  │
│              │  + Pod C IP   │              │
└──────────────┘               └──────────────┘

通信流程：
1. Pod A 访问 my-service.default.svc.cluster.local
2. DNS 解析返回 Service ClusterIP（如 10.96.0.100）
3. kube-proxy 转发请求到后端 Pod B 或 Pod C
```

#### Service 配置示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
```

#### Pod 间通信示例

```yaml
# Pod A：前端应用
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  labels:
    app: frontend
spec:
  containers:
    - name: frontend
      image: nginx:1.25
      env:
        - name: BACKEND_URL
          value: "http://backend-service.default.svc.cluster.local"
      ports:
        - containerPort: 80
---
# Pod B：后端应用
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend
spec:
  containers:
    - name: backend
      image: mybackend:1.0
      ports:
        - containerPort: 8080
          name: http
```

### 3.8.3 通信方式总结

| 通信场景 | 方式 | 说明 |
|----------|------|------|
| 同 Pod 内容器 | `localhost` | 共享网络命名空间 |
| 同命名空间 Pod | `<service-name>` | 短名称解析 |
| 跨命名空间 Pod | `<service-name>.<namespace>` | 包含命名空间 |
| 完全限定域名 | `<service-name>.<namespace>.svc.cluster.local` | 完整域名 |
| 外部访问 | NodePort / LoadBalancer / Ingress | 暴露到集群外 |

---

## 3.9 实战练习

### 实战 1：创建单个 Nginx Pod

**目标：** 创建一个最简单的 Nginx Pod，并验证其运行状态。

```bash
# 第 1 步：创建 Pod YAML 文件
cat > /tmp/nginx-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
          name: http
EOF

# 第 2 步：创建 Pod
kubectl apply -f /tmp/nginx-pod.yaml

# 第 3 步：查看 Pod 创建状态
kubectl get pods -l app=nginx

# 第 4 步：查看 Pod 详细信息
kubectl describe pod nginx-pod

# 第 5 步：查看 Pod 状态变化
kubectl get pod nginx-pod -o wide

# 第 6 步：验证 Pod IP
kubectl get pod nginx-pod -o jsonpath='{.status.podIP}'
echo ""

# 第 7 步：使用端口转发访问 Pod
kubectl port-forward pod/nginx-pod 8080:80 &
# 另开终端访问
curl http://localhost:8080

# 第 8 步：清理资源
kubectl delete pod nginx-pod
```

### 实战 2：创建多容器 Pod（Nginx + busybox sidecar 共享日志）

**目标：** 演示 Sidecar 模式，Nginx 写入日志到共享卷，busybox 读取并输出日志。

```bash
# 第 1 步：创建 Sidecar Pod YAML
cat > /tmp/sidecar-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
  labels:
    app: sidecar-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
      ports:
        - containerPort: 80
          name: http

    - name: log-reader
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "等待 Nginx 日志文件生成..."
          until [ -f /var/log/nginx/access.log ]; do
            sleep 1
          done
          echo "开始读取日志..."
          tail -f /var/log/nginx/access.log
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/nginx
  volumes:
    - name: shared-logs
      emptyDir: {}
EOF

# 第 2 步：创建 Pod
kubectl apply -f /tmp/sidecar-pod.yaml

# 第 3 步：等待 Pod 就绪
kubectl wait --for=condition=Ready pod/sidecar-pod --timeout=60s

# 第 4 步：访问 Nginx 产生日志
kubectl port-forward pod/sidecar-pod 8080:80 &
curl http://localhost:8080
curl http://localhost:8080/index.html

# 第 5 步：查看 Sidecar 日志（log-reader 容器）
kubectl logs sidecar-pod -c log-reader

# 第 6 步：查看 Nginx 容器日志
kubectl logs sidecar-pod -c nginx

# 第 7 步：进入容器查看共享文件
kubectl exec sidecar-pod -c nginx -- ls -la /var/log/nginx/
kubectl exec sidecar-pod -c log-reader -- cat /var/log/nginx/access.log

# 第 8 步：清理
kubectl delete pod sidecar-pod
```

### 实战 3：配置资源限制并观察 QoS 等级

**目标：** 创建三种 QoS 等级的 Pod，观察 QoS 标记。

```bash
# 第 1 步：创建 Guaranteed QoS Pod
cat > /tmp/qos-guaranteed.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
  labels:
    app: qos-demo
    qos: guaranteed
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      resources:
        requests:
          cpu: "200m"
          memory: "256Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
EOF

# 第 2 步：创建 Burstable QoS Pod
cat > /tmp/qos-burstable.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
  labels:
    app: qos-demo
    qos: burstable
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
EOF

# 第 3 步：创建 BestEffort QoS Pod
cat > /tmp/qos-besteffort.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
  labels:
    app: qos-demo
    qos: besteffort
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
EOF

# 第 4 步：创建所有 Pod
kubectl apply -f /tmp/qos-guaranteed.yaml
kubectl apply -f /tmp/qos-burstable.yaml
kubectl apply -f /tmp/qos-besteffort.yaml

# 第 5 步：查看 QoS 等级
echo "=== Guaranteed QoS ==="
kubectl describe pod qos-guaranteed | grep -i "qos"

echo "=== Burstable QoS ==="
kubectl describe pod qos-burstable | grep -i "qos"

echo "=== BestEffort QoS ==="
kubectl describe pod qos-besteffort | grep -i "qos"

# 第 6 步：查看资源请求和限制
kubectl get pods -l app=qos-demo -o custom-columns=\
"NAME:.metadata.name,\
"CPU_REQUEST:.spec.containers[0].resources.requests.cpu,\
"CPU_LIMIT:.spec.containers[0].resources.limits.cpu,\
"MEM_REQUEST:.spec.containers[0].resources.requests.memory,\
"MEM_LIMIT:.spec.containers[0].resources.limits.memory

# 第 7 步：清理
kubectl delete pod -l app=qos-demo
```

### 实战 4：配置健康检查，手动制造探针失败场景

**目标：** 配置 livenessProbe 和 readinessProbe，手动制造失败场景观察 Pod 重启和流量摘除行为。

```bash
# 第 1 步：创建带健康检查的 Pod
cat > /tmp/health-check-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: health-check-pod
  labels:
    app: health-check-demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
          name: http
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
        successThreshold: 1
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 3
        failureThreshold: 3
        successThreshold: 1
      startupProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 0
        periodSeconds: 2
        failureThreshold: 10
EOF

# 第 2 步：创建 Pod
kubectl apply -f /tmp/health-check-pod.yaml

# 第 3 步：等待 Pod 就绪
kubectl wait --for=condition=Ready pod/health-check-pod --timeout=60s

# 第 4 步：验证探针状态
kubectl get pod health-check-pod -o jsonpath='{.status.containerStatuses[0].ready}'
echo ""

# 第 5 步：制造 livenessProbe 失败场景
# 删除健康检查文件，导致 /healthz 返回 404
kubectl exec health-check-pod -- rm /etc/nginx/conf.d/default.conf

# 第 6 步：观察 Pod 重启行为
echo "=== 观察重启次数 ==="
for i in $(seq 1 20); do
  RESTARTS=$(kubectl get pod health-check-pod -o jsonpath='{.status.containerStatuses[0].restartCount}')
  STATE=$(kubectl get pod health-check-pod -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')
  echo "Time ${i}: Restarts=${RESTARTS}, State=${STATE}"
  sleep 3
done

# 第 7 步：恢复配置
kubectl delete pod health-check-pod
kubectl apply -f /tmp/health-check-pod.yaml
kubectl wait --for=condition=Ready pod/health-check-pod --timeout=60s

# 第 8 步：制造 readinessProbe 失败场景
# 手动将应用置为不可就绪
kubectl exec health-check-pod -- sh -c "echo 'server { listen 80; return 503; }' > /etc/nginx/conf.d/default.conf"

# 第 9 步：观察就绪状态
for i in $(seq 1 10); do
  READY=$(kubectl get pod health-check-pod -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')
  echo "Time ${i}: Ready=${READY}"
  sleep 2
done

# 第 10 步：恢复并清理
kubectl delete pod health-check-pod
```

### 实战 5：使用 kubectl 调试 Pod

**目标：** 熟练使用 kubectl logs、exec、describe 等命令调试 Pod。

```bash
# ========== 准备工作 ==========

# 创建一个多容器测试 Pod
cat > /tmp/debug-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  labels:
    app: debug-demo
  annotations:
    description: "调试演示 Pod"
spec:
  containers:
    - name: web
      image: nginx:1.25-alpine
      ports:
        - containerPort: 80
      env:
        - name: ENVIRONMENT
          value: "development"
        - name: LOG_LEVEL
          value: "debug"
    - name: sidecar
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo 'Sidecar running...'; sleep 10; done"]
      env:
        - name: SIDECAR_NAME
          value: "debug-sidecar"
  initContainers:
    - name: init-check
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Init container running...'; sleep 1; echo 'Init complete'"]
EOF

kubectl apply -f /tmp/debug-pod.yaml
kubectl wait --for=condition=Ready pod/debug-pod --timeout=60s

# ========== kubectl describe 详解 ==========

# 查看 Pod 完整信息
kubectl describe pod debug-pod

# 输出说明：
# - Name: Pod 名称
# - Namespace: 命名空间
# - Priority: 优先级
# - Node: 所在节点
# - Start Time: 创建时间
# - Labels: 标签
# - Annotations: 注解
# - Status: Pod 状态（Running）
# - IP: Pod IP
# - Containers: 容器列表
#   - Container 名称
#   - Image: 镜像
#   - Port: 端口
#   - Requests/Limits: 资源配置
#   - State: 容器状态
#   - Ready: 就绪状态
#   - Restart Count: 重启次数
#   - Liveness/Readiness Probe: 探针配置
# - Conditions: Pod 条件详情
# - Events: 事件（重要！排查问题首选）

# ========== kubectl logs 使用 ==========

# 查看默认容器日志
kubectl logs debug-pod

# 指定容器查看日志
kubectl logs debug-pod -c sidecar

# 查看前 100 行日志
kubectl logs --tail=100 debug-pod

# 查看最近 10 分钟的日志
kubectl logs --since=10m debug-pod

# 实时跟随日志输出
kubectl logs -f debug-pod

# 查看上一次崩溃的日志
kubectl logs --previous debug-pod

# 同时查看所有容器日志
kubectl logs --all-containers debug-pod

# ========== kubectl exec 使用 ==========

# 进入默认容器
kubectl exec -it debug-pod -- /bin/sh

# 指定容器进入
kubectl exec -it debug-pod -c web -- /bin/sh

# 执行单条命令
kubectl exec debug-pod -- nginx -t

# 在 sidecar 容器中执行命令
kubectl exec debug-pod -c sidecar -- echo "Hello from sidecar"

# 查看环境变量
kubectl exec debug-pod -c web -- env

# 查看文件系统
kubectl exec debug-pod -c web -- ls -la /etc/nginx/

# 查看进程
kubectl exec debug-pod -c web -- ps aux

# ========== 端口转发 ==========

# 转发 Pod 端口到本地
kubectl port-forward debug-pod 8080:80

# 在后台运行端口转发
kubectl port-forward debug-pod 8080:80 &
PF_PID=$!

# 访问服务
curl http://localhost:8080

# 停止端口转发
kill $PF_PID

# ========== 使用 ephemeral 容器调试 ==========

# 对于没有 shell 的容器，使用临时调试容器
kubectl debug debug-pod -it --image=busybox:1.36 --target=web

# 附加到 Pod 的命名空间进行调试
kubectl debug debug-pod -it --image=busybox:1.36 --share-process-namespace

# ========== 查看 Pod 资源使用 ==========

# 查看 Pod 的 YAML 输出
kubectl get pod debug-pod -o yaml

# 查看 Pod 的 JSON 输出
kubectl get pod debug-pod -o json

# 查看特定字段
kubectl get pod debug-pod -o jsonpath='{.status.podIP}'
kubectl get pod debug-pod -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl get pod debug-pod -o jsonpath='{.metadata.labels}'

# 自定义列输出
kubectl get pod debug-pod -o custom-columns=\
"NAME:.metadata.name,\
"STATUS:.status.phase,\
"READY:.status.conditions[?(@.type=="Ready")].status,\
"RESTARTS:.status.containerStatuses[*].restartCount,\
"IP:.status.podIP

# ========== 清理 ==========

kubectl delete pod debug-pod
```

---

## 3.10 常见问题排查

### Q1：Pod 一直处于 Pending 状态

```bash
# 查看事件
kubectl describe pod <pod-name>
# 关注 Events 部分，常见原因：
# 1. 资源不足（Insufficient CPU/Memory）
# 2. 节点选择器不匹配
# 3. 污点/容忍不匹配
# 4. PVC 未绑定

# 排查资源不足
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes

# 排查节点选择器
kubectl get nodes -l <label-key>=<label-value>
```

### Q2：Pod 频繁重启

```bash
# 查看重启次数
kubectl get pod <pod-name>

# 查看事件
kubectl describe pod <pod-name> | tail -30

# 查看上一次崩溃的日志
kubectl logs --previous <pod-name>

# 常见原因：
# 1. livenessProbe 失败（检查探针配置和应用健康接口）
# 2. OOMKilled（调整内存 limits）
# 3. 应用自身崩溃（查看应用日志）
```

### Q3：健康检查一直失败

```bash
# 手动测试健康接口
kubectl port-forward pod/<pod-name> 8080:80
curl -v http://localhost:8080/healthz

# 检查探针配置
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].livenessProbe}'

# 确认容器内应用实际监听的端口
kubectl exec <pod-name> -- netstat -tlnp
# 或
kubectl exec <pod-name> -- ss -tlnp
```

### Q4：Init Container 执行失败

```bash
# 查看 init 容器日志
kubectl logs <pod-name> -c <init-container-name>

# 查看 Pod 状态
kubectl describe pod <pod-name>
# 关注 Init Containers 部分的 State 字段

# 常见原因：
# 1. 依赖服务未就绪（调整等待逻辑）
# 2. 配置错误（检查 init container 的 command）
# 3. 网络问题（检查网络策略和 DNS）
```

### Q5：Pod OOMKilled

```bash
# 增加内存限制
kubectl patch pod <pod-name> -p '{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"512Mi"}}}]}}'

# 或者修改 YAML 后重新部署
# 建议设置合理的 requests 和 limits
```

### Q6：容器无法访问外部服务

```bash
# 检查 DNS
kubectl exec <pod-name> -- nslookup google.com
kubectl exec <pod-name> -- cat /etc/resolv.conf

# 检查网络策略
kubectl get networkpolicies
kubectl describe networkpolicy

# 检查 egress 规则
kubectl exec <pod-name> -- curl -v http://external-service.com
```

### Q7：Pod 之间无法通信

```bash
# 检查 Service 是否存在
kubectl get svc

# 检查 Service Endpoints
kubectl get endpoints <service-name>

# 测试 DNS 解析
kubectl exec <pod-name> -- nslookup <service-name>.<namespace>.svc.cluster.local

# 检查网络策略
kubectl get networkpolicy -n <namespace>
```

---

## 3.11 章节小结

### 核心概念

1. **Pod 是 K8s 最小的部署单元**，由一个或多个紧密相关的容器组成
2. **Pod 设计哲学**：容器间共享网络和存储，联合调度，同生共死
3. **Pod 生命周期**：Pending → Running → Succeeded / Failed / Unknown
4. **容器模式**：Sidecar 模式（辅助容器）、Init Container 模式（前置初始化）
5. **资源管理**：Requests（保证最小）、Limits（限制最大）、QoS 等级决定驱逐优先级
6. **健康检查**：livenessProbe（存活）、readinessProbe（就绪）、startupProbe（启动）
7. **重启策略**：Always（服务类）、OnFailure（任务类）、Never（一次性）
8. **通信机制**：同 Pod 内 localhost，跨 Pod 通过 Service + DNS

### 实战命令速查

```bash
# 创建 Pod
kubectl apply -f pod.yaml

# 查看 Pod 状态
kubectl get pods -o wide

# 查看 Pod 详情
kubectl describe pod <name>

# 查看日志
kubectl logs <pod> -c <container> -f

# 进入容器
kubectl exec -it <pod> -c <container> -- /bin/sh

# 端口转发
kubectl port-forward <pod> <local-port>:<container-port>

# 删除 Pod
kubectl delete pod <name>

# 使用临时容器调试
kubectl debug <pod> -it --image=busybox:1.36
```

### 最佳实践

- **资源配置**：始终设置合理的 requests 和 limits，避免资源争抢
- **健康检查**：配置三种探针，特别是 startupProbe 处理慢启动场景
- **Init Container**：用于初始化依赖、等待外部服务、下载配置等
- **Sidecar 模式**：日志收集、配置热更新、监控代理等
- **标签规范**：使用清晰的标签体系（app、env、version 等）
- **命名规范**：Pod 命名应具有描述性，方便排查
