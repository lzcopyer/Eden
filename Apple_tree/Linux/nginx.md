[docs.nginx](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

# 安装

## 包管理器安装

参考[官方安装教程](https://nginx.org/en/linux_packages.html)

## 编译安装

前往[nginx](https://nginx.org/)下载源码包

### **安装依赖**

``` Bash
# 安装 gcc 环境
yum install gcc-c++
# 安装 PCRE 库
yum install -y pcre pcre-devel
# 安装 zlib 压缩和解压缩依赖
yum install -y zlib zlib-devel
# 安装 SSL 安全的加密的套接字协议层，用于 HTTP 安全传输
yum install -y openssl openssl-devel
```

### **编译**

``` Bash
# 解压
tar -zxvf nginx.tar.gz -C /tmp
# 创建临时目录
mkdir /var/temp/nginx -p
# 开始编译
./configure \
    --prefix=/usr/local/nginx \
    --pid-path=/var/run/nginx/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --with-http_gzip_static_module \
    --http-client-body-temp-path=/var/temp/nginx/client \
    --http-proxy-temp-path=/var/temp/nginx/proxy \
    --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
    --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
    --http-scgi-temp-path=/var/temp/nginx/scgi
make && make install
```

### **编译参数解释**

``` TOML
–prefix 指定nginx安装目录

–pid-path 指向nginx的pid

–lock-path 锁定安装文件，防止被恶意篡改或误操作

–error-log 错误日志

–http-log-path http日志

–with-http_gzip_static_module 启用gzip模块，在线实时压缩输出数据流

–http-client-body-temp-path 设定客户端请求的临时目录

–http-proxy-temp-path 设定http代理临时目录

–http-fastcgi-temp-path 设定fastcgi临时目录

–http-uwsgi-temp-path 设定uwsgi临时目录

–http-scgi-temp-path 设定scgi临时目录
```

### 启停命令

``` Bash
# 启动：
./nginx
# 停止：
./nginx -s stop
# 重新加载：
./nginx -s reload
# 查看nginx版本信息
./nginx -v
# 查看编译信息
./nginx -V
# 热升级
kill -USR2 $OLD_MASTER_PID
kill -WINCH $OLD_MASTER_PID
kill -QUIT $OLD_MASTER_PID
# 热升级回滚
kill -HUP $OLD_MASTER_PID
kill -QUIT $NEW_MASTER_PID
mv /usr/local/nginx/sbin/nginx.old /usr/local/nginx/sbin/nginx
```

# 配置文件详解

Nginx 的配置文件（默认为 `/etc/nginx/nginx.conf`）是一个纯文本文件，其语法简洁且具有高度的逻辑性。

---

## 1. 核心概念：指令与指令块

Nginx 的配置由两种基本单元组成：

1. **指令 (Directive):**
    
    - 这是 Nginx 配置的基本单元，由**名称**和**参数**组成，以**分号 (`;`)** 结尾。
        
    - 示例：`worker_processes 4;`
        
    - `worker_processes` 是指令名称，`4` 是参数。
        
2. **指令块 (Block Directive / Context):**
    
    - 这是一个容器，可以包含其他的指令和指令块。
        
    - 它使用**花括号 (`{ }`)** 来定义一个上下文（Context）。
        
    - 示例：`http { ... }`
        
    - `http` 是指令块的名称，`{ ... }` 内部是该块的配置内容。
        

**核心规则：** 所有的 Nginx 配置都遵循这种由**指令**和**指令块**嵌套组成的层次结构。

---

## 2. Nginx 配置文件的主要结构

一个典型的 `nginx.conf` 文件按顺序包含以下几个主要的指令块（上下文）：

``` Nginx
# 1. 全局块 (Main / Global Context)
# (这里的指令不属于任何花括号)
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# 2. events 块
events {
    worker_connections 1024;
}

# 3. http 块
http {
    # ... http 全局配置 ...
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    log_format main '...';
    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    keepalive_timeout 65;

    # 3a. upstream 块 (可选, 用于负载均衡)
    upstream my_backend {
        server backend1.example.com;
        server backend2.example.com;
    }

    # 3b. server 块 (定义虚拟主机)
    server {
        # ... server 1 的配置 ...
        listen 80;
        server_name example.com www.example.com;

        # 3c. location 块 (定义 URI 匹配规则)
        location / {
            root /var/www/html;
            index index.html index.htm;
        }
    }
    
    server {
        # ... server 2 的配置 ...
        listen 443 ssl;
        server_name api.example.com;
        
        location /api/v1/ {
            proxy_pass http://my_backend;
        }
    }
    
    # 包含其他配置文件
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## 3. 各个结构的作用和配置详解

下面我们详细分解每个部分的功能和常用配置。

### 1. 全局块 (Global Block)

**作用：** 这是配置文件的顶层，这里的指令不属于任何 `{}` 块。它们用于配置 Nginx 服务器**最基本**的全局特性，影响整个 Nginx 实例。

**常用指令：**

- `user <user> [group];`
    
    - **作用：** 指定 Nginx worker 进程运行的用户和用户组。出于安全考虑，通常会指定为一个低权限的用户（如 `nginx` 或 `www-data`）。
        
    - **示例：** `user nginx;`
        
- `worker_processes <number | auto>;`
    
    - **作用：** Nginx 启动的 worker 进程数。worker 进程是实际处理用户请求的进程。
        
    - **配置：**
        
        - `auto`: Nginx 会自动检测 CPU 核心数并设置为该值，这是最推荐的设置。
            
        - `<number>`: 手动指定数量，通常设置为等于 CPU 核心数。
            
- `error_log <file> [level];`
    
    - **作用：** 定义全局的错误日志文件路径和日志级别。
        
    - **级别：** `debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, `emerg` (从低到高)。生产环境建议使用 `warn` 或 `error`。
        
    - **示例：** `error_log /var/log/nginx/error.log warn;`
        
- `pid <file>;`
    
    - **作用：** 指定存储 Nginx master 进程 ID (PID) 的文件路径。
        
    - **示例：** `pid /var/run/nginx.pid;`
        

### 2. `events` 块

**作用：** `events` 块用于配置 Nginx 的**网络连接处理机制**。它定义了 Nginx 如何处理客户端的连接。

**常用指令：**

- `worker_connections <number>;`
    
    - **作用：** **（最重要）** 设置每个 worker 进程允许的**最大并发连接数**。
        
    - **注意：** 这不是指 Nginx 总的并发连接数。Nginx 的最大并发数约等于 `worker_processes * worker_connections`。
        
    - **示例：** `worker_connections 1024;`
        
- `use <method>;`
    
    - **作用：** 指定 Nginx 使用的事件模型。
        
    - **配置：** Nginx 会自动选择最高效的模型。在 Linux 上通常是 `epoll`，在 BSD 上是 `kqueue`。一般**不需要**手动配置。
        
- `multi_accept on | off;`
    
    - **作用：** 设置为 `on` 时，允许 worker 进程一次性接收（accept）所有新的连接，而不是一个一个地接收。这在有大量并发连接时可以提高效率。
        

### 3. `http` 块

**作用：** 这是 Nginx 配置中**最核心、最庞大**的块。它定义了 HTTP 协议相关的所有配置。Nginx 作为一个 Web 服务器或反向代理，绝大多数配置都在这个块内部。

**`http` 块内的常用指令 (作为 `server` 块的默认值)：**

- `include <file | mask>;`
    
    - **作用：** 包含其他配置文件，用于模块化管理。
        
    - **示例：** `include /etc/nginx/mime.types;` (加载 Mime Type 定义)
        
    - **示例：** `include /etc/nginx/conf.d/*.conf;` (加载 `conf.d` 目录下的所有 `.conf` 文件)
        
- `default_type <mime-type>;`
    
    - **作用：** 设置默认的 Mime Type。如果 `mime.types` 文件中没有找到匹配的类型，则使用此默认值。
        
    - **示例：** `default_type application/octet-stream;`
        
- `log_format <name> '<string>';`
    
    - **作用：** 自定义日志格式。
        
    - **示例：** `log_format main '$remote_addr - $remote_user [$time_local] "$request" $status';`
        
- `access_log <path> [format];`
    
    - **作用：** 定义访问日志的路径和所用的格式（由 `log_format` 定义）。
        
    - **示例：** `access_log /var/log/nginx/access.log main;`
        
- `sendfile on | off;`
    
    - **作用：** 启用或禁用 `sendfile()` 系统调用来传输文件，这可以极大提高静态文件的传输效率（零拷贝）。**强烈建议开启 `on`**。
        
- `keepalive_timeout <timeout>;`
    
    - **作用：** 设置 HTTP "keep-alive" (长连接) 的超时时间。在此时间内客户端可以复用同一个连接发送多个请求。
        
    - **示例：** `keepalive_timeout 65;`
        

---

#### `http` 块的子块

`http` 块内部可以包含 `upstream`, `server` 等子块。

#### 3a. `upstream` 块

**作用：** 定义一组后端服务器（服务器池），主要用于**负载均衡 (Load Balancing)**。

**配置方法：**

- `upstream` 块必须定义在 `http` 块内，`server` 块之外。
    
- 它定义了一个**名称**（如下例中的 `my_backend`），这个名称稍后可以在 `location` 块的 `proxy_pass` 指令中被引用。
    

**常用指令：**

- `server <address> [parameters];`
    
    - **作用：** 定义后端服务器的地址（可以是 IP:Port、域名或 Unix Sockets）。
        
    - **参数：**
        
        - `weight=number`: 设置权重，数字越大，分配到的请求越多。
            
        - `max_fails=number`: 最大失败次数。
            
        - `fail_timeout=time`: 失败超时时间。
            
        - `down`: 标记服务器永久不可用。
            
        - `backup`: 标记为备份服务器。
            
- **负载均衡策略 (写在 `upstream` 块顶部)：**
    
    - `(默认)`: **Round Robin (轮询)**。
        
    - `least_conn;`: **最少连接**。请求被转发到当前活动连接数最少的服务器。
        
    - `ip_hash;`: **IP 哈希**。根据客户端的 IP 地址进行哈希计算，确保来自同一客户端的请求始终被转发到同一台后端服务器（用于解决 Session 共享问题）。
        

**示例：**

``` Nginx
http {
    upstream my_backend {
        # 轮询 (默认)
        server 192.168.1.100:8080 weight=3; # 权重为3
        server 192.168.1.101:8080;         # 权重为1
    }
    
    upstream api_servers {
        ip_hash; # IP 哈希策略
        server 10.0.0.1:80;
        server 10.0.0.2:80;
    }
    
    server {
        # ...
        location /app/ {
            proxy_pass http://my_backend; # 引用 upstream
        }
    }
}
```

#### 3b. `server` 块

**作用：** `server` 块是 Nginx **虚拟主机 (Virtual Host)** 的实现。Nginx 可以通过 `server` 块在同一台物理机、同一个 IP 和端口上托管多个不同的网站。

**`http` 块可以包含一个或多个 `server` 块。**

**常用指令：**

- `listen <port | address:port>;`
    
    - **作用：** 指定 Nginx 监听的端口和（可选的）IP 地址。
        
    - **示例：**
        
        - `listen 80;` (监听所有 IP 的 80 端口)
            
        - `listen 192.168.1.10:80;` (只监听特定 IP 的 80 端口)
            
        - `listen 443 ssl http2;` (监听 443 端口，并启用 SSL 和 HTTP/2)
            
- `server_name <name1> [name2] ...;`
    
    - **作用：** **（最重要）** 指定此 `server` 块对应的**域名**。Nginx 通过请求头中的 `Host` 字段来匹配 `server_name`，从而决定使用哪个 `server` 块来处理该请求。
        
    - **示例：**
        
        - `server_name example.com www.example.com;` (匹配 `example.com` 和 `www.example.com`)
            
        - `server_name *.example.com;` (通配符)
            
        - `server_name ~^www\d+\.example\.com$;` (正则表达式)
            
        - `server_name _;` (作为默认 "catch-all" server)
            
- `root <path>;`
    
    - **作用：** 指定该虚拟主机的网站**根目录**。`location` 块中请求的 URI 会被附加到这个路径后。
        
    - **示例：** `root /var/www/my-site;`
        
- `index <file1> [file2] ...;`
    
    - **作用：** 指定索引文件（当用户请求一个目录时，Nginx 会尝试查找的文件）。
        
    - **示例：** `index index.html index.htm default.php;`
        
- `error_page <code> ... [=response] <uri>;`
    
    - **作用：** 自定义错误页面。
        
    - **示例：** `error_page 404 /404.html;` (当发生 404 错误时，内部重定向到 `/404.html`)
        
    - **示例：** `error_page 500 502 503 504 /50x.html;`
        
- `ssl_certificate <path>;` 和 `ssl_certificate_key <path>;`
    
    - **作用：** 用于 HTTPS，指定 SSL 证书和私钥的路径。
        

#### 3c. `location` 块

**作用：** `location` 块定义在 `server` 块内部，用于匹配特定的 **URI (Uniform Resource Identifier)**（即 URL 中域名后面的部分），并为这些匹配的请求应用特定的配置。这是 Nginx 中最灵活、最复杂的配置部分。

**配置方法 (匹配规则)：**

`location` 块的匹配规则有不同的优先级：

1. `location = /path`
    
    - **精确匹配**。优先级最高。只匹配 `/path` 这个 URI。
        
2. `location ^~ /path`
    
    - **优先前缀匹配**。匹配以 `/path` 开头的 URI。如果匹配成功，Nginx 将停止搜索更复杂的正则表达式匹配。
        
3. `location ~ /regex`
    
    - **正则表达式匹配 (区分大小写)**。
        
4. `location ~* /regex`
    
    - **正则表达式匹配 (不区分大小写)**。
        
5. `location /path`
    
    - **普通前缀匹配**。匹配以 `/path` 开头的 URI。这是优先级最低的匹配方式。
        
6. `location /`
    
    - **通用匹配**。匹配所有请求，通常作为最后的 "fallback" (兜底) 规则。
        

**`location` 块内的常用指令：**

- `root <path>;`
    
    - **作用：** 设置此 location 的**根目录**。
        
    - **示例：**
        
        ``` Nginx
        location /images/ {
            root /var/www/data;
        }
        # 请求 /images/logo.png -> 会查找 /var/www/data/images/logo.png
        ```
        
- `alias <path>;`
    
    - **作用：** **别名**。它用 `<path>` 替换掉 location 匹配到的 URI 部分。
        
    - **示例：**
        
        ``` Nginx
        location /images/ {
            alias /var/www/data;
        }
        # 请求 /images/logo.png -> 会查找 /var/www/data/logo.png (注意看和 root 的区别)
        ```
        
- `proxy_pass <url>;`
    
    - **作用：** **（反向代理核心）** 将请求转发到另一个服务器（可以是 `upstream` 定义的名称，或直接的 URL）。
        
    - **示例：**
        
        - `proxy_pass http://my_backend;` (转发到 upstream)
            
        - `proxy_pass http://127.0.0.1:8080;` (转发到本地 8080 端口)
            
- `try_files $uri $uri/ /index.html;`
    
    - **作用：** 按顺序尝试查找文件。这在配置**前端框架 (如 Vue/React) 的 History 模式**时非常有用。
        
    - **解释：** 尝试查找与 URI 完全匹配的文件 (`$uri`) -> 尝试查找与 URI 匹配的目录 (`$uri/`) -> 如果都找不到，则内部重定向到 `/index.html`。
        
    - **示例：**
        
        ``` Nginx
        location / {
            root /var/www/app;
            try_files $uri $uri/ /index.html;
        }
        ```
        
- `deny <address | all>;` 和 `allow <address | all>;`
    
    - **作用：** 基于 IP 地址控制访问。
        
    - **示例：**
        
        ``` Nginx
        location /admin/ {
            allow 192.168.1.0/24; # 允许局域网
            deny all;             # 拒绝其他所有
        }
        ```
        

---

## 4. 配置的继承性

Nginx 的配置具有**继承性**：

- `server` 块会继承 `http` 块中的指令（如 `access_log`, `keepalive_timeout`）。
    
- `location` 块会继承 `server` 块中的指令（如 `root`, `index`）。
    

**规则：** 如果在子块（如 `location`）中定义了与父块（如 `server`）中**相同**的指令，子块的定义会**覆盖**父块的定义。

---

## 5. 最佳实践：模块化配置

为了保持 `nginx.conf` 主文件的简洁，最佳实践是使用 `include` 指令将配置拆分到不同的文件中。

- **`conf.d` 目录：** 许多 Nginx 安装（如 CentOS/RHEL）默认会在 `http` 块的末尾包含 `include /etc/nginx/conf.d/*.conf;`。
    
    - 你可以把**所有**的 `server` 块配置（每个网站一个 `.conf` 文件）都放在 `/etc/nginx/conf.d/` 目录下（例如 `my_site.conf`）。
        
- **`sites-available` / `sites-enabled` (Debian/Ubuntu 模式)：**
    
    - `/etc/nginx/sites-available/`: 存放所有可用的 `server` 配置文件。
        
    - `/etc/nginx/sites-enabled/`: 存放**符号链接 (symbolic links)**，链接到 `sites-available` 中你**想要启用**的配置文件。
        
    - `nginx.conf` 文件中会包含 `include /etc/nginx/sites-enabled/*;`。
        

---

## 6. 测试和重载配置

在修改配置文件后，**切勿**直接重启 (restart) Nginx，因为如果配置有语法错误，Nginx 将无法启动，导致服务中断。

正确的流程是：

1. **测试配置 (Test):**
    
    ``` Bash
    nginx -t
    ```
    
    - 如果配置正确，你会看到：
        
        ```
        nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
        nginx: configuration file /etc/nginx/nginx.conf test is successful
        ```
        
    - 如果配置有误，它会明确指出错误在哪一行。
        
2. **平滑重载 (Reload):**
    
    - 只有在 `nginx -t` 测试通过后，才执行重载。
        
    - Reload 会启动新的 worker 进程来加载新配置，并平滑地关闭旧的 worker 进程（等待它们处理完当前请求），服务**不会中断**。
        
    
    ``` Bash
    nginx -s reload
    ```
    

# 模块化reverse-proxy配置实例

## 配置文件目录结构

我们将创建和编辑以下文件：

```
/etc/nginx/
├── nginx.conf                   # (1) 主配置文件 (我们将修改它以包含 sites-enabled)
|
├── conf.d/                      # (通常用于全局 http 相关的模块配置)
|
├── snippets/                    # (2) 新建! 用于存放可重用的配置片段
│   ├── ssl-params.conf          # (3) SSL 安全参数
│   └── proxy-params.conf        # (4) 反向代理通用头部
|
├── sites-available/             # (存放所有站点的配置文件)
│   └── your_domain.com.conf     # (5) 我们的站点配置文件
|
└── sites-enabled/               # (存放已激活站点的符号链接)
    └── your_domain.com.conf     # (这是一个指向 ../sites-available/your_domain.com.conf 的
```

## 主配置文件

`nginx.conf`是 Nginx 的入口文件

``` Nginx
user www-data; # 或 work 启动用户
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log warn;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;

    # 性能调优
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    
    # Gzip 压缩 (可选, 但推荐)
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # ==========================================================
    # 核心模块化配置：包含 conf.d 和 sites-enabled
    # ==========================================================
    
    # 包含其他全局 HTTP 配置
    include /etc/nginx/conf.d/*.conf;
    
    # 包含所有已启用的虚拟主机（站点）
    include /etc/nginx/sites-enabled/*;
}
```

## SSL/HTTPS配置文件

SSL/TLS 安全设置放在`ssl-params.conf`

``` Nginx
# 来自 https://ssl-config.mozilla.org/ (现代兼容性)
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers on;

# 启用 HSTS (HTTP Strict Transport Security)
# 强制浏览器在指定时间内 (6 个月) 只能使用 HTTPS 访问
add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;

# (可选) 其他安全头部
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;

# SSL 会话缓存
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;

# 禁用旧版 SSLv3 (已由 ssl_protocols 覆盖, 此处为双重保险)
ssl_session_protocols TLSv1.2 TLSv1.3;
```

## 反向代理配置

`proxy-params.conf` 配置的是反向代理时**必须**设置的头部，用于将客户端的真实信息传递给后端应用。

``` Nginx
# 传递原始主机名
proxy_set_header Host $host;

# 传递客户端真实 IP
proxy_set_header X-Real-IP $remote_addr;

# 传递代理链中的所有 IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

# 传递原始协议 (http 还是 https)
# 这对于后端应用正确生成 URL 至关重要
proxy_set_header X-Forwarded-Proto $scheme;

# 支持 WebSocket (如果您的应用需要)
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

## 后端服务代理配置

``` Nginx
# -------------------------------------------------
# 1. 定义后端应用 (Upstream)
# -------------------------------------------------
# 这使得后端地址易于管理和负载均衡
upstream my_app_backend {
    # 被代理的应用运行在本地 8000 端口
    server 127.0.0.1:8000;

    # 如果有多个实例，可以像这样添加：
    # server 127.0.0.1:8001;
    # server 10.0.0.10:8000;
}


# -------------------------------------------------
# 2. HTTP (80 端口) -> 永久重定向到 HTTPS
# -------------------------------------------------
server {
    listen 80;
    listen [::]:80;
    
    server_name your_domain.com www.your_domain.com;
    
    # 针对 Let's Encrypt 续订的特殊 location (Certbot)
    # Certbot 的 http-01 质询需要通过 80 端口访问
    location /.well-known/acme-challenge/ {
        root /var/www/html; # Certbot 通常使用的临时验证目录
        allow all;
    }

    # 将所有其他 HTTP 请求重定向到 HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}


# -------------------------------------------------
# 3. HTTPS (443 端口) -> 实际的反向代理
# -------------------------------------------------
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    server_name your_domain.com www.your_domain.com;
    
    # ------------------
    # SSL 证书
    # ------------------
    # (使用 Certbot/Let's Encrypt 的推荐路径)
    ssl_certificate /etc/letsencrypt/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain.com/privkey.pem;
    
    # ------------------
    # 包含我们的 SSL 安全配置片段
    # ------------------
    include snippets/ssl-params.conf;
    
    # (可选) 访问和错误日志
    access_log /var/log/nginx/your_domain.com.access.log main;
    error_log /var/log/nginx/your_domain.com.error.log warn;

    # ------------------
    # 核心：反向代理配置
    # ------------------
    location / {
        # 将请求转发到我们定义的 upstream
        proxy_pass http://my_app_backend;
        
        # 包含我们的反向代理通用头部片段
        include snippets/proxy-params.conf;
    }
    
    # (可选) 如果想让 Nginx 直接处理静态文件 (性能更高)
    # location /static/ {
    #     alias /path/to/your/app/static/;
    #     expires 30d;
    #     add_header Cache-Control "public";
    # }
}
```

## SSL证书获取

``` Bash
sudo apt install certbot python3-certbot-nginx
# 注意：这里使用 --nginx 插件会自动修改配置，
# 但我们是手动配置，所以推荐使用 'certonly'
sudo certbot certonly --webroot -w /var/www/html -d your_domain.com -d www.your_domain.com
```