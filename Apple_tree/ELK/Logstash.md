# Logstash 部署与配置

>Logstash 是一个开源的数据收集引擎，具备实时管道功能，能够从多个来源采集数据，转换数据，然后将数据发送到指定的存储库。

# 体系结构

## 数据流模型

Logstash 的工作流程分为三个阶段：输入（Input）、过滤（Filter）和输出（Output），简称 IFO 模型。

- **输入 (Input)**: 定义数据源，可以从文件、数据库、消息队列等多种来源获取数据
- **过滤 (Filter)**: 对数据进行解析、修改和转换，支持条件判断、字段映射等功能
- **输出 (Output)**: 将处理后的数据发送到指定目标，如 Elasticsearch、文件、数据库等

> [!tip] 提示
Logstash 的核心是事件处理管道，输入插件从源系统获取数据，过滤插件处理数据（解析、标记、转换），输出插件将数据写入目标系统。理解这三个阶段对于编写有效的配置文件至关重要。一个典型的 Logstash 实例包含多个输入、过滤器和输出，它们按顺序处理事件。

## 部署环境准备

### 依赖准备

Logstash 环境依赖 Java 8 或更高版本（推荐 OpenJDK）。

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
# 设置 JAVA_HOME（通常会自动检测）
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64  # Ubuntu 示例
```

## 安装 Logstash

### 通过 apt/yum 安装（推荐）

```bash
# Ubuntu/Debian
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update && sudo apt install logstash

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
sudo yum install logstash
```

### 通过 tar.gz 包安装

```bash
# 下载 Logstash
cd /tmp
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.11.0-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.11.0-linux-x86_64.tar.gz.sha512

# 验证包完整性
sha512sum -c logstash-8.11.0-linux-x86_64.tar.gz.sha512
tar -zxf logstash-8.11.0-linux-x86_64.tar.gz

# 移动到合适的位置
sudo mv logstash-8.11.0 /opt/logstash
sudo chown -R 1000:1000 /opt/logstash
```

## 配置

### 主配置文件

Logstash 的配置文件使用特殊的 DSL（领域特定语言）编写，包含 input、filter 和 output 三个部分。

#### 基本配置示例

```ruby
input {
  # 从文件读取数据
  file {
    path => "/var/log/*.log"
    start_position => "beginning"
    type => "syslog"
  }

  # 从标准输入读取
  stdin {}
}

filter {
  # 如果是 syslog 类型，使用 grok 解析
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_tag => [ "syslog_parsed" ]
    }
    
    # 解析时间戳
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }

  # 移除不需要的字段
  mutate {
    remove_field => [ "syslog_timestamp" ]
  }
}

output {
  # 输出到 Elasticsearch
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }

  # 同时输出到标准输出（用于调试）
  stdout {
    codec => rubydebug
  }
}
```

### 环境配置

logstash.yml 是 Logstash 的环境配置文件，位于 config/ 目录下。

```yaml
# HTTP API 监听地址
http.host: "127.0.0.1"
http.port: 9600-9700

# JVM 选项
path.settings: /etc/logstash

# 管道配置路径
pipeline.config.string: |
  input { stdin { } }
  output { stdout { codec => dots } }

# 管道工作线程数（通常设为 CPU 核心数）
pipeline.workers: 4

# 批处理大小
pipeline.batch.size: 125
pipeline.batch.delay: 50

# 死信队列
dead_letter_queue.enable: true
path.dead_letter_queue: /var/lib/logstash/dead_letter_queue

# 持久队列
queue.type: persisted
path.queue: /var/lib/logstash/queue
queue.max_bytes: 1024mb
```

### JVM 配置

jvm.options 文件位于 config/ 目录下，用于配置 Logstash 使用的 JVM 参数。

```bash
# 设置堆内存大小
-Xms1g
-Xmx1g

# GC 配置
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
-XX:+AlwaysPreTouch
-XX:+UseParNewGC
-XX:+CMSParallelRemarkEnabled
-XX:CMSExceptionTrackAfterFullGC=on
```

## 启动

### systemd 方式（推荐）

```bash
# 启动 Logstash
sudo systemctl start logstash

# 设置开机自启
sudo systemctl enable logstash

# 查看状态
sudo systemctl status logstash

# 查看日志
sudo journalctl -u logstash.service -f
```

### 直接运行方式（仅限 tar 包安装）

```bash
# 测试配置文件
sudo -u logstash /opt/logstash/bin/logstash -f /path/to/config.conf --config.test_and_exit

# 运行 Logstash（指定配置文件）
sudo -u logstash /opt/logstash/bin/logstash -f /path/to/config.conf

# 运行 Logstash（使用管道配置目录）
sudo -u logstash /opt/logstash/bin/logstash -f /etc/logstash/conf.d/
```

## 常用插件

### 输入插件 (Input Plugins)

#### File 输入插件

```ruby
input {
  file {
    path => ["/var/log/messages", "/var/log/*.log"]
    type => "system-logs"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/sincedb"
  }
}
```

#### Beats 输入插件

```ruby
input {
  beats {
    port => 5044
  }
}
```

#### JDBC 输入插件

```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector-java.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydatabase"
    jdbc_user => "username"
    jdbc_password => "password"
    statement => "SELECT * FROM mytable WHERE modified_date > :sql_last_value ORDER BY modified_date"
    use_column_value => true
    tracking_column => "modified_date"
    schedule => "* * * * *"
  }
}
```

### 过滤插件 (Filter Plugins)

#### Grok 过滤插件

```ruby
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
}
```

#### Mutate 过滤插件

```ruby
filter {
  mutate {
    # 字段重命名
    rename => [ "oldfield", "newfield" ]
    
    # 字段类型转换
    convert => [ "field1", "integer" ]
    
    # 添加字段
    add_field => { "new_field" => "value" }
    
    # 添加标签
    add_tag => [ "production", "nginx" ]
    
    # 删除字段
    remove_field => [ "unwanted_field" ]
  }
}
```

#### Date 过滤插件

```ruby
filter {
  date {
    match => [ "timestamp", "ISO8601", "UNIX", "UNIX_MS" ]
    target => "@timestamp"
  }
}
```

### 输出插件 (Output Plugins)

#### Elasticsearch 输出插件

```ruby
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "myapp-%{+YYYY.MM.dd}"
    template_name => "myapp"
    template_pattern => "myapp-*"
  }
}
```

#### File 输出插件

```ruby
output {
  file {
    path => "/tmp/output_%{+YYYYMMdd}.txt"
    codec => line { format => "custom format: %{message}" }
  }
}
```

## 性能优化

### 硬件建议

- **CPU**: 至少 4 核，Logstash 是 CPU 密集型应用
- **内存**: 至少 4GB，建议 8GB 或更多（取决于处理的数据量）
- **存储**: SSD 存储，特别是对于持久队列
- **网络**: 高速网络（数据传输密集）

### 配置优化

```yaml
# 优化管道处理
pipeline.workers: 4                    # 设置为 CPU 核心数
pipeline.batch.size: 125               # 根据处理能力调整
pipeline.batch.delay: 50               # 默认值通常合适

# 启用持久队列
queue.type: persisted
queue.max_bytes: 4gb                  # 根据可用磁盘空间调整

# 优化 JVM
-Xms2g                                 # 设置合适的堆大小
-Xmx2g                                 # 通常设置为相同值
```

## 常见配置示例

### 收集 Nginx 日志并发送到 Elasticsearch

```ruby
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    type => "nginx-access"
  }
}

filter {
  if [type] == "nginx-access" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
  }
}
```

### 从 Kafka 读取数据并写入 Elasticsearch

```ruby
input {
  kafka {
    bootstrap_servers => "localhost:9092"
    topics => ["my-topic"]
    consumer_threads => 4
    decorate_events => true
  }
}

filter {
  json {
    source => "message"
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "kafka-data-%{+YYYY.MM.dd}"
  }
}
```

# 参考链接

[Logstash 官方文档](https://www.elastic.co/guide/en/logstash/current/index.html)
[Logstash 插件列表](https://www.elastic.co/guide/en/logstash/current/input-plugins.html)