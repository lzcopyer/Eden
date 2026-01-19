## 一、Kubernetes 概述

**Kubernetes（简称 K8s）** 是 Google 开源的容器编排平台，用于自动化容器化应用的部署、扩展和管理。它的核心功能包括：

- ✅ **服务发现与负载均衡**
    
- ✅ **存储编排**
    
- ✅ **自动部署与回滚**
    
- ✅ **自动扩缩容**
    
- ✅ **自我修复**
    
- ✅ **密钥与配置管理**

## 二、Kubernetes 架构

### 1. 整体架构图

``` mermaid
graph LR
    A[Master Node] --> B[Worker Node 1]
    A --> C[Worker Node 2]
    A --> D[Worker Node N]
```

### 2. Master 节点组件

Master 节点是集群的控制平面，负责全局决策：

|组件|功能描述|
|---|---|
|**API Server**|集群的入口点，处理 REST 操作，验证配置数据|
|**etcd**|分布式键值存储，保存集群所有配置数据|
|**Scheduler**|负责将 Pod 分配到合适的 Node 上运行|
|**Controller Manager**|运行各种控制器进程（节点、副本、端点控制器等）|

### 3. Worker 节点组件

Worker 节点是工作负载的实际运行环境：

|组件|功能描述|
|---|---|
|**kubelet**|节点代理，管理 Pod 和容器生命周期|
|**kube-proxy**|网络代理，实现 Service 抽象|
|**Container Runtime**|容器运行环境（Docker, containerd 等）|
|**Pods**|最小调度单元，包含一个或多个容器|

## 三、核心概念解析

### 1. Pod

**最小部署单元**，包含：

- 一个或多个容器
    
- 共享存储卷（Volumes）
    
- 网络命名空间（同一Pod内容器通过localhost通信）

``` yaml
# Pod 示例
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

### 2. Service

**服务的抽象定义**，提供：

- 稳定的 IP 地址和 DNS 名称
    
- 负载均衡到后端 Pod
    
- 服务发现机制

``` mermaid
graph LR
    S[Service] --> P1[Pod 1]
    S --> P2[Pod 2]
    S --> P3[Pod 3]
```

### 3. Deployment

**声明式更新管理**，提供：

- Pod 的副本数量管理
    
- 滚动更新和回滚能力
    
- 版本控制

``` yaml
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80

```

```  bash
# 创建Deployment
kubectl create deployment nginx --image=nginx:1.14.2
```

### 4. StatefulSet

StatefulSet 是 Kubernetes 中用于管理**有状态应用**的核心工作负载 API 对象，与 Deployment 不同，它为每个 Pod 提供：

- **稳定的网络标识**（唯一主机名）
    
- **持久化存储**（生命周期独立于 Pod）
    
- **有序的部署和扩缩容**（顺序启动/终止）
    

### 典型应用场景

- 数据库集群（MySQL, PostgreSQL, MongoDB）
    
- 消息队列（Kafka, RabbitMQ）
    
- 分布式存储系统（Elasticsearch, ETCD）
    
- 需要持久化数据的应用


### 5. 其他关键概念

|概念|说明|
|---|---|
|**ConfigMap**|存储非机密配置数据|
|**Secret**|存储敏感信息（密码、令牌等）|
|**Volume**|容器数据持久化存储|
|**Namespace**|虚拟集群，资源逻辑隔离|

## 四、Kubernetes 工作流程

当用户提交一个部署请求时的处理流程：

``` mermaid
sequenceDiagram
    participant User as 用户
    participant API as API Server
    participant Sched as Scheduler
    participant etcd as etcd
    participant Kubelet as kubelet
    
    User->>API: kubectl apply -f deployment.yaml
    API->>etcd: 存储对象状态
    API->>Sched: 通知有新的Pod需要调度
    Sched->>API: 选择合适节点
    API->>etcd: 更新Pod绑定信息
    API->>Kubelet: 通知目标节点
    Kubelet->>Container Runtime: 创建容器
    Kubelet->>API: 更新Pod状态
```

## 五、Kubernetes 网络模型

### 核心原则

- 每个 Pod 拥有唯一 IP 地址（IP-per-Pod）
    
- Pod 跨节点通信无需 NAT
    
- 节点与 Pod 之间通信无需 NAT
    

### 网络组件

|组件|功能|
|---|---|
|**CNI 插件**|实现网络连接（Calico, Flannel 等）|
|**Service IP**|虚拟 IP，由 kube-proxy 维护|
|**Ingress**|管理外部访问集群服务的入口|

## 六、Kubernetes 存储架构

### 持久化存储流程

``` mermaid
graph LR
    A[Pod] --> B[PersistentVolumeClaim]
    B --> C[PersistentVolume]
    C --> D[Storage Class]
    D --> E[云存储/AzureDisk/NFS等]
```

## 七、最佳实践建议

1. **声明式配置**：使用 YAML 文件管理资源
    
2. **资源限制**：为容器设置 CPU/内存请求和限制
    
3. **健康检查**：配置 liveness 和 readiness 探针
    
4. **标签管理**：合理使用标签组织资源
    
5. **滚动更新**：使用 Deployment 实现零停机更新

``` yaml
# 资源限制示例
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

## 八、总结

Kubernetes 提供了强大的容器编排能力，其架构设计具有：

- 高度模块化（组件可插拔）
    
- 声明式API（期望状态管理）
    
- 自我修复能力（自动替换故障容器）
    
- 水平扩展架构（轻松应对大规模集群）
    

通过掌握其核心概念和架构原理，您可以构建健壮、可扩展的云原生应用系统。

> **学习资源推荐**：
> 
> - [Kubernetes 官方文档](https://kubernetes.io/docs)
>     
> - 《Kubernetes in Action》
>     
> - K8s 交互式教程：Katacoda Kubernetes Playground

