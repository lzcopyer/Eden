# Kibana 部署与配置

>Kibana 是一款开源的数据可视化和分析工具，可让您探索、可视化和分享实时数据见解，建立和共享仪表盘。

# 体系结构

## 核心功能

Kibana 提供了以下核心功能模块：

- **Discover**: 探索 Elasticsearch 中存储的数据
- **Visualize Library**: 创建各种图表和可视化组件
- **Dashboard**: 构建交互式仪表盘
- **Dev Tools**: 使用 Kibana 控制台与 Elasticsearch 交互
- **Stack Management**: 管理索引、角色、用户等

> [!tip] 提示
Kibana 是 ELK Stack 中的数据可视化组件，它连接到 Elasticsearch 并提供直观的界面来探索和分析数据。Kibana 不直接存储数据，而是从 Elasticsearch 中读取数据并以图表、地图、表格等形式展示。理解 Kibana 与 Elasticsearch 的关系很重要：Kibana 是前端可视化工具，Elasticsearch 是后端数据存储和搜索引擎。

## 部署环境准备

### 依赖准备

Kibana 环境依赖 Node.js 和 Java，但在大多数预构建包中这些依赖已被打包。

#### 安装 Java（Kibana 本身不需要，但通常与 Elasticsearch 一起部署）

```bash
# Ubuntu/Debian 安装 OpenJDK 11
sudo apt update
sudo apt install openjdk-11-jdk

# CentOS/RHEL 安装 OpenJDK 11
sudo yum install java-11-openjdk-devel

# 验证安装
java -version
```

## 安装 Kibana

### 通过 apt/yum 安装（推荐）

```bash
# Ubuntu/Debian
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update && sudo apt install kibana

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
sudo yum install kibana
```

### 通过 tar.gz 包安装

```bash
# 下载 Kibana
cd /tmp
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.11.0-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.11.0-linux-x86_64.tar.gz.sha512

# 验证包完整性
sha512sum -c kibana-8.11.0-linux-x86_64.tar.gz.sha512
tar -zxf kibana-8.11.0-linux-x86_64.tar.gz

# 移动到合适的位置
sudo mv kibana-8.11.0-linux-x86_64 /opt/kibana
sudo chown -R 1000:1000 /opt/kibana
```

## 配置

### 主配置文件

kibana.yml 是 Kibana 的主配置文件，位于 config/ 目录下。

```yaml
# 服务器配置
server.port: 5601
server.host: "0.0.0.0"
server.name: "kibana-server"

# Elasticsearch 配置
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "your_password"

# Kibana 配置
elasticsearch.requestHeadersWhitelist: [ "authorization" ]

# 区域和时间设置
i18n.locale: "zh-CN"
server.defaultRoute: "/app/home"

# 日志配置
logging.dest: /var/log/kibana/kibana.log
logging.level: info
```

### 服务配置

在生产环境中，可能还需要配置 SSL 和身份验证。

```yaml
# SSL 配置
server.ssl.enabled: true
server.ssl.certificate: /path/to/kibana.crt
server.ssl.key: /path/to/kibana.key

# Elasticsearch SSL 配置
elasticsearch.ssl.certificateAuthorities: [ "/path/to/ca.crt" ]
elasticsearch.ssl.verificationMode: certificate
```

## 启动

### systemd 方式（推荐）

```bash
# 启动 Kibana
sudo systemctl start kibana

# 设置开机自启
sudo systemctl enable kibana

# 查看状态
sudo systemctl status kibana

# 查看日志
sudo journalctl -u kibana.service -f
```

### 直接运行方式（仅限 tar 包安装）

```bash
# 启动 Kibana
sudo -u kibana /opt/kibana/bin/kibana

# 后台运行
sudo -u kibana nohup /opt/kibana/bin/kibana > /var/log/kibana/kibana.log 2>&1 &
```

## 基本操作

### 访问 Kibana

打开浏览器访问 `http://your_server_ip:5601`

### 配置 Elasticsearch 连接

首次访问时，Kibana 会提示配置 Elasticsearch 连接。确保 Elasticsearch 已启动并可访问。

### 添加数据视图 (Index Pattern)

1. 进入 "Stack Management" > "Index Patterns"
2. 点击 "Create index pattern"
3. 输入索引模式（如 `logstash-*`）
4. 选择时间字段（如果有）
5. 点击 "Create index pattern"

### 创建可视化图表

1. 进入 "Analytics" > "Visualize Library"
2. 点击 "Create visualization"
3. 选择图表类型（柱状图、折线图、饼图等）
4. 选择数据视图
5. 配置图表的 X 轴、Y 轴等参数
6. 保存图表

### 创建仪表盘

1. 进入 "Analytics" > "Dashboard"
2. 点击 "Create new dashboard"
3. 点击 "Add from library" 选择之前创建的可视化图表
4. 调整图表布局和大小
5. 保存仪表盘

## 主要功能详解

### Discover 功能

Discover 允许您浏览 Elasticsearch 中的数据，支持：
- 搜索特定字段或全文搜索
- 应用过滤器缩小结果范围
- 查看原始 JSON 文档
- 添加/删除显示的字段

### Visualize 功能

Kibana 提供多种可视化类型：
- **柱状图 (Bar chart)**: 显示分类数据的对比
- **折线图 (Line chart)**: 显示数据随时间的变化趋势
- **饼图 (Pie chart)**: 显示各部分占整体的比例
- **地图 (Coordinate map)**: 在地图上显示地理位置数据
- **区域图 (Area chart)**: 显示数据量和变化趋势
- **表格 (Data table)**: 以表格形式显示详细数据

### Dashboard 功能

仪表盘功能允许您：
- 将多个可视化图表整合到一个页面
- 应用全局过滤器
- 添加文本说明
- 调整布局和尺寸
- 分享和导出仪表盘

## 高级功能

### Timelion 时间序列分析

Timelion 是 Kibana 内置的时间序列数据可视化工具，支持复杂的表达式语法。

```javascript
// 示例：过去 24 小时的平均响应时间
.es(index='web-logs*', metric='avg:response_time').label('Average Response Time')

// 示例：同比分析
.es(index='web-logs*', metric='sum:requests').label('Current') 
.divide(.es(index='web-logs*', metric='sum:requests', offset='1w')).label('Ratio')
```

### Canvas 画布

Canvas 允许您创建动态报告和演示文稿，将数据可视化与文本、图像结合。

### Machine Learning 机器学习

Kibana 集成了机器学习功能，可以：
- 自动检测数据中的异常
- 创建预测模型
- 监控数据模式变化

## 安全配置

### 启用安全功能

```yaml
# 在 elasticsearch.yml 中启用安全功能
xpack.security.enabled: true

# 在 kibana.yml 中配置用户凭据
elasticsearch.username: "kibana_system"
elasticsearch.password: "generated_password"
```

### 创建 Kibana 用户

```bash
# 在 Elasticsearch 中创建 Kibana 用户
curl -X POST "localhost:9200/_security/user/kibana_system" -H 'Content-Type: application/json' -d'
{
  "password" : "your_strong_password",
  "roles" : [ "kibana_system" ],
  "full_name" : "Kibana System User"
}'
```

## 性能优化

### 硬件建议

- **CPU**: 至少 2 核，推荐 4 核
- **内存**: 至少 4GB，推荐 8GB（取决于并发用户数）
- **存储**: 至少 1GB 可用空间
- **网络**: 高速网络（与 Elasticsearch 通信）

### 配置优化

```yaml
# 优化 Kibana 性能
server.maxPayloadBytes: 1048576      # 最大请求负载
server.shutdownTimeout: 5000          # 关闭超时时间
elasticsearch.requestTimeout: 30000   # 请求超时时间
elasticsearch.pingTimeout: 30000      # Ping 超时时间
elasticsearch.logQueries: false       # 不记录 ES 查询到日志
ops.interval: 5000                    # 操作指标间隔
```

## 常见使用场景

### 日志分析仪表盘

创建一个综合的日志分析仪表盘，包含：
- 错误率随时间变化的折线图
- 访问量统计的柱状图
- 地理位置分布的地图
- 响应时间分布的直方图

### 系统监控仪表盘

监控系统性能指标，如：
- CPU 使用率
- 内存使用情况
- 磁盘 I/O
- 网络流量

### 业务分析仪表盘

分析业务数据，如：
- 用户活跃度
- 转化率
- 收入趋势
- 产品销售情况

## 故障排除

### 常见问题及解决方法

#### 连接 Elasticsearch 失败
- 检查 Elasticsearch 是否正在运行
- 检查网络连接是否正常
- 验证用户名和密码是否正确
- 检查防火墙设置

#### 页面加载缓慢
- 检查 Elasticsearch 集群状态
- 优化查询语句，避免全表扫描
- 增加 Kibana 服务器资源

#### 图表无法显示数据
- 检查索引是否存在
- 验证索引模式配置是否正确
- 确认时间字段设置无误

# 参考链接

[Kibana 官方文档](https://www.elastic.co/guide/en/kibana/current/index.html)
[Kibana 用户手册](https://www.elastic.co/guide/en/kibana/current/getting-started.html)