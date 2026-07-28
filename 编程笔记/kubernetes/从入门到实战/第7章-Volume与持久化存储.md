# 第7章：Volume 与持久化存储

## 学习目标

完成本章学习后，你将能够：

1. 理解 Volume 的生命周期及与 Pod 的绑定关系
2. 掌握常见 Volume 类型（emptyDir、hostPath、configMap、secret、nfs、pvc）的使用场景
3. 理解 PersistentVolume（PV）与 PersistentVolumeClaim（PVC）的概念和关系
4. 掌握 PV 的生命周期状态流转
5. 理解 PV 的访问模式与回收策略
6. 掌握 StorageClass 的动态供给机制
7. 了解 CSI（Container Storage Interface）的作用
8. 通过实战掌握持久化存储的完整配置流程

---

## 7.1 Volume 概述

### 7.1.1 为什么需要 Volume

容器的文件系统天生是**临时**的：

- 容器销毁时，其内部写入的文件和数据会丢失
- 容器之间无法直接共享数据
- 容器重启后数据不会保留

```
没有 Volume 的问题：
┌──────────────────────────────────────┐
│  Pod                                  │
│  ┌────────────┐  ┌────────────┐      │
│  │ Container A │  │ Container B │      │
│  │ 数据写死    │  │ 数据不可见  │      │
│  └────────────┘  └────────────┘      │
│       ↑                ↑              │
│       └── 重启/销毁后数据丢失 ──┘      │
└──────────────────────────────────────┘
```

**Volume 解决方案**：

```
使用 Volume 后：
┌──────────────────────────────────────┐
│  Pod                                  │
│  ┌────────────┐  ┌────────────┐      │
│  │ Container A │  │ Container B │      │
│  └─────┬──────┘  └─────┬──────┘      │
│        │    读写共享     │             │
│        └───────┬───────┘             │
│                ▼                      │
│         ┌──────────────┐              │
│         │   Volume      │              │
│         │  (共享存储)   │              │
│         └──────┬───────┘              │
│                │                      │
│        ┌───────┴───────┐              │
│        ▼               ▼              │
│   临时存储          持久存储           │
│   (emptyDir)       (PV/PVC)          │
└──────────────────────────────────────┘
```

### 7.1.2 Volume 的生命周期

Volume 与 Pod 的生命周期紧密绑定：

```
Pod 生命周期:
  创建 ──▶ 运行 ──▶ 销毁
    │        │        │
    ▼        ▼        ▼
  Volume   Volume   Volume
  挂载     使用     卸载/清除
  创建     读写     数据丢失（除 PV）
```

| Volume 类型 | Pod 重建后数据 | Pod 销毁后数据 |
|-------------|---------------|---------------|
| emptyDir | ❌ 丢失 | ❌ 丢失 |
| hostPath | ✅ 保留（节点本地） | ✅ 保留（节点本地） |
| configMap/secret | ✅ 保留（ConfigMap/Secret 仍存在） | ✅ 保留 |
| nfs/cifs | ✅ 保留 | ✅ 保留 |
| persistentVolumeClaim | ✅ 保留 | ✅ 保留 |

---

## 7.2 常见 Volume 类型详解

### 7.2.1 emptyDir：临时存储

#### 特点

- Pod 启动时创建，Pod 销毁时删除
- 容器重启后**数据清空**（但 Volume 保留）
- 用于临时数据、容器间共享

#### 使用场景

- 多容器间临时共享数据（如日志传递）
- 计算中间结果缓存
- 临时工作目录

#### YAML 示例

```yaml
# emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
  labels:
    app: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date > /shared/data.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "while true; do cat /shared/data.txt; sleep 5; done"]
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
  volumes:
    - name: shared-volume
      emptyDir: {}
  restartPolicy: Never
```

#### emptyDir 的 medium 选项

```yaml
volumes:
  - name: shared-volume
    emptyDir:
      medium: Memory              # 使用内存作为临时存储
      sizeLimit: "100Mi"          # 限制大小（K8s 1.10+）
```

| medium 选项 | 说明 | 性能 | 持久性 |
|-------------|------|------|--------|
| 未设置（默认） | 使用磁盘 | 较快 | 节点重启后仍存在 |
| Memory | 使用 tmpfs | 最快 | 节点重启后丢失 |

### 7.2.2 hostPath：节点本地存储

#### 特点

- 使用 Pod 所在节点的本地文件系统
- Pod 跨节点调度时**数据不可迁移**
- 适合单机测试和特定场景

#### 使用场景

- 需要访问节点上的文件（如 Docker socket）
- 单机持久化（不推荐多节点场景）
- 日志文件直接写入节点

#### YAML 示例

```yaml
# hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: host-data
          mountPath: /data
        - name: docker-sock
          mountPath: /var/run/docker.sock
  volumes:
    - name: host-data
      hostPath:
        path: /tmp/k8s-hostpath-data      # 节点上的路径
        type: DirectoryOrCreate            # 可选类型
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
  restartPolicy: Never
```

#### hostPath 的 type 选项

| 类型 | 说明 |
|------|------|
| 未设置 | 不做任何检查 |
| Directory | 必须是已存在的目录 |
| DirectoryOrCreate | 目录不存在时自动创建 |
| File | 必须是已存在的文件 |
| FileOrCreate | 文件不存在时自动创建 |
| Socket | 必须是已存在的 UNIX Socket |
| CharDevice | 必须是已存在的字符设备 |

#### 风险与限制

- **数据隔离**：多个 Pod 使用同一 hostPath 可能相互影响
- **节点依赖**：Pod 绑定到特定节点，无法自由调度
- **安全风险**：可访问节点任意路径，存在安全隐患

### 7.2.3 configMap / secret：配置注入

#### configMap Volume

```yaml
# configmap-volume.yaml
volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
        - key: nginx.conf
          path: nginx.conf
      defaultMode: 0644
```

#### secret Volume

```yaml
# secret-volume.yaml
volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
      items:
        - key: password
          path: db-password
      defaultMode: 0400
```

#### 挂载特点

- ConfigMap/Secret 以文件形式挂载
- 修改源对象后自动热更新（约 60 秒延迟）
- 使用 `subPath` 可避免全量更新

### 7.2.4 nfs / cifs：网络文件系统

#### NFS（Network File System）

```yaml
# nfs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: nfs-volume
          mountPath: /data
  volumes:
    - name: nfs-volume
      nfs:
        server: 192.168.1.100      # NFS 服务器地址
        path: /exported/data         # NFS 导出路径
        readOnly: false              # 是否只读
  restartPolicy: Never
```

#### CIFS（Common Internet File System，即 SMB）

```yaml
# cifs-pod.yaml
volumes:
  - name: cifs-volume
    cifs:
      server: 192.168.1.200         # SMB 服务器地址
      share: /data                  # 共享目录
      username: admin               # SMB 用户名
      password: secret              # SMB 密码
      # 或使用 secretName 引用 Secret
      # secretName: smb-credentials
```

#### 网络文件系统特点

- 多 Pod 可同时读写同一卷
- 数据持久化，与 Pod 生命周期无关
- 需要网络连接到 NFS/SMB 服务器
- 存在单点故障风险

### 7.2.5 persistentVolumeClaim：使用 PV

这是最核心的持久化存储方式，将在后续章节详细讲解。

```yaml
# pvc-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
    - name: app
      image: mysql:8.0
      volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumes:
    - name: mysql-data
      persistentVolumeClaim:
        claimName: mysql-pvc
  restartPolicy: Never
```

---

## 7.3 PersistentVolume 与 PersistentVolumeClaim

### 7.3.1 PV 与 PVC 的概念

```
┌─────────────────────────────────────────────────────┐
│                    Kubernetes                       │
│                                                     │
│  ┌─────────────┐         ┌─────────────┐           │
│  │  PVC        │         │  PV         │           │
│  │  (Claim)    │ ──绑定──▶│  (Volume)   │           │
│  │  使用者     │         │  提供者     │           │
│  └─────────────┘         └──────┬──────┘           │
│                                  │                  │
│                                  ▼                  │
│                         ┌─────────────┐            │
│                         │  存储后端    │            │
│                         │  NFS/云盘等  │            │
│                         └─────────────┘            │
│                                                     │
│  关系：PVC 申请资源，PV 提供资源，两者通过绑定关联   │
└─────────────────────────────────────────────────────┘
```

#### PersistentVolume（PV）

- 集群中的一块**存储资源**
- 由管理员创建或由 StorageClass 动态供给
- 独立于 Pod 生命周期
- 包含容量、访问模式、回收策略等信息

#### PersistentVolumeClaim（PVC）

- 用户的**存储请求**
- 声明需要的存储大小、访问模式
- K8s 自动将 PVC 绑定到满足条件的 PV
- PVC 必须与 PV 在同一命名空间（但 PV 是集群级资源）

### 7.3.2 PV 的生命周期

```
┌──────────┐    Binding    ┌──────────┐    使用    ┌──────────┐
│Provisioning│────────────▶│  Bound   │─────────▶│  Using   │
│ (供给)     │              │ (绑定)   │          │ (使用)   │
└──────────┘              └──────────┘          └────┬─────┘
                                                      │
                                           删除 PVC    │
                                                      ▼
                                               ┌──────────┐
                                               │Releasing │
                                               │ (释放)   │
                                               └────┬─────┘
                                                    │
                                     ┌──────────────┼──────────────┐
                                     ▼              ▼              ▼
                              ┌──────────┐  ┌──────────┐  ┌──────────┐
                              │Available │  │  Deleted │  │  Failed  │
                              │ (可用)   │  │  (删除)  │  │  (失败)  │
                              └──────────┘  └──────────┘  └──────────┘
```

| 阶段 | 说明 |
|------|------|
| Provisioning | PV 创建中（管理员手动或 StorageClass 动态创建） |
| Binding | PVC 与 PV 正在绑定 |
| Bound | PVC 已成功绑定到 PV |
| Using | PV 正被 Pod 使用 |
| Releasing | PVC 已删除，PV 正在清理数据 |
| Available | PV 清理完成，可被新的 PVC 使用 |
| Deleting | PV 正在被删除 |
| Failed | PV 回收策略执行失败 |

### 7.3.3 PV 访问模式

| 访问模式 | 说明 | 适用场景 |
|---------|------|---------|
| ReadWriteOnce (RWO) | 单个节点可读写 | 单机应用（如 MySQL） |
| ReadOnlyMany (ROX) | 多节点只读 | 共享只读数据（如静态资源） |
| ReadWriteMany (RWX) | 多节点可读写 | 多 Pod 共享写入 |
| ReadWriteOncePod (RWOP) | 单个 Pod 可读写 | K8s 1.22+，更严格的隔离 |

**访问模式对比**：

```
RWO (ReadWriteOnce):
  Node A ──▶ 读写 ──▶ PV
  Node B ──▶ 只读 ◀── PV  （或不可访问）

RWX (ReadWriteMany):
  Node A ──▶ 读写 ──▶ PV
  Node B ──▶ 读写 ──▶ PV
  Node C ──▶ 读写 ──▶ PV

ROX (ReadOnlyMany):
  Node A ──▶ 只读 ──▶ PV
  Node B ──▶ 只读 ──▶ PV
  Node C ──▶ 只读 ──▶ PV
```

### 7.3.4 PV 回收策略

回收策略决定 PVC 删除后 PV 如何处理数据：

| 策略 | 说明 | 行为 |
|------|------|------|
| Retain | 保留数据 | PV 保留，需手动清理才能重用 |
| Delete | 删除数据 | 自动删除 PV 和后端存储 |
| Recycle | 回收利用 | 清除数据，PV 可立即重用（已废弃） |

```
Retain 流程：
  PVC 删除 ──▶ PV 变为 Released ──▶ 手动删除数据 ──▶ PV 变为 Available

Delete 流程：
  PVC 删除 ──▶ PV 变为 Released ──▶ 自动删除后端存储 ──▶ PV 变为 Deleted
```

**各存储后端支持的回收策略**：

| 存储类型 | Retain | Delete | Recycle |
|---------|--------|--------|---------|
| NFS | ✅ | ❌ | ✅ |
| CIFS | ✅ | ❌ | ✅ |
| HostPath | ✅ | ✅ | ✅ |
| AWS EBS | ✅ | ✅ | ❌ |
| GCE PD | ✅ | ✅ | ❌ |
| Azure Disk | ✅ | ✅ | ❌ |
| CSI 驱动 | ✅ | ✅ | ❌ |

### 7.3.5 创建 PV 示例

**静态创建 NFS PV**：

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    app: shared-storage
    type: nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  nfs:
    server: 192.168.1.100
    path: /exported/k8s-data
```

**静态创建 HostPath PV**：

```yaml
# hostpath-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
  labels:
    app: local-storage
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: manual
  hostPath:
    path: /data/k8s-pv
    type: DirectoryOrCreate
```

### 7.3.6 创建 PVC 示例

```yaml
# mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: manual
  volumeName: nfs-pv              # 可选：指定绑定的 PV
```

**PVC 与 PV 绑定流程**：

```bash
# 创建 PV
kubectl apply -f nfs-pv.yaml

# 创建 PVC
kubectl apply -f mysql-pvc.yaml

# 查看绑定状态
kubectl get pv
kubectl get pvc
# PVC 状态应为 Bound

# 查看详细信息
kubectl describe pv nfs-pv
kubectl describe pvc mysql-pvc
```

---

## 7.4 StorageClass 与动态供给

### 7.4.1 动态供给概述

StorageClass 定义了**如何动态创建 PV**。当用户创建 PVC 时，系统根据 StorageClass 自动创建对应的 PV。

```
静态供给（传统方式）:
  管理员手动创建 PV ──▶ 用户创建 PVC ──▶ 手动绑定

动态供给（StorageClass）:
  用户创建 PVC ──▶ StorageClass 自动创建 PV ──▶ 自动绑定
```

### 7.4.2 创建 StorageClass

**示例：基于 NFS 的 StorageClass**

```yaml
# nfs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: kubernetes.io/nfs-provisioner
parameters:
  server: 192.168.1.100          # NFS 服务器
  path: /exported/k8s            # NFS 导出根路径
  archiveOnDelete: "true"        # 删除 PVC 时是否归档
reclaimPolicy: Delete            # 默认回收策略
allowVolumeExpansion: true       # 允许扩容
```

**示例：基于云厂商的 StorageClass**

```yaml
# aws-ebs-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zones: us-east-1a
reclaimPolicy: Delete
allowVolumeExpansion: true

# gce-pd-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
reclaimPolicy: Delete

# azure-disk-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: kubernetes.io/azure-disk
parameters:
  storageAccountType: Premium_LRS
  cachingmode: ReadOnly
reclaimPolicy: Delete
```

### 7.4.3 使用 StorageClass 的 PVC

```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client    # 指定 StorageClass
  resources:
    requests:
      storage: 5Gi
```

### 7.4.4 设置默认 StorageClass

```bash
# 标记默认 StorageClass
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class": "true"}}}'

# 取消默认标记
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

```yaml
# 标记默认的 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/nfs-provisioner
# ...
```

### 7.4.5 动态供给工作流程

```
1. 用户创建 PVC ──▶ kubectl apply -f pvc.yaml
        │
        ▼
2. K8s 查找 PVC 指定的 StorageClass
        │
        ▼
3. K8s 调用 StorageClass 的 provisioner
        │
        ▼
4. Provisioner 在后端存储创建卷
        │
        ▼
5. Provisioner 创建对应的 PV 对象
        │
        ▼
6. K8s 将 PVC 绑定到新创建的 PV
        │
        ▼
7. 用户在 Pod 中使用 PVC
```

---

## 7.5 CSI（Container Storage Interface）简介

### 7.5.1 CSI 概述

CSI 是 K8s 定义的**存储驱动标准接口**，使存储厂商可以开发独立于 K8s 核心的存储插件。

```
┌───────────────────────────────────────────────┐
│                  Kubernetes                   │
│                                               │
│  ┌─────────────┐   ┌─────────────────────┐   │
│  │   Pod/PVC    │   │  PV Controller     │   │
│  └──────┬──────┘   └─────────┬───────────┘   │
│         │                    │                │
│         ▼                    ▼                │
│  ┌─────────────────────────────────────┐     │
│  │           CSI Plugin (gRPC)         │     │
│  │  ┌────────────┐  ┌────────────┐    │     │
│  │  │  Controller │  │   Node      │    │     │
│  │  │  Plugin    │  │   Plugin    │    │     │
│  │  └────────────┘  └────────────┘    │     │
│  └─────────────────────────────────────┘     │
│                      │                        │
│                      ▼                        │
│  ┌─────────────────────────────────────┐     │
│  │           存储后端                   │     │
│  │  (NFS/Ceph/AWS EBS/本地磁盘等)      │     │
│  └─────────────────────────────────────┘     │
└───────────────────────────────────────────────┘
```

### 7.5.2 CSI 的架构

| 组件 | 说明 | 职责 |
|------|------|------|
| Controller Plugin | 控制器插件 | 创建/删除卷、扩容卷、快照 |
| Node Plugin | 节点插件 | 挂载/卸载卷到节点 |
| Identity Plugin | 身份插件 | 返回插件信息、健康检查 |

### 7.5.3 CSI 驱动示例

```yaml
# csi-example.yaml - 以 AWS EBS CSI 驱动为例
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# csi-snapshotclass.yaml - 创建快照类
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete
```

### 7.5.4 CSI 的优势

- **解耦**：存储驱动与 K8s 版本解耦
- **标准化**：统一的存储接口标准
- **可扩展**：支持更多存储功能（快照、扩容、克隆）
- **社区活跃**：主流存储厂商都已提供 CSI 驱动

---

## 7.6 实战演练

### 实战1：使用 emptyDir 实现多容器间的临时共享存储

**步骤 1**：创建 Pod 配置

```yaml
# emptydir-share.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-share-pod
  labels:
    app: emptydir-demo
spec:
  containers:
    - name: producer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"[producer] $(date) - data-$(date +%s)\" > /shared/data.log; sleep 2; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
    - name: consumer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do cat /shared/data.log 2>/dev/null || echo 'waiting...'; sleep 2; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /shared
  volumes:
    - name: shared-data
      emptyDir:
        medium: ""
        sizeLimit: "100Mi"
  restartPolicy: Never
```

**步骤 2**：部署并验证

```bash
kubectl apply -f emptydir-share.yaml
kubectl get pod emptydir-share-pod

# 查看两个容器的日志
kubectl logs emptydir-share-pod -c producer
kubectl logs emptydir-share-pod -c consumer

# 进入容器验证共享
kubectl exec emptydir-share-pod -c producer -- sh -c "echo 'test-from-producer' > /shared/test.txt"
kubectl exec emptydir-share-pod -c consumer -- cat /shared/test.txt
# 应该输出: test-from-producer

# 清理
kubectl delete pod emptydir-share-pod
```

**步骤 3**：验证 Pod 重启后数据丢失

```yaml
# emptydir-restart.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-restart-pod
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "echo 'data-before-restart' > /data/info.txt && cat /data/info.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
  restartPolicy: Always
```

```bash
kubectl apply -f emptydir-restart.yaml
kubectl exec emptydir-restart-pod -- cat /data/info.txt
# 输出: data-before-restart

# 模拟容器重启
kubectl delete pod emptydir-restart-pod
# 等待 Pod 重建后...
kubectl exec emptydir-restart-pod -- cat /data/info.txt
# 空文件！数据已丢失
```

### 实战2：使用 hostPath 实现节点本地持久化

**步骤 1**：在节点上创建目录

```bash
# 查看当前节点
kubectl get nodes

# 在节点上创建目录（通过 SSH 或节点上执行）
# ssh <node-name>
mkdir -p /data/k8s-hostpath
chmod 777 /data/k8s-hostpath

# 或者在 Pod 中使用 initContainer 初始化
```

**步骤 2**：创建 Pod 使用 hostPath

```yaml
# hostpath-persist.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-persist-pod
  labels:
    app: hostpath-demo
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: host-data
          mountPath: /usr/share/nginx/html
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"<h1>HostPath Demo - $(date)</h1>\" > /data/index.html; sleep 5; done"]
      volumeMounts:
        - name: host-data
          mountPath: /data
  volumes:
    - name: host-data
      hostPath:
        path: /data/k8s-hostpath
        type: DirectoryOrCreate
  nodeSelector:
    kubernetes.io/hostname: <node-name>    # 可选：绑定到特定节点
  restartPolicy: Never
```

**步骤 3**：验证持久化

```bash
kubectl apply -f hostpath-persist.yaml

# 访问应用
kubectl port-forward hostpath-persist-pod 8080:80
curl http://localhost:8080
# 应该显示时间戳的 HTML

# 删除 Pod 并重建
kubectl delete pod hostpath-persist-pod
kubectl apply -f hostpath-persist.yaml

# 数据应该仍然存在（因为存储在节点上）
```

### 实战3：创建 NFS PV 并使用 PVC 申请

**步骤 1**：搭建 NFS 服务器（如果没有现成的）

```bash
# 在一台节点上安装 NFS 服务
# Ubuntu/Debian
apt-get update && apt-get install -y nfs-kernel-server

# CentOS/RHEL
yum install -y nfs-utils

# 创建导出目录
mkdir -p /exported/k8s-data
echo "nfs-test-content" > /exported/k8s-data/test.txt
chmod 777 /exported/k8s-data

# 配置 NFS 导出
cat >> /etc/exports << 'EOF'
/exported/k8s-data 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

# 启动 NFS 服务
systemctl enable nfs-server
systemctl start nfs-server
exportfs -r
```

**步骤 2**：创建 NFS PV

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-10gi
  labels:
    storage-type: nfs
    app: nfs-demo
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  mountOptions:
    - nfsvers=4.1
    - hard
  nfs:
    server: 192.168.1.100          # NFS 服务器 IP
    path: /exported/k8s-data
```

```bash
kubectl apply -f nfs-pv.yaml
kubectl get pv nfs-pv-10gi
# 状态应为 Available
```

**步骤 3**：创建 PVC 申请存储

```yaml
# nfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  labels:
    app: nfs-demo
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
  selector:
    matchLabels:
      storage-type: nfs
```

```bash
kubectl apply -f nfs-pvc.yaml
kubectl get pvc nfs-pvc
# 状态应为 Bound

# 查看绑定的 PV
kubectl get pvc nfs-pvc -o jsonpath='{.spec.volumeName}'
```

**步骤 4**：在 Pod 中使用 PVC

```yaml
# nfs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-pod
  labels:
    app: nfs-demo
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: nfs-storage
          mountPath: /data
    - name: init
      image: busybox:1.36
      command: ["sh", "-c", "echo 'Hello from NFS Pod!' > /data/greeting.txt && cat /data/greeting.txt && sleep 3600"]
      volumeMounts:
        - name: nfs-storage
          mountPath: /data
  volumes:
    - name: nfs-storage
      persistentVolumeClaim:
        claimName: nfs-pvc
  restartPolicy: Never
```

```bash
kubectl apply -f nfs-pod.yaml

# 验证数据
kubectl exec nfs-test-pod -c init -- cat /data/greeting.txt
# 输出: Hello from NFS Pod!

# 从另一个 Pod 验证数据共享
kubectl run nfs-test-pod-2 --image=busybox:1.36 --overrides='
{
  "spec": {
    "containers": [{
      "name": "app",
      "image": "busybox:1.36",
      "command": ["sh", "-c", "cat /data/greeting.txt && sleep 3600"],
      "volumeMounts": [{
        "name": "nfs",
        "mountPath": "/data"
      }]
    }],
    "volumes": [{
      "name": "nfs",
      "persistentVolumeClaim": {
        "claimName": "nfs-pvc"
      }
    }]
  }
}'

# 两个 Pod 应该能看到相同的数据
```

### 实战4：创建 StorageClass 实现动态供给

**步骤 1**：安装 NFS 动态供给器

```bash
# 添加 NFS provisioner（基于 external-provisioner）
# 使用 Helm 安装
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.1.100 \
  --set nfs.path=/exported/k8s \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=true

# 或者使用 YAML 手动安装
```

**步骤 2**：创建 StorageClass

```yaml
# dynamic-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dynamic-nfs
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/nfs-provisioner
parameters:
  server: 192.168.1.100
  path: /exported/k8s/dynamic
  archiveOnDelete: "false"
  pathPattern: "${.PVC.namespace}-${.PVC.name}"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
kubectl apply -f dynamic-sc.yaml
kubectl get storageclass
```

**步骤 3**：创建 PVC 触发动态供给

```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc-demo
  labels:
    app: dynamic-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: dynamic-nfs
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f dynamic-pvc.yaml

# 等待几秒，PV 应该自动创建
kubectl get pv
# 应该看到一个新的 PV，名称类似 pvc-xxxxxx-xxxxxx

# PVC 状态应为 Bound
kubectl get pvc dynamic-pvc-demo
```

**步骤 4**：使用动态创建的存储

```yaml
# dynamic-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: dynamic-storage
          mountPath: /data
      command: ["sh", "-c", "echo 'dynamic-provisioning-works!' > /data/status.txt && cat /data/status.txt && sleep 3600"]
  volumes:
    - name: dynamic-storage
      persistentVolumeClaim:
        claimName: dynamic-pvc-demo
  restartPolicy: Never
```

```bash
kubectl apply -f dynamic-pod.yaml
kubectl exec dynamic-pod -- cat /data/status.txt
# 输出: dynamic-provisioning-works!
```

**步骤 5**：删除 PVC 观察动态回收

```bash
# 删除 PVC
kubectl delete pvc dynamic-pvc-demo

# 观察 PV 状态变化
kubectl get pv
# 1. PV 先变为 Released
# 2. 然后被自动删除（如果回收策略为 Delete）

# 清理 StorageClass
kubectl delete storageclass dynamic-nfs
```

### 实战5：部署 MySQL 并配置持久化存储

**步骤 1**：创建 PV

```yaml
# mysql-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/mysql
    type: DirectoryOrCreate
```

```bash
kubectl apply -f mysql-pv.yaml
kubectl get pv mysql-pv
# 确认状态为 Available
```

**步骤 2**：创建 Secret 存储密码

```bash
kubectl create secret generic mysql-secret \
  --from-literal=mysql-root-password=Root123! \
  --from-literal=mysql-password=User123! \
  --from-literal=mysql-database=wordpress
```

**步骤 3**：创建 PVC

```yaml
# mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: manual
```

```bash
kubectl apply -f mysql-pvc.yaml
kubectl get pvc mysql-pvc
# 状态应为 Bound
```

**步骤 4**：创建 MySQL 部署

```yaml
# mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-root-password
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-password
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-database
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
            - name: mysql-config
              mountPath: /etc/mysql/conf.d
          livenessProbe:
            exec:
              command:
                - mysqladmin
                - ping
                - -h
                - localhost
                - -u
                - root
                - -pRoot123!
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - mysqladmin
                - ping
                - -h
                - localhost
                - -u
                - root
                - -pRoot123!
            initialDelaySeconds: 10
            periodSeconds: 5
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
        - name: mysql-config
          configMap:
            name: mysql-config
      nodeSelector:
        kubernetes.io/os: linux
```

**步骤 5**：创建 MySQL 配置

```yaml
# mysql-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    max_connections=200
    innodb_buffer_pool_size=256M
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
    default-authentication-plugin=mysql_native_password
    log-error=/var/lib/mysql/error.log
    slow_query_log=1
    slow_query_log_file=/var/lib/mysql/slow.log

    [client]
    character-set=utf8mb4

    [mysql]
    default-character-set=utf8mb4
```

**步骤 6**：创建 Service

```yaml
# mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  type: ClusterIP
  ports:
    - port: 3306
      targetPort: 3306
      protocol: TCP
  selector:
    app: mysql
```

**步骤 7**：部署并验证

```bash
kubectl apply -f mysql-config.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml

# 等待 MySQL 启动
kubectl get pods -l app=mysql -w

# 进入 MySQL 验证
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') -- mysql -uroot -pRoot123! -e "SHOW DATABASES;"
```

### 实战6：模拟 Pod 重建后数据仍可访问

**步骤 1**：在 MySQL 中创建测试数据

```bash
# 获取 Pod 名称
POD_NAME=$(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}')

# 连接 MySQL 创建测试表和数据
kubectl exec -it $POD_NAME -- mysql -uroot -pRoot123! -e "
USE wordpress;
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO users (username, email) VALUES
  ('alice', 'alice@example.com'),
  ('bob', 'bob@example.com'),
  ('charlie', 'charlie@example.com');
SELECT * FROM users;
"
```

**步骤 2**：删除 Pod 模拟重建

```bash
# 删除 Pod（Deployment 会自动重建）
kubectl delete pod -l app=mysql

# 等待新 Pod 启动
kubectl get pods -l app=mysql -w

# 新 Pod 启动后，验证数据仍然存在
NEW_POD=$(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $NEW_POD -- mysql -uroot -pRoot123! -e "USE wordpress; SELECT * FROM users;"
# 应该还能看到之前插入的三条记录
```

**步骤 3**：验证持久化的关键点

```bash
# 检查 PV/PVC 状态
kubectl get pv mysql-pv
kubectl get pvc mysql-pvc
# 两者都应为 Bound 状态

# 检查 Pod 使用的 PVC
kubectl get pod $NEW_POD -o jsonpath='{.spec.volumes[?(@.name=="mysql-data")].persistentVolumeClaim.claimName}'
# 应该输出: mysql-pvc

# 即使删除 Deployment，PV 数据仍然保留
kubectl delete deployment mysql
kubectl get pv mysql-pv
# PV 仍然存在，状态为 Released

# 重新创建 Deployment，数据依然可用
kubectl apply -f mysql-deployment.yaml
kubectl get pods -l app=mysql -w

# 验证数据
kubectl exec -it $(kubectl get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') -- mysql -uroot -pRoot123! -e "USE wordpress; SELECT COUNT(*) FROM users;"
# 应该输出 3
```

**步骤 4**：完整清理流程

```bash
# 1. 删除 Deployment（Pod 自动销毁）
kubectl delete deployment mysql

# 2. 删除 PVC（PV 根据回收策略处理）
kubectl delete pvc mysql-pvc
# PV 变为 Released 状态

# 3. 清理 PV 数据（如果使用 Retain 策略）
# ssh 到节点删除数据
# rm -rf /data/mysql/*

# 4. 删除 PV
kubectl delete pv mysql-pv

# 5. 删除其他资源
kubectl delete secret mysql-secret
kubectl delete configmap mysql-config
kubectl delete service mysql
```

---

## 7.7 常见问题排查

### 7.7.1 PVC 一直处于 Pending 状态

**问题**：PVC 创建后长时间 Pending，无法绑定到 PV。

**排查步骤**：

```bash
# 1. 查看 PVC 状态
kubectl get pvc
kubectl describe pvc <name>
# 关注 Events 部分的错误信息

# 2. 检查是否有匹配的 PV
kubectl get pv
# 确保 PV 存在且状态为 Available

# 3. 检查 StorageClass 是否匹配
kubectl get storageclass
# PVC 和 PV 的 storageClassName 必须一致

# 4. 检查容量和访问模式
kubectl get pv <name> -o jsonpath='{.spec.capacity.storage}'
kubectl get pvc <name> -o jsonpath='{.spec.resources.requests.storage}'
# PV 容量必须 >= PVC 请求的容量
# 访问模式必须兼容

# 5. 如果使用动态供给，检查 Provisioner 日志
kubectl logs -n kube-system -l app=<provisioner-name>

# 6. 检查 PV 的 nodeAffinity（如使用本地存储）
kubectl get pv <name> -o jsonpath='{.spec.nodeAffinity}'
```

### 7.7.2 Volume 挂载失败

**问题**：Pod 无法正常启动，Volume 挂载失败。

**排查步骤**：

```bash
# 1. 查看 Pod 事件
kubectl describe pod <name>
# Events 部分会显示挂载错误

# 2. 检查 NFS 连接
# 在节点上测试 NFS 连接
mount -t nfs -o v4.1 192.168.1.100:/exported/path /mnt/test
ls /mnt/test
umount /mnt/test

# 3. 检查 hostPath 目录
# 在节点上检查目录是否存在且权限正确
ls -la /data/k8s-hostpath
chmod 777 /data/k8s-hostpath

# 4. 检查 PV/PVC 状态
kubectl get pv,pvc
# 确保 PV 状态为 Available，PVC 状态为 Bound

# 5. 检查 kubelet 日志
journalctl -u kubelet -f
# 或
kubectl logs -n kube-system -l k8s-app=kubelet
```

### 7.7.3 PV 删除失败

**问题**：PVC 删除后 PV 一直卡在 Released 状态。

**排查步骤**：

```bash
# 1. 查看 PV 状态
kubectl describe pv <name>
# 关注 Status 和 ClaimRef 部分

# 2. 手动清理数据（Retain 策略）
# SSH 到存储节点，删除数据
rm -rf /data/k8s-pv/*

# 3. 移除 ClaimRef 使 PV 变为 Available
kubectl patch pv <name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'

# 4. 检查 PV 是否变为 Available
kubectl get pv <name>

# 5. 强制删除 PV（谨慎使用）
kubectl delete pv <name> --grace-period=0 --force
# 注意：这不会清理后端存储数据
```

### 7.7.4 NFS 权限问题

**问题**：NFS 挂载成功但容器内无法写入数据。

**排查步骤**：

```bash
# 1. 检查 NFS 导出配置
cat /etc/exports
# 确保配置了读写权限 (rw)

# 2. 检查节点上的挂载选项
mount | grep nfs
# 确认是读写挂载

# 3. 检查 NFS 服务器上的目录权限
ls -la /exported/path
chmod 777 /exported/path
# 或
chown -R nfsnobody:nfsnobody /exported/path

# 4. 检查 root_squash 配置
# 如果配置了 root_squash，容器内的 root 用户会被映射为 nfsnobody
# 解决方案：配置 no_root_squash 或调整目录权限
```

### 7.7.5 PV 扩容失败

**问题**：PVC 扩容后容量没有变化。

**排查步骤**：

```bash
# 1. 检查 StorageClass 是否允许扩容
kubectl get storageclass <name> -o jsonpath='{.allowVolumeExpansion}'
# 必须为 true

# 2. 修改 PVC 容量
kubectl patch pvc <name> -p '{"spec": {"resources": {"requests": {"storage": "20Gi"}}}}'

# 3. 检查 PVC 状态
kubectl get pvc <name>
# 查看 STATUS 和 CAPACITY 列

# 4. 重建 Pod（某些存储类型需要重建 Pod 才能生效）
kubectl delete pod <pod-name>
# 等待 Pod 重建

# 5. 检查底层存储是否支持扩容
# NFS 不支持在线扩容
# 云盘（AWS EBS/GCE PD）支持扩容
```

### 7.7.6 数据丢失问题

**问题**：Pod 重建后数据丢失。

**排查步骤**：

```bash
# 1. 确认使用的 Volume 类型
# emptyDir：Pod 销毁后数据丢失（正常行为）
# hostPath：数据保留在节点上
# PVC：数据保留在后端存储

# 2. 检查 Pod 是否使用了 PVC
kubectl get pod <name> -o yaml | grep -A 5 persistentVolumeClaim

# 3. 检查 PV 的回收策略
kubectl get pv <name> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'

# 4. 检查 PV/PVC 是否仍然存在
kubectl get pv,pvc

# 5. 如果使用 hostPath，检查节点是否变化
kubectl get pod <name> -o jsonpath='{.spec.nodeName}'
# 如果 Pod 调度到其他节点，数据会丢失
```

---

## 7.8 章节小结

### 核心知识点回顾

1. **Volume 基础**：
   - Volume 与 Pod 生命周期绑定
   - emptyDir：临时共享，Pod 销毁即丢失
   - hostPath：节点本地存储，跨节点不可用
   - configMap/secret：配置注入
   - nfs/cifs：网络文件系统
   - persistentVolumeClaim：持久化存储

2. **PV/PVC 核心**：
   - PV 是存储提供者，PVC 是存储请求者
   - 生命周期：Provisioning → Binding → Using → Releasing → Available
   - 访问模式：RWO、ROX、RWX、RWOP
   - 回收策略：Retain、Delete、Recycle（已废弃）

3. **StorageClass 与动态供给**：
   - StorageClass 定义如何自动创建 PV
   - 动态供给流程：PVC → StorageClass → Provisioner → PV
   - 支持扩容、快照等高级功能

4. **CSI 标准**：
   - 存储驱动的标准化接口
   - 解耦 K8s 与存储实现
   - 支持 Controller/Node/Identity 三种插件

### 常用命令速查

```bash
# PV 管理
kubectl get pv
kubectl get pv <name> -o yaml
kubectl describe pv <name>
kubectl apply -f pv.yaml
kubectl delete pv <name>
kubectl patch pv <name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'

# PVC 管理
kubectl get pvc
kubectl get pvc <name> -o yaml
kubectl describe pvc <name>
kubectl apply -f pvc.yaml
kubectl delete pvc <name>
kubectl patch pvc <name> -p '{"spec": {"resources": {"requests": {"storage": "20Gi"}}}}'

# StorageClass 管理
kubectl get storageclass
kubectl get storageclass <name> -o yaml
kubectl apply -f sc.yaml
kubectl delete storageclass <name>

# 资源清理
kubectl delete pvc <name>           # 删除 PVC
kubectl delete pv <name>             # 删除 PV
kubectl delete pod <name>            # 删除 Pod（PV 数据根据回收策略处理）
```

### 最佳实践总结

| 场景 | 推荐方案 |
|------|---------|
| 临时数据共享 | emptyDir（Pod 级别生命周期） |
| 节点本地存储 | hostPath（仅限单节点） |
| 配置注入 | configMap/secret + Volume 挂载 |
| 多 Pod 共享 | NFS/CIFS + PVC |
| 单机持久化 | PVC + 云盘（RWO） |
| 多机读写 | PVC + NFS（RWX） |
| 生产环境 | StorageClass + CSI 驱动 + 云盘 |
| 多环境管理 | 动态供给 + 默认 StorageClass |

### Volume 选择决策树

```
                    ┌─────────────────────┐
                    │  数据生命周期？      │
                    └──────────┬──────────┘
                               │
                    ┌────Pod───┴───持久────┐
                    ▼                     ▼
              ┌──────────┐          ┌──────────┐
              │ emptyDir │          │ 需要共享？│
              └──────────┘          └─────┬────┘
                                    YES │    NO
                                         │    │
                                         ▼    ▼
                                   ┌────────┐ ┌──────────┐
                                   │RWX需要？│ │ RWO 即可 │
                                   └───┬────┘ └─────┬────┘
                                    YES │   NO  │     │
                                        │       │     │
                                        ▼       ▼     ▼
                                  ┌────────┐ ┌──────┐ ┌──────┐
                                  │ NFS    │ │ROX需要│ │ PVC  │
                                  │ CIFS   │ │      │ │ +云盘│
                                  └────────┘ └──┬───┘ └──────┘
                                               │
                                               ▼
                                          ┌────────────┐
                                          │ 静态只读PV  │
                                          └────────────┘
```
