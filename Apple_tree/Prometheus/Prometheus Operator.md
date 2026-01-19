
---

# Prometheus简明运维手册

文档版本：{{document_version}} (Draft v1.0)

适用版本：Prometheus {{prometheus_version}} (建议 v2.30+)

部署模式：{{deployment_mode}} (Linux Binary/Systemd)

---

## 1. 核心概念与环境准备

### 1.1 架构简图

Prometheus 采用 Pull（拉取）模型，核心组件交互如下：

``` mermaid
sequenceDiagram
    participant T as Target (Exporter)
    participant P as Prometheus Server
    participant TS as TSDB (Local Disk)
    participant A as Alertmanager
    participant G as Grafana

    T->>P: 1. Expose Metrics (/metrics)
    P->>T: 2. Scrape (Pull Model)
    P->>TS: 3. Store Time Series
    P->>P: 4. Evaluate Rules (Recording/Alerting)
    P->>A: 5. Push Alerts (if triggered)
    G->>P: 6. PromQL Query
    P->>G: 7. Return Data
```

### 1.2 核心配置文件 (`prometheus.yml`)

关键字段说明：

``` YAML
global:
  scrape_interval: 15s      # 抓取频率，建议与 evaluation_interval 保持一致
  evaluation_interval: 15s  # 告警规则评估频率

# 告警与记录规则文件加载
rule_files:
  - "rules/*.yml"

# 抓取目标配置
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets: ["localhost:9100", "{{target_ip}}:9100"]
    # 标签重写（可选）
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

### 1.3 服务状态检查

在 {{deployment_mode}} 模式下，常用检查命令：

- **健康检查**: `curl -f http://localhost:9090/-/healthy` (返回 `Prometheus is Healthy.` 即正常)
    
- **重载配置**: `curl -X POST http://localhost:9090/-/reload` (无需重启进程)
    
- **语法校验**: `promtool check config /etc/prometheus/prometheus.yml`
    

### 1.4 常见配置错误 (Top 3)

1. **YAML缩进错误**: 使用 Tab 而非空格。**修复**: 使用 IDE 插件或 `yamllint` 检查，统一 2 空格缩进。
    
2. **Target 不可达**: 防火墙拦截端口。**修复**: 检查目标机器防火墙及网络连通性 (`telnet {{target_ip}} 9100`)。
    
3. **时间不同步**: 客户端与服务端时间偏差过大。**修复**: 所有节点配置 NTP 同步。
    

---

## 2. PromQL 语法精要

### 2.1 核心函数语义对比

在 {{evaluation_interval}} 评估周期下的差异：

|**函数**|**语义**|**适用场景**|**注意事项**|
|---|---|---|---|
|**rate(v[5m])**|计算区间内每秒平均增长率|长期趋势分析、告警规则（平滑）|必须作用于 Counter 类型，自动处理重置|
|**irate(v[5m])**|计算区间最后两个点的增长率|瞬时尖峰检测（高灵敏度）|图表可能剧烈波动，不建议用于长时间跨度告警|
|**increase(v[5m])**|计算区间内的增量总和|统计总请求量、总错误数|实际上是 `rate() * 300` (5m的秒数)|
|**histogram_quantile**|计算直方图的分位数 (P99/P95)|服务延迟分析|需配合 `le` (less equal) 标签使用|

### 2.2 TOP 10 实用查询模板

#### 1. 节点存活状态

``` promQL
up{job="node_exporter"} == 0
```

- **用途**: 告警目标宕机。
    

#### 2. CPU 使用率 (排除空闲时间)

``` promQL
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- **用途**: 获取节点 CPU 总体负载百分比。
    

#### 3. 内存使用率 (百分比)
 
``` promQL
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

- **用途**: Linux 节点内存真实使用情况。
    

#### 4. 磁盘剩余空间预测 (4小时后是否满)

``` promQL
predict_linear(node_filesystem_free_bytes{fstype!~"tmpfs"}[1h], 4 * 3600) < 0
```

- **用途**: 基于最近1小时趋势，预测未来4小时磁盘是否耗尽。
    

#### 5. 接口 P99 延迟

``` promQL
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler))
```

- **用途**: 查看各接口 99% 的请求耗时。
    

#### 6. 接口错误率 (HTTP 5xx)

``` promQL
sum(rate(http_requests_total{status=~"5.."}[5m])) 
  / 
sum(rate(http_requests_total[5m])) * 100
```

- **用途**: 计算 5xx 错误占总请求的百分比。
    

#### 7. 流量突增检测 (入网流量 > 100MB/s)

``` promQL
rate(node_network_receive_bytes_total[1m]) > 100 * 1024 * 1024
```

- **用途**: 网络带宽饱和预警。
    

#### 8. 容器/进程重启频繁

``` promQL
changes(kube_pod_status_ready[15m]) > 2
```

- **用途**: 检测应用是否处于 CrashLoopBackOff 状态。
    

#### 9. 饱和度监控 (Apdex)

_(需根据具体应用指标定制，通常结合 error rate 和 latency)_

#### 10. 高基数维度排查 (Top 10 Metrics)

``` promQL
topk(10, count by (__name__) ({__name__=~".+"}))
```

- **用途**: 找出哪个指标占用了最多的时间序列资源。
    

---

## 3. 运维管理实践

### 3.1 数据保留策略

修改启动参数以调整 TSDB 策略：

- **保留时间**: `--storage.tsdb.retention.time=15d` (默认15天)
    
- **保留大小**: `--storage.tsdb.retention.size=50GB` (推荐设置，防止磁盘撑爆)
    

### 3.2 告警规则管理 (`rules/alerting_rules.yml`)

**规则编写范例**：

``` YAML
groups:
- name: host_monitoring
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m  # 持续 5 分钟才触发
    labels:
      severity: warning
    annotations:
      summary: "Instance {{ $labels.instance }} CPU usage high"
      description: "CPU usage is above 80% (current value: {{ $value | printf \"%.2f\" }})"
```

**加载验证**:

1. 保存文件。
    
2. `promtool check rules rules/alerting_rules.yml`
    
3. `curl -X POST http://localhost:9090/-/reload`
    

### 3.3 性能瓶颈与优化

- **高基数问题 (High Cardinality)**: 避免在 Label 中使用 IP、User ID、Request ID 等无界数据。如果查询变慢，请检查 `prometheus_tsdb_head_series` 指标。
    
- **查询延迟**: 避免在大时间跨度上使用 `irate` 或复杂的正则匹配。
    

### 3.4 紧急故障处理

- **磁盘满 (Disk Full)**:
    
    1. 停止服务。
        
    2. 删除旧的 block 目录 (`/data/01XXXX...`) 或调整 retention 参数。
        
    3. 启动服务。
        
- **WAL 损坏 (Write Ahead Log Corruption)**:
    
    - 现象: 启动失败，日志报错 `corrupted WAL`。
        
    - 处理: 删除 `/data/wal` 目录 (会丢失最近几小时未落盘数据)。
        

---

## 4. 集成与扩展

### 4.1 Exporter 标准接入流程 (Node Exporter)

1. **下载**: `wget https://github.com/prometheus/node_exporter/releases/download/v{{version}}/node_exporter...`
    
2. **运行**: `./node_exporter` (默认监听 9100)
    
3. **配置**: 在 `prometheus.yml` 添加 job (见 1.2 节)。
    
4. **验证**: 查询 `up{job="node_exporter"}` 值为 1。
    

### 4.2 Grafana 数据源关联

1. Configuration -> Data Sources -> Add data source -> Prometheus.
    
2. **URL**: `http://{{prometheus_server_ip}}:9090`.
    
3. Save & Test -> 显示 "Data source is working"。
    

### 4.3 自定义指标命名规范

命名应遵循 `{{app}}_{{noun}}_{{verb}}_{{unit}}` 格式，以确保可读性。

- **正例**: `http_request_duration_seconds`
    
- **反例**: `my_app_latency` (缺乏单位和具体语义)
    

---

## 5. 附录

### 常用命令速查表

|**动作**|**命令**|
|---|---|
|热加载|`curl -X POST 'http://localhost:9090/-/reload'`|
|启动服务|`systemctl start prometheus`|
|查看日志|`journalctl -u prometheus -f`|
|检查版本|`prometheus --version`|
|API删除数据|`curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={job="old_job"}'` (需开启 `--web.enable-admin-api`)|