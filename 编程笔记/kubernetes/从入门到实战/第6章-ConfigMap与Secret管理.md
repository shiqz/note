# 第6章：ConfigMap 与 Secret 管理

## 学习目标

完成本章学习后，你将能够：

1. 理解 ConfigMap 和 Secret 的使用场景与区别
2. 掌握 ConfigMap 的三种创建方式
3. 熟练使用环境变量注入和 Volume 挂载两种使用方式
4. 理解 ConfigMap 热更新机制
5. 掌握 Secret 的编码方式和使用场景
6. 创建和管理 TLS Secret、Basic Auth Secret
7. 遵循 K8s 配置管理的最佳实践
8. 通过实战掌握配置管理的完整流程

---

## 6.1 ConfigMap 概述

### 6.1.1 为什么需要 ConfigMap

在 K8s 中，应用配置与容器镜像应该**分离**：

- **镜像不可变**：容器镜像是不可变的，不同环境（开发/测试/生产）的配置应该通过外部注入
- **配置可独立管理**：修改配置不需要重新构建镜像
- **敏感信息不暴露**：密码、密钥等不应硬编码在镜像中

```
传统方式的问题：
┌─────────────────────┐
│  Docker Image        │
│  ┌─────────────────┐│
│  │  App Config     ││  ← 配置写死在镜像里
│  │  DB_PASSWORD=xxx││  ← 密码泄露风险
│  └─────────────────┘│
└─────────────────────┘

K8s 方式：
┌─────────────────────┐     ┌─────────────────────┐
│  Docker Image        │     │  ConfigMap / Secret  │
│  ┌─────────────────┐│     │  ┌─────────────────┐ │
│  │  App (代码)     ││     │  │  Config / 密码   │ │
│  └─────────────────┘│     │  └─────────────────┘ │
└─────────┬───────────┘     └─────────┬───────────┘
          │                            │
          └──────── 注入 ─────────────┘
```

### 6.1.2 ConfigMap 的使用场景

| 场景 | 示例 |
|------|------|
| 应用配置文件 | `application.properties`、`nginx.conf` |
| 环境变量 | `DB_HOST=localhost`、`LOG_LEVEL=debug` |
| 命令行参数 | 启动脚本中的配置项 |
| 非敏感配置 | 数据库地址、端口、服务 URL |

### 6.1.3 ConfigMap 的特点

- **键值对存储**：以 key-value 形式存储数据
- **非敏感数据**：不应用于存储密码、密钥等敏感信息
- **Namespace 隔离**：ConfigMap 属于特定命名空间
- **热更新支持**：通过 Volume 挂载方式支持热更新

---

## 6.2 ConfigMap 的三种创建方式

### 6.2.1 使用 --from-literal（直接值）

适用于创建简单的键值对配置。

```bash
# 创建单个键值对
kubectl create configmap app-config \
  --from-literal=app_name=my-app

# 创建多个键值对
kubectl create configmap app-config \
  --from-literal=app_name=my-app \
  --from-literal=app_version=1.0.0 \
  --from-literal=log_level=debug \
  --from-literal=max_connections=100

# 创建并查看
kubectl create configmap app-config \
  --from-literal=app_name=my-app \
  --from-literal=app_version=1.0.0 \
  --from-literal=log_level=debug
kubectl get configmap app-config -o yaml
```

**YAML 等价写法**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  app_name: my-app
  app_version: "1.0.0"
  log_level: debug
```

### 6.2.2 使用 --from-file（从文件创建）

将文件内容作为 ConfigMap 的值。

```bash
# 创建配置文件
cat > application.properties << 'EOF'
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=admin
spring.jpa.hibernate.ddl-auto=update
logging.level.root=INFO
EOF

# 从单个文件创建（key 为文件名）
kubectl create configmap app-config \
  --from-file=application.properties

# 指定 key 名称
kubectl create configmap app-config \
  --from-file=app_config=application.properties

# 从多个文件创建
kubectl create configmap app-config \
  --from-file=application.properties \
  --from-file=logback.xml
```

**YAML 等价写法**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.properties: |
    server.port=8080
    spring.datasource.url=jdbc:mysql://localhost:3306/mydb
    spring.datasource.username=admin
    spring.jpa.hibernate.ddl-auto=update
    logging.level.root=INFO
  logback.xml: |
    <configuration>
      <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
          <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
      </appender>
    </configuration>
```

### 6.2.3 使用 --from-directory（从目录创建）

将目录下所有文件导入 ConfigMap。

```bash
# 创建配置目录
mkdir -p config/
cat > config/nginx.conf << 'EOF'
server {
    listen 80;
    server_name localhost;
    location / {
        proxy_pass http://backend:8080;
    }
}
EOF

cat > config/upstream.conf << 'EOF'
upstream backend {
    server backend-1:8080;
    server backend-2:8080;
}
EOF

# 从目录创建（每个文件名为 key，文件内容为 value）
kubectl create configmap nginx-config \
  --from-directory=config/

# 查看创建结果
kubectl get configmap nginx-config -o yaml
```

### 6.2.4 创建方式对比

| 方式 | 适用场景 | 示例 |
|------|---------|------|
| `--from-literal` | 简单键值对 | `--from-literal=key=value` |
| `--from-file` | 单个配置文件 | `--from-file=config.properties` |
| `--from-directory` | 多个配置文件 | `--from-directory=./config/` |

---

## 6.3 ConfigMap 的两种使用方式

### 6.3.1 环境变量注入（envFrom）

将 ConfigMap 中的 key-value 注入为环境变量。

```yaml
# configmap-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      envFrom:
        - configMapRef:
            name: app-config       # 引用 ConfigMap 的名称
      # 也可以单独引用某个 key
      env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: app_name
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log_level
  restartPolicy: Never
```

**关键点**：
- `envFrom`：批量注入 ConfigMap 中所有 key
- `configMapKeyRef`：单独注入某个 key
- 环境变量注入**不支持热更新**

```bash
# 创建并验证
kubectl apply -f configmap-env.yaml
kubectl exec env-demo-pod -- env | grep -E "app_name|log_level|app_version"
```

### 6.3.2 Volume 挂载

将 ConfigMap 挂载为文件系统中的文件。

```yaml
# configmap-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config           # 整个 ConfigMap 挂载到该目录
        - name: app-config-volume
          mountPath: /etc/app               # 使用 subPath 挂载单个文件
      command: ["sh", "-c", "cat /etc/config/application.properties && echo '---' && cat /etc/app/conf"]
  volumes:
    - name: config-volume
      configMap:
        name: app-config                   # ConfigMap 名称
        # items:                           # 可选：选择性挂载
        #   - key: application.properties
        #     path: config.properties
    - name: app-config-volume
      configMap:
        name: app-config
        items:
          - key: log_level
            path: conf
  restartPolicy: Never
```

**挂载后的文件结构**：

```
/etc/config/
├── application.properties    # key 为文件名，value 为文件内容
└── logback.xml

/etc/app/
└── conf                      # 只挂载了 log_level 这一个 key
```

### 6.3.3 使用 subPath 避免全量更新

**问题**：直接挂载整个 ConfigMap 到目录时，修改 ConfigMap 会导致挂载目录下的所有文件重新加载，可能引起应用重启。

**解决方案**：使用 `subPath` 挂载单个文件。

```yaml
# subpath-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: subpath-demo-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf          # 关键：只挂载单个文件
        - name: app-properties
          mountPath: /etc/app/config.properties
          subPath: config.properties
      command: ["sh", "-c", "sleep 3600"]
  volumes:
    - name: nginx-conf
      configMap:
        name: nginx-config
        items:
          - key: default.conf
            path: default.conf
    - name: app-properties
      configMap:
        name: app-config
        items:
          - key: config.properties
            path: config.properties
  restartPolicy: Never
```

**subPath 的作用**：
- 只更新被修改的文件，其他文件不受影响
- 避免配置变更导致应用意外重启
- 实现了更细粒度的配置管理

### 6.3.4 两种使用方式对比

| 特性 | 环境变量注入 | Volume 挂载 |
|------|-------------|------------|
| 热更新 | ❌ 不支持 | ✅ 支持 |
| 数据量 | 小（环境变量长度限制） | 大（可存储配置文件） |
| 文件格式 | 键值对 | 支持任意格式 |
| 适用场景 | 简单配置 | 复杂配置文件 |
| 重启需求 | 修改后需重启 Pod | 修改后自动生效 |

---

## 6.4 ConfigMap 热更新机制

### 6.4.1 Volume 挂载方式的热更新

当 ConfigMap 被更新时，Volume 挂载方式会自动更新 Pod 中的文件。

**更新流程**：

```
1. kubectl apply -f configmap.yaml  (更新 ConfigMap)
           │
           ▼
2. K8s 检测到 ConfigMap 变化
           │
           ▼
3. kubelet 同步更新挂载点的文件内容
           │
           ▼
4. 应用读取文件时获得最新配置
```

**热更新的限制**：
- 只有挂载方式支持热更新
- 使用 `subPath` 时，挂载的文件会自动更新
- 应用需要定期重新读取文件（或使用 inotify 监听）

### 6.4.2 环境变量方式的限制

```yaml
# 环境变量注入 - 不支持热更新
envFrom:
  - configMapRef:
      name: app-config
```

- 环境变量在容器启动时加载
- 修改 ConfigMap 后，容器内的环境变量不会变化
- 需要**重启 Pod**才能生效

### 6.4.3 验证热更新

```bash
# 创建 ConfigMap
kubectl create configmap hot-reload-demo \
  --from-literal=message="version-1"

# 创建 Pod 挂载 ConfigMap
kubectl run hot-reload-pod --image=nginx:1.25 --overrides='
{
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx:1.25",
      "volumeMounts": [{
        "name": "config",
        "mountPath": "/etc/config"
      }]
    }],
    "volumes": [{
      "name": "config",
      "configMap": {
        "name": "hot-reload-demo"
      }
    }]
  }
}'

# 查看初始内容
kubectl exec hot-reload-pod -- cat /etc/config/message

# 更新 ConfigMap
kubectl create configmap hot-reload-demo \
  --from-literal=message="version-2" \
  -o yaml | kubectl apply -f -

# 等待一会儿再查看（约 1 秒）
sleep 2
kubectl exec hot-reload-pod -- cat /etc/config/message
# 应该显示 "version-2"
```

---

## 6.5 Secret 概述

### 6.5.1 Secret 的使用场景

Secret 用于存储敏感信息，与 ConfigMap 的主要区别是**编码存储**。

| 场景 | 示例 |
|------|------|
| 认证凭证 | 用户名、密码 |
| API 密钥 | Access Key、Secret Key |
| TLS 证书 | SSL 证书和私钥 |
| Token | OAuth Token、JWT Secret |
| SSH 密钥 | 用于拉取私有镜像 |

### 6.5.2 Secret 的类型

| 类型 | 说明 | 用途 |
|------|------|------|
| Opaque | 用户自定义的任意数据 | 通用密钥/密码存储 |
| kubernetes.io/tls | TLS 证书和私钥 | Ingress TLS 终止 |
| kubernetes.io/basic-auth | HTTP Basic Auth 凭证 | 认证配置 |
| kubernetes.io/ssh-auth | SSH 密钥 | 拉取私有镜像 |
| kubernetes.io/dockerconfigjson | Docker Registry 凭证 | 拉取私有镜像 |
| kubernetes.io/service-account-token | ServiceAccount Token | 自动创建 |

### 6.5.3 Secret 与 ConfigMap 的区别

| 特性 | ConfigMap | Secret |
|------|-----------|--------|
| 数据类型 | 非敏感配置 | 敏感信息 |
| 存储方式 | 明文 | Base64 编码（注意：不是加密！） |
| 访问控制 | 无额外限制 | 更严格的 RBAC |
| 用途 | 应用配置 | 密码、证书、密钥 |

**重要提醒**：Secret 使用 Base64 编码，但**不是加密**。Base64 只是编码格式，任何人都可以解码。生产环境应结合 Encryption at Rest 等机制使用。

---

## 6.6 Secret 的编码方式

### 6.6.1 Base64 编码/解码

```bash
# 编码
echo -n "my-password" | base64
# 输出: bXktcGFzc3dvcmQ=

# 解码
echo "bXktcGFzc3dvcmQ=" | base64 -d
# 输出: my-password
```

**注意**：
- 使用 `echo -n` 避免添加换行符
- Base64 编码是可逆的，**不是加密**
- 不要将 Base64 编码的 Secret 提交到 Git 仓库

### 6.6.2 创建 Secret 时的编码

```bash
# 创建 Secret（需要 Base64 编码）
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123

# 查看 Secret 内容（显示的是 Base64 编码值）
kubectl get secret db-credentials -o yaml

# 查看解码后的内容
kubectl get secret db-credentials -o jsonpath='{.data.username}' | base64 -d
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

---

## 6.7 Secret 的创建与使用

### 6.7.1 创建 Secret

**方式一：使用 kubectl create**

```bash
# Opaque 类型 - 通用密钥
kubectl create secret generic db-password \
  --from-literal=mysql-root-password=root123 \
  --from-literal=mysql-user-password=user123

# 从文件创建
kubectl create secret generic app-secrets \
  --from-file=api-key.txt \
  --from-file=secret-key.txt

# 从目录创建
kubectl create secret generic all-secrets \
  --from-directory=./secrets/
```

**方式二：使用 YAML 文件**

```yaml
# secret-db.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque
data:
  username: YWRtaW4=           # echo -n "admin" | base64
  password: c2VjcmV0MTIz       # echo -n "secret123" | base64
stringData:                     # 可选：明文数据，K8s 会自动编码
  extra-key: plain-text-value   # 方便开发时使用
```

```bash
kubectl apply -f secret-db.yaml
```

### 6.7.2 使用 Secret

**方式一：环境变量注入**

```yaml
# secret-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      envFrom:
        - secretRef:
            name: db-credentials
  restartPolicy: Never
```

**方式二：Volume 挂载**

```yaml
# secret-volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
        items:
          - key: username
            path: db-username
          - key: password
            path: db-password
  restartPolicy: Never
```

**挂载后文件结构**：

```
/etc/secrets/
├── db-username    # 内容为 "admin"
└── db-password    # 内容为 "secret123"
```

### 6.7.3 创建 TLS Secret

TLS Secret 用于存储 SSL/TLS 证书和私钥，主要用于 Ingress。

```bash
# 创建自签名证书（用于测试）
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt -days 365 -nodes -subj '/CN=example.com'

# 创建 TLS Secret
kubectl create secret tls example-tls \
  --key=tls.key \
  --cert=tls.crt

# 查看 TLS Secret
kubectl get secret example-tls -o yaml
```

**YAML 等价写法**：

```yaml
# tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-tls
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # Base64 编码的证书
  tls.key: LS0tLS1CRUdJTi...  # Base64 编码的私钥
```

**在 Ingress 中使用 TLS Secret**：

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls       # 引用 TLS Secret
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
```

### 6.7.4 创建 Basic Auth Secret

```bash
# 创建 Basic Auth Secret
kubectl create secret generic basic-auth \
  --from-literal=username=admin \
  --from-literal=password=admin123 \
  --type=kubernetes.io/basic-auth

# 在 Ingress 中使用
kubectl create ingress basic-auth-ingress \
  --rule="/auth=web-service:80" \
  --annotations="nginx.ingress.kubernetes.io/auth-type=basic,nginx.ingress.kubernetes.io/auth-secret=basic-auth"
```

### 6.7.5 创建 Docker Registry Secret

```bash
# 创建 Docker Registry Secret
kubectl create secret docker-registry docker-registry-secret \
  --docker-server=myregistry.example.com \
  --docker-username=myusername \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# 在 Pod 中引用
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-pod
spec:
  imagePullSecrets:
    - name: docker-registry-secret
  containers:
    - name: app
      image: myregistry.example.com/myapp:1.0
```

---

## 6.8 K8s 配置管理最佳实践

### 6.8.1 ConfigMap vs Secret 选择指南

```
                    ┌─────────────────────┐
                    │  数据是否敏感？      │
                    └──────────┬──────────┘
                               │
                    ┌──YES─────┴────NO──┐
                    ▼                   ▼
              ┌──────────┐      ┌──────────┐
              │   Secret   │      │ ConfigMap │
              └──────────┘      └──────────┘
                    │                   │
         ┌──────────┼──────────┐       │
         ▼          ▼          ▼       ▼
      密码/密钥  TLS证书  Token/API   应用配置
      凭证        SSH密钥           环境变量
```

### 6.8.2 最佳实践清单

1. **ConfigMap 用于非敏感配置**
   - 应用配置文件
   - 环境变量（非敏感）
   - 日志级别、服务地址等

2. **Secret 用于敏感信息**
   - 密码、密钥
   - TLS 证书
   - API Token

3. **不要将 Secret 提交到 Git**
   - 使用 `.gitignore` 排除 Secret 文件
   - 使用模板和占位符

4. **使用命名空间隔离**
   - 不同环境使用不同命名空间
   - 不同团队使用不同命名空间

5. **RBAC 控制访问**
   - 限制 Secret 的读取权限
   - 遵循最小权限原则

6. **生产环境加密**
   - 启用 Encryption at Rest
   - 使用 External Secrets Operator（ESO）
   - 集成 Vault 等密钥管理服务

### 6.8.3 目录结构建议

```
k8s-config/
├── configmaps/
│   ├── app-config.yaml
│   ├── nginx-config.yaml
│   └── logging-config.yaml
├── secrets/
│   ├── .gitignore              # 排除所有 Secret
│   ├── db-credentials.yaml
│   └── tls-secret.yaml
├── deployments/
│   └── app-deployment.yaml
└── services/
    └── app-service.yaml
```

### 6.8.4 使用 .gitignore 保护 Secret

```gitignore
# secrets/
# *
# !.gitkeep
# !README.md

# 或者使用占位符方式
secrets/*.yaml
!secrets/*.template.yaml
```

### 6.8.5 使用 Kustomize 管理多环境

```yaml
# base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=${SERVER_PORT}
    db.host=${DB_HOST}
    db.port=${DB_PORT}
```

```yaml
# overlays/production/configmap-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    db.host=prod-db.example.com
    db.port=3306
    log.level=ERROR
```

---

## 6.9 实战演练

### 实战1：创建 ConfigMap 并通过环境变量注入到 Pod

**步骤 1**：创建 ConfigMap

```yaml
# cm-env-demo.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-env-demo
  labels:
    app: env-demo
data:
  APP_NAME: "EnvDemoApp"
  APP_VERSION: "2.0.0"
  LOG_LEVEL: "debug"
  MAX_CONNECTIONS: "200"
  DB_HOST: "mysql.default.svc.cluster.local"
```

```bash
kubectl apply -f cm-env-demo.yaml
kubectl get configmap cm-env-demo
kubectl describe configmap cm-env-demo
```

**步骤 2**：创建 Pod 引用 ConfigMap

```yaml
# env-demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo-pod
  labels:
    app: env-demo
spec:
  containers:
    - name: demo
      image: nginx:1.25
      envFrom:
        - configMapRef:
            name: cm-env-demo
      env:
        - name: CUSTOM_ENV
          valueFrom:
            configMapKeyRef:
              name: cm-env-demo
              key: APP_NAME
      command: ["sh", "-c", "echo '=== Environment Variables ===' && env | grep -E 'APP_|LOG_|MAX_|DB_' && echo '=== Done ===' && sleep 3600"]
  restartPolicy: Never
```

```bash
kubectl apply -f env-demo-pod.yaml
kubectl get pod env-demo-pod
```

**步骤 3**：验证环境变量

```bash
kubectl logs env-demo-pod
# 输出：
# === Environment Variables ===
# APP_NAME=EnvDemoApp
# APP_VERSION=2.0.0
# LOG_LEVEL=debug
# MAX_CONNECTIONS=200
# DB_HOST=mysql.default.svc.cluster.local
# === Done ===

# 进入 Pod 检查
kubectl exec -it env-demo-pod -- env | grep APP_
```

### 实战2：创建 ConfigMap 并挂载为 Volume，使用 subPath 避免全量更新

**步骤 1**：创建多文件 ConfigMap

```yaml
# cm-volume-demo.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-volume-demo
data:
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
  app.properties: |
    server.port=8080
    app.name=VolumeDemo
    log.level=INFO
    max.threads=100
  logback.xml: |
    <configuration>
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="STDOUT" />
        </root>
    </configuration>
```

```bash
kubectl apply -f cm-volume-demo.yaml
```

**步骤 2**：创建 Pod 使用 Volume 挂载 + subPath

```yaml
# volume-demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: nginx.conf
        - name: app-config
          mountPath: /etc/app/app.properties
          subPath: app.properties
        - name: log-config
          mountPath: /etc/app/logback.xml
          subPath: logback.xml
      command: ["sh", "-c", "echo '=== Nginx Config ===' && cat /etc/nginx/conf.d/default.conf && echo '=== App Config ===' && cat /etc/app/app.properties && echo '=== Log Config ===' && cat /etc/app/logback.xml && sleep 3600"]
  volumes:
    - name: nginx-conf
      configMap:
        name: cm-volume-demo
        items:
          - key: nginx.conf
            path: nginx.conf
    - name: app-config
      configMap:
        name: cm-volume-demo
        items:
          - key: app.properties
            path: app.properties
    - name: log-config
      configMap:
        name: cm-volume-demo
        items:
          - key: logback.xml
            path: logback.xml
  restartPolicy: Never
```

```bash
kubectl apply -f volume-demo-pod.yaml
kubectl logs volume-demo-pod
```

**步骤 3**：验证 subPath 效果

```bash
# 初始内容
kubectl exec volume-demo-pod -- cat /etc/app/app.properties

# 修改 ConfigMap 中的 app.properties
kubectl create configmap cm-volume-demo -o yaml | \
  kubectl set env configmap/cm-volume-demo --containers='' 2>/dev/null || true

# 使用 kubectl 直接更新某个 key
kubectl patch configmap cm-volume-demo --type='json' -p='[{"op":"replace","path":"/data/app.properties","value":"server.port=8080\napp.name=VolumeDemo\nlog.level=DEBUG\nmax.threads=200"}]'

# 等待热更新生效
sleep 2

# 验证文件已更新
kubectl exec volume-demo-pod -- cat /etc/app/app.properties
# 应该显示 log.level=DEBUG, max.threads=200

# 验证其他文件未受影响
kubectl exec volume-demo-pod -- cat /etc/nginx/conf.d/default.conf
# 应该还是原来的 nginx.conf 内容
```

### 实战3：创建 Secret 存储数据库密码

**步骤 1**：创建数据库 Secret

```bash
# 方式一：直接创建
kubectl create secret generic db-credentials \
  --from-literal=mysql-root-password=Root123! \
  --from-literal=mysql-user-password=User123! \
  --from-literal=mysql-database=mydb

# 方式二：YAML 文件创建
kubectl create secret generic db-credentials -o yaml \
  --from-literal=mysql-root-password=Root123! \
  --from-literal=mysql-user-password=User123! \
  --from-literal=mysql-database=mydb \
  | kubectl apply -f -
```

**步骤 2**：查看 Secret 内容

```bash
# 查看 Secret 基本信息
kubectl get secret db-credentials

# 查看 YAML 格式（Base64 编码）
kubectl get secret db-credentials -o yaml

# 解码查看实际值
kubectl get secret db-credentials -o jsonpath='{.data.mysql-root-password}' | base64 -d
echo ""
kubectl get secret db-credentials -o jsonpath='{.data.mysql-user-password}' | base64 -d
echo ""
```

**步骤 3**：在 Pod 中使用 Secret（环境变量方式）

```yaml
# mysql-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
    - name: mysql
      image: mysql:8.0
      ports:
        - containerPort: 3306
      env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: mysql-root-password
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: mysql-user-password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: mysql-database
      volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
    - name: app
      image: nginx:1.25
      envFrom:
        - secretRef:
            name: db-credentials
  volumes:
    - name: mysql-data
      emptyDir: {}
  restartPolicy: Never
```

```bash
kubectl apply -f mysql-pod.yaml

# 验证环境变量注入
kubectl exec mysql-pod -c mysql -- env | grep MYSQL
kubectl exec mysql-pod -c app -- env
```

### 实战4：创建 TLS Secret

**步骤 1**：生成测试证书

```bash
# 创建临时目录
mkdir -p tls-demo && cd tls-demo

# 生成自签名证书
openssl req -x509 -newkey rsa:4096 \
  -keyout tls.key \
  -out tls.crt \
  -days 365 \
  -nodes \
  -subj '/CN=myapp.example.com/O=MyOrg/C=CN'

# 验证证书
openssl x509 -in tls.crt -text -noout
openssl rsa -in tls.key -check
```

**步骤 2**：创建 TLS Secret

```bash
# 创建 TLS Secret
kubectl create secret tls myapp-tls \
  --key=tls.key \
  --cert=tls.crt

# 查看 Secret
kubectl get secret myapp-tls
kubectl get secret myapp-tls -o yaml

# 解码验证
kubectl get secret myapp-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

**步骤 3**：在 Ingress 中使用 TLS Secret

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

```bash
# 注意：需要先安装 Ingress Controller
kubectl apply -f tls-ingress.yaml

# 验证 Ingress
kubectl get ingress myapp-ingress
kubectl describe ingress myapp-ingress
```

**步骤 4**：使用 YAML 方式创建 TLS Secret

```yaml
# tls-secret-manual.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls-manual
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWZ3Z0...
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUV...
```

### 实战5：验证 ConfigMap 热更新

**步骤 1**：创建 ConfigMap 和 Pod

```bash
# 创建 ConfigMap（版本1）
kubectl create configmap hot-reload-cm \
  --from-literal=config.txt="version-1-content"

# 创建 Pod 挂载 ConfigMap
kubectl run hot-reload-pod \
  --image=nginx:1.25 \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx:1.25",
      "volumeMounts": [{
        "name": "config-volume",
        "mountPath": "/etc/config"
      }],
      "command": ["sh", "-c", "while true; do cat /etc/config/config.txt; sleep 2; done"]
    }],
    "volumes": [{
      "name": "config-volume",
      "configMap": {
        "name": "hot-reload-cm"
      }
    }]
  }
}'

# 查看初始内容
kubectl exec hot-reload-pod -- cat /etc/config/config.txt
# 输出: version-1-content
```

**步骤 2**：更新 ConfigMap

```bash
# 查看当前 Pod 日志（持续输出内容）
kubectl logs -f hot-reload-pod &

# 更新 ConfigMap
kubectl patch configmap hot-reload-cm --type='json' \
  -p='[{"op":"replace","path":"/data/config.txt","value":"version-2-content"}]'

# 等待热更新生效
sleep 3

# 查看 Pod 中的文件是否已更新
kubectl exec hot-reload-pod -- cat /etc/config/config.txt
# 应该输出: version-2-content

# 再次更新
kubectl patch configmap hot-reload-cm --type='json' \
  -p='[{"op":"replace","path":"/data/config.txt","value":"version-3-content"}]'

sleep 3
kubectl exec hot-reload-pod -- cat /etc/config/config.txt
# 应该输出: version-3-content
```

**步骤 3**：对比环境变量方式（不支持热更新）

```bash
# 创建使用环境变量的 Pod
kubectl run env-reload-pod \
  --image=nginx:1.25 \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "nginx",
      "image": "nginx:1.25",
      "env": [{
        "name": "CONFIG_VALUE",
        "valueFrom": {
          "configMapKeyRef": {
            "name": "hot-reload-cm",
            "key": "config.txt"
          }
        }
      }],
      "command": ["sh", "-c", "while true; do echo $CONFIG_VALUE; sleep 2; done"]
    }]
  }
}'

# 查看初始值
kubectl logs env-reload-pod | head -5
# 输出: version-3-content

# 再次更新 ConfigMap
kubectl patch configmap hot-reload-cm --type='json' \
  -p='[{"op":"replace","path":"/data/config.txt","value":"version-4-content"}]'

sleep 3

# 查看环境变量是否更新（应该没有更新）
kubectl exec env-reload-pod -- env | grep CONFIG_VALUE
# 仍然是 version-3-content

# 查看 Volume 挂载方式（应该已更新）
kubectl exec hot-reload-pod -- cat /etc/config/config.txt
# 应该是 version-4-content
```

**步骤 4**：清理资源

```bash
kubectl delete pod hot-reload-pod env-reload-pod
kubectl delete configmap hot-reload-cm
```

---

## 6.10 常见问题排查

### 6.10.1 ConfigMap 更新后 Pod 未生效

**问题**：修改 ConfigMap 后，Pod 中的配置没有更新。

**排查步骤**：

```bash
# 1. 检查 ConfigMap 是否已更新
kubectl get configmap <name> -o yaml

# 2. 如果使用环境变量方式 - 必须重启 Pod
kubectl rollout restart deployment <name>

# 3. 如果使用 Volume 挂载 - 等待热更新（最长可能需要 60 秒）
# K8s 默认同步周期约 60 秒
kubectl exec <pod-name> -- cat /path/to/file

# 4. 检查是否使用了 subPath
# subPath 方式需要确保应用重新读取文件

# 5. 检查 kubelet 日志
kubectl logs -n kube-system -l k8s-app=kubelet | grep configmap
```

### 6.10.2 Secret 解码失败

**问题**：Secret 中的值解码后不正确。

**排查步骤**：

```bash
# 1. 确认使用 echo -n 编码（不要加换行符）
echo -n "password" | base64
# 正确: cGFzc3dvcmQ=

echo "password" | base64
# 错误: cGFzc3dvcmQK  (包含换行符)

# 2. 使用正确的解码命令
echo "cGFzc3dvcmQ=" | base64 -d

# 3. 检查是否有不可见字符
echo -n "cGFzc3dvcmQ=" | base64 -d | od -c
```

### 6.10.3 ConfigMap/Secret 挂载失败

**问题**：Pod 无法挂载 ConfigMap 或 Secret。

**排查步骤**：

```bash
# 1. 检查 ConfigMap/Secret 是否存在
kubectl get configmap <name>
kubectl get secret <name>

# 2. 检查命名空间是否正确
kubectl get configmap <name> -n <namespace>
kubectl get pods -n <namespace>

# 3. 检查 Pod 的事件
kubectl describe pod <name>
# 查看 Events 部分的错误信息

# 4. 检查引用的 key 是否存在
kubectl get configmap <name> -o jsonpath='{.data}'
kubectl get secret <name> -o jsonpath='{.data}'

# 5. 检查 YAML 语法
kubectl apply --dry-run=client -f <file.yaml>
```

### 6.10.4 TLS Secret 创建失败

**问题**：创建 TLS Secret 时报错。

**排查步骤**：

```bash
# 1. 检查证书文件格式
openssl x509 -in tls.crt -text -noout
# 必须是有效的 X.509 证书

# 2. 检查私钥格式
openssl rsa -in tls.key -check
# 必须是有效的 RSA 私钥

# 3. 检查证书和私钥是否匹配
openssl x509 -in tls.crt -noout -modulus | md5sum
openssl rsa -in tls.key -noout -modulus | md5sum
# 两个 MD5 值必须相同

# 4. 使用正确的命令
kubectl create secret tls <name> --key=tls.key --cert=tls.crt
```

### 6.10.5 环境变量未注入

**问题**：Pod 中看不到 ConfigMap/Secret 的环境变量。

**排查步骤**：

```bash
# 1. 检查 envFrom 和 env 配置
kubectl get pod <name> -o yaml | grep -A 10 envFrom
kubectl get pod <name> -o yaml | grep -A 10 env:

# 2. 检查容器是否正确引用
kubectl exec <pod-name> -c <container-name> -- env

# 3. 检查是否有拼写错误
kubectl exec <pod-name> -- env | grep -i <expected-key>

# 4. 检查容器的 shell
# 某些镜像使用非标准 shell，需要用 /bin/sh 或 /bin/bash
kubectl exec <pod-name> -- /bin/sh -c 'env'
```

### 6.10.6 ConfigMap 大小限制

**问题**：ConfigMap 创建失败或过大。

**排查步骤**：

```bash
# ConfigMap 大小限制：1MB（1,048,576 字节）
# 检查 ConfigMap 大小
kubectl get configmap <name> -o json | wc -c

# 如果超过限制，考虑：
# 1. 拆分为多个 ConfigMap
# 2. 使用 ConfigMap + Volume 挂载方式
# 3. 外部存储（如对象存储）
```

---

## 6.11 章节小结

### 核心知识点回顾

1. **ConfigMap**：存储非敏感配置
   - 三种创建方式：`--from-literal`、`--from-file`、`--from-directory`
   - 两种使用方式：环境变量注入、Volume 挂载
   - 热更新：Volume 挂载支持，环境变量不支持

2. **Secret**：存储敏感信息
   - 类型：Opaque、tls、basic-auth、dockerconfigjson、ssh-auth
   - 编码：Base64（注意：不是加密）
   - 使用方式与 ConfigMap 相同

3. **最佳实践**：
   - ConfigMap 存非敏感，Secret 存敏感
   - 生产环境启用 Encryption at Rest
   - 使用 subPath 实现细粒度配置更新
   - 不要将 Secret 提交到 Git

### 常用命令速查

```bash
# ConfigMap 创建
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=path/to/file
kubectl create configmap <name> --from-directory=path/to/dir

# ConfigMap 使用
kubectl get configmap <name>
kubectl get configmap <name> -o yaml
kubectl get configmap <name> -o jsonpath='{.data.<key>}'

# ConfigMap 更新
kubectl edit configmap <name>
kubectl patch configmap <name> --type='json' -p='[...]'

# Secret 创建
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret tls <name> --key=key.pem --cert=cert.pem
kubectl create secret docker-registry <name> --docker-server=... --docker-username=... --docker-password=...

# Secret 查看/解码
kubectl get secret <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data.<key>}' | base64 -d

# 删除
kubectl delete configmap <name>
kubectl delete secret <name>
```

### 最佳实践总结

| 场景 | 推荐方式 |
|------|---------|
| 应用配置文件 | ConfigMap + Volume 挂载 + subPath |
| 数据库密码 | Secret + 环境变量注入 |
| TLS 证书 | TLS Secret + Ingress 引用 |
| 简单键值对 | ConfigMap + envFrom |
| 敏感 Token | Secret + Volume 挂载 |
| 多环境配置 | Kustomize / Helm Values |
| 大规模配置 | 外部配置中心（Nacos、Apollo） |
