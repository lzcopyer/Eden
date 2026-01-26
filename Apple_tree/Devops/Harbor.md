# Harbor 源码编译安装与使用指南

## 简介

>Harbor 是一个企业级私有容器镜像仓库，提供安全、合规、高效的镜像管理服务。支持多租户管理、镜像复制、漏洞扫描等高级功能，适用于 Kubernetes 和 Docker 生态系统。

## 源码编译安装

### 系统要求
- 操作系统：Linux（推荐 Ubuntu 20.04+ 或 CentOS 7+）
- 内存：至少 8GB RAM
- 存储：至少 50GB 可用空间
- 依赖组件：Docker、Docker Compose、Go 1.16+、Python 3.6+

### 编译步骤

#### 1. 获取源码
```bash
git clone https://github.com/goharbor/harbor.git
cd harbor
git checkout v2.7.0  # 选择稳定版本
```

#### 2. 安装依赖
```bash
# Ubuntu/Debian
sudo apt-get install -y make python3-pip

# CentOS
sudo yum install -y python3-pip

pip3 install --no-cache-dir -r requirements.txt
```

#### 3. 配置编译参数
编辑 `make/photon/Makefile` 文件：
```makefile
# 修改镜像仓库地址（可选）
HARBOR_REPO = my-registry.com/harbor
```

#### 4. 执行编译
```bash
make prepare
make build
```

#### 5. 部署服务
```bash
sudo make install
sudo systemctl start harbor
```

## HTTPS 配置

### 生成证书
```bash
mkdir -p /etc/cert/harbor
cd /etc/cert/harbor

openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout harbor.key \
  -out harbor.crt \
  -subj "/C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=harbor.example.com"
```

### 配置 Harbor
编辑 `/etc/harbor/harbor.yml`：
```yaml
https:
  port: 443
  certificate: /etc/cert/harbor/harbor.crt
  private_key: /etc/cert/harbor/harbor.key
```

### 重启服务
```bash
sudo harbor-ctl stop
sudo harbor-ctl start
```

## 客户端信任配置

### 1. 分发证书
将 `harbor.crt` 复制到客户端：
```bash
sudo cp harbor.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### 2. Docker 客户端配置
编辑 Docker 守护配置：
```json
{
  "insecure-registries": ["harbor.example.com"]
}
```
重启 Docker 服务：
```bash
sudo systemctl restart docker
```

## 容器镜像管理

### 1. 登录仓库
```bash
docker login harbor.example.com -u admin -p 密码
```

### 2. 推送镜像
```bash
docker tag ubuntu harbor.example.com/library/ubuntu:latest
docker push harbor.example.com/library/ubuntu:latest
```

### 3. Kubernetes 集成
创建 Secret：
```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.example.com \
  --docker-username=admin \
  --docker-password=密码
```

## 常见问题

### 证书信任失败
检查证书路径：
```bash
openssl x509 -in /etc/docker/certs.d/harbor.example.com/ca.crt -text -noout
```

### 镜像推送拒绝
确认项目权限：
```bash
# 检查项目策略
curl -u admin:密码 https://harbor.example.com/api/v2/projects
```

## 参考链接
- [Harbor 官方文档](https://goharbor.io/docs/)
- [源码编译指南](https://github.com/goharbor/harbor/blob/main/docs/compile_guide.md)