# 第14章 Helm 包管理与应用部署

## 学习目标

- 理解 Helm 的核心概念：Chart、Release、Repository
- 掌握 Helm 与 kubectl 的区别与互补关系
- 熟悉 Chart 目录结构和模板引擎基础
- 掌握 values.yaml 的多环境配置方法
- 熟练使用 Helm 三大功能：Install、Upgrade、Uninstall
- 理解 Helm 仓库管理和常用公共 Chart 的使用
- 通过 7 个实战练习，掌握 Helm 从安装到自定义 Chart 的完整流程

---

## 14.1 Helm 核心概念

### 14.1.1 什么是 Helm

Helm 是 Kubernetes 的包管理工具，类似于 Linux 的 `apt`、`yum` 或 Node.js 的 `npm`。它将 Kubernetes 应用及其依赖打包成标准化的 Chart，实现应用的一键部署、升级和回滚。

```
┌─────────────────────────────────────────────────────────────┐
│                    Helm 在 K8s 生态中的定位                    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  传统方式（kubectl）                    │   │
│  │                                                     │   │
│  │  kubectl apply -f deployment.yaml                    │   │
│  │  kubectl apply -f service.yaml                      │   │
│  │  kubectl apply -f configmap.yaml                    │   │
│  │  kubectl apply -f ingress.yaml                      │   │
│  │  ... 多个 YAML 文件手动管理                            │   │
│  │  问题：版本管理困难、多环境配置繁琐、重复劳动           │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                   │
│                          │  进化                             │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Helm 方式                            │   │
│  │                                                     │   │
│  │  helm install my-app ./my-chart                     │   │
│  │  helm upgrade my-app ./my-chart -f prod-values.yaml  │   │
│  │  helm rollback my-app 1                             │   │
│  │                                                     │   │
│  │  优势：模板化、版本管理、一键部署、多环境配置           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 14.1.2 Helm vs kubectl

| 维度 | kubectl | Helm |
|------|---------|------|
| **定位** | Kubernetes 资源管理工具 | Kubernetes 包管理工具 |
| **操作对象** | 单个资源文件 | 整个应用（多个资源） |
| **模板支持** | 不支持 | 支持 Go Template 模板 |
| **版本管理** | 无原生支持 | 支持 Release 版本管理 |
| **多环境** | 手动维护多套 YAML | 一套模板 + 多套 values |
| **部署复杂度** | 需要逐个 apply | 一键 install/upgrade |
| **回滚能力** | 需要手动重建 | `helm rollback` 一键回滚 |
| **依赖管理** | 手动处理 | Chart 依赖声明 |
| **适用场景** | 简单资源操作 | 应用级部署和管理 |

### 14.1.3 Helm 三大核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                    Helm 核心概念关系图                         │
│                                                             │
│  ┌──────────────────┐                                      │
│  │   Repository     │                                      │
│  │   （仓库）        │                                      │
│  │                  │                                      │
│  │  Chart 的存储位置  │                                      │
│  │  类似 npm registry │                                      │
│  └────────┬─────────┘                                      │
│           │ 存放                                            │
│           ▼                                                 │
│  ┌──────────────────┐     helm install        ┌──────────────┐ │
│  │      Chart        │ ──────────────────────▶│   Release    │ │
│  │   （包/模板）      │                        │ （部署实例）   │ │
│  │                  │                        │              │ │
│  │  应用的描述模板    │                        │ Chart 的实例化 │ │
│  │  包含 YAML 模板    │                        │ 包含配置值     │ │
│  │ 和默认配置值       │                        │ 独立的生命周期 │ │
│  └──────────────────┘                        └──────────────┘ │
│           ▲                                                  │
│           │  包含                                            │
│           │                                                  │
│  ┌──────────────────┐                                      │
│  │    values.yaml    │                                      │
│  │   （配置值）       │                                      │
│  │                  │                                      │
│  │  Chart 的默认配置  │                                      │
│  │  可被外部 values   │                                      │
│  │  文件覆盖          │                                      │
│  └──────────────────┘                                      │
└─────────────────────────────────────────────────────────────┘
```

| 概念 | 说明 | 类比 |
|------|------|------|
| **Chart** | 应用的模板包，包含 Kubernetes 资源定义和默认配置 | Docker Image |
| **Release** | Chart 的一次部署实例，具有独立的名称和版本 | Docker Container |
| **Repository** | Chart 的仓库，存储和分发 Chart 的地方 | Docker Registry |

### 14.1.4 Helm 三大功能

| 功能 | 命令 | 说明 |
|------|------|------|
| **Install** | `helm install` | 部署一个新的 Release |
| **Upgrade** | `helm upgrade` | 更新已有的 Release，修改配置或升级版本 |
| **Uninstall** | `helm uninstall` | 删除一个 Release，清理所有相关资源 |

---

## 14.2 Chart 目录结构详解

### 14.2.1 标准目录结构

```
my-chart/
├── Chart.yaml              # Chart 元信息（必填）
├── values.yaml             # 默认配置值
├── values/                 # 多环境配置目录（可选）
│   ├── dev.yaml
│   ├── staging.yaml
│   └── production.yaml
├── templates/              # 模板文件
│   ├── _helpers.tpl        # 公共模板函数（以下划线开头）
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── serviceaccount.yaml
│   └── NOTES.txt            # 部署后显示的提示信息
├── charts/                 # 依赖的子 Chart
│   └── subchart/
│       ├── Chart.yaml
│       └── values.yaml
└── .helmignore             # Helm 忽略文件（类似 .gitignore）
```

### 14.2.2 Chart.yaml 详解

```yaml
# Chart.yaml - Chart 元信息
apiVersion: v2                # Chart API 版本，v2 对应 Helm 3
name: my-app                 # Chart 名称
description: A Helm chart for Kubernetes  # 描述
type: application             # 类型：application / library
version: 1.0.0                # Chart 自身版本（遵循 SemVer）
appVersion: "1.0.0"           # 应用版本
kubeVersion: ">=1.24.0"      # 兼容的 Kubernetes 版本

# 维护者信息
maintainers:
  - name: Platform Team
    email: platform@example.com
    url: https://example.com

# 依赖声明
dependencies:
  - name: redis
    version: 17.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
    tags:
      - cache
      - database
  - name: mysql
    version: 9.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: mysql.enabled

# 关键词
keywords:
  - web
  - api
  - restful

# 首页
home: https://example.com

# 图标
icon: https://example.com/icon.png

# 许可证
license: Apache-2.0
```

### 14.2.3 values.yaml 详解

```yaml
# values.yaml - 默认配置值

# 全局配置
global:
  environment: development
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""

# 应用副本数
replicaCount: 3

# 镜像配置
image:
  repository: my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

# 容器配置
container:
  name: my-app
  port: 8080
  env:
    - name: NODE_ENV
      value: production
    - name: LOG_LEVEL
      value: info

# 资源配置
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

# 自动扩缩容
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# 服务配置
service:
  type: ClusterIP
  port: 80
  annotations: {}
  loadBalancerIP: ""

# Ingress 配置
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - my-app.example.com
      secretName: my-app-tls

# 持久化存储
persistence:
  enabled: true
  size: 10Gi
  accessMode: ReadWriteOnce

# 健康检查
probes:
  liveness:
    path: /health
    port: 8080
    initialDelaySeconds: 15
    periodSeconds: 10
    timeoutSeconds: 3
    failureThreshold: 3
  readiness:
    path: /ready
    port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3

# Pod 注解
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"

# Pod 标签
podLabels:
  app: my-app

# 节点选择
nodeSelector: {}

# 容忍度
tolerations: []

# 亲和性
affinity: {}

# 网络策略
networkPolicy:
  enabled: false

# ServiceMonitor（Prometheus 集成）
serviceMonitor:
  enabled: false
  interval: 30s
  path: /metrics

# 子 Chart 配置
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
  master:
    persistence:
      size: 5Gi
  replica:
    replicaCount: 2

mysql:
  enabled: false
  auth:
    rootPassword: ""
    database: mydb
  primary:
    persistence:
      size: 10Gi
```

### 14.2.4 模板文件示例

#### _helpers.tpl（公共模板）

```
{{/* vim: set filetype=mustache: */}}

{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/part-of: {{ .Release.Name }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

#### deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Values.container.name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          {{- if .Values.global.imageRegistry }}
          image: "{{ .Values.global.imageRegistry }}/{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          {{- else }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          {{- end }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            {{- range .Values.container.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          ports:
            - name: http
              containerPort: {{ .Values.container.port }}
              protocol: TCP
          {{- with .Values.livenessProbe }}
          livenessProbe:
            httpGet:
              path: {{ .path }}
              port: {{ .port }}
            initialDelaySeconds: {{ .initialDelaySeconds }}
            periodSeconds: {{ .periodSeconds }}
            timeoutSeconds: {{ .timeoutSeconds }}
            failureThreshold: {{ .failureThreshold }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            httpGet:
              path: {{ .path }}
              port: {{ .port }}
            initialDelaySeconds: {{ .initialDelaySeconds }}
            periodSeconds: {{ .periodSeconds }}
            timeoutSeconds: {{ .timeoutSeconds }}
            failureThreshold: {{ .failureThreshold }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if .Values.serviceMonitor.enabled }}
          metrics:
            - path: {{ .Values.serviceMonitor.path }}
              port: metrics
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- if .Values.priorityClassName }}
      priorityClassName: {{ .Values.priorityClassName }}
      {{- end }}
```

#### service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.container.port }}
      protocol: TCP
      name: http
    {{- if .Values.serviceMonitor.enabled }}
    - port: 9090
      targetPort: 9090
      protocol: TCP
      name: metrics
    {{- end }}
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
  {{- if and (eq .Values.service.type "LoadBalancer") .Values.service.loadBalancerIP }}
  loadBalancerIP: {{ .Values.service.loadBalancerIP }}
  {{- end }}
```

#### ingress.yaml

```
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-app.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

#### NOTES.txt

```
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "my-app.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "my-app.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "my-app.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "my-app.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

---

## 14.3 Helm 模板引擎

### 14.3.1 Go Template 基础语法

Helm 使用 Go Template 引擎，基础语法如下：

| 语法 | 说明 | 示例 |
|------|------|------|
| `{{ .Values.key }}` | 引用 values 中的值 | `{{ .Values.replicaCount }}` |
| `{{ .Release.Name }}` | 引用 Release 名称 | `{{ .Release.Name }}` |
| `{{ .Chart.Name }}` | 引用 Chart 名称 | `{{ .Chart.Name }}` |
| `{{- ... -}}` | 去除空白 | `{{- .Values.key -}}` |
| `{{ if }}...{{ else }}...{{ end }}` | 条件判断 | 见下方示例 |
| `{{ range .items }}...{{ end }}` | 循环遍历 | 见下方示例 |
| `{{ define "name" }}` | 定义模板 | 见下方示例 |
| `{{ include "name" . }}` | 引用模板 | 见下方示例 |
| `{{ template "name" . }}` | 引用模板（不传递上下文） | 见下方示例 |
| `{{ .Values.key \| func }}` | 使用管道函数 | `{{ .Values.tag \| default "latest" }}` |

### 14.3.2 常用模板函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `default` | 设置默认值 | `{{ .Values.tag \| default "latest" }}` |
| `quote` | 添加引号 | `{{ .Values.name \| quote }}` |
| `required` | 必填值检查 | `{{ .Values.required \| required "必须配置" }}` |
| `toYaml` | 转为 YAML | `{{ .Values.resources \| toYaml \| nindent 12 }}` |
| `indent` | 增加缩进 | `{{ .Values.config \| indent 2 }}` |
| `nindent` | 换行并缩进 | `{{ .Values.config \| nindent 4 }}` |
| `upper` | 转大写 | `{{ .Values.name \| upper }}` |
| `lower` | 转小写 | `{{ .Values.name \| lower }}` |
| `trim` | 去除两端空白 | `{{ .Values.name \| trim }}` |
| `contains` | 包含判断 | `{{ if contains "nginx" .Values.name }}` |
| `replace` | 字符串替换 | `{{ .Values.version \| replace "+" "_" }}` |
| `trunc` | 截断字符串 | `{{ .Values.name \| trunc 63 }}` |
| `join` | 数组拼接 | `{{ .Values.labels \| join "," }}` |
| `int` | 转整数 | `{{ .Values.count \| int }}` |
| `float64` | 转浮点数 | `{{ .Values.ratio \| float64 }}` |

### 14.3.3 模板使用示例

#### 条件判断

```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  ...
{{- end }}

{{- if eq .Values.config.mode "production" }}
  replicas: 5
{{- else if eq .Values.config.mode "staging" }}
  replicas: 2
{{- else }}
  replicas: 1
{{- end }}
```

#### 循环遍历

```
{{- range .Values.container.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

{{- range $key, $value := .Values.labels }}
{{ $key }}: {{ $value }}
{{- end }}
```

#### 定义和引用模板

```
{{- define "my-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end }}

# 引用模板
{{ include "my-app.labels" . | nindent 4 }}
```

#### 值类型转换

```
# 字符串转整数
port: {{ .Values.service.port | int }}

# 数组转 YAML
env:
  {{- toYaml .Values.container.env | nindent 2 }}

# 必填值
{{ $password := .Values.password | required "password is required" }}
```

---

## 14.4 values.yaml 多环境配置

### 14.4.1 多环境 values 文件

```
my-chart/
├── values.yaml              # 基础默认值
├── values/
│   ├── dev.yaml             # 开发环境
│   ├── staging.yaml         # 预发环境
│   └── production.yaml      # 生产环境
```

#### dev.yaml（开发环境）

```yaml
# values/dev.yaml
replicaCount: 1

image:
  tag: "latest"
  pullPolicy: Always

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi

service:
  type: NodePort

ingress:
  enabled: false

autoscaling:
  enabled: false

persistence:
  enabled: false

container:
  env:
    - name: NODE_ENV
      value: development
    - name: LOG_LEVEL
      value: debug
    - name: DB_HOST
      value: dev-db.example.com
```

#### staging.yaml（预发环境）

```yaml
# values/staging.yaml
replicaCount: 2

image:
  tag: "staging"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

service:
  type: ClusterIP

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: staging.example.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: false

persistence:
  enabled: true
  size: 5Gi

container:
  env:
    - name: NODE_ENV
      value: staging
    - name: LOG_LEVEL
      value: info
    - name: DB_HOST
      value: staging-db.example.com
```

#### production.yaml（生产环境）

```yaml
# values/production.yaml
replicaCount: 5

image:
  tag: "1.0.0"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi

service:
  type: ClusterIP

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: production.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - production.example.com
      secretName: production-tls

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

persistence:
  enabled: true
  size: 50Gi

container:
  env:
    - name: NODE_ENV
      value: production
    - name: LOG_LEVEL
      value: warn
    - name: DB_HOST
      value: prod-db.example.com
    - name: REDIS_HOST
      value: prod-redis.example.com

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
```

### 14.4.2 部署时指定 values 文件

```bash
# 使用默认 values 部署
helm install my-app ./my-chart

# 使用指定环境的 values 文件
helm install my-app ./my-chart -f values/dev.yaml

# 使用多个 values 文件（后者覆盖前者）
helm install my-app ./my-chart \
  -f values/production.yaml \
  -f values/secrets.yaml

# 直接通过命令行覆盖单个值
helm install my-app ./my-chart \
  -f values/production.yaml \
  --set replicaCount=10 \
  --set image.tag="v2.0.0"

# 使用 --set-string 强制字符串类型
helm install my-app ./my-chart \
  --set-string image.tag="1.0.0"

# 使用 --set-file 从文件读取值
helm install my-app ./my-chart \
  --set-file configmap.data=./config.json

# 使用 --set 嵌套值
helm install my-app ./my-chart \
  --set container.env[0].value=new-value
```

### 14.4.3 Secrets 管理

```yaml
# values/secrets.yaml（注意：不应提交到 Git！）
# 推荐使用 Sealed Secrets 或 External Secrets Operator
container:
  env:
    - name: DB_PASSWORD
      value: "super-secret-password"
    - name: API_KEY
      value: "sk-xxxxx"
```

```bash
# 方式一：通过命令行传入（不安全，仅临时使用）
helm install my-app ./my-chart \
  --set container.env[0].value=secret123

# 方式二：使用 Secret 资源引用
# 在模板中引用已存在的 Secret
```

```
# 在 deployment.yaml 中引用 Secret
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ include "my-app.fullname" . }}-secrets
        key: db-password
```

---

## 14.5 常用 Helm 仓库

### 14.5.1 添加和管理仓库

```bash
# 添加官方仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jetstack https://charts.jetstack.io
helm repo add elastic https://helm.elastic.co
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo add argo https://argoproj.github.io/argo-helm

# 更新所有仓库
helm repo update

# 查看已添加的仓库
helm repo list

# 移除仓库
helm repo remove old-repo

# 重命名仓库
helm repo rename old-name new-name
```

### 14.5.2 常用公共 Chart

| 仓库 | URL | 常用 Chart |
|------|-----|-----------|
| **Bitnami** | `https://charts.bitnami.com/bitnami` | nginx、redis、mysql、postgresql、mongodb、kafka |
| **Ingress NGINX** | `https://kubernetes.github.io/ingress-nginx` | ingress-nginx |
| **Prometheus Community** | `https://prometheus-community.github.io/helm-charts` | kube-prometheus-stack、prometheus、grafana |
| **Grafana** | `https://grafana.github.io/helm-charts` | loki、promtail、tempo、grafana |
| **Jetstack (Cert Manager)** | `https://charts.jetstack.io` | cert-manager |
| **Elastic** | `https://helm.elastic.co` | elasticsearch、kibana、filebeat |
| **Argo** | `https://argoproj.github.io/argo-helm` | argo-cd、argo-workflows |

### 14.5.3 搜索和查看 Chart

```bash
# 搜索 Chart
helm search repo nginx
helm search repo redis
helm search repo mysql --versions

# 查看 Chart 详细信息
helm show chart bitnami/redis
helm show values bitnami/redis
helm show all bitnami/redis

# 查看可用版本
helm search repo bitnami/redis --versions

# 下载 Chart 到本地
helm pull bitnami/redis
helm pull bitnami/redis --version 17.11.3
helm pull bitnami/redis --untar
```

---

## 14.6 实战演练

### 实战 1：安装 Helm

```bash
# macOS
brew install helm

# 使用官方脚本安装（通用）
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 从二进制包安装
# 访问 https://github.com/helm/helm/releases 下载对应平台的包
tar -xzf helm-v3.14.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

# 验证安装
helm version
# version.BuildInfo{Version:"v3.14.0", GitCommit:"xxx", GoVersion:"go1.21.5"}
```

### 实战 2：使用 Helm 安装 Nginx Ingress Controller

```bash
# 1. 添加 Ingress NGINX 仓库
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 2. 创建命名空间
kubectl create namespace ingress-nginx

# 3. 自定义配置 values 文件
cat > nginx-ingress-values.yaml << 'EOF'
controller:
  replicaCount: 2
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:xxx
  config:
    use-forwarded-headers: "true"
    proxy-body-size: "50m"
    server-tokens: "false"
    ssl-redirect: "true"
  metrics:
    enabled: true
    service:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "10254"
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "10254"
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 80
  admissionWebhooks:
    enabled: true
  ingressClassResource:
    enabled: true
    name: nginx
    default: true
EOF

# 4. 安装 Ingress NGINX
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  -f nginx-ingress-values.yaml

# 5. 查看部署状态
helm list -n ingress-nginx
kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller
kubectl get svc -n ingress-nginx

# 6. 获取负载均衡 IP
kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 实战 3：搜索和使用公共 Chart

```bash
# 1. 搜索 MySQL Chart
helm search repo mysql
helm show chart bitnami/mysql

# 2. 查看默认 values
helm show values bitnami/mysql

# 3. 创建 MySQL 配置
cat > mysql-values.yaml << 'EOF'
architecture: replication
auth:
  rootPassword: "root123"
  database: mydb
  username: appuser
  password: "app123"
primary:
  persistence:
    size: 20Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 1Gi
secondary:
  replicaCount: 2
  persistence:
    size: 20Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: "1"
      memory: 1Gi
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
EOF

# 4. 安装 MySQL
helm install my-mysql bitnami/mysql \
  --namespace database \
  --create-namespace \
  -f mysql-values.yaml

# 5. 搜索和安装 Redis
helm search repo redis

cat > redis-values.yaml << 'EOF'
architecture: sentinel
auth:
  enabled: false
master:
  persistence:
    size: 10Gi
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
sentinel:
  quorum: 2
  downAfterMilliseconds: 60000
  parallelSyncs: 1
  failoverTimeout: 180000
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
EOF

helm install my-redis bitnami/redis \
  --namespace database \
  -f redis-values.yaml

# 6. 验证部署
helm list -n database
kubectl get pods -n database
kubectl get svc -n database
```

### 实战 4：创建自定义 Chart

```bash
# 1. 使用 helm create 创建 Chart 骨架
helm create my-app-chart

# 2. 查看生成的目录结构
find my-app-chart -type f
# my-app-chart/
# ├── Chart.yaml
# ├── templates/
# │   ├── NOTES.txt
# │   ├── _helpers.tpl
# │   ├── deployment.yaml
# │   ├── ingress.yaml
# │   ├── service.yaml
# │   ├── serviceaccount.yaml
# │   └── tests/
# │       └── test-connection.yaml
# ├── values.yaml
# └── values.schema.json（可选的 values 校验）

# 3. 自定义 Chart.yaml
cat > my-app-chart/Chart.yaml << 'EOF'
apiVersion: v2
name: my-app-chart
description: A Helm chart for My Application
type: application
version: 1.0.0
appVersion: "1.0.0"
kubeVersion: ">=1.24.0"
maintainers:
  - name: Platform Team
    email: platform@example.com
keywords:
  - web
  - api
home: https://example.com
EOF

# 4. 简化 values.yaml（移除不需要的配置）
cat > my-app-chart/values.yaml << 'EOF'
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"

podSecurityContext: {}
securityContext: {}

service:
  type: ClusterIP
  port: 80
  annotations: {}

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: my-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

nodeSelector: {}
tolerations: []
affinity: {}
EOF

# 5. 添加自定义模板文件

# 添加 ConfigMap 模板
cat > my-app-chart/templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-app-chart.fullname" . }}-config
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
data:
  nginx.conf: |
    server {
      listen 80;
      server_name {{ .Values.ingress.hosts[0].host }};
      location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
EOF

# 添加 HPA 模板
cat > my-app-chart/templates/hpa.yaml << 'EOF'
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app-chart.fullname" . }}
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app-chart.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
EOF

# 添加 ServiceMonitor 模板（Prometheus 集成）
cat > my-app-chart/templates/servicemonitor.yaml << 'EOF'
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "my-app-chart.fullname" . }}
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "my-app-chart.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: {{ .Values.serviceMonitor.path }}
      interval: {{ .Values.serviceMonitor.interval }}
{{- end }}
EOF

# 6. 本地部署测试
helm install my-app ./my-app-chart --dry-run --debug
helm install my-app ./my-app-chart

# 7. 打包 Chart
helm package my-app-chart
# 生成 my-app-chart-1.0.0.tgz

# 8. 创建本地仓库并服务
mkdir -p local-repo
cp my-app-chart-1.0.0.tgz local-repo/
helm repo index local-repo/
# 生成 index.yaml
helm serve --repo-root local-repo/
# 默认监听 :8080
```

### 实战 5：使用 values.yaml 配置部署参数

```bash
# 1. 开发环境部署
helm install my-app-dev ./my-app-chart \
  --namespace dev \
  --create-namespace \
  -f values/dev.yaml

# 2. 预发环境部署
helm install my-app-stg ./my-app-chart \
  --namespace staging \
  --create-namespace \
  -f values/staging.yaml

# 3. 生产环境部署
helm install my-app-prod ./my-app-chart \
  --namespace production \
  --create-namespace \
  -f values/production.yaml

# 4. 覆盖特定参数
helm install my-app ./my-app-chart \
  --namespace production \
  -f values/production.yaml \
  --set replicaCount=10 \
  --set image.tag="v2.0.0" \
  --set container.env[0].value="new-value"

# 5. 使用 Kustomize 与 Helm 结合
# kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
helm template ./my-app-chart -f values/production.yaml | kubectl apply -f -

# 6. 使用环境变量
export IMAGE_TAG="v1.5.0"
helm install my-app ./my-app-chart \
  --set image.tag="${IMAGE_TAG}"
```

### 实战 6：Helm 的升级、回滚、卸载操作

```bash
# 1. 升级 Release
helm upgrade my-app ./my-app-chart
helm upgrade my-app ./my-app-chart -f values/production.yaml

# 使用 --install 参数自动安装（如果不存在）
helm upgrade --install my-app ./my-app-chart -f values/production.yaml

# 升级并等待就绪
helm upgrade my-app ./my-app-chart \
  --wait \
  --timeout 10m \
  --atomic  # 失败则自动回滚

# 2. 查看 Release 历史
helm history my-app
# REVISION  STATUS   CHART                 APP VERSION   DESCRIPTION
# 1        superseded   my-app-chart-1.0.0   1.0.0         Install complete
# 2        superseded   my-app-chart-1.0.1   1.0.1         Upgrade complete
# 3        deployed     my-app-chart-1.0.2   1.0.2         Upgrade complete

# 3. 回滚到指定版本
helm rollback my-app 2

# 回滚并等待就绪
helm rollback my-app 2 --wait --timeout 5m

# 4. 查看 Release 状态
helm status my-app
# NAME: my-app
# LAST DEPLOYED: 2024-01-15 10:30:00
# NAMESPACE: production
# STATUS: deployed
# REVISION: 3
# TEST SUITE: None
# NOTES: ...

# 5. 查看 Release 所有 values
helm get values my-app
helm get values my-app --all

# 6. 卸载 Release
helm uninstall my-app
helm uninstall my-app --keep-history  # 保留历史

# 7. 查看所有 Release
helm list
helm list --all
helm list --all-namespaces
helm list --namespace production
helm list --deleted  # 查看已卸载的 Release
```

### 实战 7：使用 helm template 预览渲染结果

```bash
# 1. 预览模板渲染结果（不部署）
helm template my-app ./my-app-chart

# 2. 使用指定 values 文件预览
helm template my-app ./my-app-chart -f values/production.yaml

# 3. 指定命名空间和 Release 名称
helm template my-app ./my-app-chart \
  --namespace production \
  --set fullnameOverride=my-custom-name

# 4. 输出到文件
helm template my-app ./my-app-chart -f values/production.yaml > rendered.yaml

# 5. 使用 --show-only 查看特定模板
helm template my-app ./my-app-chart \
  --show-only templates/deployment.yaml

# 6. 使用 --dry-run --debug 模拟安装过程
helm install my-app ./my-app-chart \
  --dry-run \
  --debug \
  -f values/production.yaml

# 7. 使用 diff 插件对比变更
# 安装 helm diff 插件
helm plugin install https://github.com/databus23/helm-diff

# 对比本地 Chart 与已部署版本的差异
helm diff upgrade my-app ./my-app-chart

# 对比两个版本之间的 values 差异
helm diff version my-app 2 3

# 8. 使用 kubeval 校验渲染结果
helm template my-app ./my-app-chart -f values/production.yaml | kubeval --strict
```

---

## 14.7 常见问题排查

### 14.7.1 模板渲染问题

| 问题 | 排查方法 |
|------|---------|
| `template: ... unexpected "}"` | 检查模板语法，确保 `{{ }}` 正确配对<br>使用 `helm template` 预览渲染结果 |
| `no required values` | 检查 `required` 函数引用的 values 是否存在<br>设置合理的默认值 |
| `wrong type for value` | 使用 `int`、`float64`、`string` 转换类型<br>使用 `--set-string` 强制字符串类型 |
| 缩进不正确 | 使用 `nindent` 替代 `indent`（自动换行）<br>检查 YAML 缩进是否符合预期 |

### 14.7.2 部署问题

| 问题 | 排查方法 |
|------|---------|
| Chart 依赖下载失败 | 检查网络连接和仓库 URL<br>使用 `helm dependency update` 更新依赖<br>检查 `Chart.yaml` 中 `dependencies` 配置 |
| 资源创建失败 | 使用 `helm install --dry-run --debug` 查看详情<br>检查模板渲染结果是否正确<br>检查 Kubernetes 资源配置是否合法 |
| Release 状态卡住 | 使用 `helm status` 查看状态<br>检查 Pod 日志和事件<br>考虑使用 `--wait` 和 `--timeout` 参数 |
| 升级失败回滚 | 使用 `helm history` 查看历史<br>使用 `helm rollback` 回滚到上一个版本<br>检查升级的变更内容 |

### 14.7.3 仓库管理问题

| 问题 | 排查方法 |
|------|---------|
| 仓库无法连接 | 检查网络和代理设置<br>使用 `helm repo update` 刷新<br>检查仓库 URL 是否正确 |
| Chart 版本找不到 | 使用 `helm search repo <name> --versions` 查看可用版本<br>确认版本号格式正确 |
| Chart 签名验证失败 | 检查 GPG 密钥<br>使用 `--verify` 参数验证签名 |

### 14.7.4 版本管理最佳实践

1. **使用语义化版本**：遵循 MAJOR.MINOR.PATCH 格式
2. **values.yaml 版本化**：不同环境使用不同 values 文件
3. **使用 `helm diff` 插件**：在升级前对比变更差异
4. **利用 `--wait` 和 `--atomic`**：确保部署成功后再标记完成
5. **保留 Release 历史**：避免使用 `--keep-history` 时丢失关键信息
6. **CI/CD 集成**：在流水线中使用 `helm template` + `kubeval` 校验

---

## 14.8 章节小结

### 知识图谱

```
Helm 包管理
├── 核心概念
│   ├── Chart（包/模板）
│   ├── Release（部署实例）
│   └── Repository（仓库）
├── 三大功能
│   ├── Install（安装）
│   ├── Upgrade（升级）
│   └── Uninstall（卸载）
├── Chart 结构
│   ├── Chart.yaml（元信息）
│   ├── values.yaml（配置值）
│   ├── templates/（模板）
│   │   ├── _helpers.tpl（公共模板）
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── NOTES.txt
│   └── charts/（子 Chart）
├── 模板引擎
│   ├── Go Template 语法
│   ├── 常用函数
│   └── 条件/循环/定义模板
├── 多环境配置
│   ├── values/dev.yaml
│   ├── values/staging.yaml
│   ├── values/production.yaml
│   └── --set / -f 参数覆盖
└── 仓库管理
    ├── Bitnami
    ├── Ingress-nginx
    ├── Prometheus Community
    └── Grafana
```

### 核心要点

1. **Helm 核心概念**：Chart 是模板包，Release 是部署实例，Repository 是分发仓库，三者关系类似 Docker 的 Image、Container、Registry
2. **Helm vs kubectl**：Helm 是 kubectl 的增强，提供模板化、版本管理、多环境配置能力，适用于应用级部署
3. **Chart 目录结构**：`Chart.yaml`（元信息）、`values.yaml`（默认配置）、`templates/`（模板文件）是核心三要素
4. **模板引擎**：掌握 `{{ }}` 语法、`include`/`template` 引用、`range`/`if` 控制流、管道函数（`default`、`toYaml`、`nindent` 等）
5. **多环境配置**：一套模板 + 多套 values 文件，通过 `-f` 和 `--set` 参数灵活切换
6. **版本管理**：`helm history` 查看历史、`helm rollback` 回滚、`helm diff` 对比变更

### 下一章预告

第15章将学习 Kubernetes 集群管理与运维实战，包括集群扩缩容、节点管理、资源监控等生产级运维操作。
