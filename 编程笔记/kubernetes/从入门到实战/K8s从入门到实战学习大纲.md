# Kubernetes 从入门到实战：完整知识学习大纲

## 一、为什么需要系统化学习 K8s

Kubernetes（K8s）已经成为容器编排的事实标准，几乎所有云原生应用的部署和管理都离不开它。但 K8s 的学习曲线陡峭，概念多、组件复杂、上手困难。

本大纲采用**"理论 + 实战"双轨模式**，每章都包含核心原理讲解和可操作的实战练习，帮助你从零基础到能够独立部署和管理生产级 K8s 集群。

---

## 二、学习路线总览

```text
第 1 章：K8s 概述与架构理解
  ↓
第 2 章：环境搭建（Minikube / Kind 实战）
  ↓
第 3 章：Pod 核心概念与实战
  ↓
第 4 章：Deployment 与应用部署
  ↓
第 5 章：Service 与网络通信
  ↓
第 6 章：ConfigMap 与 Secret 管理
  ↓
第 7 章：Volume 与持久化存储
  ↓
第 8 章：Namespace 与资源管理
  ↓
第 9 章：调度机制与节点管理
  ↓
第 10 章：Ingress 与流量路由
  ↓
第 11 章：控制器进阶（DaemonSet / StatefulSet / Job / CronJob）
  ↓
第 12 章：RBAC 权限与安全
  ↓
第 13 章：监控、日志与可观测性
  ↓
第 14 章：Helm 包管理与应用部署
  ↓
第 15 章：综合实战：完整微服务项目部署
```

---

## 三、各章节详细规划

### 第 1 章：K8s 概述与架构理解

**目标**：理解 K8s 是什么、为什么需要它、核心架构是什么。

**核心内容**：
- 容器化与容器编排的演进历史
- K8s 核心概念：Cluster、Node、Pod、Container
- K8s 架构：Control Plane + Worker Node
- 核心组件详解：API Server、etcd、Scheduler、Controller Manager、Kubelet、Kube-Proxy
- K8s 工作流程全景图
- kubectl 基础使用

**实战**：
- 安装 kubectl 并配置自动补全
- 理解 kubectl 命令的资源操作模式

---

### 第 2 章：环境搭建

**目标**：搭建可用的 K8s 学习环境。

**核心内容**：
- 方案对比：Minikube vs Kind vs k3s vs Docker Desktop
- Minikube 安装与配置（macOS / Linux）
- Kind（Kubernetes in Docker）安装与配置
- 集群状态验证
- kubectl 与集群的交互

**实战**：
- 使用 Minikube 搭建单节点集群
- 使用 Kind 搭建多节点集群
- 验证集群状态和基本功能
- 部署第一个 Hello World 应用

---

### 第 3 章：Pod 核心概念与实战

**目标**：掌握 K8s 中最核心的部署单元——Pod。

**核心内容**：
- Pod 的定义与特点
- Pod 的生命周期与状态（Pending / Running / Succeeded / Failed / Unknown）
- Pod 的 YAML 结构详解
- Pod 中的容器模式（sidecar、init container）
- Pod 的资源限制（requests / limits）
- Pod 的健康检查（liveness / readiness / startup probe）
- Pod 的重启策略（Always / OnFailure / Never）
- Pod 之间的通信

**实战**：
- 创建单个 Pod（nginx）
- 创建多容器 Pod（主容器 + sidecar 日志收集）
- 配置资源限制并观察调度
- 配置健康检查并模拟故障场景
- 使用 kubectl logs / exec / describe 调试 Pod

---

### 第 4 章：Deployment 与应用部署

**目标**：掌握 K8s 中最常用的应用控制器——Deployment。

**核心内容**：
- Deployment 的核心功能：声明式更新、滚动升级、版本回滚
- Deployment 与 ReplicaSet 的关系
- Deployment YAML 结构详解
- 滚动更新策略（RollingUpdate / Recreate）
- 回滚与版本历史
- 扩缩容操作
- Pod 模板变更触发滚动更新
- 部署策略选择

**实战**：
- 部署 Nginx Deployment
- 手动扩缩容副本数
- 配置滚动更新策略（maxUnavailable / maxSurge）
- 触发版本更新并观察滚动过程
- 执行回滚操作恢复到上一个版本
- 实现蓝绿部署和金丝雀发布

---

### 第 5 章：Service 与网络通信

**目标**：掌握 K8s 中为 Pod 提供稳定访问入口的 Service。

**核心内容**：
- Service 的核心作用：Pod IP 会变，Service IP 不变
- Service 的类型：ClusterIP / NodePort / LoadBalancer / ExternalName
- Service YAML 结构详解
- Service 与 Pod 的连接：Label Selector
- Service 的会话亲和性（Session Affinity）
- 无头服务（Headless Service）
- K8s 网络模型：CNI、Overlay / Underlay
- kube-proxy 的工作模式（iptables / ipvs / userspace）

**实战**：
- 创建 ClusterIP Service 暴露 Pod
- 创建 NodePort Service 外部访问
- 观察 Service 的 Endpoints 变化
- 实现 Service 之间的互相访问
- 使用 Headless Service 实现 StatefulSet 服务发现

---

### 第 6 章：ConfigMap 与 Secret 管理

**目标**：掌握配置管理的两种核心方式。

**核心内容**：
- ConfigMap 的使用场景
- ConfigMap 的三种创建方式：直接值 / 配置文件 / 目录
- ConfigMap 的 Volume 挂载 vs 环境变量注入
- Secret 的使用场景与类型
- Secret 的编码方式（Base64）
- Secret 的创建与使用
- K8s 中的配置管理最佳实践

**实战**：
- 创建 ConfigMap 并注入到 Pod（环境变量方式）
- 创建 ConfigMap 并挂载为 Volume
- 创建 Secret 存储密码
- 创建 TLS Secret
- 使用 kubectl create configmap/secret 快速创建

---

### 第 7 章：Volume 与持久化存储

**目标**：理解 K8s 中数据持久化的解决方案。

**核心内容**：
- Volume 的生命周期
- 常见 Volume 类型：emptyDir / hostPath / configMap / secret / nfs / hostPath
- PersistentVolume（PV）与 PersistentVolumeClaim（PVC）
- PV 的生命周期：Provisioning / Binding / Using / Releasing / Available
- PV 的访问模式（ReadWriteOnce / ReadOnlyMany / ReadWriteMany）
- PV 的回收策略（Retain / Delete / Recycle）
- StorageClass 与动态供给
- CSI（Container Storage Interface）

**实战**：
- 使用 emptyDir 实现临时共享存储
- 使用 hostPath 实现节点本地持久化
- 创建 NFS PV 并使用 PVC 申请
- 创建 StorageClass 实现动态供给
- 部署 MySQL 并配置持久化存储
- 模拟 Pod 重建后数据仍可访问

---

### 第 8 章：Namespace 与资源管理

**目标**：掌握 K8s 的多租户隔离和资源管理。

**核心内容**：
- Namespace 的作用与使用场景
- Namespace 中的资源隔离
- ResourceQuota：命名空间级资源配额
- LimitRange：默认资源限制配置
- 标签（Label）与注解（Annotation）
- 标签选择器的使用
- 资源清理策略

**实战**：
- 创建多个 Namespace（dev / staging / production）
- 为 dev 命名空间设置 ResourceQuota
- 为命名空间配置 LimitRange
- 使用 Label 对资源进行分类管理
- 使用 kubectl 按 Namespace 隔离操作

---

### 第 9 章：调度机制与节点管理

**目标**：理解 K8s 的调度决策过程并进行节点管理。

**核心内容**：
- Scheduler 的调度流程
- 调度器的决策因素：资源请求 / 亲和性 / 反亲和性 / 污点容忍
- Node 亲和性（Node Affinity）
- Pod 亲和性 / 反亲和性（Pod Affinity / Anti-Affinity）
- 污点（Taint）与容忍（Toleration）
- Node 的管理：标记 / 状态维护
- 节点维护：cordon / drain / uncordon

**实战**：
- 设置 Node 标签并配置 Node Affinity
- 使用 Pod Anti-Affinity 将 Pod 分散到不同节点
- 设置 Taint 并配置 Toleration
- 维护节点：drain 节点进行升级
- 使用 nodeSelector 限制 Pod 调度

---

### 第 10 章：Ingress 与流量路由

**目标**：掌握 K8s 中 HTTP(S) 流量的管理方式。

**核心内容**：
- Ingress 的作用
- Ingress Controller 概述（Nginx / Traefik / HAProxy / Kong）
- Ingress YAML 结构详解
- 路径路由与主机路由
- TLS 终止配置
- 路径重写
- 默认后端
- Ingress 与 Service 的关系

**实战**：
- 部署 Nginx Ingress Controller
- 创建路径路由 Ingress
- 创建主机路由 Ingress
- 配置 TLS 证书
- 实现路径重写规则
- 使用 Ingress 实现蓝绿部署流量切换

---

### 第 11 章：控制器进阶

**目标**：掌握 K8s 中其他重要的控制器。

**核心内容**：
- DaemonSet：每个 Node 上运行一个 Pod
- DaemonSet 的典型应用：日志收集 / 监控 Agent / 网络插件
- StatefulSet：有状态应用部署
- StatefulSet 的特性：稳定标识 / 有序部署 / 稳定存储
- Job：一次性任务
- CronJob：定时任务
- 控制器选择指南

**实战**：
- 部署 DaemonSet 运行 Fluentd 日志收集
- 部署 StatefulSet 运行 Redis Cluster
- 创建 Job 执行一次性数据迁移
- 创建 CronJob 定时执行备份任务
- 对比 Deployment / StatefulSet / DaemonSet 的适用场景

---

### 第 12 章：RBAC 权限与安全

**目标**：掌握 K8s 的角色访问控制。

**核心内容**：
- RBAC 的核心资源：Role / ClusterRole / RoleBinding / ClusterRoleBinding
- ServiceAccount
- RBAC 的作用域：Namespace 级 / Cluster 级
- 常见权限场景：只读 / 命名空间管理员 / 集群管理员
- 最小权限原则
- RBAC YAML 编写

**实战**：
- 创建只读 Role 和 RoleBinding
- 创建命名空间管理员权限
- 创建 ServiceAccount 并绑定权限
- 使用 kubectl auth can-i 验证权限
- 创建跨命名空间的 ClusterRole

---

### 第 13 章：监控、日志与可观测性

**目标**：构建 K8s 的可观测性体系。

**核心内容**：
- 可观测性三大支柱：Metrics / Logs / Traces
- Metrics：Prometheus 架构与部署
- Grafana 可视化仪表盘
- 日志收集：ELK / Loki / Fluentd
- 分布式追踪：Jaeger / Zipkin
- kube-state-metrics
- 常用监控指标

**实战**：
- 部署 Prometheus + Grafana
- 配置 Pod 的资源 Metrics 采集
- 部署 Loki 实现日志聚合
- 配置 Grafana 仪表盘
- 实现简单的告警规则
- 查看 Pod 日志并分析

---

### 第 14 章：Helm 包管理与应用部署

**目标**：使用 Helm 简化 K8s 应用管理。

**核心内容**：
- Helm 的核心概念：Chart / Release / Repository
- Helm vs kubectl 的区别
- Helm 的模板引擎
- Chart 的目录结构
- values.yaml 的使用
- 常用 Helm 仓库

**实战**：
- 安装 Helm
- 使用 Helm 安装 Nginx
- 搜索和使用公共 Chart
- 创建自定义 Chart
- 使用 values.yaml 配置部署参数
- Helm 的升级、回滚、卸载操作

---

### 第 15 章：综合实战：完整微服务项目部署

**目标**：将所学知识综合运用，部署一个完整的微服务项目。

**实战项目**：部署一个包含 API 网关 + 后端服务 + MySQL + Redis + Nginx 的完整 Web 应用。

**覆盖知识点**：
- 使用 Namespace 隔离不同服务
- 使用 ConfigMap 和 Secret 管理配置
- 使用 Deployment 部署无状态服务
- 使用 StatefulSet 部署 MySQL
- 使用 Service 实现服务间通信
- 使用 Ingress 实现外部访问
- 使用 Volume 实现数据持久化
- 使用 HPA 实现自动扩缩容
- 使用 RBAC 配置权限
- 使用 Metrics 进行监控

**实战步骤**：
1. 创建项目 Namespace
2. 部署 MySQL（StatefulSet + PV + Service）
3. 部署 Redis（Deployment + Service）
4. 准备 ConfigMap 和 Secret
5. 部署后端 API 服务（Deployment + Service）
6. 配置 HPA 自动扩缩容
7. 部署 Nginx 前端（Deployment + Service）
8. 配置 Ingress 实现路由
9. 配置 RBAC 权限
10. 部署 Prometheus + Grafana 监控
11. 验证整体服务可用性
12. 故障排查与恢复演练

---

## 四、学习建议

### 4.1 前置知识

- 基本的 Linux 命令行操作
- Docker 容器基础（镜像、容器、Dockerfile）
- 网络基础（TCP/IP、HTTP、端口）
- YAML 语法

### 4.2 学习顺序

建议严格按照大纲顺序学习，每完成一章都要完成对应的实战练习。K8s 的知识依赖关系强，跳过前面的章节会导致后续难以理解。

### 4.3 学习方法

- **动手优先**：每一个知识点都要亲手操作，不要只看文档。
- **报错是最好的老师**：遇到错误先自己排查，这是掌握 K8s 的最佳方式。
- **多看 YAML**：阅读和编写 YAML 是 K8s 的核心技能。
- **理解原理**：不仅要会操作，还要理解背后的原理。
- **模拟故障**：主动制造故障并排查，深度理解 K8s 的自愈能力。

### 4.4 常用命令速查

```bash
# 集群信息
kubectl cluster-info
kubectl get nodes

# Pod 操作
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs -f <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# 部署操作
kubectl create deployment <name> --image=<image>
kubectl scale deployment <name> --replicas=<n>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# 服务操作
kubectl get svc
kubectl expose deployment <name> --type=NodePort --port=80

# 配置操作
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=password=xxx

# 调试
kubectl top nodes
kubectl top pods
kubectl get events --sort-by=.lastTimestamp
```
