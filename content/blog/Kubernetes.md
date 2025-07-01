---
title: "Kubernetes 深入浅出"
date: 2020-03-24T13:16:53+08:00
tags: ["Kubernetes", "Docker"]
---

## Kubernetes 核心概念
### 1. 基本概念
   * Pod：最小的部署单元，包含一个或多个容器
   * Node：工作节点，运行 Pod
   * Cluster：由多个 Node 组成的集群
   * Namespace：资源隔离和分组
### 2. 核心组件
   * Master 组件：
      * API Server：提供 Restful API，是集群的前端接口
      * Scheduler：负责调度 Pod 到合适的 Node
      * Controller Manager：运行各种控制器，如 Replication Controller、Node Controller 等
      * etcd： 分布式键值存储，保存集群的所有配置数据
   * Node 组件
      * Kubelet：负责管理 Pod 和容器的生命周期
      * Kube Proxy：负责网络代理和负载均衡
      * Container Runtime： 运行容器的软件，如 Docker、containerd
### 3. 资源对象
   * Pod
      * 最小的部署单元
      * 包含一个或多个容器
      * 共享网络和存储
   * Deployment
      * 声明式更新 Pod 和 ReplicaSet
      * 支持滚动更新和回滚
      * 示例：

```yaml
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
   * Service
      * 定义 Pod 的访问策略
      * 支持 ClusterIP、NodePort、LoadBalancer 等类型
      * 示例：

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
   * ConfigMap 和 Secret
      * 存储配置数据
      * 示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
data:
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
```
   * Secret：
      * 存储敏感信息

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```
## 网络模型
### 1. Pod 网络
   * 每个 Pod 有唯一的 IP 地址
   * Pod 之间可以直接通信
   * 使用 CNI（Container Network Interface）插件
### 2. Service 网络
   * ClusterIP：集群内部访问
   * NodePort：通过节点访问
   * LoadBalancer：通过云服务商的负载均衡访问
### 3. DNS 服务
   * 内置 DNS 服务
   * 通过服务名进行服务发现
   * 示例：`my-service.my-namespace.svc.cluster.local`

## 存储管理
### 1. Volume
   * 提供持久化存储
   * 支持多种类型（如 emptyDir、hostPath、NFS、云存储）
   * 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
      type: Directory
```
### 2. PersistentVolume(PV) 和 PersistentVolumeClaim(PVC)
   * PV：集群范围的存储资源
   * PVC：用户对存储资源的请求
   * 示例：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
```
```yaml
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
## 调度策略
### 1. 节点选择器（nodeSelector）
   * 根据节点标签选择节点
   * 示例：

```yaml
spec:
  nodeSelector:
    disktype: ssd
```
### 2. 亲和性和反亲和性（affinity/anti-affinity）
   * 控制 Pod 的调度位置
   * 示例：

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/e2e-az-name
          operator: In
          values:
          - e2e-az1
          - e2e-az2
```
### 3. 污点与容忍（taint/toleration）
   * 控制 Pod 是否可以调度到特定节点
   * 示例：

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```
## 安全机制
### 1. 认证（Authentication）
   * 支持多种认证方式（如证书、令牌、用户名密码）
### 2. 授权（Authorization）
   * 使用 RBAC （Role-Based Access Controll）
   * 示例：

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
```
### 3. 网络策略（Network Policy）
   * 控制 Pod 之间的网络通信
   * 示例：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
## 监控和日志
### 1. Metrics Server
   * 收集集群资源使用情况
   * 支持 Horizontal Pod Autoscaler（HPA）
### 2. Prometheus 和 Grafana
   * 提供强大的监控和可视化功能
   * 示例：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  resources:
    requests:
      memory: 400Mi
```
### 3. Kubernetes Dashboard
   * 提供 Web 界面管理集群
   * 示例：

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`

## Kubernetes 架构
### 1. Master 节点
```Plain Text
+-------------------+
|    API Server     |
+-------------------+
|    Scheduler      |
+-------------------+
| Controller Manager|
+-------------------+
|      etcd         |
+-------------------+
```
### 2. Node 节点
```Plain Text
+-------------------+
|     Kubelet       |
+-------------------+
|   Kube Proxy      |
+-------------------+
| Container Runtime |
+-------------------+
```
### 3. 整体架构
```Plain Text
+-------------------+     +-------------------+
|      Master       | <-> |       Node        |
+-------------------+     +-------------------+
      ^                         ^
      |                         |
      v                         v
+-------------------+     +-------------------+
|      etcd         |     |      Pods         |
+-------------------+     +-------------------+

```
## 常见问题
### 1. Kubernetes 的核心组件有哪些？
   * Master 组件：API Server、Scheduler、Controller Manager、etcd
   * Node 组件：Kubelet、Kube Proxy、Container Runtime
### 2. Pod 和容器区别是什么
   * Pob 是 Kubernetes 的最小部署单元
   * 一个 Pod 可以包含一个或者多个容器
   * Pod 内的容器共享网络和存储
### 3. 如何实现 Kubernetes 的高可用？
   * 部署多个 Master 节点
   * 使用负载均衡
   * 配置 etcd 集群
   * 定期备份数据
### 4. Kubernetes 的网络模型是怎么样的？
   * 每个 Pod 有唯一的 IP 地址
   * Pod 之间可以直接通信
   * 使用 CNI （Container Network Interface）
### 5. 如何实现 Kubernetes 的自动扩展？
   * 使用 Horizontal Pod Autoscaler （HPA）
   * 配置资源请求和限制
   * 监控应用指标
### 6. 如何管理 Kubernetes 的配置
   * 使用 ConfigMap 存储配置数据
   * 使用 Secret 存储敏感信息
   * 通过环境变量或 Volume 挂载到 Pod
### 7. Kubernetes 的调度策略有哪些？
   * 节点选择器（nodeSelector）
   * 亲和性和反亲和性（affinity/anti-affinity）
   * 污点和容忍（taint/toleration）
   * 优先级和抢占（priority/preempiton）
### 8. 如何实现 Kubernetes 的持久化存储？
   * 使用 PersistentVolume（PV）
   * 使用 PersistentVolumeClaim（PVC）
   * 支持多种存储后端（如 NFS、iSCI、云存储）
### 9. 如何监控 Kubernetes 集群？
   * 使用 Metrics Server
   * 集成 Prometheus 和 Grafana
   * 使用 Kubernetes Dashborad
   * 配置告警规则
### 10. 如何实现 Kubernetes 的滚动更新？
   * 使用 Deployment
   * 配置更新策略（RollingUpdate）
   * 设置最大不可用和最大超出副本数