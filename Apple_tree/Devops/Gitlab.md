# GitLab CE 部署与配置指南

## 简介

> GitLab 是一个基于 Git 的完整 DevOps 平台，提供从代码托管、CI/CD 持续集成交付到安全扫描的单一应用程序。它支持自托管部署，能够完全掌控代码资产与构建流程，广泛应用于企业级软件开发生命周期管理。

## 安装部署 (Omnibus 方式)

### 系统要求

* **操作系统**：Linux（推荐 Ubuntu 20.04+ 或 CentOS 7/Rocky Linux 8+）
* **硬件配置**：
* CPU：4 Core+ (推荐)
* 内存：至少 4GB RAM（推荐 8GB+ 以避免 Swap 频繁交换）
* 存储：至少 50GB 可用空间（取决于代码仓库大小）


* **依赖组件**：OpenSSH, Postfix (用于邮件通知), Curl

### 安装步骤

#### 1. 安装基础依赖

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl

# CentOS/RHEL
sudo yum install -y curl policycoreutils-python openssh-server perl
sudo systemctl enable sshd && sudo systemctl start sshd

```

#### 2. 添加软件源

```bash
# 添加 GitLab 官方 CE (社区版) 镜像源
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash

```

*(注：CentOS 用户请将脚本替换为 `script.rpm.sh`)*

#### 3. 执行安装

```bash
# 将 EXTERNAL_URL 替换为你计划使用的域名或 IP
sudo EXTERNAL_URL="http://gitlab.example.com" apt-get install gitlab-ce

```

#### 4. 初始化配置

安装完成后，GitLab 会自动初始化。如果需要重新加载配置，执行：

```bash
sudo gitlab-ctl reconfigure

```

## HTTPS 配置

### 生成自签证书

如果内网环境没有购买商业证书，可以使用 OpenSSL 生成自签证书。

```bash
mkdir -p /etc/gitlab/ssl
cd /etc/gitlab/ssl

# 1. 生成私钥
openssl genrsa -out gitlab.key 4096

# 2. 生成证书签名请求 (CSR)
openssl req -new -key gitlab.key -out gitlab.csr \
  -subj "/C=CN/ST=Shanghai/L=Shanghai/O=DevOps/CN=gitlab.example.com"

# 3. 生成自签名证书 (有效期 10 年)
openssl x509 -req -days 3650 -in gitlab.csr -signkey gitlab.key -out gitlab.crt

```

### 配置 GitLab

编辑 GitLab 核心配置文件 `/etc/gitlab/gitlab.rb`：

```ruby
# 修改外部 URL 为 https
external_url 'https://gitlab.example.com'

# 开启 Nginx SSL 支持
nginx['enable'] = true
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.key"

# 优化 HTTPS 协议 (可选)
nginx['ssl_protocols'] = "TLSv1.2 TLSv1.3"

```

### 重启服务

使配置生效：

```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart

```

## 服务管理 (Systemd 配置)

GitLab Omnibus 安装包默认集成了 Runit 服务管理，并自动注册了 Systemd 单元。

### 1. 核心服务控制

使用 `gitlab-ctl` 工具管理组件（Nginx, Puma, Sidekiq 等）：

```bash
# 启动所有组件
sudo gitlab-ctl start

# 停止所有组件
sudo gitlab-ctl stop

# 查看组件状态
sudo gitlab-ctl status

```

### 2. Systemd 守护进程

GitLab 的主守护进程由 Systemd 接管，确保开机自启。

检查 `/usr/lib/systemd/system/gitlab-runsvdir.service`：

```ini
[Unit]
Description=GitLab Runit supervision process
After=basic.target network.target
Wants=systemd-journald-dev-log-socket.socket

[Service]
ExecStart=/opt/gitlab/embedded/bin/gitlab-runsvdir-start
Restart=always

[Install]
WantedBy=multi-user.target

```

**管理命令：**

```bash
# 允许开机自启
sudo systemctl enable gitlab-runsvdir

# 如果 gitlab-ctl 命令失效，尝试重启主服务
sudo systemctl restart gitlab-runsvdir

```

## Runner 客户端信任配置

GitLab Runner 是执行 CI/CD 作业的代理节点。当 GitLab 使用自签证书时，Runner 必须配置信任。

### 1. 安装 Runner

```bash
# 下载二进制文件 (Linux x64)
sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"

# 赋予执行权限
sudo chmod +x /usr/local/bin/gitlab-runner

# 创建 CI 用户并安装服务
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start

```

### 2. 配置证书信任

在 Runner 机器上操作：

```bash
# 获取 GitLab 服务器证书
scp root@gitlab.example.com:/etc/gitlab/ssl/gitlab.crt /tmp/

# 方式一：注册时指定 (推荐)
# 将证书放入特定目录，命名需与域名一致
sudo mkdir -p /etc/gitlab-runner/certs/
sudo cp /tmp/gitlab.crt /etc/gitlab-runner/certs/gitlab.example.com.crt

```

### 3. 注册 Runner

```bash
sudo gitlab-runner register \
  --url "https://gitlab.example.com/" \
  --registration-token "PROJECT_REGISTRATION_TOKEN" \
  --description "docker-runner" \
  --executor "docker" \
  --docker-image "docker:stable" \
  --tls-ca-file "/etc/gitlab-runner/certs/gitlab.example.com.crt"

```

## 代码仓库管理

### 1. 初始登录

* **地址**：`https://gitlab.example.com`
* **账号**：`root`
* **初始密码**：查看 `/etc/gitlab/initial_root_password` (仅在安装后 24 小时内有效)

### 2. 推送代码

在开发人员机器上配置 Git 全局设置：

```bash
git config --global user.name "Administrator"
git config --global user.email "admin@example.com"

# 解决自签证书 SSL 验证报错
git config --global http.sslVerify false

```

推送现有项目：

```bash
cd my-project
git init
git remote add origin https://gitlab.example.com/root/my-project.git
git add .
git commit -m "Initial commit"
git push -u origin master

```

## 常见问题

### 502 Bad Gateway

通常是因为 Unicorn/Puma 服务启动缓慢或端口冲突。

* **检查日志**：`sudo gitlab-ctl tail`
* **内存检查**：确保可用内存大于 4GB，否则进程会被 OOM Kill。

### Runner 无法拉取代码 (x509 error)

* **现象**：CI Job 失败，提示 `SSL certificate problem: self signed certificate`。
* **解决**：确保在 `/etc/gitlab-runner/certs/` 下放置了正确的 `.crt` 文件，且文件名必须匹配 GitLab 的域名（如 `gitlab.example.com.crt`）。

### 忘记 Root 密码

进入 Rails 控制台重置：

```bash
sudo gitlab-rails console -e production

# 在控制台中执行：
user = User.where(id: 1).first
user.password = 'NewPassword123!'
user.password_confirmation = 'NewPassword123!'
user.save!
exit

```

## 参考链接

* [GitLab 官方文档](https://docs.gitlab.com/)
* [Omnibus 安装指南](https://docs.gitlab.com/omnibus/)
* [GitLab Runner 配置](https://docs.gitlab.com/runner/)

---
