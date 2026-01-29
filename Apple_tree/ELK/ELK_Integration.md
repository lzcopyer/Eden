# ELK 集成配置示例

>本文档展示如何将 Elasticsearch、Logstash 和 Kibana 集成在一起，构建完整的日志分析系统。

# 集成概述

## 端到端数据流

ELK 集成的核心是构建一个从数据收集到可视化的完整管道：

1. **数据源**: 应用程序日志、系统日志、数据库日志等
2. **Logstash**: 收集、解析、转换数据
3. **Elasticsearch**: 存储、索引、搜索数据
4. **Kibana**: 可视化、分析、监控数据

> [!tip] 提示
ELK 集成的关键在于确保三个组件之间的顺畅通信。Logstash 将处理后的数据发送到 Elasticsearch，Elasticsearch 存储数据并提供搜索能力，Kibana 从 Elasticsearch 读取数据并提供可视化界面。这种设计使每个组件专注于自己的职责，同时形成一个强大的数据分析平台。

## 集成架构

### 基础架构

```
应用程序日志 → Logstash → Elasticsearch → Kibana
     ↓              ↓            ↓           ↓
  Filebeat     数据处理      数据存储     数据可视化
```

### 高可用架构

```
[Filebeat] → [Logstash Cluster] → [Elasticsearch Cluster] → [Kibana Instances]
     ↓             ↓                     ↓                      ↓
  多个数据源    负载均衡处理         分布式存储              负载均衡访问
```

# 实际部署示例

## 部署前准备

### 系统要求

- **硬件**: 至少 8GB RAM，4核 CPU，50GB 可用磁盘空间
- **操作系统**: Linux (Ubuntu 18.04+, CentOS 7+)
- **Java**: OpenJDK 11
- **网络**: 确保各组件间网络互通

### 安全设置

```bash
# 创建专用用户
sudo useradd -r -s /bin/false elasticsearch
sudo useradd -r -s /bin/false logstash
sudo useradd -r -s /bin/false kibana

# 设置目录权限
sudo mkdir -p /var/lib/elasticsearch
sudo mkdir -p /var/lib/logstash
sudo mkdir -p /var/lib/kibana
sudo mkdir -p /var/log/elasticsearch
sudo mkdir -p /var/log/logstash
sudo mkdir -p /var/log/kibana

sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch
sudo chown -R elasticsearch:elasticsearch /var/log/elasticsearch
sudo chown -R logstash:logstash /var/lib/logstash
sudo chown -R logstash:logstash /var/log/logstash
sudo chown -R kibana:kibana /var/lib/kibana
sudo chown -R kibana:kibana /var/log/kibana
```

## 完整配置示例

### 1. Elasticsearch 配置 (elasticsearch.yml)

```yaml
# 集群配置
cluster.name: elk-cluster
node.name: elk-node-1
node.roles: ["master", "data", "ingest"]

# 网络配置
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# 集群发现
discovery.type: single-node

# 内存锁定
bootstrap.memory_lock: true

# 路径配置
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# 安全配置
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

### 2. Logstash 配置 (logstash.conf)

```ruby
input {
  # 从 Filebeat 接收数据
  beats {
    port => 5044
  }
  
  # 从标准输入接收（用于测试）
  stdin {}
}

filter {
  # 根据日志类型进行不同的处理
  if [type] == "nginx-access" {
    grok {
      match => { "message" => "%{IPORHOST:client_ip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:method} %{NOTSPACE:request}(?: HTTP/%{NUMBER:http_version})?|%{DATA:raw_request})\" %{NUMBER:response_code:int} (?:%{NUMBER:bytes:int}|-) %{QS:referrer} %{QS:agent}" }
    }
    
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
      target => "@timestamp"
    }
    
    mutate {
      convert => [ "response_code", "integer" ]
      convert => [ "bytes", "integer" ]
    }
  }
  
  if [type] == "system-logs" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:hostname} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:message_body}" }
    }
    
    date {
      match => [ "timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      target => "@timestamp"
    }
  }
  
  # 添加地理信息（如果包含 IP）
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geo_location"
    }
  }
}

output {
  # 发送到 Elasticsearch
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "elk-logs-%{+YYYY.MM.dd}"
    template_name => "elk-logs"
    template_pattern => "elk-logs-*"
    manage_template => true
    template_overwrite => true
  }
  
  # 调试输出
  stdout {
    codec => rubydebug
  }
}
```

### 3. Kibana 配置 (kibana.yml)

```yaml
# 服务器配置
server.port: 5601
server.host: "0.0.0.0"
server.name: "kibana-server"
server.publicBaseUrl: "http://your-domain.com:5601"

# Elasticsearch 配置
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "your_kibana_password"

# 区域设置
i18n.locale: "zh-CN"

# 日志配置
logging.dest: /var/log/kibana/kibana.log
logging.level: info

# 安全配置
elasticsearch.ssl.certificateAuthorities: ["/path/to/ca.crt"]
xpack.security.enabled: true
```

### 4. Filebeat 配置 (filebeat.yml)

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
  fields:
    type: nginx-access
  fields_under_root: true
  
- type: log
  enabled: true
  paths:
    - /var/log/syslog
    - /var/log/messages
  fields:
    type: system-logs
  fields_under_root: true

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

setup.template.settings:
  index.number_of_shards: 1
  index.number_of_replicas: 1

output.logstash:
  hosts: ["localhost:5044"]

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

## 部署步骤

### 1. 启动 Elasticsearch

```bash
# 启动 Elasticsearch
sudo systemctl start elasticsearch

# 检查状态
curl -X GET "localhost:9200/_cluster/health?pretty"

# 设置初始密码（首次启动后）
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-passwords auto
```

### 2. 配置 Logstash

```bash
# 测试 Logstash 配置
sudo -u logstash /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/ --config.test_and_exit

# 启动 Logstash
sudo systemctl start logstash
```

### 3. 配置 Kibana

```bash
# 启动 Kibana
sudo systemctl start kibana

# 等待 Kibana 初始化完成
sudo tail -f /var/log/kibana/kibana.log
```

### 4. 配置 Filebeat

```bash
# 测试 Filebeat 配置
sudo /usr/share/filebeat/bin/filebeat test config -c /etc/filebeat/filebeat.yml

# 启动 Filebeat
sudo systemctl start filebeat
```

## 数据验证

### 检查数据流入

```bash
# 检查 Elasticsearch 中的索引
curl -X GET "localhost:9200/_cat/indices?v"

# 检查 Logstash 统计信息
curl -X GET "localhost:9600/_node/stats/pipeline?pretty"

# 检查索引中的文档数量
curl -X GET "localhost:9200/elk-logs-*/_count?pretty"
```

### 在 Kibana 中验证

1. 登录 Kibana 界面
2. 进入 "Stack Management" > "Index Patterns"
3. 创建新的索引模式 `elk-logs-*`
4. 进入 "Discover" 标签页，验证数据是否正确显示

## 性能调优

### Elasticsearch 优化

```yaml
# 线程池优化
thread_pool.index.queue_size: 1000
thread_pool.search.queue_size: 1000
thread_pool.bulk.queue_size: 500

# 缓存优化
indices.fielddata.cache.size: 40%
indices.query.bool.max_clause_count: 1024

# 分片优化
index.number_of_shards: 5
index.number_of_replicas: 1
```

### Logstash 优化

```yaml
# 管道优化
pipeline.workers: 4
pipeline.batch.size: 125
pipeline.batch.delay: 50

# 队列优化
queue.type: persisted
queue.max_bytes: 1gb
```

## 监控和维护

### 常用监控命令

```bash
# 检查集群健康
curl -X GET "localhost:9200/_cluster/health?pretty"

# 检查节点状态
curl -X GET "localhost:9200/_cat/nodes?v"

# 检查索引状态
curl -X GET "localhost:9200/_cat/indices?v&h=index,docs.count,store.size"

# 检查 Logstash 统计
curl -X GET "localhost:9600/_node/stats?pretty"
```

### 日志轮转配置

```bash
# Elasticsearch 日志轮转 (/etc/logrotate.d/elasticsearch)
/var/log/elasticsearch/*.log {
    rotate 4
    weekly
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        systemctl kill --signal=USR1 elasticsearch.service
    endscript
}
```

## 故障排除

### 常见问题

1. **Logstash 无法连接 Elasticsearch**
   - 检查 Elasticsearch 是否运行
   - 检查网络连接
   - 验证用户名密码

2. **Kibana 无法显示数据**
   - 检查索引模式配置
   - 验证时间范围设置
   - 检查 Elasticsearch 权限

3. **性能问题**
   - 检查系统资源使用情况
   - 优化配置参数
   - 考虑水平扩展

### 调试技巧

```bash
# 启用 Logstash 调试模式
sudo -u logstash /usr/share/logstash/bin/logstash -f /path/to/config.conf --log.level debug

# 查看 Elasticsearch 慢查询日志
curl -X PUT "localhost:9200/_settings?pretty" -H 'Content-Type: application/json' -d'
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.fetch.warn": "1s"
}'
```

# 参考链接

[ELK Stack 官方指南](https://www.elastic.co/guide/en/elastic-stack/current/index.html)
[ELK 部署最佳实践](https://www.elastic.co/guide/en/elastic-stack-deployment/current/elastic-stack-deploy-best-practices.html)