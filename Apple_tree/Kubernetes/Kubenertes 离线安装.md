**仙人指路**
- [easzlab](https://github.com/easzlab/kubeasz)

# Linux离线环境下Kubernetes高可用集群部署深度指南

本报告旨在为系统管理员、DevOps工程师及SRE提供一份详尽、权威的指南，用于在无互联网连接的Linux离线环境中，使用`kubeadm`部署一个生产级的高可用（HA）Kubernetes集群。报告将从在线环境的资源准备、离线环境的基础配置，到高可用负载均衡器搭建、集群引导与验证，提供端到端的详细操作步骤，并深入剖析每个步骤背后的技术原理与设计考量。

## 1. 概述与架构概览

在高度安全或隔离的网络环境中，传统的Kubernetes在线部署方式因无法访问外部依赖（如容器镜像仓库、软件包源）而无法实施。离线部署的核心挑战在于解决**依赖隔离**和**资源完整性**的问题，确保所有必需的软件包、二进制文件和容器镜像在安装前已妥善收集并打包。同时，生产级集群的部署必须具备**高可用性**，以消除单点故障（Single Point of Failure, SPOF），保证核心服务的持续运行。

### 1.1. Kubernetes核心组件概览与高可用角色定位

一个Kubernetes集群由控制平面和工作节点两大部分组成 1。理解其组件在HA架构中的角色，是成功部署的基础：

- **控制平面 (Control Plane)**：负责集群的“决策”和“管理”。
    
    - `kube-apiserver`：集群的统一入口，验证和配置API对象数据，所有组件都通过它进行交互。在HA架构中，多个`kube-apiserver`实例通过负载均衡器提供冗余。
        
    - `etcd`：集群状态的分布式键值存储，是Kubernetes的唯一数据持久化后端。HA架构中，`etcd`集群由奇数个成员（通常是3或5个）构成，通过Raft协议保持数据一致性和高可用性 3。
        
    - `kube-scheduler`：负责将新的Pod调度到可用的工作节点上。
        
    - `kube-controller-manager`：管理多种控制器，如节点控制器（监测节点状态）、副本控制器（维护副本集数量）等，确保集群状态与期望状态一致 1。
        
- **工作节点 (Worker Node)**：负责运行用户容器负载。
    
    - `kubelet`：运行在每个节点上的代理，负责管理节点上的Pod和容器生命周期，并与控制平面通信 1。
        
    - `kube-proxy`：每个节点上的网络代理，负责维护网络规则，实现Service的负载均衡和网络路由。
        
    - **容器运行时（CRI）**：如Containerd或CRI-O，负责管理容器的生命周期，例如拉取镜像、启动和停止容器 5。
        

一个稳健的生产级HA架构并非简单的组件复制，而是多层面的冗余设计。这包括API服务器的高可用、API访问入口的高可用，以及集群状态存储的高可用。通过将`HAProxy`和`Keepalived`作为外部负载均衡器，可以为多个`kube-apiserver`实例提供一个统一的虚拟IP（VIP）入口，从而解决了API访问的单点故障。同时，`etcd`集群通过奇数个成员的配置，确保了数据存储的冗余与一致性。这种多层面的冗余设计是抵御不同层面故障、保障集群稳定运行的关键。

### 1.2. 部署架构设计

本部署方案采用“堆叠式`etcd`拓扑”（Stacked etcd Topology），即`etcd`成员与控制平面组件部署在同一组节点上 4。这种架构在3个控制平面节点时，既满足

`etcd`集群的最小高可用要求（奇数个成员），又提供了控制平面的高可用性，是一种资源高效且广泛应用的生产模式。

以下架构图清晰地展示了各组件之间的关系与数据流向：

代码段

``` mermaid
graph TD
    subgraph "客户端"
        A[运维主机]
    end

    subgraph "高可用负载均衡层"
        LB_VIP(fa:fa-globe 虚拟IP VIP)
        LB1[HAProxy & Keepalived 主]
        LB2[HAProxy & Keepalived 备]
    end

    subgraph "控制平面节点"
        direction LR
        subgraph Master1
            M1_KAPI(fa:fa-server kube-apiserver)
            M1_KETCD(etcd)
            M1_KCM(kube-controller-manager)
            M1_KSCH(kube-scheduler)
            M1_KUBELET(kubelet)
            M1_KPROXY(kube-proxy)
        end
        subgraph Master2
            M2_KAPI(fa:fa-server kube-apiserver)
            M2_KETCD(etcd)
            M2_KCM(kube-controller-manager)
            M2_KSCH(kube-scheduler)
            M2_KUBELET(kubelet)
            M2_KPROXY(kube-proxy)
        end
        subgraph Master3
            M3_KAPI(fa:fa-server kube-apiserver)
            M3_KETCD(etcd)
            M3_KCM(kube-controller-manager)
            M3_KSCH(kube-scheduler)
            M3_KUBELET(kubelet)
            M3_KPROXY(kube-proxy)
        end
    end

    subgraph "工作节点"
        WorkerNode1(Worker 1)
        WorkerNode2(Worker 2)
        WorkerNodeN(fa:fa-ellipsis-h...)
    end

    A -- "kubectl" --> LB_VIP
    LB_VIP -- "TCP 6443" --> LB1
    LB1 -- "TCP 6443" --> M1_KAPI
    LB1 -- "TCP 6443" --> M2_KAPI
    LB1 -- "TCP 6443" --> M3_KAPI
    LB2 -- "TCP 6443" --> M1_KAPI
    LB2 -- "TCP 6443" --> M2_KAPI
    LB2 -- "TCP 6443" --> M3_KAPI

    M1_KUBELET --> M1_KAPI
    M2_KUBELET --> M2_KAPI
    M3_KUBELET --> M3_KAPI
    WorkerNode1 --> LB_VIP
    WorkerNode2 --> LB_VIP
    WorkerNodeN --> LB_VIP

    M1_KAPI -- "读/写" --> M1_KETCD
    M2_KAPI -- "读/写" --> M2_KETCD
    M3_KAPI -- "读/写" --> M3_KETCD
    M1_KETCD <--> M2_KETCD
    M1_KETCD <--> M3_KETCD
    M2_KETCD <--> M3_KETCD
```

## 2. 预备阶段：在线环境资源打包

本阶段为整个离线部署的基石，必须在有互联网连接的机器上，完整地收集所有必要的资源。这个过程不仅是简单的文件下载，更是一个关键的工程决策过程，涉及到组件间的版本兼容性管理。

### 2.1. 硬件与软件前置条件

在准备资源之前，必须确保所有集群节点满足以下最低要求：

|角色|最低vCPU|最低RAM|最低存储|推荐操作系统|前置配置项|
|---|---|---|---|---|---|
|控制平面|2|2GB|40GB+|Ubuntu/CentOS|禁用Swap, 配置内核参数|
|工作节点|1|2GB|40GB+|Ubuntu/CentOS|禁用Swap, 配置内核参数|
|负载均衡器|1|1GB|10GB+|Ubuntu/CentOS|禁用Swap, 配置内核参数|

所有节点必须满足以下条件：

- 所有节点都拥有唯一的`hostname`、MAC地址和`product_uuid` 7。
    
- 各节点之间具备完全的网络连通性 6。
    
- 已开放必要的端口，例如API服务器的6443端口，`etcd`的2379和2380端口等 7。
    

在选择Kubernetes版本后，必须查阅官方文档或兼容性矩阵，以确定配套的容器运行时（Containerd）和网络插件（CNI，如Calico）版本，这是离线部署成功的关键。以下是根据研究资料整理的部分兼容性参考：

|Kubernetes 版本|Containerd 版本|Docker 版本|Calico 版本|
|---|---|---|---|
|v1.27.4|1.6.4|-|v3.26.1|
|v1.26.7|1.6.4|-|v3.26.1|
|v1.25.4|1.6.4|-|v3.22.4|
|v1.24.8|1.6.4|-|v3.22.4|
|v1.23.9|1.6.4|20.10.20|v3.22.4|
|v1.22.12|1.6.4|20.10.20|v3.22.4|
|v1.21.14|1.6.4|20.10.20|v3.22.4|

### 2.2. 核心二进制文件与软件包下载

此步骤需要在一个可以访问互联网的Linux机器上执行。

1. **下载K8s核心组件包**：根据操作系统类型，下载`kubeadm`、`kubelet`和`kubectl`的`rpm`或`deb`软件包 7。例如，对于CentOS，可以使用
    
    `wget`下载`https://packages.cloud.google.com/yum/pool/...`路径下的rpm包。
    
2. **下载容器运行时包**：选择并下载Containerd或CRI-O的软件包。例如，可以从GitHub或其他官方源下载相应的tarball。
    
3. **下载其他工具**：`HAProxy`和`Keepalived`也需要提前下载相应的软件包。
    
4. **下载CNI网络插件清单**：以Calico为例，从其GitHub仓库下载最新的`custom-resources.yaml`清单文件 8。此文件通常引用了多个Calico的容器镜像，需要仔细检查。
    

### 2.3. 容器镜像预拉取与打包

所有Kubernetes核心组件和附加组件都是以容器形式运行的。离线部署要求将所有这些镜像预先拉取并打包。

1. **获取镜像列表**：使用以下命令获取当前`kubeadm`版本所需的所有核心镜像列表 9。
    
    ``` Bash
    kubeadm config images list --kubernetes-version=v1.27.4
    ```
    
2. **手动补充镜像列表**：检查CNI插件（如Calico）的`YAML`清单，提取其中引用的所有镜像，并将它们添加到镜像列表中。同时，不要忘记其他可能需要的镜像，例如`nginx:alpine`用于测试等 11。
    
3. **拉取镜像**：使用`docker pull`命令逐个拉取镜像。
    
4. **打包镜像**：使用`docker save`命令将所有镜像打包成一个或多个`tar`文件。
    
    ``` Bash
    # 将单个镜像打包
    docker save -o hello-world.tar hello-world:latest
    
    # 将多个镜像打包到一个文件中
    docker save -o k8s_images.tar image1:tag1 image2:tag2...
    ```
    
    此命令的优点在于可以完整保留镜像的仓库和标签信息，便于后续离线环境中的加载 12。
    

### 2.4. 资源包组织与转移

将所有下载的二进制文件、软件包、容器镜像`tar`包和配置文件（如CNI清单）组织成一个清晰的目录结构，然后压缩成单个文件（例如`offline-k8s.tar.gz`），并通过USB、SCP或专用网络设备转移到离线环境中的所有节点上。

## 3. 离线环境：节点基础配置

此阶段将在离线环境中的所有节点上执行。

### 3.1. 所有节点通用配置

1. **禁用Swap**：`kubelet`要求所有节点必须禁用`swap`，否则`kubeadm preflight`检查会失败。
    
    ``` Bash
    # 临时禁用
    sudo swapoff -a
    
    # 永久禁用，编辑 /etc/fstab 文件，注释掉所有swap分区行
    # /dev/mapper/centos-swap swap swap defaults 0 0
    ```
    
2. **配置内核参数**：`kubelet`和`CNI`网络插件要求配置特定的内核参数，以确保容器网络正常工作。
    
    ``` Bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    
    sudo modprobe overlay
    sudo modprobe br_netfilter
    
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1
    EOF
    
    sudo sysctl --system
    ```
    
    这些参数使得`iptables`能够正确处理桥接流量 5。
    
3. **配置主机名与Hosts文件**：确保每个节点都拥有一个唯一的`hostname`，并在所有节点的`/etc/hosts`文件中添加所有节点的IP地址与主机名映射，以确保内部通信畅通。
    

### 3.2. 安装容器运行时与加载镜像

1. **安装Containerd**：从离线资源包中安装Containerd软件包。
    
2. **配置`cgroup`驱动**：`kubelet`和容器运行时必须使用相同的`cgroup`驱动。现代Linux系统默认使用`systemd`作为`cgroup`驱动。需编辑`containerd`的配置文件（`/etc/containerd/config.toml`），确保其`SystemdCgroup`设置为`true` 14。
    
3. **启动服务**：启动`containerd`服务。
    
4. **加载镜像**：使用`docker load`命令，将之前打包的`tar`镜像包加载到所有节点的本地镜像仓库。
    
    ``` Bash
    sudo docker load -i k8s_images.tar
    ```
    
    在离线环境中，加载本地镜像文件是唯一可行的方案 13。
    

### 3.3. 安装`kubelet`、`kubeadm`和`kubectl`

1. **安装软件包**：在所有节点上，使用`rpm -ivh`或`dpkg -i`命令安装离线软件包。
    
2. **启动`kubelet`**：`kubelet`是节点上的核心代理，需要在`kubeadm`初始化前启动并设置为开机自启。
    
    ``` Bash
    sudo systemctl enable --now kubelet
    ```
    

这个阶段的工作是为`kubeadm`初始化做铺垫。`kubeadm`在启动时会执行一系列预检（preflight checks），包括`swap`状态、内核模块、端口可用性等。任何一个环节的配置失误，都会导致`kubeadm`预检失败。因此，严格遵循此阶段的每一个配置步骤，是确保后续安装过程顺利进行的根本保障。

## 4. 高可用负载均衡器与VIP配置

高可用负载均衡器是HA架构的“入口”。本方案使用`HAProxy`作为TCP层负载均衡器，负责将对API服务器的请求分发到多个控制平面节点；同时使用`Keepalived`管理一个虚拟IP（VIP），确保`HAProxy`服务本身的高可用性。

### 4.1. HAProxy配置指南

1. **安装**：在负载均衡器节点上安装`HAProxy`。
    
2. **配置文件`haproxy.cfg`**：提供一个HAProxy配置文件模板，并逐行解释关键配置项。该配置将监听VIP地址的6443端口，并将流量分发到所有控制平面节点的`kube-apiserver`。
    
    ``` Bash
    global
        log         /dev/log local0 warning
        chroot      /var/lib/haproxy
        stats socket /var/run/haproxy.sock user haproxy group haproxy mode 660 level admin expose-fd listeners
        maxconn     4000
        user        haproxy
        group       haproxy
        daemon
    
    defaults
        log global
        mode tcp
        option tcplog
        option dontlognull
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms
    
    frontend kube-apiserver
        bind *:6443
        mode tcp
        default_backend kube-apiserver-backend
    
    backend kube-apiserver-backend
        mode tcp
        balance roundrobin
        server master1 <Master1_IP>:6443 check
        server master2 <Master2_IP>:6443 check
        server master3 <Master3_IP>:6443 check
    ```
    
    其中，`<MasterX_IP>`需要替换为实际控制平面节点的IP地址 15。
    
3. **启动服务**：启动并设置`HAProxy`服务开机自启。
    

### 4.2. Keepalived配置指南

1. **安装**：在负载均衡器节点上安装`Keepalived`。
    
2. **配置文件`keepalived.conf`**：配置`Keepalived`以管理VIP。以下是主节点的配置示例，备用节点的`state`应为`BACKUP`，`priority`应低于主节点。
    
    ``` Bash
    global_defs {
        router_id LVS_DEVEL
    }
    
    vrrp_script chk_haproxy {
        script "killall -0 haproxy"
        interval 2
        weight 2
    }
    
    vrrp_instance haproxy-vip {
        state MASTER
        interface eth0  # 替换为实际网卡名称
        virtual_router_id 60
        priority 101     # 主节点优先级较高
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
    
        virtual_ipaddress {
            <VIP_Address>/24
        }
    
        track_script {
            chk_haproxy
        }
    }
    ```
    
    `vrrp_script`用于周期性检查`HAProxy`进程状态，如果主节点上的`HAProxy`宕机，`Keepalived`将自动进行VIP漂移 14。
    
3. **启动服务**：启动并设置`Keepalived`服务开机自启。
    

负载均衡器的配置是连接外部客户端与内部集群的桥梁。这里配置的虚拟IP（VIP）地址，将作为整个集群的统一入口，也即后续`kubeadm`配置中的`--control-plane-endpoint`参数。这个VIP不仅是网络层面的地址，更是API服务器证书中的一个`SAN`（Subject Alternative Name），确保所有通过此VIP进行的通信都具备正确的身份认证 14。

## 5. 集群引导与节点加入

本阶段使用`kubeadm`工具，在离线环境中完成集群的初始化和节点加入。

### 5.1. 在第一个控制平面节点上初始化集群

为了实现HA部署，推荐使用配置文件而非命令行参数进行初始化。

1. **创建配置文件**：在第一个控制平面节点上创建`kubeadm-config.yaml`文件。
    
    ``` YAML
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: InitConfiguration
    localAPIEndpoint:
      advertiseAddress: <Master1_IP>
      bindPort: 6443
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: <Master1_IP>
    ---
    apiVersion: kubeadm.k8s.io/v1beta3
    kind: ClusterConfiguration
    kubernetesVersion: v1.27.4 # 替换为实际版本
    controlPlaneEndpoint: "<VIP_Address>:6443"
    networking:
      podSubnet: "10.244.0.0/16" # CNI插件的Pod网络段
    apiServer:
      certSANs:
      - "<VIP_Address>"
      - "<Master1_IP>"
      - "<Master2_IP>"
      - "<Master3_IP>"
      - "<LoadBalancer1_IP>"
      - "<LoadBalancer2_IP>"
    etcd:
      local:
        dataDir: /var/lib/etcd
    ```
    
    此配置中，`controlPlaneEndpoint`被设置为负载均衡器的VIP地址，`certSANs`列表则包含了VIP和所有控制平面节点的IP地址，这是HA部署的关键 6。
    
2. **执行初始化**：
    
    ``` Bash
    sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
    ```
    
    `--upload-certs`参数至关重要，它会自动将集群CA证书和密钥上传至一个名为`kubeadm-certs`的Secret中，并生成一个有时效性的密钥。其他控制平面节点可以使用这个密钥安全地加入集群，并解密下载这些证书。这是一种自动化且安全的证书分发机制，避免了在离线环境中手动拷贝敏感文件的风险 6。
    
3. **保存加入命令**：`kubeadm init`命令执行成功后，将打印出两个关键的`kubeadm join`命令：一个用于加入其他控制平面节点，一个用于加入工作节点。请务必将这两个命令保存下来。
    

### 5.2. 加入其他控制平面节点

在第二个和第三个控制平面节点上，执行上一步保存的加入命令。请注意，该命令必须包含`--control-plane`和`--certificate-key`两个参数 6。

``` Bash
# kubeadm join命令示例
sudo kubeadm join <VIP_Address>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <certificate-key>
```

`--certificate-key`是用于解密和下载共享证书的密钥，其默认有效期为两小时，因此必须在此期间内完成所有控制平面节点的加入操作 6。

### 5.3. 加入工作节点

在所有工作节点上，执行上一步保存的工作节点加入命令。此命令与控制平面节点命令不同，不包含`--control-plane`和`--certificate-key`参数。

## 6. 离线集群配置与验证

完成集群初始化和节点加入后，最后一步是安装网络插件并验证集群状态。

### 6.1. 安装容器网络接口（CNI）插件

`kubeadm`本身不安装CNI，因此需要手动部署。

1. **编辑CNI清单**：以Calico为例，打开之前下载的`custom-resources.yaml`文件。根据`kubeadm`初始化时指定的`podSubnet`（例如`10.244.0.0/16`），确保Calico配置中的`IPPools`也使用相同的网段。
    
2. **应用清单**：在第一个控制平面节点上，使用`kubectl`应用`YAML`清单。
    
    ``` Bash
    kubectl apply -f custom-resources.yaml
    ```
    
3. **配置`iptables`后端**：如果操作系统使用的是`nftables`，Calico可能无法正常工作。此时，需要在Calico的`custom-resources.yaml`文件中，为`calico-node`容器添加`FELIX_IPTABLESBACKEND: "NFT"`环境变量 8。
    

### 6.2. 验证集群健康状态

1. **节点状态**：检查所有节点是否已成功加入集群。
    
    ``` Bash
    kubectl get nodes
    ```
    
    预期输出是所有节点都处于`Ready`状态。
    
2. **核心组件状态**：检查`kube-system`命名空间下所有Pod是否正常运行。
    
    ``` Bash
    kubectl get pods -A
    ```
    
    只有当CNI插件成功部署并运行后，如`coredns`等依赖于网络通信的Pod才能正常启动。因此，检查所有Pod的状态是验证集群内网络连通性的重要一步 5。
    
3. **获取`kubeconfig`**：`kubeconfig`文件位于`/etc/kubernetes/admin.conf`。将此文件安全地复制到需要访问集群的客户端机器上，并设置`KUBECONFIG`环境变量。
    

## 7. 运维与灾备

离线环境的运维和灾备策略必须具备自给自足的能力，无法依赖外部云服务或在线工具。

### 7.1. `etcd`集群快照备份与恢复

`etcd`是集群状态的唯一真理来源，定期备份至关重要 3。

- **备份命令**：在任一`etcd`节点上执行`etcdctl`命令进行快照备份。
    
    ``` Bash
    ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd/backup.db
    ```
    
- **恢复步骤**：在灾难发生时，可以使用`etcdctl`从快照恢复数据，然后重新初始化控制平面。
    

### 7.2. 节点维护与故障排查简述

- **证书管理**：使用`kubeadm certs check-expiration`命令定期检查集群证书的有效期，并在过期前进行续期 6。
    
- **调试工具**：在离线环境中，`crictl`工具对于调试容器运行时是不可或缺的，它可以直接与CRI接口交互，用于检查容器和镜像状态 5。
    
    `etcdctl`则用于检查`etcd`集群的健康状态和数据 3。掌握这些本地化工具和方法，是离线环境中有效运维的根本。