# Jenkins 高可用部署与流水线实践指南

## 简介

>Jenkins 是一款开源持续集成与持续交付（CI/CD）工具，支持自动化构建、测试和部署。通过分布式架构和插件扩展，可满足企业级复杂流水线需求。

## 高可用部署

### 架构设计
支持三种主流部署方案：
1. **主从架构**：1个主节点+多个构建节点
2. **Kubernetes 部署**：基于Helm Chart实现容器化部署
3. **云原生方案**：结合Jenkins Operator实现自动化运维

### 主从架构部署步骤
```bash
# 安装主节点
wget -qO - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update && sudo apt-get install jenkins

# 配置从节点（通过SSH或JNLP）
ssh jenkins@slave-node java -jar agent.jar -jnlpUrl http://master/jenkins/computer/node1/jenkins-agent.jnlp -secret your_secret
```

### Kubernetes 部署
```bash
# 添加Helm仓库
helm repo add jenkins https://charts.jenkins.io
helm repo update

# 安装Jenkins
helm install jenkins jenkins/jenkins \
  --set controller.replicas=3 \
  --set agent.replicas=5 \
  --set persistence.enabled=true
```

## HTTPS 证书配置

### 证书生成
```bash
# 使用Let's Encrypt
sudo apt install certbot
sudo certbot certonly --standalone -d jenkins.example.com

# 配置Nginx反向代理
sudo vim /etc/nginx/sites-available/jenkins
```
Nginx配置示例：
```nginx
server {
    listen 443 ssl;
    server_name jenkins.example.com;

    ssl_certificate /etc/letsencrypt/live/jenkins.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
    }
}
```

## Pipeline 编写规范

### 基本结构
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'make' }
        }
        stage('Test') {
            steps { sh 'make test' }
            when { branch 'develop' }
        }
    }
}
```

### 最佳实践
- 使用`agent none`在stage级别指定构建节点
- 添加`options { disableConcurrentBuilds() }`防止并发构建
- 通过`environment`块管理敏感参数：
```groovy
environment {
    AWS_ACCESS_KEY_ID = credentials('aws-key-id')
}
```

## Pipeline 用例

### Git触发自动构建
```groovy
pipeline {
    agent any
    triggers {
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true)
    }
    stages {
        stage('Checkout') {
            steps { git 'https://gitlab.com/your/repo.git' }
        }
        stage('Build') {
            steps { sh './mvnw package' }
        }
        stage('Deploy') {
            steps {
                sshagent(['server-ssh-credentials']) {
                    sh 'scp target/app.jar user@server:/opt/app'
                    sh 'ssh user@server "systemctl restart app"'
                }
            }
        }
    }
}
```

### 功能测试流水线
```groovy
pipeline {
    agent dockerfile { filename 'Dockerfile.test' }
    stages {
        stage('Unit Test') {
            steps {
                sh 'npm install'
                sh 'npm test'
                junit 'test-results/*.xml'
            }
        }
        stage('Integration Test') {
            steps {
                docker.image('test-container').run('-p 8080:8080')
                sh 'npm run integration-tests'
            }
        }
    }
}
```

## 常见问题

### 节点连接失败
检查从节点状态：
```bash
# 查看节点日志
journalctl -u jenkins-agent -n node1

# 验证网络连通性
telnet master-node 50000
```

### 证书过期处理
自动续期配置：
```bash
# 添加定时任务
echo "0 0 * * * root certbot renew --quiet && systemctl reload nginx" | sudo tee -a /etc/crontab
```

## 参考链接
- [Jenkins 官方文档](https://www.jenkins.io/doc/)
- [流水线最佳实践](https://www.jenkins.io/blog/tags/pipeline/)
- [Kubernetes 部署指南](https://www.jenkins.io/projects/jenkins-on-kubernetes/)