---
title: "Kubernetes 深入浅出"
date: 2020-03-24T13:16:53+08:00
tags: ["Kubernetes", "Docker"]
---

## Kubernetes 核心概念

### 为什么需要 Kubernetes？

容器化解决了环境一致性问题，但生产环境需要管理成百上千个容器——自动部署、扩缩容、负载均衡、服务发现、滚动更新。Kubernetes 提供了这些能力的声明式编排框架。

### 架构总览

```
┌─────────────────────────────────────────┐
│              Control Plane              │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ API Svr  │ │Scheduler │ │Ctrl Mgr │ │
│  └────┬─────┘ └────┬─────┘ └────┬────┘ │
│       │            │            │       │
│  ┌────▼──────────────────────────▼────┐ │
│  │              etcd                  │ │
│  └───────────────────────────────────┘ │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│               Node                      │
│  ┌──────────┐  ┌──────────┐ ┌────────┐ │
│  │ Kubelet  │  │Kube-Proxy│ │Runtime │ │
│  └────┬─────┘  └────┬─────┘ └───┬────┘ │
│       │             │           │       │
│  ┌────▼─────────────▼───────────▼────┐ │
│  │          Pods                     │ │
│  └──────────────────────────────────┘ │
└────────────────────────────────────────┘
```

### 核心组件设计原理

| 组件 | 作用 | 为什么这样设计 |
|------|------|--------------|
| **API Server** | 提供 RESTful API，是集群入口 | 所有组件只与 API Server 通信，避免组件间直接耦合——这是 K8s 的核心设计哲学：**声明式 + 控制循环** |
| **etcd** | 分布式 KV 存储，保存集群全部状态 | 基于 Raft 共识算法保证一致性。为什么选 etcd？强一致、watch 机制（支持控制器监听变更）、CNCF 项目 |
| **Scheduler** | 将 Pod 分配到最优 Node | 筛选（Predicates）→ 打分（Priorities）两阶段算法，而非简单轮询，因为需要考虑资源、亲和性、污点等多维约束 |
| **Controller Manager** | 运行各种控制器（Deployment、Node、Endpoints 等） | 控制循环模式（Reconciliation Loop）：期望状态 vs 实际状态，不断调谐直到一致 |
| **Kubelet** | 管理 Pod 生命周期 | 节点上的 Agent，负责创建/删除容器、健康检查、上报节点状态 |
| **Kube-Proxy** | 网络代理和负载均衡 | 维护 iptables/IPVS 规则，将 Service IP 转发到后端 Pod |

#### 关键设计：声明式 API + 控制循环

K8s 的设计核心不是命令式（"启动 3 个 nginx"），而是声明式（"确保有 3 个 nginx 在运行"）。用户提交期望状态（通过 YAML），控制器监听 API Server 的变化，不断调谐直到实际状态与期望状态一致。这种模式的好处：
- 自动恢复：Pod 挂了，ReplicaSet 控制器自动补一个
- 幂等：同样的 YAML 应用多次结果一致
- 可审计：所有变更记录在 etcd 中

## 核心资源对象详解

### Pod

Pod 是最小调度单元。为什么不是容器？因为有些场景多个容器需要共享网络栈（如 Sidecar 模式），K8s 将它们打包为 Pod，共享 IP、端口、存储卷。

**Pod 生命周期**：`Pending → Running → Succeeded/Failed → Unknown`

```
关键阶段说明：
- Pending：API Server 已接受，但 Pod 尚未调度到 Node（或镜像拉取中）
- Running：至少有一个容器在运行
- Unknown：Node 失联（kubelet 停止上报心跳）
```

### Deployment

Deployment 管理 ReplicaSet，ReplicaSet 管理 Pod。为什么中间加一层 ReplicaSet？为了实现**滚动更新与回滚**——每发布一次，创建一个新 RS，逐步扩容新 RS 并缩容旧 RS，回滚时直接指回旧 RS。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  revisionHistoryLimit: 10          # 保留最近的 10 个版本用于回滚
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                   # 滚动期间最多允许超出 1 个副本
      maxUnavailable: 0             # 滚动期间不允许有不可用副本
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

**滚动更新流程**：
1. 创建新 ReplicaSet（副本数为 0）
2. 新 RS +1，旧 RS -1（maxSurge=1, maxUnavailable=0 意味着始终保持 3 个可用 Pod）
3. 重复直到新 RS 达到 3，旧 RS 缩到 0

### Service

Pod 的 IP 不固定（重启、扩缩容后会变），Service 提供稳定的访问入口。实现方式：kube-proxy 在节点上维护 iptables/IPVS 规则。

**Service 类型选择**：

| 类型 | 访问范围 | 实现方式 | 场景 |
|------|---------|---------|------|
| ClusterIP | 集群内 | iptables/IPVS 转发 | 内部服务调用 |
| NodePort | 集群外（节点 IP:端口） | 每个节点监听端口 → iptables | 开发调试、无 LB |
| LoadBalancer | 公网 | 云厂商 LB + NodePort | 生产对外服务 |
| ExternalName | DNS 映射 | CoreDNS 返回 CNAME | 访问外部服务 |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

### ConfigMap 和 Secret

为什么需要 ConfigMap？将配置与容器镜像解耦，镜像不变、配置可变。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yml: |
    server:
      port: 8080
    database:
      url: jdbc:mysql://db:3306/app
```

Secret 存储敏感信息，数据以 Base64 编码（非加密——需要配合 RBAC 和加密存储）。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

**注意**：ConfigMap/Secret 更新后，Pod 中的环境变量不会自动更新，只有通过 Volume 挂载的方式才会自动同步（默认约 60 秒）。

## 网络模型

K8s 网络模型的核心约定——**每个 Pod 都有独立 IP，Pod 之间可以直接通信**（无需 NAT）。

### CNI 插件原理

K8s 不自己实现网络，而是通过 CNI（Container Network Interface）插件接入。工作流程：

```
Pod 创建 → kubelet 调用 CNI 插件 → 插件创建 veth pair → 一端 Pod 内，一端连到网桥/路由
```

常见 CNI 对比：

| CNI | 数据平面 | 特点 |
|-----|---------|------|
| Flannel | VXLAN/UDP | 简单，性能一般，适合小集群 |
| Calico | BGP + iptables | 纯三层，性能好，支持 NetworkPolicy |
| Cilium | eBPF | 性能极高，支持 L7 策略，但要求内核 5.8+ |
| Weave | VXLAN + fastdp | 自动组网，跨主机通信简单 |

**Service 网络**：ClusterIP 通过 kube-proxy 的 iptables/IPVS 规则将虚拟 IP 映射到后端 Pod。IPVS 比 iptables 更适合大规模集群（iptables 的规则复杂度为 O(n)，IPVS 为 O(1)）。

**DNS**：CoreDNS 为 Service 解析域名，格式为 `<service>.<namespace>.svc.cluster.local`。

## 存储管理

### Volumes 类型与选择

| 类型 | 生命周期 | 场景 |
|------|---------|------|
| emptyDir | Pod 生命周期 | 临时缓存、Sidecar 数据交换 |
| hostPath | Node 生命周期 | 需要读取宿主机数据 |
| PersistentVolumeClaim | 独立于 Pod | 数据库等需持久化存储 |

### PV/PVC 设计原理

为什么设计 PV 和 PVC 两层抽象？将**存储的提供**和**存储的使用**分离：
- **PV**：由集群管理员预先创建或通过 StorageClass 动态供应
- **PVC**：用户声明存储需求（大小、访问模式），与 PV 匹配后绑定

```yaml
# PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

**PV 回收策略**：`Retain`（管理员手动回收）→ 生产环境推荐 / `Delete`（自动删除）→ 动态供应场景 / `Recycle`（已废弃，不建议使用）

## 调度原理

### 两阶段调度算法

Scheduler 对每个待调度 Pod 执行两步：

1. **Predicates（筛选）**：过滤不满足条件的 Node。包括：
   - `PodFitsResources`：节点资源足够
   - `MatchNodeSelector`：匹配 `nodeSelector` 标签
   - `PodToleratesNodeTaints`：Pod 能容忍 Node 的污点
   - `CheckNodePortConflict`：端口不冲突
   - `CheckNodeAffinity`：满足节点亲和性

2. **Priorities（打分）**：对候选 Node 按策略评分：
   - `LeastRequestedPriority`：资源利用率低的 Node 得分高（分散 Pod）
   - `BalancedResourceAllocation`：CPU 和内存均衡使用
   - `ImageLocalityPriority`：Pod 镜像已有的 Node 得分高
   - NodeAffinity、TaintToleration 等优先级

### 节点选择器与亲和性

```yaml
# nodeSelector：简单的标签匹配
spec:
  nodeSelector:
    disktype: ssd

# 节点亲和性：更灵活的表达
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 硬约束
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:  # 软约束
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1"]
```

### Pod 亲和性与反亲和性

```yaml
spec:
  affinity:
    podAntiAffinity:  # 确保同一服务的 Pod 分散在不同节点
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: nginx
        topologyKey: kubernetes.io/hostname
```

**topologyKey** 指定了"分散的范围"——这里是 `hostname`，即不同宿主机。

### 污点与容忍

污点（Taint）阻止 Pod 调度到特定 Node，容忍（Toleration）允许 Pod"忍受"污点。

```yaml
# 给节点打污点
# kubectl taint nodes node1 key=value:NoSchedule

# Pod 容忍
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```

污点效果：`NoSchedule`（不调度新 Pod）/ `PreferNoSchedule`（尽量不调度）/ `NoExecute`（驱逐已有 Pod）

## 安全机制

### RBAC

K8s 的授权体系基于 RBAC（最小权限原则）。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole vs Role**：Role 作用于特定 Namespace，ClusterRole 作用于集群范围。

## 高可用设计

### Master 高可用

```
Load Balancer → API Server × 3 → etcd cluster × 3（奇数节点）
```

etcd 基于 Raft 协议，3 节点允许 1 个故障，5 节点允许 2 个故障。Scheduler 和 Controller Manager 通过 Leader Election 选主。

### Pod 高可用

- Deployment 多副本 + 反亲和性确保跨节点分布
- PDB（PodDisruptionBudget）确保更新时最少存活副本数
- HPA 根据指标自动伸缩

### 节点故障处理

Node 失联后：
1. 默认 40 秒后 Node 变为 `NotReady`（`node-monitor-period` 5 秒 × `node-monitor-grace-period` 40 秒 × 8 次失败）
2. Controller Manager 的 NodeLifecycleController 驱逐 Pod（`pod-eviction-timeout` 默认 5 分钟）
3. 被驱逐的 Pod 在其他 Node 上重建

## 常见问题排查

1. **Pod 一直 Pending**
   - `kubectl describe pod <name>` 查看 Events
   - 常见原因：节点资源不足、PVC 未绑定、镜像拉取失败、端口冲突
   - 检查 `kubectl get nodes` 是否有 Ready 节点

2. **Pod 不断 CrashLoopBackOff**
   - `kubectl logs <pod>` 查看日志
   - `kubectl logs <pod> --previous` 查看上次启动的日志
   - 常见原因：启动参数错误、依赖服务未就绪、配置错误

3. **Node 一直 NotReady**
   - 排查 kubelet 是否运行：`systemctl status kubelet`
   - 检查 kubelet 日志：`journalctl -u kubelet -f`
   - `docker info`（或 `crictl info`）确认容器运行时正常
   - 检查磁盘空间：`df -h`（磁盘满会导致 Node NotReady）

4. **Service 不通**
   - 检查 Endpoint：`kubectl get endpoints <service>`——为空说明 Pod 未匹配到 Service selector
   - 检查 CoreDNS：`kubectl run -it test --image=busybox -- wget <service>`——DNS 解析是否正常
   - 检查 kube-proxy 能否正常工作：`iptables-save | grep <service-name>`

5. **存储异常**
   - 检查 PV/PVC 状态：`kubectl get pv,pvc`——Pending 说明未绑定
   - 检查 StorageClass 是否正确创建——动态供应时需要默认 StorageClass
   - 检查宿主机目录权限——hostPath 场景需确认目录存在且权限正确

6. **etcd 性能问题**
   - `etcdctl check perf` 检查读写延迟
   - 避免单 etcd 成员存储过大（建议 8GB 限制）
   - 避免频繁大量写入（如频繁创建/删除 Pod）
