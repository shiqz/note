# 第10章：Ingress 与流量路由

## 学习目标

完成本章学习后，你将能够：

1. 理解 Ingress 的核心作用：统一管理集群内 HTTP(S) 流量的入口
2. 了解主流 Ingress Controller（Nginx、Traefik、HAProxy、Kong）的优缺点
3. 部署 Nginx Ingress Controller 并配置基本路由规则
4. 编写 Ingress YAML 实现路径路由和主机路由
5. 配置 TLS 终止，使用 cert-manager 自动签发证书
6. 实现路径重写、默认后端等高级功能
7. 通过 Ingress 实现蓝绿部署流量切换

---

## 10.1 Ingress 核心概念

### 10.1.1 为什么需要 Ingress

在 K8s 中，暴露服务给外部访问有多种方式：

| 方式 | 特点 | 局限性 |
|------|------|--------|
| `ClusterIP` | 集群内部访问 | 外部无法访问 |
| `NodePort` | 通过节点端口访问 | 端口范围有限（30000-32767），每个服务需分配一个端口 |
| `LoadBalancer` | 云厂商提供负载均衡 | 每个服务需要独立的 LB，成本高 |
| **Ingress** | HTTP(S) 路由 | 单个入口管理多个服务，支持路径/主机路由 |

```
没有 Ingress 的情况：
    客户端 ──→ NodePort:30080 (web)
    客户端 ──→ NodePort:30081 (api)
    客户端 ──→ NodePort:30082 (admin)
    问题：端口分散、难以统一管理、无法实现 TLS 终止

使用 Ingress 的情况：
    客户端 ──→ 80/443 (Ingress)
                  ├── /web/*   → web-service
                  ├── /api/*   → api-service
                  └── /admin/* → admin-service
    优势：统一入口、TLS 终止、灵活路由、低成本
```

**Ingress 的核心作用**：
1. **统一入口**：所有外部流量通过一个入口进入集群
2. **HTTP 路由**：基于路径或主机名将流量转发到不同服务
3. **TLS 终止**：在 Ingress 层处理 HTTPS，后端服务只需 HTTP
4. **灵活配置**：支持路径重写、默认后端、多种路由规则

### 10.1.2 Ingress 与 Service 的关系

```
                    外部客户端
                        │
                        ▼
                ┌─────────────┐
                │   Ingress   │  ← 流量入口，HTTP(S) 路由
                │  Controller │     运行在 Pod 中，由 Service 暴露
                └──────┬──────┘
                       │
          根据 Ingress 规则路由
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Service  │ │ Service  │ │ Service  │  ← 集群内部服务
    │  (api)   │ │  (web)   │ │ (admin)  │     ClusterIP 类型
    └────┬─────┘ └────┬─────┘ └────┬─────┘
         │            │            │
         ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Pod      │ │ Pod      │ │ Pod      │  ← 后端应用实例
    │ Pod      │ │ Pod      │ │ Pod      │
    └──────────┘ └──────────┘ └──────────┘
```

**数据流**：Ingress Controller 根据 Ingress 规则 → 找到对应 Service → 通过 Service 的 Endpoints → 转发到后端 Pod。

---

## 10.2 Ingress Controller 概述

### 10.2.1 主流 Ingress Controller 对比

| Controller | 特点 | 适用场景 |
|------------|------|----------|
| **Nginx Ingress** | 基于 Nginx，社区活跃，功能全面，配置简单 | 通用场景，主流选择 |
| **Traefik** | 云原生，动态配置，内置 Dashboard，支持自动发现 | 微服务架构，需要灵活配置 |
| **HAProxy** | 高性能负载均衡，配置复杂 | 高性能需求，传统架构 |
| **Kong** | API 网关，丰富的插件生态（限流、认证、日志） | API 管理，需要网关功能 |
| **Istio Gateway** | Service Mesh 集成，HTTP/HTTPS/gRPC 路由 | 服务网格架构 |

### 10.2.2 Nginx Ingress vs Traefik 选择

| 维度 | Nginx Ingress | Traefik |
|------|---------------|---------|
| 语言 | Go | Go |
| 性能 | 高（基于 Nginx） | 高 |
| 配置方式 | 主要通过 Ingress YAML | Ingress YAML + CRD |
| Dashboard | 有 | 有（更丰富） |
| 热更新 | 重新加载 Nginx 配置 | 动态无需重启 |
| 插件生态 | 丰富 | 丰富 |
| 学习曲线 | 平缓 | 中等 |
| 社区支持 | 最活跃 | 活跃 |

**选择建议**：
- 新手/通用场景 → **Nginx Ingress**
- 需要动态配置/API 网关 → **Traefik** 或 **Kong**
- Service Mesh 架构 → **Istio Gateway**

---

## 10.3 部署 Nginx Ingress Controller

### 10.3.1 使用 YAML 部署

```bash
# Step 1: 创建命名空间
kubectl create namespace ingress-nginx

# Step 2: 下载官方 YAML（简化版）
# Controller Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/component: controller
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/component: controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/component: controller
    spec:
      serviceAccountName: ingress-nginx
      containers:
      - name: controller
        image: k8s.gcr.io/ingress-nginx/controller:v1.9.4
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-nginx-controller
        - --ingress-class=nginx
        - --controller-class=k8s.io/ingress-nginx
        - --validating-webhook=:8443/validate
        - --validating-webhook-certificate=/usr/local/certs/validating-webhook/ca.crt
        - --validating-webhook-key=/usr/local/certs/validating-webhook/ca.key
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          runAsUser: 101
          seccompProfile:
            type: RuntimeDefault
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: metrics
          containerPort: 10254
        - name: webhook
          containerPort: 8443
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
          initialDelaySeconds: 10
          periodSeconds: 10
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      nodeSelector:
        kubernetes.io/os: linux
EOF
```

### 10.3.2 使用 Helm 部署（推荐）

```bash
# Step 1: 添加 Helm 仓库
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Step 2: 安装（裸金属环境使用 NodePort）
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443

# 云环境使用 LoadBalancer
# helm install ingress-nginx ingress-nginx/ingress-nginx \
#   --namespace ingress-nginx \
#   --create-namespace

# Step 3: 查看部署状态
kubectl get pods -n ingress-nginx
# NAME                                                   READY   STATUS
# ingress-nginx-controller-5d4b7c6f4c-abc12              1/1     Running
# ingress-nginx-controller-5d4b7c6f4c-def34              1/1     Running

# Step 4: 查看 Service
kubectl get svc -n ingress-nginx
# NAME                                   TYPE       CLUSTER-IP       PORT(S)
# ingress-nginx-controller               NodePort   10.96.100.50     80:30080/TCP,443:30443/TCP
# ingress-nginx-controller-metrics        ClusterIP  10.96.200.100    10254/TCP

# Step 5: 测试访问
curl http://<node-ip>:30080
# 返回 404 表示 Controller 正常运行（因为还没有配置 Ingress 规则）
```

---

## 10.4 Ingress YAML 结构详解

### 10.4.1 基本结构

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    # 注解配置（Nginx Ingress 特有）
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx  # 指定 Ingress Class
  tls:  # TLS 配置（可选）
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com  # 主机名
    http:
      paths:
      - path: /api  # 路径
        pathType: Prefix  # 路径匹配类型
        backend:
          service:
            name: api-service  # 后端 Service 名
            port:
              number: 80  # Service 端口
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### 10.4.2 pathType 说明

| pathType | 说明 | 示例 |
|----------|------|------|
| `Prefix` | 前缀匹配，匹配以该路径开头的所有请求 | `/api` 匹配 `/api`、`/api/v1`、`/api/v1/users` |
| `Exact` | 精确匹配，只匹配完全相同的路径 | `/api` 只匹配 `/api`，不匹配 `/api/v1` |
| `ImplementationSpecific` | 由 Ingress Controller 自定义 | 取决于具体实现 |

**推荐使用 `Prefix`**，覆盖大多数场景。

### 10.4.3 注解（Annotations）

Nginx Ingress 支持丰富的注解配置：

```yaml
metadata:
  annotations:
    # 路径重写
    nginx.ingress.kubernetes.io/rewrite-target: /
    
    # SSL 重定向
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # 强制 HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # 超时设置
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    
    # 请求体大小限制
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    
    # 负载均衡策略
    nginx.ingress.kubernetes.io/load-balance: "least_conn"
    
    # 会话亲和性
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "balanced"
    
    # 限流
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "1000"
    
    # 认证（Basic Auth）
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
```

---

## 10.5 路径路由与主机路由

### 10.5.1 路径路由

根据 URL 路径将流量分发到不同后端服务。

```
客户端请求：
  https://example.com/api/users  → api-service
  https://example.com/web/home  → web-service
  https://example.com/admin/dash → admin-service
```

**YAML 配置**：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

### 10.5.2 主机路由

根据 HTTP Host Header 将流量分发到不同后端服务。

```
客户端请求：
  https://app.example.com/   → app-service
  https://api.example.com/   → api-service
  https://admin.example.com/ → admin-service
```

**YAML 配置**：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

### 10.5.3 混合路由

路径路由和主机路由可以混合使用：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mixed-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

---

## 10.6 TLS 终止配置

### 10.6.1 手动创建 TLS Secret

```bash
# Step 1: 获取或生成证书
# 方式1：使用已有证书
# 方式2：生成自签名证书（仅测试用）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout example.com.key \
  -out example.com.crt \
  -subj "/CN=example.com"

# Step 2: 创建 TLS Secret
kubectl create secret tls example-tls \
  --key=example.com.key \
  --cert=example.com.crt

# Step 3: 验证 Secret
kubectl get secret example-tls
# NAME          TYPE                DATA   AGE
# example-tls   kubernetes.io/tls   2      10s
```

### 10.6.2 Ingress 配置 TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### 10.6.3 使用 cert-manager 自动签发证书

cert-manager 可以自动向 Let's Encrypt 签发证书，并自动续期。

```bash
# Step 1: 安装 cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Step 2: 创建 ClusterIssuer（Let's Encrypt 自动签发）
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# 测试环境使用 staging：
# apiVersion: cert-manager.io/v1
# kind: ClusterIssuer
# metadata:
#   name: letsencrypt-staging
# spec:
#   acme:
#     server: https://acme-staging-v02.api.letsencrypt.org/directory
#     ...

# Step 3: Ingress 添加 cert-manager 注解
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cert-manager-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# Step 4: 验证证书签发
kubectl get certificates -n default
# NAME          READY   SECRET        AGE
# example-tls   True    example-tls   1m

# Step 5: 检查证书 Secret
kubectl get secret example-tls
# NAME          TYPE                DATA   AGE
# example-tls   kubernetes.io/tls   2      1m
```

---

## 10.7 路径重写与默认后端

### 10.7.1 路径重写

路径重写可以将请求路径修改后再转发到后端服务。

**场景**：
```
请求: https://example.com/api/users
重写: /api/users → /users
转发到: http://api-service:80/users
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**重写规则说明**：

| 注解值 | 说明 | 示例 |
|--------|------|------|
| `/` | 所有路径重写为 `/` | `/api/users` → `/` |
| `/$1` | 使用捕获组 | `/api/users` → `/users` |
| `/prefix/$1` | 添加前缀 | `/users` → `/prefix/users` |
| `http://example.com$1` | 重写为完整 URL | `/api` → `http://example.com/api` |

**高级重写示例**：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-rewrite-ingress
  annotations:
    # 将 /v1/* 重写为 /api/v1/*
    nginx.ingress.kubernetes.io/rewrite-target: /api/$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /v1/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: gateway-service
            port:
              number: 80
```

### 10.7.2 默认后端

当请求不匹配任何 Ingress 规则时，会转发到默认后端。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**默认后端的作用**：
- 匹配所有未被其他 Ingress 规则覆盖的请求
- 可以返回友好的 404 页面
- 可以作为健康检查的目标

**全局默认后端**：

```bash
# 使用 Helm 配置全局默认后端
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.defaultBackend.enabled=true \
  --set controller.defaultBackend.service=default-backend/default-backend-service
```

---

## 10.8 实战1：部署 Nginx Ingress Controller

### 完整步骤

```bash
# Step 1: 创建命名空间
kubectl create namespace ingress-nginx

# Step 2: 使用 Helm 安装
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443 \
  --set controller.replicaCount=2

# Step 3: 验证部署
kubectl get pods -n ingress-nginx
# NAME                   READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-xxx  1/1   Running   0          30s
# ingress-nginx-controller-yyy  1/1   Running   0          30s

# Step 4: 查看 Service
kubectl get svc -n ingress-nginx
# NAME                           TYPE       CLUSTER-IP      PORT(S)
# ingress-nginx-controller       NodePort   10.96.100.50    80:30080/TCP,443:30443/TCP

# Step 5: 测试访问
curl http://<node-ip>:30080
# 预期返回：404 Not Found（因为还没有配置任何 Ingress 规则）
# 这证明 Controller 已经正常运行

# Step 6: 查看 Ingress Class
kubectl get ingressclasses
# NAME    CONTROLLER                    PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx                        30s
```

---

## 10.9 实战2：创建路径路由 Ingress

### 步骤说明

创建两个后端服务（api-service 和 web-service），然后通过 Ingress 实现路径路由。

### 完整步骤

```bash
# Step 1: 创建 API 服务 Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
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
      containers:
      - name: api
        image: nginx:1.25
        ports:
        - containerPort: 80
        env:
        - name: SERVICE_NAME
          value: "API Service"
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo "<h1>API Service</h1>" > /usr/share/nginx/html/index.html
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
EOF

# Step 2: 创建 API Service
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 3: 创建 Web 服务 Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
spec:
  replicas: 2
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
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo "<h1>Web Service</h1>" > /usr/share/nginx/html/index.html
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
EOF

# Step 4: 创建 Web Service
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 5: 创建路径路由 Ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# Step 6: 验证 Ingress 状态
kubectl get ingress
# NAME                   CLASS   HOSTS         ADDRESS   PORTS   AGE
# path-routing-ingress   nginx   example.com             80      30s

# Step 7: 查看 Ingress 详情
kubectl describe ingress path-routing-ingress

# Step 8: 测试路径路由（在节点上执行或配置 DNS）
# 配置 hosts 或使用 curl --resolve
curl -H "Host: example.com" http://<node-ip>:30080/api
# <h1>API Service</h1>

curl -H "Host: example.com" http://<node-ip>:30080/web
# <h1>Web Service</h1>

curl -H "Host: example.com" http://<node-ip>:30080/other
# 404 Not Found

# Step 9: 清理
kubectl delete ingress path-routing-ingress
kubectl delete deployment api-deployment web-deployment
kubectl delete service api-service web-service
```

---

## 10.10 实战3：创建主机路由 Ingress

### 步骤说明

创建基于主机名的路由，不同域名指向不同服务。

### 完整步骤

```bash
# Step 1: 创建 App 服务
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo "<h1>App Service (app.example.com)</h1>" > /usr/share/nginx/html/index.html
EOF

# Step 2: 创建 App Service
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: app
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 3: 创建 Admin 服务
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-deployment
  labels:
    app: admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: nginx:1.25
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo "<h1>Admin Service (admin.example.com)</h1>" > /usr/share/nginx/html/index.html
EOF

# Step 4: 创建 Admin Service
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  selector:
    app: admin
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 5: 创建主机路由 Ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
EOF

# Step 6: 验证
kubectl get ingress host-routing-ingress

# Step 7: 测试（使用 --resolve 模拟 DNS）
curl --resolve app.example.com:30080:<node-ip> http://app.example.com:30080/
# <h1>App Service (app.example.com)</h1>

curl --resolve admin.example.com:30080:<node-ip> http://admin.example.com:30080/
# <h1>Admin Service (admin.example.com)</h1>

# Step 8: 清理
kubectl delete ingress host-routing-ingress
kubectl delete deployment app-deployment admin-deployment
kubectl delete service app-service admin-service
```

---

## 10.11 实战4：配置 TLS 证书

### 步骤说明

为 Ingress 配置 HTTPS，支持自签名证书（测试）和 cert-manager 自动签发。

### 完整步骤

```bash
# ==================== 方式1：自签名证书（测试用）====================

# Step 1: 创建自签名证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/example.com.key \
  -out /tmp/example.com.crt \
  -subj "/CN=example.com/O=Test/C=CN" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com"

# Step 2: 创建 TLS Secret
kubectl create secret tls example-tls \
  --key=/tmp/example.com.key \
  --cert=/tmp/example.com.crt

# Step 3: 创建带 TLS 的 Ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# Step 4: 测试 HTTPS
curl -k --resolve example.com:30443:<node-ip> https://example.com:30443/
# 使用 -k 跳过证书验证（因为是自签名）

# ==================== 方式2：cert-manager 自动签发（生产用）====================

# Step 1: 安装 cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Step 2: 等待 cert-manager 就绪
kubectl wait --for=condition=Available deployment/cert-manager \
  -n cert-manager --timeout=120s

# Step 3: 创建 ClusterIssuer（Staging 测试）
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Step 4: 创建 Ingress 并添加 cert-manager 注解
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cert-manager-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# Step 5: 验证证书签发
kubectl get certificates
kubectl get challenges

# Step 6: 检查 Certificate 状态
kubectl describe certificate example-tls
# Status:
#   Conditions:
#   - Last Transition Time:  ...
#     Status: True
#     Type: Ready

# 清理
kubectl delete ingress tls-ingress cert-manager-ingress
kubectl delete secret example-tls
```

---

## 10.12 实战5：实现路径重写规则

### 步骤说明

实现路径重写，将请求路径转换为后端服务期望的路径。

### 完整步骤

```bash
# Step 1: 创建后端服务
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-web-content
data:
  index.html: |
    <h1>API Backend</h1>
    <p>Path: /</p>
EOF

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rewrite-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rewrite-backend
  template:
    metadata:
      labels:
        app: rewrite-backend
    spec:
      containers:
      - name: backend
        image: nginx:1.25
        ports:
        - containerPort: 80
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo '<h1>Backend Root</h1>' > /usr/share/nginx/html/index.html
                echo '<h1>Backend /api</h1>' > /usr/share/nginx/html/api.html
                mkdir -p /usr/share/nginx/html/v1
                echo '<h1>Backend /v1</h1>' > /usr/share/nginx/html/v1/index.html
      volumes:
      - name: content
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: rewrite-backend
spec:
  selector:
    app: rewrite-backend
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 2: 创建路径重写 Ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    # 重写规则：捕获组 $1 作为新路径
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: rewrite-backend
            port:
              number: 80
      - path: /v1/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: rewrite-backend
            port:
              number: 80
EOF

# Step 3: 测试重写效果
# 请求 /api/api.html → 重写为 /api.html → 返回 "Backend /api"
curl -H "Host: example.com" http://<node-ip>:30080/api/api.html

# 请求 /v1/index.html → 重写为 /index.html → 返回 "Backend /v1"
curl -H "Host: example.com" http://<node-ip>:30080/v1/index.html

# Step 4: 复杂重写示例（带前缀）
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-rewrite-ingress
  annotations:
    # 重写规则：/old-prefix/* → /new-prefix/*
    nginx.ingress.kubernetes.io/rewrite-target: /new-prefix/$1
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /old-prefix/(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: rewrite-backend
            port:
              number: 80
EOF

# Step 5: 清理
kubectl delete ingress rewrite-ingress advanced-rewrite-ingress
kubectl delete deployment rewrite-backend
kubectl delete service rewrite-backend
```

---

## 10.13 实战6：使用 Ingress 实现蓝绿部署

### 步骤说明

蓝绿部署是一种零停机部署策略：
- **蓝版本**：当前生产版本
- **绿版本**：新版本
- 通过 Ingress 切换流量，实现无缝发布

### 完整步骤

```bash
# ==================== 阶段1：部署蓝版本（当前生产）====================

# Step 1: 创建蓝版本 Deployment
cat <<'EOF' | kubectl apply -f -
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
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo '<h1>Blue Version (v1.0)</h1>' > /usr/share/nginx/html/index.html
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
EOF

# Step 2: 创建蓝版本 Service
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-blue-service
spec:
  selector:
    app: web
    version: blue
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 3: 创建 Ingress，指向蓝版本
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue-green-ingress
  annotations:
    nginx.ingress.kubernetes.io/service-upstream: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-blue-service
            port:
              number: 80
EOF

# Step 4: 验证蓝版本
curl --resolve app.example.com:30080:<node-ip> http://app.example.com:30080/
# <h1>Blue Version (v1.0)</h1>

# ==================== 阶段2：部署绿版本 ====================

# Step 5: 创建绿版本 Deployment
cat <<'EOF' | kubectl apply -f -
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
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - echo '<h1>Green Version (v2.0)</h1>' > /usr/share/nginx/html/index.html
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
EOF

# Step 6: 创建绿版本 Service
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-green-service
spec:
  selector:
    app: web
    version: green
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 7: 验证绿版本独立访问
kubectl get pods -l app=web,version=green
# 等待绿版本就绪

# ==================== 阶段3：流量切换 ====================

# Step 8: 更新 Ingress，切换到绿版本
kubectl patch ingress blue-green-ingress -p '
{
  "spec": {
    "rules": [{
      "host": "app.example.com",
      "http": {
        "paths": [{
          "path": "/",
          "pathType": "Prefix",
          "backend": {
            "service": {
              "name": "web-green-service",
              "port": { "number": 80 }
            }
          }
        }]
      }
    }]
  }
}'

# Step 9: 验证绿版本已生效
curl --resolve app.example.com:30080:<node-ip> http://app.example.com:30080/
# <h1>Green Version (v2.0)</h1>

# ==================== 阶段4：验证与清理 ====================

# Step 10: 全量测试绿版本
for i in $(seq 1 10); do
  curl -s --resolve app.example.com:30080:<node-ip> http://app.example.com:30080/
  echo ""
done

# Step 11: 确认无误后，清理蓝版本
kubectl delete deployment web-blue
kubectl delete service web-blue-service

# Step 12: 如果出问题，快速回滚
# kubectl patch ingress blue-green-ingress -p '
# {
#   "spec": {
#     "rules": [{
#       "host": "app.example.com",
#       "http": {
#         "paths": [{
#           "path": "/",
#           "pathType": "Prefix",
#           "backend": {
#             "service": {
#               "name": "web-blue-service",
#               "port": { "number": 80 }
#             }
#           }]
#         }
#       }
#     }]
#   }
# }'

# 清理
kubectl delete ingress blue-green-ingress
kubectl delete deployment web-green
kubectl delete service web-green-service
```

**蓝绿部署流程图**：

```
                     ┌─────────────────────┐
                     │  Ingress Controller  │
                     └──────────┬──────────┘
                                │
              流量切换前         │
              ┌────────────────┤
              ▼                ▼
     ┌──────────────────┐  ┌──────────────────┐
     │  web-blue-service │  │ web-green-service │
     │  (蓝版本 v1.0)    │  │  (绿版本 v2.0)   │
     │  ← 生产流量      │  │  ← 空闲          │
     └──────────────────┘  └──────────────────┘
                                │
              流量切换后         │
              ┌────────────────┤
              ▼                ▼
     ┌──────────────────┐  ┌──────────────────┐
     │  web-blue-service │  │ web-green-service │
     │  (蓝版本 v1.0)    │  │  (绿版本 v2.0)   │
     │  ← 空闲          │  │  ← 生产流量      │
     └──────────────────┘  └──────────────────┘
```

---

## 10.14 常见问题排查

### 10.14.1 Ingress 不生效

```bash
# 1. 检查 Ingress 资源状态
kubectl get ingress
# 检查 ADDRESS 列是否有值（NodePort 类型可能为空）

# 2. 检查 Ingress 事件
kubectl describe ingress my-ingress | grep -A 20 "Events:"

# 3. 检查 Ingress Controller 日志
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100

# 4. 检查 Service 是否存在
kubectl get service api-service
# 确认 Service 存在且有 Endpoints

# 5. 检查 Service Endpoints
kubectl get endpoints api-service
# 确认有后端 Pod 的 IP:Port

# 6. 检查后端 Pod 是否正常
kubectl get pods -l app=api
kubectl describe pod -l app=api
```

### 10.14.2 404 Not Found

```bash
# 1. 确认路径匹配
# 检查 Ingress 的 path 和 pathType
# Prefix 匹配：/api 匹配 /api、/api/v1，但不匹配 /apix

# 2. 确认 Service 名称和端口
kubectl get ingress my-ingress -o yaml | grep -A 5 backend
# 检查 service.name 和 service.port.number

# 3. 确认 Service 有后端 Pod
kubectl get endpoints <service-name>
# 如果为空，说明 Pod 没有正确匹配 label

# 4. 检查路径重写
# 如果使用了 rewrite-target，确认重写规则正确
kubectl get ingress my-ingress -o yaml | grep rewrite
```

### 10.14.3 502 Bad Gateway

```bash
# 1. 检查后端 Pod 是否正常运行
kubectl get pods -l app=my-app
# Pod 状态应为 Running，READY 应为 1/1

# 2. 检查 Service Endpoints
kubectl get endpoints <service-name>
# 应有可用的后端地址

# 3. 检查网络连通性
# 在 Ingress Controller Pod 内测试
kubectl exec -n ingress-nginx <controller-pod> -- curl http://<service-name>
# 如果失败，说明 Service 或 Pod 有问题

# 4. 检查 Service 端口
kubectl get service <service-name> -o yaml | grep -A 5 ports
# 确保 port 和 targetPort 正确

# 5. 查看 Ingress Controller 日志
kubectl logs -n ingress-nginx <controller-pod> --tail=50 | grep error
```

### 10.14.4 TLS 证书问题

```bash
# 1. 检查 TLS Secret
kubectl get secret example-tls
# 确认 Secret 存在且类型为 kubernetes.io/tls

# 2. 检查证书有效期
kubectl describe secret example-tls
# 查看证书过期时间

# 3. cert-manager 签发失败排查
kubectl get certificates
kubectl describe certificate example-tls
kubectl get challenges
kubectl describe challenge <challenge-name>
kubectl logs -n cert-manager -l app=cert-manager --tail=100

# 4. 手动测试证书
openssl s_client -connect <node-ip>:30443 -servername example.com
# 检查证书链和有效期
```

### 10.14.5 路径重写不生效

```bash
# 1. 确认注解正确
kubectl get ingress my-ingress -o yaml | grep rewrite
# 注解应为 nginx.ingress.kubernetes.io/rewrite-target

# 2. 检查正则匹配
# ImplementationSpecific 类型配合 (.*) 捕获组
# 示例：/api/(.*) → 捕获 $1

# 3. 查看实际重写结果
# 在 Ingress Controller 日志中查看
kubectl logs -n ingress-nginx <controller-pod> | grep rewrite

# 4. 测试重写结果
curl -v -H "Host: example.com" http://<node-ip>:30080/api/test
# 查看转发的路径
```

### 10.14.6 Ingress Controller 日志排查

```bash
# 查看 Controller 日志
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=200

# 查看 Nginx 配置生成
kubectl logs -n ingress-nginx <controller-pod> | grep "configuration is now valid"

# 调试单个 Ingress
kubectl describe ingress my-ingress

# 使用 Ingress 调试工具
kubectl ingress-nginx lint --with-error-logging
# 需要安装：kubectl krew install ingress-nginx
```

---

## 10.15 章节小结

### 核心知识图谱

```
                    ┌──────────────────────────────┐
                    │       Ingress 流量路由        │
                    └──────────────┬───────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │  Ingress      │      │  Ingress     │      │  Ingress      │
    │  Controller   │      │  路由规则    │      │  TLS 终止     │
    └───────────────┘      └───────────────┘      └───────────────┘
           │                       │                       │
    ┌──────┴──────┐          ┌──────┴──────┐          ┌──────┴──────┐
    │Nginx        │          │路径路由     │          │手动证书     │
    │Traefik      │          │主机路由     │          │cert-manager │
    │HAProxy      │          │混合路由     │          │自动签发     │
    │Kong         │          │默认后端     │          │自动续期     │
    └─────────────┘          └─────────────┘          └─────────────┘

    ┌─────────────────────────────────────────────────────────────┐
    │                    高级功能                                  │
    ├─────────────────────────────────────────────────────────────┤
    │ 路径重写    → rewrite-target 注解                          │
    │ 蓝绿部署    → 通过更新 Ingress Service 实现流量切换         │
    │ 注解配置    → timeout、限流、认证、会话亲和等              │
    └─────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────┐
    │                    数据流                                    │
    ├─────────────────────────────────────────────────────────────┤
    │ 客户端 → Ingress Controller → Service → Pod                 │
    │          (HTTP/S路由)      (负载均衡)   (应用实例)          │
    └─────────────────────────────────────────────────────────────┘
```

### 关键要点回顾

1. **Ingress 核心作用**：统一 HTTP(S) 流量入口，实现路径/主机路由、TLS 终止
2. **Ingress Controller**：推荐使用 Nginx Ingress，社区活跃、功能全面
3. **路径路由 vs 主机路由**：
   - 路径路由：`example.com/api` → api-service，`example.com/web` → web-service
   - 主机路由：`api.example.com` → api-service，`web.example.com` → web-service
4. **TLS 终止**：在 Ingress 层处理 HTTPS，后端服务只需 HTTP，简化证书管理
5. **路径重写**：使用 `rewrite-target` 注解改变转发路径
6. **蓝绿部署**：通过更新 Ingress 的 Service 引用实现零停机流量切换
7. **排查顺序**：Ingress 规则 → Service → Endpoints → Pod 日志

### 下一章预告

下一章将学习 **配置管理与 Secret**，深入理解 ConfigMap 和 Secret 的使用、配置热更新、以及与 Volume 的结合，实现应用配置与代码的分离管理。
