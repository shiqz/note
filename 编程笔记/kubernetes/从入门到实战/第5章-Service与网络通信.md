# 第5章：Service 与网络通信

## 学习目标

完成本章学习后，你将能够：

1. 理解 Service 的核心作用：为什么 Pod 之上还需要 Service
2. 掌握 Service 的四种类型及其适用场景
3. 编写完整的 Service YAML 配置文件
4. 理解 Service 与 Pod 的连接机制（Label Selector + Endpoints）
5. 配置 Service 的会话亲和性
6. 创建无头服务（Headless Service）并理解其在 StatefulSet 中的应用
7. 了解 K8s 网络模型和 kube-proxy 工作模式
8. 通过实战掌握 Service 的创建、调试和使用

---

## 5.1 Service 核心概念

### 5.1.1 为什么需要 Service

在 K8s 中，Pod 是临时的、会变化的：

- **Pod IP 会变**：Pod 随时可能被销毁和重建，每次重建都会获得新的 IP 地址
- **Pod 数量会变**：Deployment 可以动态扩缩容，Pod 副本数量不固定
- **Pod 本身不稳定**：Pod 是易失性资源，不适合直接作为访问入口

```
传统方式的问题：
客户端 → Pod IP (10.244.1.5)  ← Pod 挂了就访问不到了！

Service 提供的解决方案：
客户端 → Service IP (10.96.0.100) → 后端 Pods  ← 稳定可靠
```

**Service 的核心作用**：为一组 Pod 提供统一的、稳定的访问入口。Service IP 不会改变，后端的 Pod 可以任意变化。

### 5.1.2 Service 的工作原理

```
                    ┌─────────────────────┐
                    │    Service (VIP)    │
                    │   10.96.100.100:80  │
                    └──────────┬──────────┘
                               │ 负载均衡
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
        │   Pod-1     │ │   Pod-2     │ │   Pod-3     │
        │ 10.244.1.5  │ │ 10.244.2.7  │ │ 10.244.3.9  │
        └─────────────┘ └─────────────┘ └─────────────┘
            (Running)      (Running)      (Running)
```

- 客户端请求到达 Service 的虚拟 IP（VIP）
- kube-proxy 将流量转发到后端健康的 Pod
- 当 Pod 变化时，Service 自动更新后端列表

---

## 5.2 Service 的四种类型

### 5.2.1 ClusterIP（默认类型）

**特点**：只能在集群内部访问，K8s 会分配一个集群内部的虚拟 IP。

**适用场景**：
- 微服务之间的互相调用
- 数据库、缓存等内部服务
- 不需要外部直接访问的服务

**YAML 示例**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-clusterip
  namespace: default
  labels:
    app: my-app
spec:
  type: ClusterIP                    # 可以省略，默认就是 ClusterIP
  selector:
    app: my-app                      # 选择带有 app=my-app 标签的 Pod
  ports:
    - name: http
      protocol: TCP
      port: 80                       # Service 暴露的端口
      targetPort: 8080               # 后端 Pod 的端口
```

### 5.2.2 NodePort

**特点**：在 ClusterIP 的基础上，在每个节点上开放一个端口（默认 30000-32767），外部可通过 `NodeIP:NodePort` 访问。

**适用场景**：
- 开发测试环境快速暴露服务
- 单节点临时访问
- 配合外部负载均衡器使用

**YAML 示例**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
  labels:
    app: my-app
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080                # 可选，不指定则自动分配 30000-32767
```

**访问方式**：`http://<NodeIP>:30080`

### 5.2.3 LoadBalancer

**特点**：在 NodePort 的基础上，向云服务商申请一个外部负载均衡器，将流量转发到集群节点。

**适用场景**：
- 生产环境对外暴露服务
- 云平台（AWS、GCP、Azure）环境
- 需要固定的外部访问 IP

**YAML 示例**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-loadbalancer
  labels:
    app: my-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"    # AWS NLB
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
```

**注意**：LoadBalancer 依赖云服务商的支持，裸金属环境需要额外配置（如 MetalLB）。

### 5.2.4 ExternalName

**特点**：将 Service 映射到一个外部域名，不分配集群 IP。

**适用场景**：
- 引用外部数据库
- 跨命名空间引用服务
- 渐进式迁移

**YAML 示例**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  labels:
    app: my-app
spec:
  type: ExternalName
  externalName: database.external-service.com    # 外部服务的 DNS 名称
```

四种类型对比：

| 类型 | 访问范围 | IP 分配 | 使用场景 |
|------|---------|---------|---------|
| ClusterIP | 集群内部 | 集群虚拟 IP | 内部服务间通信 |
| NodePort | 外部（节点IP） | 节点端口 | 开发测试 |
| LoadBalancer | 外部（LB IP） | 云 LB 地址 | 生产环境 |
| ExternalName | DNS 映射 | 无 IP | 引用外部服务 |

---

## 5.3 Service YAML 结构详解

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
  labels:
    app: web
    tier: frontend
  annotations:
    description: "Web frontend service"
spec:
  # Service 类型
  type: ClusterIP    # ClusterIP | NodePort | LoadBalancer | ExternalName

  # 标签选择器：决定哪些 Pod 作为后端
  selector:
    app: web
    tier: frontend

  # 端口配置
  ports:
    - name: http              # 端口名称，便于识别
      protocol: TCP           # 协议：TCP / UDP / SCTP
      port: 80                # Service 对外暴露的端口
      targetPort: 8080        # 后端 Pod 的端口
      nodePort: 30080         # 仅 NodePort / LoadBalancer 类型使用

    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443

  # 会话亲和性
  sessionAffinity: None       # None | ClientIP
  sessionAffinityConfig:      # 可选配置
    clientIP:
      timeoutSeconds: 10800   # 会话保持超时时间（秒）

  # 不指定 selector 时使用
  # endpoints:
  #   - addresses:
  #       - ip: 10.0.0.1
  #     ports:
  #       - port: 8080

  # 发布配置（仅 LoadBalancer）
  # externalTrafficPolicy: Cluster   # Cluster | Local
  # healthCheckNodePort: 30000
```

---

## 5.4 Service 与 Pod 的连接机制

### 5.4.1 Label Selector 选择后端 Pod

Service 通过 **Label Selector** 来匹配后端 Pod：

```yaml
# Service 定义
spec:
  selector:
    app: web          # 选择 app=web 的 Pod
    tier: frontend    # 且 tier=frontend 的 Pod
```

**匹配规则**：Pod 必须同时满足所有 selector 条件才会被选中。

```
Service Selector: {app: web, tier: frontend}

Pod-A: labels: {app: web, tier: frontend}    ✅ 匹配
Pod-B: labels: {app: web, tier: backend}     ❌ 不匹配（tier 不同）
Pod-C: labels: {app: web, tier: frontend}    ✅ 匹配
Pod-D: labels: {app: api, tier: frontend}    ❌ 不匹配（app 不同）
```

### 5.4.2 Endpoints 对象

Endpoints 是 K8s 自动维护的对象，记录了 Service 后端的 Pod IP 和端口列表。

```bash
# 查看 Service 的 Endpoints
kubectl get endpoints web-service

# 查看详情
kubectl describe endpoints web-service
```

**示例输出**：

```
NAME            ENDPOINTS                          AGE
web-service     10.244.1.5:8080,10.244.2.7:8080    5m
```

**工作流程**：

```
1. Service Controller 监控 Service 和 Pod 的变化
2. 当 Pod 标签匹配 Service Selector 时，将 Pod IP 加入 Endpoints
3. 当 Pod 不健康或标签不匹配时，从 Endpoints 中移除
4. kube-proxy 根据 Endpoints 更新 iptables/ipvs 规则
```

**无 Selector 的 Service**：如果 Service 不指定 selector，需要手动维护 Endpoints。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: manual-service
spec:
  type: ClusterIP
  # 没有 selector
---
apiVersion: v1
kind: Endpoints
metadata:
  name: manual-service
subsets:
  - addresses:
      - ip: 10.0.0.1
      - ip: 10.0.0.2
    ports:
      - port: 8080
        protocol: TCP
```

### 5.4.3 EndpointSlice

EndpointSlice 是 Endpoints 的下一代替代品，具有更好的扩展性：

```bash
# 查看 EndpointSlice
kubectl get endpointslices
kubectl get endpointslices -l kubernetes.io/service-name=web-service
```

**优势**：
- 分片存储，解决大规模 Endpoints 性能问题
- 支持更丰富的信息（拓扑、条件等）
- 增量更新，减少网络传输

---

## 5.5 Service 的会话亲和性

### 5.5.1 Session Affinity 概述

默认情况下，Service 的负载均衡是随机的或轮询的。会话亲和性可以让来自同一客户端的请求始终路由到同一个 Pod。

### 5.5.2 None（默认）

```yaml
spec:
  sessionAffinity: None
```

- 无会话保持
- 每次请求可能路由到不同 Pod
- 适用于无状态服务

### 5.5.3 ClientIP

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800    # 3小时会话保持
```

- 同一客户端 IP 的请求始终路由到同一 Pod
- 适用于需要会话保持的服务（如购物车、登录状态）
- 缺点：IP 哈希可能导致负载不均衡

### 5.5.4 对比

| 特性 | None | ClientIP |
|------|------|----------|
| 负载均衡 | 随机/轮询 | 基于客户端 IP |
| 会话保持 | 无 | 同 IP 同后端 |
| 适用场景 | 无状态服务 | 有状态服务 |
| 扩展性 | 好 | 一般 |

---

## 5.6 无头服务（Headless Service）

### 5.6.1 什么是无头服务

无头服务是一种特殊的 Service，`clusterIP: None`，不分配虚拟 IP，直接返回后端 Pod 的 IP 列表。

### 5.6.2 核心作用

- **稳定的 Pod 标识**：每个 Pod 都有稳定的 DNS 名称
- **独立访问每个 Pod**：客户端可以直接访问特定 Pod
- **服务发现**：用于 StatefulSet 等需要稳定网络标识的场景

### 5.6.3 DNS 解析规则

无头服务为每个 Pod 提供 DNS 记录：

```
# 普通 Service
web-service.default.svc.cluster.local → 10.96.100.100（ClusterIP）

# 无头 Service
web-service.default.svc.cluster.local → 10.244.1.5, 10.244.2.7, 10.244.3.9（所有 Pod IP）

# 每个 Pod 的稳定 DNS
web-0.web-service.default.svc.cluster.local → 10.244.1.5
web-1.web-service.default.svc.cluster.local → 10.244.2.7
web-2.web-service.default.svc.cluster.local → 10.244.3.9
```

### 5.6.4 YAML 示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  labels:
    app: redis
spec:
  clusterIP: None              # 关键：设置为 None
  publishNotReadyAddresses: true    # 发布未就绪的 Pod（集群初始化时需要）
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
      name: client
    - port: 16379
      targetPort: 16379
      name: gossip
```

### 5.6.5 StatefulSet 中的使用

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-service    # 关联的无头服务名称
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
```

**StatefulSet 提供的稳定标识**：

```
Pod 名称：web-0, web-1, web-2（有序、稳定）
DNS 名称：web-0.web-service, web-1.web-service, web-2.web-service
网络标识：每个 Pod 有固定的 DNS 记录
存储标识：每个 Pod 绑定独立的 PVC
```

---

## 5.7 K8s 网络模型

### 5.7.1 CNI 插件

CNI（Container Network Interface）是 K8s 的网络插件标准，用于配置 Pod 网络。

**常见 CNI 插件**：

| 插件 | 类型 | 特点 |
|------|------|------|
| Flannel | Overlay | 简单、稳定，适合入门 |
| Calico | Overlay/Underlay | 支持网络策略，生产首选 |
| Cilium | eBPF | 高性能，可观测性好 |
| Weave | Overlay | 简单易用 |
| Kube-OVN | Underlay | 企业级网络方案 |

### 5.7.2 Overlay vs Underlay 网络

**Overlay 网络**：

```
┌─────────────────────────────────────────┐
│              Node A                      │
│  ┌───────────┐  ┌───────────┐            │
│  │  Pod-1    │  │  Pod-2    │            │
│  │ 10.244.1.1│  │ 10.244.1.2│            │
│  └─────┬─────┘  └─────┬─────┘            │
│        │              │                  │
│  ┌─────┴──────────────┴─────┐            │
│  │    vxlan / flannel 网络   │            │
│  │    10.244.0.0/16         │            │
│  └──────────┬───────────────┘            │
│             │                            │
│        ┌─────┴─────┐                      │
│        │ eth0      │ 192.168.1.10        │
│        └─────┬─────┘                      │
└──────────────┼────────────────────────────┘
               │ 物理网络
┌──────────────┼────────────────────────────┐
│              │     Node B                 │
│        ┌─────┴─────┐                      │
│        │ eth0      │ 192.168.1.11        │
│        └─────┬─────┘                      │
│  ┌──────────┬───────────────┐            │
│  │    flannel 网络           │            │
│  └─────┬─────┴─────┬────────┘            │
│        │           │                     │
│  ┌─────┴─────┐ ┌───┴───────┐             │
│  │  Pod-3    │ │  Pod-4    │             │
│  │ 10.244.2.1│ │ 10.244.2.2│             │
│  └───────────┘ └───────────┘             │
└─────────────────────────────────────────┘
```

- Pod 流量封装在额外的包头中（VXLAN、UDP 等）
- 跨节点通信需要解封装
- 优点：简单，不需要修改物理网络
- 缺点：性能开销

**Underlay 网络**：

- Pod IP 直接在物理网络中路由
- 无需封装，性能好
- 优点：高性能、低延迟
- 缺点：需要物理网络支持，路由配置复杂

### 5.7.3 K8s 网络通信流程

```
Pod A (10.244.1.5)                    Pod B (10.244.2.7)
     │                                    │
     │ 1. 应用发起请求                     │
     ▼                                    │
┌───────────┐                             │
│ kube-proxy │                             │
│ (iptables) │                             │
└─────┬─────┘                             │
      │ 2. DNAT: ServiceIP → PodB_IP      │
      ▼                                    │
┌───────────┐    3. CNI 转发              │
│  CNI 插件  │ ──────────────────────────► │
└───────────┘                             │
      │                                    │
      │                                    ▼
      │                              ┌───────────┐
      │                              │  Pod B    │
      │                              └───────────┘
```

---

## 5.8 kube-proxy 工作模式

kube-proxy 是运行在每个节点上的代理组件，负责将 Service 请求转发到后端 Pod。

### 5.8.1 iptables 模式（默认）

**工作原理**：

```
Service VIP:Port → iptables NAT 规则 → Pod IP:Port
```

- 使用 iptables DNAT 规则实现端口转发
- 每个 Service 创建多条 iptables 规则
- 当 Service/Pod 变化时，增量更新 iptables 规则

**优缺点**：

| 优点 | 缺点 |
|------|------|
| 无需额外内核模块 | 大量 Service 时 iptables 规则多 |
| 稳定可靠 | 更新规则较慢 |
| 调试简单 | 性能随规模下降 |

**查看规则**：

```bash
# 查看某个 Service 的 iptables 规则
iptables-save | grep web-service

# 查看所有 K8s 相关规则
iptables-save | grep KUBE-SERVICES
```

### 5.8.2 ipvs 模式（高性能）

**工作原理**：

```
Service VIP:Port → IPVS 负载均衡 → Pod IP:Port
```

- 使用内核级 IPVS（IP Virtual Server）实现负载均衡
- 支持多种调度算法：rr、wrr、lc、wlc、ip_hash
- 适用于大规模集群

**优缺点**：

| 优点 | 缺点 |
|------|------|
| 高性能（内核态） | 需要内核模块支持 |
| 支持复杂调度算法 | 调试相对复杂 |
| 适合大规模集群 | 配置稍复杂 |

**调度算法**：

| 算法 | 说明 |
|------|------|
| rr（Round Robin） | 轮询调度，默认 |
| wrr（Weighted Round Robin） | 加权轮询 |
| lc（Least Connection） | 最少连接数 |
| wlc（Weighted Least Connection） | 加权最少连接 |
| ip_hash | 基于源 IP 哈希 |

### 5.8.3 userspace 模式

**工作原理**：

```
Service VIP:Port → userspace proxy → Pod IP:Port
```

- 在用户空间实现代理
- 性能最差，已基本淘汰
- 仅用于特殊场景或调试

### 5.8.4 切换 kube-proxy 模式

```bash
# 查看当前模式
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# 切换为 ipvs 模式（修改 configmap）
kubectl patch configmap kube-proxy -n kube-system --patch '{"data":{"config.conf":"mode: ipvs"}}'

# 重启 kube-proxy
kubectl rollout restart daemonset kube-proxy -n kube-system

# 验证模式
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep "Using ipvs"
```

### 5.8.5 三种模式对比

| 特性 | iptables | ipvs | userspace |
|------|----------|------|-----------|
| 性能 | 中 | 高 | 低 |
| 内核支持 | iptables | IPVS | 无 |
| 规模扩展性 | 一般 | 好 | 差 |
| 调度算法 | 随机/轮询 | 多种 | 轮询 |
| 适用场景 | 中小规模 | 大规模 | 调试/特殊 |

---

## 5.9 实战演练

### 实战1：创建 ClusterIP Service 暴露 Nginx Pod

**步骤 1**：创建 Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -l app=nginx
```

**步骤 2**：创建 ClusterIP Service

```yaml
# nginx-service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

```bash
kubectl apply -f nginx-service-clusterip.yaml
kubectl get svc nginx-clusterip
```

**步骤 3**：在集群内部访问

```bash
# 启动一个临时 Pod 用于测试
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- sh

# 在 Pod 内部执行
curl http://nginx-clusterip
curl http://nginx-clusterip.default.svc.cluster.local

# 查看 DNS 解析
nslookup nginx-clusterip
```

### 实战2：创建 NodePort Service 实现外部访问

**步骤 1**：创建 NodePort Service

```yaml
# nginx-service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f nginx-service-nodeport.yaml
kubectl get svc nginx-nodeport
```

**步骤 2**：获取节点 IP 并访问

```bash
# 获取节点 IP
kubectl get nodes -o wide

# 外部访问（假设节点 IP 为 192.168.1.100）
curl http://192.168.1.100:30080
```

**步骤 3**：使用 Minikube 快速访问

```bash
# Minikube 环境
minikube service nginx-nodeport
```

### 实战3：观察 Service 的 Endpoints 随 Pod 变化

**步骤 1**：查看初始 Endpoints

```bash
kubectl get endpoints nginx-clusterip -o wide
kubectl describe endpoints nginx-clusterip
```

**步骤 2**：扩展 Deployment 副本数

```bash
# 扩展到 5 个副本
kubectl scale deployment nginx-deployment --replicas=5

# 等待新 Pod 启动
kubectl get pods -l app=nginx -w

# 再次查看 Endpoints（应该增加了 2 个）
kubectl get endpoints nginx-clusterip
```

**步骤 3**：减少副本数

```bash
# 减少到 1 个副本
kubectl scale deployment nginx-deployment --replicas=1

# 查看 Endpoints（应该只剩 1 个）
kubectl get endpoints nginx-clusterip
```

**步骤 4**：查看 EndpointSlice

```bash
kubectl get endpointslices -l kubernetes.io/service-name=nginx-clusterip
```

### 实战4：实现 Service 之间的互相访问

**步骤 1**：创建后端服务

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.25
          ports:
            - containerPort: 80
          env:
            - name: SERVER_NAME
              value: "Backend Service"
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
          lifecycle:
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - 'echo "<h1>Backend Service Response</h1>" > /usr/share/nginx/html/index.html'
      volumes:
        - name: html
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f backend-deployment.yaml
```

**步骤 2**：在前端 Pod 中访问后端 Service

```bash
# 进入 nginx Pod
kubectl exec -it deploy/nginx-deployment -- sh

# 在 Pod 内部 curl 后端服务
curl http://backend-service
curl http://backend-service.default.svc.cluster.local

# 使用环境变量
env | grep BACKEND
```

**步骤 3**：使用 Service 环境变量

```bash
# 新创建的 Pod 会自动注入 Service 环境变量
kubectl exec -it deploy/backend-deployment -- env | grep BACKEND_SERVICE
```

**注意**：环境变量方式需要 Service 在 Pod 创建之前存在。推荐使用 DNS 解析方式。

### 实战5：创建 Headless Service

**步骤 1**：创建无头服务

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-nginx
  labels:
    app: headless-nginx
spec:
  clusterIP: None
  selector:
    app: headless-nginx
  ports:
    - port: 80
      targetPort: 80
```

**步骤 2**：创建 StatefulSet 使用无头服务

```yaml
# statefulset-nginx.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: headless-nginx
  replicas: 3
  selector:
    matchLabels:
      app: headless-nginx
  template:
    metadata:
      labels:
        app: headless-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f headless-service.yaml
kubectl apply -f statefulset-nginx.yaml

# 查看 Pod（注意有序编号 web-0, web-1, web-2）
kubectl get pods -l app=headless-nginx
```

**步骤 3**：验证 DNS 解析

```bash
# 启动测试 Pod
kubectl run dns-test --image=curlimages/curl --rm -it --restart=Never -- sh

# 解析无头服务（应返回所有 Pod IP）
nslookup headless-nginx

# 解析单个 Pod
nslookup web-0.headless-nginx
nslookup web-1.headless-nginx

# 直接访问特定 Pod
curl http://web-0.headless-nginx
curl http://web-1.headless-nginx
```

---

## 5.10 常见问题排查

### 5.10.1 Service 无法访问 Pod

**问题**：创建了 Service 但访问不到后端 Pod。

**排查步骤**：

```bash
# 1. 检查 Service 的 selector 是否正确
kubectl get svc <service-name> -o yaml | grep -A 5 selector

# 2. 检查 Pod 标签是否匹配
kubectl get pods -l <selector-label> --show-labels

# 3. 检查 Endpoints 是否为空
kubectl get endpoints <service-name>
# 如果 ENDPOINTS 为空，说明 selector 没有匹配到任何 Pod

# 4. 检查 Pod 是否健康
kubectl get pods -l <selector-label>
# STATUS 应为 Running，READY 应为 1/1

# 5. 检查端口是否正确
kubectl get svc <service-name> -o yaml | grep -A 5 ports
# targetPort 应与 Pod 的 containerPort 一致

# 6. 手动验证 Pod 服务是否正常
kubectl port-forward <pod-name> 8080:80
curl http://localhost:8080
```

### 5.10.2 NodePort 无法外部访问

**问题**：NodePort Service 创建后外部无法访问。

**排查步骤**：

```bash
# 1. 检查 NodePort 是否正确分配
kubectl get svc <service-name>
# 查看 PORT(S) 列，确认 NodePort 值

# 2. 检查节点防火墙
# 在节点上开放对应端口
sudo iptables -A INPUT -p tcp --dport 30080 -j ACCEPT

# 3. 检查节点 IP 是否可达
ping <node-ip>

# 4. 使用 kubectl 端口转发测试
kubectl port-forward svc/<service-name> 8080:80
curl http://localhost:8080
```

### 5.10.3 DNS 解析失败

**问题**：在 Pod 内无法解析 Service 名称。

**排查步骤**：

```bash
# 1. 检查 CoreDNS 是否正常
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. 检查 Service 是否存在
kubectl get svc <service-name>

# 3. 在 Pod 内检查 DNS 配置
kubectl exec -it <pod-name> -- cat /etc/resolv.conf

# 4. 测试 DNS 解析
kubectl run dns-test --image=curlimages/curl --rm -it --restart=Never -- nslookup <service-name>

# 5. 检查 CoreDNS 日志
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 5.10.4 Service 负载不均衡

**问题**：流量集中到少数 Pod。

**排查步骤**：

```bash
# 1. 检查会话亲和性设置
kubectl get svc <service-name> -o yaml | grep sessionAffinity

# 2. 如果使用 ClientIP，考虑切换为 None
kubectl patch svc <service-name> -p '{"spec":{"sessionAffinity":"None"}}'

# 3. 检查 Pod 健康状态
kubectl get pods -l <label> -o wide

# 4. 使用 ipvs 模式获得更好的负载均衡
```

### 5.10.5 Headless Service 不返回 Pod IP

**问题**：无头服务 DNS 解析不返回 Pod IP 列表。

**排查步骤**：

```bash
# 1. 确认 clusterIP 设置为 None
kubectl get svc <service-name> -o yaml | grep clusterIP

# 2. 检查 Pod 是否就绪
kubectl get pods -l <label>

# 3. 如果使用 StatefulSet，检查 serviceName 配置
kubectl get statefulset <name> -o yaml | grep serviceName

# 4. 检查 publishNotReadyAddresses 设置
# 集群初始化时可能需要设置为 true
```

---

## 5.11 章节小结

### 核心知识点回顾

1. **Service 的本质**：为 Pod 提供稳定的访问入口，解决 Pod IP 易变的问题
2. **四种类型**：
   - ClusterIP：默认类型，集群内部访问
   - NodePort：在节点上开放端口，外部可访问
   - LoadBalancer：云服务商提供外部负载均衡
   - ExternalName：映射到外部 DNS
3. **连接机制**：Label Selector 匹配 Pod → Endpoints 维护后端列表 → kube-proxy 实现转发
4. **会话亲和性**：None（无状态）vs ClientIP（有状态服务）
5. **无头服务**：`clusterIP: None`，用于 StatefulSet 等需要稳定网络标识的场景
6. **kube-proxy 模式**：iptables（默认）→ ipvs（高性能）→ userspace（已淘汰）

### 常用命令速查

```bash
# 创建 Service
kubectl expose deployment <name> --type=ClusterIP --port=80 --target-port=8080

# 查看 Service
kubectl get svc
kubectl get svc <name> -o yaml

# 查看 Endpoints
kubectl get endpoints <service-name>
kubectl describe endpoints <service-name>

# 查看 EndpointSlice
kubectl get endpointslices -l kubernetes.io/service-name=<service-name>

# 删除 Service
kubectl delete svc <name>

# 端口转发调试
kubectl port-forward svc/<service-name> 8080:80

# Minikube 快速访问
minikube service <service-name>

# 修改 Service 类型
kubectl patch svc <name> -p '{"spec":{"type":"NodePort"}}'

# 设置会话亲和性
kubectl patch svc <name> -p '{"spec":{"sessionAffinity":"ClientIP"}}'
```

### 最佳实践

1. **优先使用 DNS**：使用 Service DNS 名称而非环境变量
2. **合理选择类型**：内部服务用 ClusterIP，对外暴露用 LoadBalancer
3. **使用标签管理**：通过 label selector 灵活控制后端
4. **监控 Endpoints**：定期检查 Endpoints 确保后端 Pod 正常
5. **大规模场景使用 ipvs**：当 Service 数量超过 1000 时考虑切换 ipvs 模式
6. **StatefulSet 配合无头服务**：有状态应用必须使用 Headless Service
