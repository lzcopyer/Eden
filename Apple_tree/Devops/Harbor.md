# Harbor 源码编译安装与高级配置指南

## 简介

> Harbor 是一个企业级私有容器镜像仓库，提供安全、合规、高效的镜像管理服务。它不仅支持多租户管理、镜像复制、漏洞扫描等高级功能，还完美适配 Kubernetes 和 Docker/Containerd 生态系统。

## 一、源码编译与安装

### 1. 系统要求

* **操作系统**：Linux（推荐 Ubuntu 20.04+ 或 CentOS 7.9+）
* **硬件配置**：
* CPU：2 Core+
* 内存：至少 8GB RAM
* 存储：至少 50GB 可用空间（生产环境建议挂载独立数据盘）


* **依赖组件**：
* Docker Engine 20.10+
* Docker Compose v2.0+ (或独立的 `docker-compose`)
* Go 1.18+ (根据 Harbor 版本要求，v2.7+ 建议较高版本)
* Python 3.6+
* Make



### 2. 编译步骤

#### 2.1 获取源码

```bash
git clone https://github.com/goharbor/harbor.git
cd harbor
git checkout v2.7.0  # 切换至稳定分支

```

#### 2.2 安装编译依赖

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y make python3-pip

# CentOS/RHEL
sudo yum install -y python3-pip

# 安装 Python 依赖（用于生成配置文件）
pip3 install --no-cache-dir -r requirements.txt

```

#### 2.3 执行编译

Harbor 的编译过程会构建各个组件的 Docker 镜像并生成安装包。

```bash
# 1. 预处理
make prepare

# 2. 编译并生成离线安装包（包含镜像）
# 如果需要包含 Notary 或 Trivy，可追加 COMPILETAG=notary,trivy
make install 

```

*注：此处的 `make install` 在源码目录中旨在编译代码并生成发布包，通常位于 `make` 结束后的发布目录中，或者直接生成 docker 镜像到本地。*

#### 2.4 初始化配置

在部署目录（通常是源码根目录或解压后的目录）进行配置：

```bash
cp harbor.yml.tmpl harbor.yml

```

编辑 `harbor.yml`，这是 Harbor 的核心配置文件。

## 二、HTTPS 与 Harbor 服务配置

### 1. 生成自签证书

为了确保通信安全，强烈建议启用 HTTPS。

```bash
mkdir -p /data/cert
cd /data/cert

# 生成私钥
openssl genrsa -out harbor.key 4096

# 生成证书签名请求 (CSR)
# 注意：CN (Common Name) 必须填写你的域名或 IP
openssl req -new -key harbor.key -out harbor.csr \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=harbor.example.com"

# 生成自签名证书 (有效期 10 年)
openssl x509 -req -days 3650 -in harbor.csr -signkey harbor.key -out harbor.crt

```

### 2. 配置 harbor.yml

编辑 `/path/to/harbor/harbor.yml`：

```yaml
hostname: harbor.example.com

# http:
#   port: 80

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key

# Harbor Services 基础配置
# 配置数据存储路径
data_volume: /data/harbor

# 外部数据库配置 (可选，生产环境推荐使用外部数据库)
# external_database:
#   harbor:
#     host: db.example.com
#     port: 5432
#     db_name: registry
#     username: harbor
#     password: "password"

```

### 3. 安装与启动

运行安装脚本以生成 `docker-compose.yml` 并启动服务：

```bash
# 安装并启动（包含 Notary 和 Trivy 扫描器）
sudo ./install.sh --with-notary --with-trivy

```

### 4. 配置 Systemd 管理 (Harbor.services)

为了方便使用 `systemctl` 管理 Harbor，我们需要创建一个服务单元文件。

创建文件 `/etc/systemd/system/harbor.service`：

```ini
[Unit]
Description=Harbor Container Registry
Requires=docker.service
After=docker.service

[Service]
Restart=always
User=root
Group=root
# 指向你的 harbor 安装目录
WorkingDirectory=/path/to/harbor
# 使用 docker compose 启动和停止
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target

```

**启动服务：**

```bash
sudo systemctl enable harbor
sudo systemctl start harbor

```

## 三、Docker 客户端配置

### 1. 分发证书

将 CA 证书（自签证书即为自身的 CA）复制到 Docker 信任目录：

```bash
# 创建对应的域名目录
sudo mkdir -p /etc/docker/certs.d/harbor.example.com/

# 复制证书（注意：Docker 客户端识别的是 .crt 扩展名）
sudo cp /data/cert/harbor.crt /etc/docker/certs.d/harbor.example.com/ca.crt

# 如果是系统级信任（可选）
sudo cp /data/cert/harbor.crt /usr/local/share/ca-certificates/harbor.crt
sudo update-ca-certificates

```

### 2. 验证登录

不需要重启 Docker 守护进程（如果使用了 certs.d 方式），直接登录：

```bash
docker login harbor.example.com -u admin -p Harbor12345

```

## 四、Containerd 客户端配置 (K8s 常用)

Kubernetes 现在的默认运行时通常是 Containerd。Containerd 对自签证书的配置方式与 Docker 不同。

### 1. 方法一：使用 hosts.toml (推荐，Containerd 1.5+)

这是目前推荐的配置方式，支持热加载，无需重启 Containerd。

**创建配置目录：**

```bash
# 目录结构：/etc/containerd/certs.d/<域名或IP>
sudo mkdir -p /etc/containerd/certs.d/harbor.example.com

```

**创建 hosts.toml：**
编辑 `/etc/containerd/certs.d/harbor.example.com/hosts.toml`：

```toml
server = "https://harbor.example.com"

[host."https://harbor.example.com"]
  capabilities = ["pull", "resolve", "push"]
  # 跳过证书验证（不推荐，仅测试用）
  # skip_verify = true
  # 指定 CA 证书路径（推荐）
  ca = "/etc/containerd/certs.d/harbor.example.com/ca.crt"

```

**复制证书：**

```bash
sudo cp /data/cert/harbor.crt /etc/containerd/certs.d/harbor.example.com/ca.crt

```

**修改 config.toml 指向 certs.d (如果尚未配置)：**
检查 `/etc/containerd/config.toml`，确保包含以下配置：

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"

```

### 2. 方法二：修改 config.toml (旧版方式)

如果不支持 `hosts.toml`，则直接修改主配置文件 `/etc/containerd/config.toml`：

```toml
[plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.example.com".tls]
  ca_file = "/etc/containerd/certs.d/harbor.example.com/ca.crt"
  # 或者使用不安全模式
  # insecure_skip_verify = true

```

### 3. 验证 Containerd 拉取

使用 `ctr` 或 `crictl` 进行测试：

```bash
# 使用 ctr
ctr images pull --user admin:Harbor12345 harbor.example.com/library/ubuntu:latest

# 使用 crictl (k8s 节点常用)
crictl pull --auth "admin:Harbor12345" harbor.example.com/library/ubuntu:latest

```

## 五、Kubernetes 集成

### 创建 ImagePullSecret

为了让 K8s 能够从私有仓库拉取镜像，需要创建一个 Secret。

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-email=admin@example.com \
  -n default

```

### 在 Pod 中使用

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: harbor-test
spec:
  containers:
  - name: ubuntu
    image: harbor.example.com/library/ubuntu:latest
  imagePullSecrets:
  - name: harbor-secret

```

## 六、常见问题排查

### 1. x509: certificate signed by unknown authority

* **原因**：客户端未正确配置 CA 证书。
* **解决**：
* **Docker**：检查 `/etc/docker/certs.d/harbor.example.com/ca.crt` 内容是否与服务器端 `harbor.crt` 一致。
* **Containerd**：检查 `hosts.toml` 中的 `ca` 路径是否正确。



### 2. Harbor 服务启动失败

* **检查日志**：
```bash
cd /path/to/harbor
docker compose logs -f

```


* **数据库连接**：如果使用了 `external_database`，确保 PostgreSQL 的 `pg_hba.conf` 允许 Harbor 机器连接。

### 3. 镜像推送 401 Unauthorized

* **原因**：未登录或登录过期。
* **解决**：重新执行 `docker login`。如果是 Robot 账号，确保使用了正确的 Token 而不是密码。

## 参考链接

* [Harbor 官方文档](https://goharbor.io/docs/)
* [Harbor 安装与配置指南](https://goharbor.io/docs/2.7.0/install-config/)
* [Containerd Registry Configuration](https://github.com/containerd/containerd/blob/main/docs/hosts.md)

---

