# 企业级 Prometheus 高可用（HA）二进制部署方案

本方案基于 **Prometheus + Thanos Sidecar** 架构实现数据持久化与全局视图，结合 **Alertmanager 集群** 实现告警高可用，并提供 **联邦集群** 配置示例。所有组件均采用二进制方式部署，适用于生产级物理机或虚拟机环境。

---

## 1. 前置准备

### 1.1 系统环境要求

- **操作系统**: CentOS 7.9 / Ubuntu 20.04 LTS 或更高版本
    
- **硬件资源 (推荐)**:
    
    - CPU: 4 Core+
        
    - Memory: 16GB+ (生产环境建议内存预留量为：`Target数 * 样本大小 * 2`)
        
    - Disk: SSD (强烈建议，Prometheus 对 IOPS 敏感)
        
- **网络**: 需开放节点间 TCP 端口（9090, 9093, 10901, 10902 等）
    
- **时间同步**: 所有节点必须配置 NTP 同步（如 `chrony`），时间偏差过大会导致 Alertmanager 集群通信失败或数据写入拒绝。
    

### 1.2 架构规划

- **Prometheus 节点**: 2 台（双活 HA），配置 `replica` 标签区分。
    
- **Alertmanager 节点**: 3 台（组成 Gossip 集群）。
    
- **Thanos 组件**: Sidecar（随 Prometheus 部署）、Querier（独立部署用于查询聚合）。

Prometheus 高可用与 Thanos 架构图

``` mermaid
graph TD
    subgraph "Prometheus HA Cluster (Data Center A)"
        P1[Prometheus Server A] -- "Scrape" --> Targets[Monitoring Targets]
        S1[Thanos Sidecar A] -- "Read TSDB" --> P1
    end

    subgraph "Prometheus HA Cluster (Data Center B)"
        P2[Prometheus Server B] -- "Scrape" --> Targets
        S2[Thanos Sidecar B] -- "Read TSDB" --> P2
    end

    subgraph "Alerting System"
        AM1[Alertmanager 1] <--> AM2[Alertmanager 2]
        AM2 <--> AM3[Alertmanager 3]
        AM3 <--> AM1
        P1 -- "Push Alerts" --> AM1
        P2 -- "Push Alerts" --> AM1
    end

    subgraph "Storage & Global View"
        S3[(Object Storage / S3)]
        S1 -- "Upload Blocks" --> S3
        S2 -- "Upload Blocks" --> S3
        
        TQ[Thanos Querier] -- "gRPC Query" --> S1
        TQ -- "gRPC Query" --> S2
        TQ -- "Store Gateway" --> S3
    end

    subgraph "Visualization"
        Grafana[Grafana UI] -- "PromQL" --> TQ
    end

    style P1 fill:#f96,stroke:#333
    style P2 fill:#f96,stroke:#333
    style TQ fill:#69f,stroke:#333
    style S3 fill:#9c9,stroke:#333
    style AM1 fill:#f66,stroke:#333
```

---

## 2. 二进制包下载与校验

建议将安装包统一下载至 `/opt/software` 目录。

``` Bash
# 定义版本变量
export PROM_VER="2.45.0"
export AM_VER="0.26.0"
export THANOS_VER="0.32.0"

# 下载 Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VER}/prometheus-${PROM_VER}.linux-amd64.tar.gz
# 下载 Alertmanager
wget https://github.com/prometheus/alertmanager/releases/download/v${AM_VER}/alertmanager-${AM_VER}.linux-amd64.tar.gz
# 下载 Thanos
wget https://github.com/thanos-io/thanos/releases/download/v${THANOS_VER}/thanos-${THANOS_VER}.linux-amd64.tar.gz

# SHA256 校验 (示例)
sha256sum -c <(grep prometheus-${PROM_VER}.linux-amd64.tar.gz sha256sums.txt)
```

---

## 3. 用户与目录规划

为了安全起见，禁止使用 root 运行服务。

``` Bash
# 创建系统用户
groupadd prometheus
useradd -g prometheus -s /sbin/nologin -M prometheus

# 创建目录结构
mkdir -p /etc/prometheus/{rules,secrets}
mkdir -p /var/lib/prometheus/data
mkdir -p /var/lib/alertmanager/data
mkdir -p /etc/alertmanager/templates
mkdir -p /var/log/prometheus

# 授权
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus /var/log/prometheus
chown -R prometheus:prometheus /etc/alertmanager /var/lib/alertmanager
```

---

## 4. 系统级优化

在所有节点执行以下内核参数优化，以支撑高并发采集和查询。

### 4.1 ulimit 调优

编辑 `/etc/security/limits.conf`：

``` Plaintext
prometheus  soft  nofile  65536
prometheus  hard  nofile  65536
prometheus  soft  nproc   4096
prometheus  hard  nproc   4096
```

### 4.2 Sysctl 内核参数

创建 `/etc/sysctl.d/99-prometheus.conf`：

``` Bash
# 增加连接队列长度，防止瞬时高并发导致丢包
net.core.somaxconn = 4096
# 允许分配更多内存
vm.overcommit_memory = 1
# 尽量减少 swap 使用，Prometheus 强依赖内存，swap 会导致性能急剧下降
vm.swappiness = 1
# 增加 TCP 端口范围
net.ipv4.ip_local_port_range = 1024 65000
```

执行 `sysctl -p /etc/sysctl.d/99-prometheus.conf` 生效。

---

## 5. Prometheus HA + Thanos Sidecar 部署

Prometheus 将以双副本模式运行（Replica A 和 Replica B），数据保留在本地，Thanos Sidecar 负责将旧数据上传至对象存储并暴露 StoreAPI 供查询。

### 5.1 安装二进制文件

``` Bash
tar -xvf prometheus-${PROM_VER}.linux-amd64.tar.gz
cp prometheus-${PROM_VER}.linux-amd64/{prometheus,promtool} /usr/local/bin/
chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}

tar -xvf thanos-${THANOS_VER}.linux-amd64.tar.gz
cp thanos-${THANOS_VER}.linux-amd64/thanos /usr/local/bin/
chown prometheus:prometheus /usr/local/bin/thanos
```

### 5.2 配置文件编写 (`prometheus.yml`)

在 **节点 A** (`{{IP_PROMETHEUS_01}}`) 和 **节点 B** (`{{IP_PROMETHEUS_02}}`) 上配置，注意 `external_labels` 的差异。

``` YAML
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  # 关键：HA 标识，两个节点 cluster 名必须一致，replica 必须不同
  external_labels:
    cluster: '{{CLUSTER_NAME}}'
    replica: '{{REPLICA_NAME}}'  # 节点A填 'replica-01', 节点B填 'replica-02'

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - '{{IP_ALERTMANAGER_01}}:9093'
      - '{{IP_ALERTMANAGER_02}}:9093'
      - '{{IP_ALERTMANAGER_03}}:9093'

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  # 示例：Node Exporter
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['{{IP_TARGET_01}}:9100', '{{IP_TARGET_02}}:9100']
```

### 5.3 Thanos 对象存储配置

创建 `/etc/prometheus/bucket_config.yaml`：

``` YAML
type: S3
config:
  bucket: "{{S3_BUCKET_NAME}}"
  endpoint: "{{S3_ENDPOINT}}"
  access_key: "{{S3_ACCESS_KEY}}"
  secret_key: "{{S3_SECRET_KEY}}"
  insecure: false
```

### 5.4 Systemd 服务管理

#### Prometheus 服务单元

`/usr/lib/systemd/system/prometheus.service`：

``` TOML
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/data \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.min-block-duration=2h \
  --storage.tsdb.max-block-duration=2h \
  --web.enable-lifecycle \
  --web.enable-admin-api \
  --log.level=info

ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

_注意：`min-block-duration` 和 `max-block-duration` 设为 2h 是为了配合 Thanos Sidecar 的上传逻辑。_

#### Thanos Sidecar 服务单元

`/usr/lib/systemd/system/thanos-sidecar.service`：

``` TOML
[Unit]
Description=Thanos Sidecar
Wants=network-online.target
After=prometheus.service

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/thanos sidecar \
  --tsdb.path=/var/lib/prometheus/data \
  --prometheus.url=http://localhost:9090 \
  --objstore.config-file=/etc/prometheus/bucket_config.yaml \
  --grpc-address=0.0.0.0:10901 \
  --http-address=0.0.0.0:10902

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

启动服务：

``` Bash
systemctl daemon-reload
systemctl enable --now prometheus thanos-sidecar
```

---

## 6. Alertmanager 高可用部署

部署 3 个节点组成 Gossip 集群，防止单点故障。

### 6.1 安装与配置

将 Alertmanager 二进制解压至 `/usr/local/bin/`。

配置文件 `/etc/alertmanager/alertmanager.yml` (所有节点相同)：

``` YAML
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'web.hook'

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: '{{WEBHOOK_URL}}'
```

### 6.2 Systemd 服务配置 (集群启动)

`/usr/lib/systemd/system/alertmanager.service`：

``` TOML
[Unit]
Description=Alertmanager
After=network.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager/data \
  --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer={{IP_ALERTMANAGER_01}}:9094 \
  --cluster.peer={{IP_ALERTMANAGER_02}}:9094 \
  --cluster.peer={{IP_ALERTMANAGER_03}}:9094 \
  --web.listen-address=0.0.0.0:9093

ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

_注意：`--cluster.peer` 参数需指向集群内所有节点（包括自己，通常推荐写其他节点IP，但在脚本化部署中写全量IP列表由程序自动处理去重也可以，或者只填另外两台的IP）。_

---

## 7. 远程存储集成 (Remote Write)

如果您已有中心化的 Cortex、InfluxDB 或 Thanos Receive 集群，除了使用 Sidecar 模式外，也可以配置 Prometheus 直接推送数据。

编辑 `prometheus.yml`，添加以下段落：

``` YAML
remote_write:
  - url: "{{REMOTE_WRITE_URL}}/api/v1/push"
    # 远程写入优化配置
    queue_config:
      max_shards: 200          # 并发 shard 数
      capacity: 2500           # 队列容量
      max_samples_per_send: 500
    # 安全认证配置
    basic_auth:
      username: "{{REMOTE_AUTH_USER}}"
      password: "{{REMOTE_AUTH_PASS}}"
    # TLS 配置 (如有需要)
    tls_config:
      ca_file: /etc/prometheus/secrets/ca.crt
      cert_file: /etc/prometheus/secrets/client.crt
      key_file: /etc/prometheus/secrets/client.key
      insecure_skip_verify: false
    # 写入重标记（可选：过滤不想上传的指标）
    write_relabel_configs:
      - source_labels: [__name__]
        regex: "^go_.*"
        action: drop
```

---

## 8. 联邦集群配置 (Federation)

如果存在**上级 Prometheus** 需要从当前的 Prometheus HA 集群拉取聚合数据。

### 8.1 上级 Prometheus 配置

在上级 Prometheus 的 `prometheus.yml` 中添加：

``` YAML
scrape_configs:
  - job_name: 'federate-cluster-a'
    scrape_interval: 1m
    scrape_timeout: 1m
    honor_labels: true
    metrics_path: '/federate'
    
    params:
      'match[]':
        - '{job="node_exporter"}'   # 仅拉取特定 Job
        - '{__name__=~"job:.*"}'    # 或拉取预聚合的 Recording Rules
    
    static_configs:
      - targets:
        - '{{IP_PROMETHEUS_01}}:9090'
        - '{{IP_PROMETHEUS_02}}:9090'
```

---

## 9. 高可用数据查询视图 (Thanos Querier)

为了统一查询两个 Prometheus 副本的数据并进行去重，需部署 Thanos Querier。

### 9.1 Querier Systemd 配置

`/usr/lib/systemd/system/thanos-querier.service`：

``` TOML
[Unit]
Description=Thanos Querier
After=network.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/thanos query \
  --http-address=0.0.0.0:10903 \
  --store={{IP_PROMETHEUS_01}}:10901 \
  --store={{IP_PROMETHEUS_02}}:10901 \
  --query.replica-label=replica \
  --query.timeout=2m

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

- `--store`: 指向两个 Sidecar 的 gRPC 端口。
    
- `--query.replica-label=replica`: 关键参数，告诉 Thanos 基于哪个标签进行数据去重（Deduplication）。
    

---

## 10. 日志轮转 (Logrotate)

防止日志占满磁盘，创建 `/etc/logrotate.d/prometheus`：

``` Plaintext
/var/log/prometheus/*.log {
    daily
    rotate 7
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
}
```

---

## 11. 验证步骤

1. **服务状态检查**:
    
    ``` Bash
    systemctl status prometheus thanos-sidecar alertmanager
    ```
    
1. **健康接口检查**:
    
    ``` Bash
    curl -f http://localhost:9090/-/healthy
    curl -f http://localhost:9093/-/healthy
    ```
    
2. Alertmanager 集群验证:
    
    访问任意 Alertmanager 节点的 Web UI (Status -> Cluster)，应看到 3 个 Peer 状态均为 Settle。
    
3. **Prometheus HA 与去重验证**:
    
    - 访问 Thanos Querier UI (`http://{{IP_QUERIER}}:10903`)。
        
    - 查询 `up` 指标。
        
    - **未勾选** "Use Deduplication"：应看到双倍的数据条目（replica-01 和 replica-02）。
        
    - **勾选** "Use Deduplication"：数据合并，显示单一曲线。
        
4. **故障切换测试**:
    
    - 停止 `{{IP_PROMETHEUS_01}}` 的 Prometheus 服务。
        
    - 在 Thanos Querier 继续查询，数据应无中断（自动切换读取 `{{IP_PROMETHEUS_02}}`）。
        
    - 发送测试告警，Alertmanager 集群应能正常去重并发送通知。