# HPA 和 VPA 的区别

## 一句话理解

在 Kubernetes 中，HPA 和 VPA 都是自动扩缩容机制，但它们解决的问题不同：

- **HPA（Horizontal Pod Autoscaler）**：横向扩缩容，调整 Pod 副本数量。
- **VPA（Vertical Pod Autoscaler）**：纵向扩缩容，调整单个 Pod 的 CPU / 内存 requests 和 limits。

可以把它们理解为：

```text
HPA：人手不够，就多招几个人一起干活。
VPA：一个人干不动，就给他更好的电脑和更多内存。
```

---

## 1. 为什么需要自动扩缩容

在 Kubernetes 中，一个服务通常通过 Deployment 管理多个 Pod。业务流量会随着时间变化：

- 白天访问量高，晚上访问量低
- 大促期间流量突然上涨
- 某些任务型服务在短时间内 CPU 飙高
- 部分服务长期资源配置不合理，CPU 或内存 requests 过大/过小

如果完全靠人工扩缩容，会遇到两个问题：

1. **响应慢**：流量已经上来了，人工再扩容可能来不及。
2. **资源浪费**：为了应对峰值一直配置高资源，平时会大量空闲。

自动扩缩容的目标就是让集群根据实际负载自动调整资源，使服务在稳定性和成本之间取得平衡。

---

## 2. HPA：横向扩缩容

### 2.1 HPA 是什么

HPA 全称是 **Horizontal Pod Autoscaler**，即水平 Pod 自动扩缩容。

它会根据 CPU、内存或自定义指标，自动调整目标工作负载的副本数。常见目标包括：

- Deployment
- StatefulSet
- ReplicaSet

例如，一个 Deployment 原来有 2 个 Pod：

```text
Deployment: web-api
replicas: 2

Pod-1  Pod-2
```

当 CPU 使用率持续超过阈值时，HPA 可以把副本数扩到 5：

```text
Deployment: web-api
replicas: 5

Pod-1  Pod-2  Pod-3  Pod-4  Pod-5
```

### 2.2 HPA 的工作原理

HPA 的核心逻辑是一个控制循环：

```text
采集指标 -> 计算期望副本数 -> 修改 workload.spec.replicas -> Deployment 创建/删除 Pod
```

典型流程如下：

```text
Metrics Server / Custom Metrics API
          │
          ▼
Horizontal Pod Autoscaler Controller
          │
          ▼
计算期望副本数
          │
          ▼
修改 Deployment replicas
          │
          ▼
ReplicaSet 创建或删除 Pod
```

HPA 本身不直接创建 Pod，它只是修改 Deployment 的 `spec.replicas` 字段，真正创建 Pod 的仍然是 Deployment / ReplicaSet 控制器。

### 2.3 HPA 的基础示例

下面的 HPA 表示：

- 最少 2 个 Pod
- 最多 10 个 Pod
- CPU 平均利用率超过 70% 时扩容
- CPU 降下来后再缩容

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

对应的 Deployment 必须配置 `resources.requests.cpu`，否则 HPA 无法计算 CPU 利用率：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      containers:
        - name: web-api
          image: nginx:1.25
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

### 2.4 HPA 的计算逻辑

HPA 的核心公式可以简化理解为：

```text
期望副本数 = 当前副本数 × 当前指标值 / 目标指标值
```

例如：

- 当前副本数：2
- 当前平均 CPU 利用率：140%
- 目标 CPU 利用率：70%

那么：

```text
期望副本数 = 2 × 140% / 70% = 4
```

于是 HPA 会尝试把副本数从 2 调整到 4。

注意这里的 CPU 利用率是相对于 `requests.cpu` 计算的，不是相对于节点总 CPU 计算的。

例如某个 Pod 配置：

```yaml
resources:
  requests:
    cpu: "200m"
```

如果它实际使用了 `140m` CPU，那么利用率就是：

```text
140m / 200m = 70%
```

### 2.5 HPA 支持的指标类型

| 指标类型 | 说明 | 示例 |
|---|---|---|
| Resource Metrics | CPU、内存等资源指标 | CPU 平均利用率 70% |
| Pods Metrics | 每个 Pod 的自定义指标 | 每个 Pod QPS |
| Object Metrics | 某个 Kubernetes 对象的指标 | Ingress QPS |
| External Metrics | 集群外部指标 | 消息队列长度、Kafka Lag |

实际生产中，CPU HPA 最常见，但不一定最准确。对于 API 服务，基于 QPS、响应时间、队列长度的扩缩容通常更贴近业务。

---

## 3. VPA：纵向扩缩容

### 3.1 VPA 是什么

VPA 全称是 **Vertical Pod Autoscaler**，即垂直 Pod 自动扩缩容。

它关注的不是 Pod 数量，而是单个 Pod 的资源配置：

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

如果一个服务长期 CPU 使用率很高，VPA 可能建议把 `requests.cpu` 从 `200m` 调到 `500m`。

如果一个服务长期只使用很少内存，VPA 可能建议把 `requests.memory` 从 `1Gi` 降到 `256Mi`。

### 3.2 VPA 的工作原理

VPA 主要由三个组件组成：

| 组件 | 作用 |
|---|---|
| Recommender | 根据历史资源使用量计算推荐值 |
| Updater | 判断是否需要驱逐旧 Pod，使新配置生效 |
| Admission Controller | 在 Pod 创建时注入推荐的 resources |

简化流程如下：

```text
采集 Pod 历史资源使用量
          │
          ▼
VPA Recommender 生成推荐值
          │
          ▼
VPA Updater 判断是否需要重建 Pod
          │
          ▼
驱逐旧 Pod
          │
          ▼
Deployment 创建新 Pod
          │
          ▼
Admission Controller 注入新的 requests / limits
```

这里有一个关键点：**大多数资源配置变更不能直接在运行中的 Pod 上原地生效，需要重建 Pod。**

所以 VPA 如果处于自动更新模式，可能会驱逐 Pod，这会带来服务抖动风险。

### 3.3 VPA 的基础示例

下面的 VPA 作用于 `web-api` 这个 Deployment：

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-api-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: web-api
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "2Gi"
```

### 3.4 VPA 的 updateMode

VPA 的 `updateMode` 很重要，它决定 VPA 只是给建议，还是自动修改资源。

| updateMode | 行为 | 适用场景 |
|---|---|---|
| Off | 只生成推荐值，不修改 Pod | 资源评估、压测分析 |
| Initial | 只在 Pod 创建时设置资源，不主动驱逐已运行 Pod | 稳定服务，降低扰动 |
| Auto | 自动应用推荐值，必要时驱逐 Pod | 可接受重建的服务 |
| Recreate | 类似 Auto，通过重建 Pod 应用资源 | 老版本常见配置 |

生产环境中，通常建议先使用 `Off` 或 `Initial`，观察一段时间后再决定是否启用 `Auto`。

### 3.5 查看 VPA 推荐值

创建 VPA 后，可以通过下面的命令查看推荐资源：

```bash
kubectl describe vpa web-api-vpa
```

你通常会看到类似信息：

```text
Recommendation:
  Container Recommendations:
    Container Name: web-api
    Lower Bound:
      Cpu:     100m
      Memory: 128Mi
    Target:
      Cpu:     500m
      Memory: 512Mi
    Upper Bound:
      Cpu:     1
      Memory:  1Gi
```

这些值可以这样理解：

| 字段 | 说明 |
|---|---|
| Lower Bound | 推荐资源下限 |
| Target | 推荐使用的资源值 |
| Upper Bound | 推荐资源上限 |

一般最值得关注的是 `Target`，它可以作为调整 requests 的参考。

---

## 4. HPA 和 VPA 的核心区别

| 对比项 | HPA | VPA |
|---|---|---|
| 扩缩容方向 | 横向扩缩容 | 纵向扩缩容 |
| 调整对象 | Pod 副本数 | 单个 Pod 的 CPU / 内存 requests、limits |
| 修改字段 | `spec.replicas` | `resources.requests` / `resources.limits` |
| 是否重建 Pod | 扩容时新增 Pod，缩容时删除 Pod | 通常需要重建 Pod 才能生效 |
| 适合服务 | 无状态服务、可水平扩展服务 | 单实例资源敏感服务、资源配置优化 |
| 典型指标 | CPU、内存、QPS、队列长度 | CPU、内存历史使用量 |
| 常见风险 | 扩缩容震荡、冷启动、下游压力增加 | 驱逐 Pod、资源配置波动 |
| 生产常用程度 | 非常常用 | 更谨慎使用 |

---

## 5. 用业务场景理解 HPA 和 VPA

### 5.1 Web API 服务

Web API 通常是无状态服务，可以通过增加 Pod 数量处理更多请求。

推荐方案：

```text
优先使用 HPA
```

原因：

- Pod 多了以后可以分摊流量
- 滚动扩容对服务影响小
- 与 Service 负载均衡天然配合
- 出问题时缩容或回滚比较简单

典型配置：

```text
minReplicas: 2
maxReplicas: 20
CPU target: 60% ~ 70%
```

如果业务更成熟，可以改为基于 QPS、P95 延迟或队列长度扩缩容。

### 5.2 消费者服务

比如 Kafka Consumer、RabbitMQ Consumer、任务队列 Worker。

推荐方案：

```text
优先使用 HPA，但指标最好用队列长度或消费延迟
```

原因：

- CPU 不一定能准确反映堆积情况
- 队列堆积才是真正的扩容信号
- Pod 副本数增加后可以提高消费并发

例如：

```text
Kafka Lag 高 -> 增加 Consumer Pod
Kafka Lag 降低 -> 减少 Consumer Pod
```

### 5.3 Java 服务

Java 服务对内存比较敏感，而且 JVM 堆大小、容器内存限制、GC 行为之间存在关系。

推荐方案：

```text
HPA 为主，VPA 用于观察和资源建议
```

不要一开始就让 VPA 自动调整 Java 服务内存，因为：

- VPA 调整内存通常需要重建 Pod
- JVM 参数可能没有跟随容器 limits 自动调整
- Pod 频繁重建可能影响服务稳定性

更稳妥的做法是：

1. VPA 使用 `updateMode: Off`
2. 观察推荐值
3. 人工调整 Deployment resources
4. 通过滚动发布生效

### 5.4 数据库或有状态服务

数据库、缓存、搜索引擎等有状态组件不适合轻易横向扩缩容。

推荐方案：

```text
谨慎使用 HPA，VPA 也不建议直接 Auto
```

原因：

- 副本扩缩容可能涉及数据分片、主从关系、数据迁移
- 驱逐 Pod 可能导致主从切换或服务抖动
- 资源调整需要结合存储、连接数、缓存命中率等因素

这类服务更适合通过专门的 Operator 或人工容量规划管理。

---

## 6. HPA 和 VPA 能不能一起用

答案是：**可以，但要看指标和配置方式。**

### 6.1 不建议同时基于 CPU/内存自动调整

如果 HPA 基于 CPU 利用率扩缩容，而 VPA 又在自动调整 `requests.cpu`，两者可能互相影响。

例如：

```text
HPA 的 CPU 利用率 = 当前 CPU 使用量 / requests.cpu
```

如果 VPA 把 `requests.cpu` 调大：

```text
原来：
实际使用 CPU = 200m
requests.cpu = 200m
CPU 利用率 = 100%

VPA 调整后：
实际使用 CPU = 200m
requests.cpu = 500m
CPU 利用率 = 40%
```

这时 HPA 可能认为负载下降了，于是缩容。

所以如果 HPA 和 VPA 都自动调整 CPU / 内存，很容易出现策略互相干扰。

### 6.2 推荐组合方式

更推荐的组合是：

| 组合方式 | 是否推荐 | 说明 |
|---|---|---|
| HPA 基于 CPU + VPA Auto 调 CPU | 不推荐 | 两者会互相影响 |
| HPA 基于内存 + VPA Auto 调内存 | 不推荐 | 同样容易互相影响 |
| HPA 基于 QPS/队列长度 + VPA 调 CPU/内存 | 推荐 | 指标维度不同，冲突小 |
| HPA 负责副本数 + VPA Off 提供建议 | 推荐 | 最稳妥 |
| HPA 负责业务流量 + VPA Initial 设置初始资源 | 可用 | 适合稳定服务 |

生产环境中，一个比较稳的模式是：

```text
HPA：根据业务指标扩缩容
VPA：使用 Off 或 Initial，辅助资源建议
```

---

## 7. HPA 的生产实践

### 7.1 必须配置 resources.requests

如果使用 CPU 或内存 HPA，容器必须配置 requests：

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

没有 `requests.cpu` 时，HPA 无法计算 CPU 利用率。

### 7.2 设置合理的 minReplicas

不要把生产核心服务的 `minReplicas` 设置为 1。

更推荐：

```yaml
minReplicas: 2
```

这样至少可以保证：

- 单个 Pod 重启时还有另一个 Pod 提供服务
- 节点维护或滚动发布时可用性更好

### 7.3 控制扩容和缩容速度

HPA 支持通过 `behavior` 控制扩缩容节奏：

```yaml
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

含义：

- 扩容要快，避免流量打爆服务。
- 缩容要慢，避免负载刚下降就立刻缩容，然后又反复扩容。

### 7.4 注意冷启动问题

HPA 扩容不是瞬时完成的。完整链路包括：

```text
指标超过阈值
  -> HPA 计算新副本数
  -> Deployment 创建 Pod
  -> 调度到 Node
  -> 拉取镜像
  -> 容器启动
  -> readinessProbe 通过
  -> Service 分发流量
```

如果镜像很大、启动很慢，扩容可能需要几十秒甚至几分钟。

优化方向：

- 减小镜像体积
- 设置合理的 `readinessProbe`
- 提前保留一定 `minReplicas`
- 配合 Cluster Autoscaler 扩容节点
- 对可预测流量提前定时扩容

---

## 8. VPA 的生产实践

### 8.1 先用 Off 模式观察

生产环境不要一开始就使用：

```yaml
updateMode: "Auto"
```

更稳妥的是：

```yaml
updatePolicy:
  updateMode: "Off"
```

先观察 VPA 推荐值，再决定是否调整 Deployment。

### 8.2 给 VPA 设置上下限

一定要通过 `resourcePolicy` 限制 VPA 的调整范围：

```yaml
resourcePolicy:
  containerPolicies:
    - containerName: web-api
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "2"
        memory: "2Gi"
```

否则 VPA 可能给出过大或过小的推荐值，影响调度和稳定性。

### 8.3 谨慎使用 Auto 模式

VPA Auto 可能驱逐 Pod。对于下面这些服务要特别谨慎：

- 单副本服务
- 有状态服务
- 长连接服务
- 启动很慢的服务
- 对延迟敏感的核心服务

如果确实要使用 Auto，建议同时配置：

- 多副本
- PodDisruptionBudget
- readinessProbe
- 合理的滚动发布策略

### 8.4 VPA 更适合做资源治理

VPA 在生产中常见的价值不是“自动改资源”，而是“发现资源配置不合理”。

例如：

| 现象 | VPA 能提供的帮助 |
|---|---|
| requests 配太大 | 给出更低的推荐值，节省资源 |
| requests 配太小 | 给出更高的推荐值，减少 CPU throttling 或 OOM |
| 不知道服务该配多少资源 | 根据历史使用量给出初始参考 |

---

## 9. 常见问题

### Q1：HPA 为什么不扩容

常见原因：

1. 没有安装 Metrics Server。
2. Pod 没有配置 `resources.requests.cpu`。
3. 当前指标没有超过目标阈值。
4. 已经达到 `maxReplicas`。
5. HPA 指标获取失败。

排查命令：

```bash
kubectl get hpa
kubectl describe hpa <hpa-name>
kubectl top pods
kubectl top nodes
```

如果 `kubectl top pods` 没有数据，通常说明 Metrics Server 没有正常工作。

### Q2：HPA 为什么频繁扩容缩容

常见原因：

- 目标阈值设置过低
- 服务负载本身波动大
- 缩容稳定窗口太短
- Pod 启动慢，指标延迟明显

优化方式：

- 增大 `scaleDown.stabilizationWindowSeconds`
- 提高目标利用率阈值
- 使用业务指标代替 CPU 指标
- 设置合理的 `minReplicas`

### Q3：VPA 为什么会重启 Pod

因为 Pod 的 CPU / 内存 requests 和 limits 通常不能在运行中直接修改。

VPA 如果要让新资源配置生效，就需要：

```text
驱逐旧 Pod -> 重新创建 Pod -> 注入新的资源配置
```

这就是为什么生产环境要谨慎使用 VPA Auto。

### Q4：HPA 和 Cluster Autoscaler 有什么区别

HPA 调整的是 Pod 数量：

```text
Deployment replicas: 2 -> 10
```

Cluster Autoscaler 调整的是 Node 数量：

```text
Node: 3 -> 6
```

两者通常配合使用：

```text
HPA 创建更多 Pod
  -> 现有 Node 资源不够
  -> 部分 Pod Pending
  -> Cluster Autoscaler 增加 Node
  -> Pending Pod 被调度
```

---

## 10. 选型建议

### 10.1 优先使用 HPA 的场景

- 无状态 Web 服务
- API 服务
- Worker / Consumer 服务
- 可以通过增加副本提升吞吐的服务
- 对高可用要求较高的在线服务

### 10.2 优先使用 VPA 的场景

- 资源配置不确定，需要推荐值
- 服务副本数不适合频繁变化
- 希望降低 requests 过高造成的资源浪费
- 离线任务、批处理任务、内部工具服务

### 10.3 不建议自动扩缩容的场景

- 数据库
- 分布式存储
- 单副本核心服务
- 强状态服务
- 扩缩容会触发复杂业务副作用的服务

---

## 11. 实战推荐配置

### 11.1 通用 Web 服务

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
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

### 11.2 VPA 只做资源建议

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: web-api
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "2Gi"
```

---

## 12. 总结

HPA 和 VPA 的核心区别可以概括为：

```text
HPA 解决“要不要更多 Pod”的问题。
VPA 解决“每个 Pod 应该给多少资源”的问题。
```

生产环境中更常见的组合是：

```text
HPA：负责在线服务的自动扩缩容
VPA：负责资源推荐和容量治理
```

如果是普通无状态服务，优先考虑 HPA；如果是资源配置长期不准，先用 VPA 的 `Off` 模式观察推荐值。两者都不要盲目开启自动模式，尤其要注意 CPU / 内存指标之间的互相影响。

最终目标不是“能自动扩缩容”，而是让服务在流量变化时保持稳定，同时避免长期浪费集群资源。
