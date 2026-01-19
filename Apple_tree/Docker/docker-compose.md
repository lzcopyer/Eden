## 部署

[docker-compose项目地址](https://github.com/docker/compose/releases)  
`cp docker-compose /usr/bin && chmod o+x /usr/bin/docker-compose`

## docker-compose基础指令

[官方手册](https://docker.github.net.cn/manuals)  

``` bash
#启动服务
docker-compose -f docker-compose.yaml up -d
#停止服务
docker-compose -f docker-compose.yaml stop
#停止并移除所有服务、网络和容器
docker-compose -f docker-compose.yaml down
#移除所有服务，包括命名卷
docker-compose down --volumes
#查看所有服务的状态
docker-compose ps
#查看服务日志
docker-compose logs <服务名>
#在一个正在运行的服务中执行命令
docker-compose exec <服务名> <命令>
```

---

## Docker Compose 编排指南

`docker-compose` 是用于定义和运行多容器 Docker 应用程序的工具。它通过一个 YAML 文件来配置应用程序的所有服务，然后使用一个命令来创建和启动所有服务。对于运维而言，它极大地简化了复杂应用（如 Web 服务、数据库、缓存等）的部署和管理。

---

### 一、`docker-compose.yaml` 文件结构概览

一个典型的 `docker-compose.yaml` 文件包含三个顶级键：

- **`version`**: 指定 Compose 文件的版本。
    
- **`services`**: 定义应用中包含的所有服务，这是文件的核心部分。
    
- **`networks`** (可选): 定义自定义网络，用于服务间的通信。
    
- **`volumes`** (可选): 定义命名卷，用于持久化数据。
    

``` YAML
# 推荐使用版本3，这是为Swarm模式设计的
version: '3.8'

services:
  # 服务1
  web:
    # 服务配置

  # 服务2
  database:
    # 服务配置

volumes:
  # 命名卷定义

networks:
  # 网络定义
```

---

### 二、核心组件详解

#### 1. `version`

- **作用**: 定义 Compose 文件的版本。
    
- **实践**: 建议始终使用最新的 **`3.x` 版本**，因为它支持 Docker Swarm 模式，并且功能最完整。
    

#### 2. `services`

这是配置文件的核心，每个键都是一个服务。以下是服务配置中常用且关键的选项：

- **`image`**:
    
    - **作用**: 指定服务基于哪个 Docker 镜像。
        
    - **示例**: `image: nginx:1.21.6`。
        
- **`build`**:
    
    - **作用**: 从本地 `Dockerfile` 构建镜像。当镜像不存在时，Compose 会自动构建。
        
    - **示例**:
        
        
        
``` YAML
build:
    context: .  # 指定Dockerfile的路径
    dockerfile: Dockerfile  # 指定Dockerfile的文件名
```
        
- **`ports`**:
    
    - **作用**: 将容器内的端口映射到宿主机。
        
    - **示例**: `ports: ["8080:80"]`。这表示将宿主机的 `8080` 端口映射到容器的 `80` 端口。
        
- **`volumes`**:
    
    - **作用**: 持久化数据，实现容器和宿主机之间的数据共享。
        
    - **类型**:
        
        - **绑定挂载 (Bind Mount)**: `volumes: ["./data:/app/data"]`，将当前目录下的 `data` 目录挂载到容器的 `/app/data`。适用于配置文件和源代码。
            
        - **命名卷 (Named Volume)**: `volumes: ["my-volume:/app/data"]`，引用顶级键 `volumes` 中定义的命名卷。适用于数据库等需要持久化存储的数据。
            
- **`environment`**:
    
    - **作用**: 为容器设置环境变量。
        
    - **示例**: `environment: - DB_HOST=database`。
        
- **`depends_on`**:
    
    - **作用**: 定义服务启动的依赖关系。**它只保证依赖服务在启动**，并不保证依赖服务已经“就绪”。
        
    - **示例**: `depends_on: - database`。确保 `database` 服务先于当前服务启动。
        
- **`networks`**:
    
    - **作用**: 将服务连接到指定的网络。
        
    - **类型**:
            
        - **`Default Bridge Network`**: 不明确定义网络时，Docker Compose 会自动创建一个默认网络。
            
		- **`User-defined Bridge Network`**: 自定义桥接网络，可以实现网络隔离。
            
			``` YAML
			version: "3.8"
			services:
			  web:
			    image: nginx:latest
			    ports:
			      - "80:80"
			    networks:
			      frontend:
			        ipv4_address: 172.16.238.10
			
			networks:
			  frontend:
			    driver: bridge
			    ipam:
			      driver: default
			      config:
			        - subnet: "172.16.238.0/24"
			          gateway: "172.16.238.1"
			```
            
		- **`Host Network`**: 主机网络模式让容器直接使用宿主机的网络堆栈。
            
			``` YAML
			version: '3.8'
			services:
			  web:
			    image: my-web-app
			    # 容器直接使用宿主机的网络
			    network_mode: host
			```
            
		- **`External Network`**: 外部网络允许连接到在 `docker-compose` 项目之外创建的网络。
             
			``` YAML
			version: '3.8'
			services:
			  web:
			    image: my-web-app
			    networks:
			      - my-shared-network
			
			networks:
			  my-shared-network:
			    external: true
			```
            
			- **`Overlay Network`**: 叠加网络是为 **Docker Swarm** 集群设计的，用于连接运行在不同宿主机上的容器。
            
			``` YAML
			version: '3.8'
			services:
			  web:
			    image: my-web-app
			    deploy:
			      replicas: 3
			    networks:
			      - my-overlay
			
			networks:
			  my-overlay:
			    driver: overlay
			```
            
- **`restart`**:
    
    - **作用**: 定义服务的重启策略，这对生产环境非常重要。
        
    - **常用值**:
        
        - **`no`**: 默认值，不自动重启。
            
        - **`always`**: 无论退出状态如何，总是重启。
            
        - **`on-failure`**: 只有当退出代码非零时才重启。
            
        - **`unless-stopped`**: 除非手动停止，否则总是重启。
            

---

### 三、实战示例：一个 Web 应用与数据库

这是一个典型的 Web 应用配置，其中包含一个 Web 服务、一个数据库和一个自定义网络。

``` YAML
version: '3.8'

# 定义命名卷，用于持久化数据库数据
volumes:
  db_data: "/data/postgres/data"

# 定义自定义网络，用于隔离服务
networks:
  app_net: bridge

services:
  web:
    # 从本地的Dockerfile构建镜像
    build: .
    # 将宿主机的8080端口映射到容器的80端口
    ports:
      - "8080:80"
    # 将服务连接到自定义网络
    networks:
      - app_net
    # 定义重启策略
    restart: unless-stopped
    # 依赖数据库服务
    depends_on:
      - database
    # 设置环境变量
    environment:
      - DATABASE_URL=postgres://user:password@database/mydb
    # 开启server
    command: /workspace/nginx -c /workspace/nginx/conf/nginx.conf

  database:
    # 使用官方Postgres镜像
    image: postgres:13
    # 挂载命名卷
    volumes:
      - db_data:/var/lib/postgresql/data
    # 将服务连接到自定义网络
    networks:
      - app_net
    # 设置环境变量
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    # 定义重启策略
    restart: unless-stopped
```
