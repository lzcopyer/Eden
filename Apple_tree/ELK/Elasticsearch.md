# Elasticsearch 部署与配置

>Elasticsearch 是一个分布式的 RESTful 搜索和分析引擎，能够快速存储、搜索和分析大量数据。

# 体系结构

## 核心概念

Elasticsearch 有以下几个核心概念：

- **节点 (Node)**: 一个 Elasticsearch 实例
- **集群 (Cluster)**: 一个或多个节点的集合
- **索引 (Index)**: 文档的集合，相当于关系数据库中的数据库
- **类型 (Type)**: 索引中的分类，类似表（在新版本中已废弃）
- **文档 (Document)**: 可被索引的基础信息单元，类似记录
- **分片 (Shard)**: 索引的子集，使数据可以分布在多个节点上
- **副本 (Replica)**: 分片的拷贝，提高数据可靠性和查询性能

> [!tip] 提示
Elasticsearch 是一个分布式系统，理解这些核心概念对于正确使用它至关重要。索引是逻辑命名空间，而分片是物理存储单元。通过分片，Elasticsearch 可以将数据分散到多个节点上，实现水平扩展。副本不仅提供冗余，还能提升查询性能。

## 部署环境准备

### 依赖准备

Elasticsearch 环境依赖 Java 8 或更高版本（推荐 OpenJDK）。

#### 安装 Java

```bash
# Ubuntu/Debian 安装 OpenJDK 11
sudo apt update
sudo apt install openjdk-11-jdk

# CentOS/RHEL 安装 OpenJDK 11
sudo yum install java-11-openjdk-devel

# 验证安装
java -version
```

#### 设置 Java 环境变量

```bash
# 查找 Java 安装路径
sudo update-alternatives --config java

# 设置 JAVA_HOME
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64  # Ubuntu 示例
# 或
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk       # CentOS 示例
```

## 安装 Elasticsearch

### 通过 apt/yum 安装（推荐）

```bash
# Ubuntu/Debian
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update && sudo apt install elasticsearch

# CentOS/RHEL
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat <<EOF | sudo tee /etc/yum.repos.d/elastic.repo
[elastic-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
enabled=1
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
EOF
sudo yum install elasticsearch
```

### 通过 tar.gz 包安装

```bash
# 下载 Elasticsearch
cd /tmp
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.11.0-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.11.0-linux-x86_64.tar.gz.sha512

# 验证包完整性
sha512sum -c elasticsearch-8.11.0-linux-x86_64.tar.gz.sha512
tar -zxf elasticsearch-8.11.0-linux-x86_64.tar.gz

# 移动到合适的位置
sudo mv elasticsearch-8.11.0 /opt/elasticsearch
sudo chown -R 1000:1000 /opt/elasticsearch
```

## 配置

### 主配置文件

elasticsearch.yml 是 Elasticsearch 的主配置文件，位于 config/ 目录下。

```yaml
# 集群名称
cluster.name: my-elasticsearch-cluster

# 节点名称
node.name: node-1

# 网络配置
network.host: 0.0.0.0
http.port: 9200

# 集群发现配置（单节点模式）
discovery.type: single-node

# 生产环境配置（多节点集群）
# cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
# discovery.seed_hosts: ["192.168.1.10", "192.168.1.11", "192.168.1.12"]

# 内存配置
bootstrap.memory_lock: true

# 数据和日志路径
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
```

### JVM 配置

jvm.options 文件位于 config/ 目录下，用于配置 Elasticsearch 使用的 JVM 参数。

```bash
# 设置堆内存大小（建议设置为系统内存的一半，但不超过 32GB）
-Xms2g
-Xmx2g

# GC 配置
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+DisableExplicitGC
-XX:+AlwaysPreTouch
```

### 安全配置

在生产环境中，强烈建议启用安全功能。

```yaml
# 启用安全功能
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

## 启动

### systemd 方式（推荐）

```bash
# 启动 Elasticsearch
sudo systemctl start elasticsearch

# 设置开机自启
sudo systemctl enable elasticsearch

# 查看状态
sudo systemctl status elasticsearch

# 查看日志
sudo journalctl -u elasticsearch.service -f
```

### 直接运行方式（仅限 tar 包安装）

```bash
# 切换到 elasticsearch 用户（如果创建了）
sudo -u elasticsearch /opt/elasticsearch/bin/elasticsearch

# 后台运行
sudo -u elasticsearch /opt/elasticsearch/bin/elasticsearch -d
```

## 基本操作

### 检查集群健康状态

```bash
curl -X GET "localhost:9200/_cluster/health?pretty"
```

### 创建索引

```bash
curl -X PUT "localhost:9200/my_index?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}'
```

### 添加文档

```bash
curl -X POST "localhost:9200/my_index/_doc?pretty" -H 'Content-Type: application/json' -d'
{
  "user": "kimchy",
  "post_date": "2009-11-15T14:12:12",
  "message": "trying out Elasticsearch"
}'
```

### 搜索文档

```bash
curl -X GET "localhost:9200/my_index/_search?q=user:kimchy&pretty"
```

### 删除索引

```bash
curl -X DELETE "localhost:9200/my_index?pretty"
```

## 性能优化

### 硬件建议

- **CPU**: 至少 2 核，推荐 4 核或更多
- **内存**: 至少 4GB，推荐 8GB 或更多（一半分配给 JVM 堆）
- **存储**: SSD 存储，至少 10GB 可用空间
- **网络**: 高速网络（集群部署时）

### 配置优化

```yaml
# 线程池配置
thread_pool.index.queue_size: 1000
thread_pool.search.queue_size: 1000
thread_pool.bulk.queue_size: 500

# 缓冲区配置
indices.memory.index_buffer_size: 10%
indices.fielddata.cache.size: 40%
indices.query.bool.max_clause_count: 1024

# 文件描述符
# 在 /etc/security/limits.conf 中添加:
# elasticsearch soft nofile 65536
# elasticsearch hard nofile 65536
```

# 参考链接

[Elasticsearch 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
[Elasticsearch 权威指南](https://elasticsearch.cn/book/)