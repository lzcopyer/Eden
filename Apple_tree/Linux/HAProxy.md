## 引言
[官方网址](http://www.haproxy.org/download/)  
HAProxy 是==一款开源的负载均衡软件==，它可以运行在多种主流的Linux 操作系统上，并提供高可用性、负载均衡和应用程序代理功能。HAProxy 尤其适合纯TCP负载均衡、对性能、稳定性和连接处理能力要求极高的场景。

## HAProxy负载均衡算法
|**场景需求**|**推荐算法**|**说明**|
|---|---|---|
|**简单通用，短连接**|`roundrobin` (带权重) 或 `random`|默认轮询即可；随机在高并发下性能更好。|
|**长连接（数据库、WS、MQ）**|`leastconn` (带权重)|基于当前连接数分配，负载最均衡。|
|**基于客户端 IP 的会话保持**|`source` + `hash-type consistent`|确保同一 IP 用户访问相同后端。**务必加一致性哈希**。|
|**基于 URI 的缓存亲和性**|`uri` + `hash-type consistent`|同一资源请求到同一后端，提高缓存效率。**务必加一致性哈希**。|
|**基于 URL 参数 (如 sessionid) 的会话保持**|`url_param` + `hash-type consistent`|更精准的会话保持。注意 POST 请求需 `check_post`。**务必加一致性哈希**。|
|**基于 HTTP Header 的路由/会话保持**|`hdr(<header-name>)` + `hash-type consistent`|灵活性强，如基于 `User-Agent` 或自定义头。**务必加一致性哈希**。|
|**Microsoft RDP 负载均衡**|`rdp-cookie`|专为 RDP 协议设计。|
|**大规模服务器池，追求极致性能**|`random` (带权重)|随机选择开销最小，统计上均衡性好。无需会话保持时首选。|
|**强制主备 (非负载均衡)**|`first`|第一个可用服务器处理所有请求，失败才切备用。|

## HAProxy配置说明

``` TOML
global  
    chroot /apps/haproxy   改变根目录  
    deamon                 后台运行  
    mode tcp | http        转发类型   
    stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin  proxess 1    这是socket文件,可用通过命令向该文件执行命令  
    user  gid | username  
    group uid | groupname 指定用户和组  
    nbproc n   该命令和nginx相似，开启n个进程，不过从2.5以后被弃用  
    nbthread n  指定每个进程开启的线程，默认为1，同时两者同时开启会报错，Centos8的1.8无此问题  
    cpu-map 1 0  
    cpu-map 2 1  亲缘性，CPU绑定进程，进程从1开始，cpu从0开始  
    cpu-map auto:1/1-8 0-7   自动绑定，每个进程的1-8个线程分别绑定0-7号cpu。haproxy2.4中为nbthreads  
    #查看cpu绑定 ps axo pid,cmd,psr -L | grep haproxy  
    #或者安装sysstat,pidstat -p  haproxy-pid -t   
    maxconn n       每个进程的最大连接数为n  
    maxsslconn n    每个进程的ssl最大连接数为n  
    maxconnrate n   每个进程每秒能创建的最多连接数为n  
    spread-checks n 后端server状态检查，建议为2-5，默认为0  
    pidfile   /some/path/haproxy.pid  PID文件路径  
    log 127.0.0.1 local2 info   日志发送到本机，默认为514 udp端口  
    log x.x.x.x local2 info     定义log，和rsyslog配合使用，local3为名称，info为日志等级，需要开启UDP，最多可以定义两个  
        日志配置  
        vim /etc/rsyslog.conf  
        module(load="imudp")  
        input(type="imudp" port="514")  
  
        echo  local2.* /var/log/haproxy.log 或 local2.info  /var/log/haproxy.log > /etc/rsyslog.d/50-default.conf  
        systemctl restart rsyslog  
defaults  
    option redispatch                   重定向其他正常服务器  
    option abortonclose                 自动结束连接时长较久的连接               
    option http-keep-alive              开启与客户端的会话保持  
    option forwardfor                   透传真实IP至后端服务器,可自定义修改  
    mode http | tcp                     默认工作类型  
    timeout http-keep-alive             会话保持时间  
    timeout connect 120s                客户端请求从haproxy到后端server最长连接等待时间(TCP连接之前)默认单位 ms  
    timeout server 600s                 客户端请求从haproxy到后端服务端的请求处理超时时长(TCP连接之后)，超时502  
    timeout client 600s                 设置haproxy与客户端的最长非活动时间，默认单位ms，建议和timeoutserver相同  
    timeout check 5s                    对后端服务器的默认检测超时时间  
    default-server inter 1000 weight 3  指定后端服务器的默认设置  
  
前后端整合  
listen nginx                            使用listen替换 frontend和backend的配置方式，可以简化设置，常用于TCP协议的应用         
        bind 公网IP:端口  
        mode tcp | http  
        server web1 后端IP:port  
        server web2 后端IP:port  
  
  
前端服务器  
frontend  nginx  
    bind 公网IP：端口  
    use_backend real-nginx  
  
后端服务器                                                                                   
backend  real-nginx   
    redirect prefix http://www.baidu.com/                    全局重定向，只适合http  
    server web1 后端IP:port check inter 5000 fail 3 rise 3   每5秒检查一次，如果3次不通就挂，上线同理  
    server web2 后端IP:port weight 3                         权重  
    server web2 后端IP:port check backup                     备份服务器，可当sorry server  
        server后字段  
            check   
                addr <IP> 可指定的健康状态监测IP，可以是专门的数据网段，减少业务网络的流量  
                port <num> 指定的健康状态监测端口  
                inter <num> 健康状态检查间隔时间，默认2000 ms  
                fall <num> 后端服务器从线上转为线下的检查的连续失效次数，默认为3  
                rise <num> 后端服务器从下线恢复上线的检查的连续有效次数，默认为2  
            disable 关闭服务器  
            maxconn 后端服务器最大连接数  
            redir http://x.x.x  局部302临时重定向到某url，只适合http  
  
listen mysql  
        bind 0.0.0.0:3306           haproxy端口  
        mode tcp                    如果不指定tcp，会以http进行连接  
        server data1 x.x.x.x:3306   后端数据库地址和端口  
  
listen stats  
        mode http  
        bind 0.0.0.0:9999  
        stats enable  
        log global                  开启日志，默认不开启  
        stats uri /haproxy-status   访问后缀  
        stats auth admin:123456     开启认证  
        stats refresh 5             自动刷新，这里是每5s  
        stats admin if TURE         如果是admin就允许访问stats页面
```

## 编译安装

安装依赖
``` bash
# ubuntu
apt -y install gcc make libssl-dev libpcre3  libpcre3-dev zlib1g-dev libreadline-dev libsystemd-dev liblua5.4-dev

# centos
yum -y install gcc make openssl-devel pcre-devel systemd-devel

# 安装lua环境
wget https://www.lua.org/ftp/lua-5.4.8.tar.gz
tar xf lua-5.4.8.tar.gz -C /usr/local/src/
cd /usr/local/src/lua-5.4.8/
make linux test
mv /usr/bin/lua /usr/bin/lua.old
cp /usr/local/src/lua-5.4.8/src/lua /usr/bin/lua
```

编译HAProxy
``` bash
# 下载源码包
wget https://www.haproxy.org/download/3.3/src/devel/haproxy-3.3-dev3.tar.gz

# 编译
tar xf haproxy-3.3-dev3.tar.gz
cd haproxy-3.3
make TARGET=linux-glibc USE_OPENSSL=1 USE_PCRE=1 USE_SYSTEMD=1 USE_LUA=1 LUA_INC=/usr/local/src/lua-5.4.8/src LUA_LIB=/usr/local/src/lua-5.4.8/src
make install PREFIX=/usr/local/haproxy

# 创建软连接
ln -s /usr/local/haproxy/sbin/haproxy /usr/bin/haproxy

# 创建配置文件
mdkir -p /etc/haproxy/conf
touch /etc/haproxy/haproxy.cfg

# 创建service
cat << EOF > /usr/lib/systemd/system/haproxy.service  
[Unit]  
Description=HAProxy Load Balancer  
After=syslog.target network.target  
[Service]  
ExecStartPre=/usr/local/sbin/haproxy -f /haproxy/haproxy.cfg -f /etc/haproxy/haproxy.d/ -c -q
#-f可指定文件，也可指定目录，目录一般存放子配置文件
ExecStart=/usr/local/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/haproxy.d/ -p  /var/lib/haproxy/haproxy.pid  
ExecReload=/bin/kill -USR2 $MAINPID  
LimitNOFILE=100000  
[Install]  
WantedBy=multi-user.target  
EOF

# 重载services
systemctl daemon-reload

# 设置开机自启动
systemctl enable haproxy
```

## 部署脚本

``` bash
#!/bin/bash
# HAProxy 部署脚本 v1.2
# 支持 RHEL/CentOS 7+
# 作者: Malou
# 最后更新: 2025-07-15

# 设置错误处理
set -e
trap 'echo -e "\033[31m错误发生在第 $LINENO 行，退出状态 $?\033[0m"' ERR

# 配置参数 - 根据实际情况修改这些值
work_path="/tmp/haproxy-install"
Lua_ver="5.4.8"
Haproxy_ver="2.8"  # 使用稳定版本号
HAPROXY_USER="haproxy"
HAPROXY_GROUP="haproxy"
DOMAIN_CERT="/etc/ssl/private/yourdomain.pem"  # 修改为实际证书路径

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m' # 无颜色

# 检查是否以 root 运行
if [ "$(id -u)" != "0" ]; then
    echo -e "${RED}错误: 此脚本必须以 root 权限运行${NC}"
    exit 1
fi

# 安装依赖
echo -e "${GREEN}[1/9] 安装依赖包...${NC}"
yum -y install epel-release
yum -y install gcc make openssl-devel pcre-devel systemd-devel readline-devel tar wget

# 创建工作目录
echo -e "${GREEN}[2/9] 创建工作目录: ${work_path}${NC}"
mkdir -p "$work_path"
cd "$work_path"

# 安装 Lua
echo -e "${GREEN}[3/9] 安装 Lua ${Lua_ver}...${NC}"
if ! command -v lua &> /dev/null; then
    wget -q --show-progress "https://www.lua.org/ftp/lua-${Lua_ver}.tar.gz"
    tar xf "lua-${Lua_ver}.tar.gz" -C /usr/local/src/
    cd "/usr/local/src/lua-${Lua_ver}/"
    make -j$(nproc) linux test
    if [ -f /usr/bin/lua ]; then
        mv /usr/bin/lua /usr/bin/lua.old
    fi
    cp src/lua /usr/bin/lua
    cp src/luac /usr/bin/luac
    echo -e "${YELLOW}Lua ${Lua_ver} 安装完成${NC}"
else
    echo -e "${YELLOW}Lua 已安装，跳过安装步骤${NC}"
fi

# 安装 HAProxy
echo -e "${GREEN}[4/9] 安装 HAProxy ${Haproxy_ver}...${NC}"
cd "$work_path"
if ! command -v haproxy &> /dev/null; then
    wget -q --show-progress "https://www.haproxy.org/download/${Haproxy_ver}/src/haproxy-${Haproxy_ver}.tar.gz"
    tar xf "haproxy-${Haproxy_ver}.tar.gz"
    cd "haproxy-${Haproxy_ver}"
    
    # 编译参数优化
    make -j$(nproc) TARGET=linux-glibc \
        USE_OPENSSL=1 \
        USE_PCRE=1 \
        USE_PCRE_JIT=1 \
        USE_SYSTEMD=1 \
        USE_LUA=1 \
        LUA_INC="/usr/local/src/lua-${Lua_ver}/src" \
        LUA_LIB="/usr/local/src/lua-${Lua_ver}/src" \
        CPU=native \
        ARCH=native
    
    make install PREFIX=/usr/local/haproxy
    ln -sf /usr/local/haproxy/sbin/haproxy /usr/sbin/haproxy
    echo -e "${YELLOW}HAProxy ${Haproxy_ver} 安装完成${NC}"
else
    echo -e "${YELLOW}HAProxy 已安装，跳过安装步骤${NC}"
fi

# 创建用户和目录
echo -e "${GREEN}[5/9] 创建系统用户和目录...${NC}"
if ! id "$HAPROXY_USER" &>/dev/null; then
    useradd -r -s /sbin/nologin "$HAPROXY_USER"
fi

mkdir -p /etc/haproxy/haproxy.d
mkdir -p /var/lib/haproxy
chown -R "$HAPROXY_USER:$HAPROXY_GROUP" /var/lib/haproxy
mkdir -p /var/log/haproxy
chown -R "$HAPROXY_USER:$HAPROXY_GROUP" /var/log/haproxy

# 配置 rsyslog 日志
echo -e "${GREEN}[6/9] 配置日志系统...${NC}"
cat <<EOF > /etc/rsyslog.d/haproxy.conf
\$ModLoad imudp
\$UDPServerRun 514

local0.* /var/log/haproxy/haproxy.log
EOF

systemctl restart rsyslog

# 创建 HAProxy 配置文件
echo -e "${GREEN}[7/9] 创建 HAProxy 配置文件...${NC}"
cat <<EOF > /etc/haproxy/haproxy.cfg
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /var/lib/haproxy/haproxy.sock mode 660 level admin
    stats timeout 30s
    user $HAPROXY_USER
    group $HAPROXY_GROUP
    daemon
    maxconn 10000
    nbproc 1
    nbthread 4
    cpu-map auto:1/1-4 0-3
    tune.ssl.default-dh-param 2048
    
defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    option  redispatch
    option  tcp-smart-accept
    option  tcp-smart-connect
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    timeout http-request 10s
    timeout queue   30s
    retries 3
    
# HiveServer2 代理配置
frontend hive_frontend
    bind *:10000
    default_backend hive_backend

# HTTP/HTTPS 代理配置
frontend http_frontend
    bind *:80
    bind *:443 ssl crt $DOMAIN_CERT alpn h2,http/1.1
    mode    http
    option  httplog
    option  forwardfor
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header X-Forwarded-Port %[dst_port]
    acl is_http dst_port 80
    redirect scheme https code 301 if !{ ssl_fc } is_http
    default_backend http_backend

# HiveServer2 后端配置
backend hive_backend
    mode tcp
    option tcp-check
    tcp-check connect port 10000
    tcp-check send PING\r\n
    tcp-check expect string PONG
    balance leastconn
    timeout connect 10s
    timeout server  1h
    server hiveserver1 192.168.1.101:10000 check inter 5s rise 2 fall 3 maxconn 300
    server hiveserver2 192.168.1.102:10000 check inter 5s rise 2 fall 3 maxconn 300

# HTTP 后端配置
backend http_backend
    mode http
    balance roundrobin
    cookie SERVERID insert indirect nocache
    option httpchk HEAD / HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200
    server web1 192.168.1.201:80 check cookie web1 inter 10s
    server web2 192.168.1.202:80 check cookie web2 inter 10s

# HAProxy 统计页面
listen stats
    bind *:9000
    mode http
    stats enable
    stats hide-version
    stats refresh 10s
    stats show-node
    stats uri /haproxy?stats
    stats realm HAProxy\ Statistics
    stats auth admin:${RANDOM}SecurePwd${RANDOM}  # 随机生成安全密码
EOF

# 生成随机统计密码（实际环境中应使用安全密码）
STATS_PWD=$(grep 'stats auth' /etc/haproxy/haproxy.cfg | awk '{print $4}')
echo -e "${YELLOW}统计页面密码: $STATS_PWD${NC}"
echo -e "${YELLOW}请记录此密码并修改为安全密码${NC}"

# 创建 systemd 服务
echo -e "${GREEN}[8/9] 创建 systemd 服务...${NC}"
cat <<EOF > /usr/lib/systemd/system/haproxy.service
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/local/haproxy/doc/haproxy-en.txt
After=network.target syslog.service

[Service]
EnvironmentFile=-/etc/default/haproxy
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid"
ExecStartPre=/usr/sbin/haproxy -f \$CONFIG -c -q
ExecStart=/usr/sbin/haproxy -Ws -f \$CONFIG -p /run/haproxy.pid
ExecReload=/usr/sbin/haproxy -f \$CONFIG -c -q
ExecReload=/bin/kill -USR2 \$MAINPID
KillMode=mixed
Restart=on-failure
RestartSec=5s
TimeoutStopSec=30
LimitNOFILE=100000
User=$HAPROXY_USER
Group=$HAPROXY_GROUP

[Install]
WantedBy=multi-user.target
EOF

# 启用并启动服务
echo -e "${GREEN}[9/9] 启动 HAProxy 服务...${NC}"
systemctl daemon-reload
systemctl enable haproxy

# 配置检查
if ! /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c; then
    echo -e "${RED}配置文件检查失败，请检查错误${NC}"
    exit 1
fi

systemctl start haproxy

# 验证安装
echo -e "\n${GREEN}安装完成!${NC}"
echo -e "${YELLOW}验证 HAProxy 状态:${NC}"
systemctl status haproxy --no-pager

echo -e "\n${YELLOW}监听端口:${NC}"
ss -tulpn | grep haproxy

echo -e "\n${YELLOW}统计页面 URL: http://$(hostname -I | awk '{print $1}'):9000/haproxy?stats${NC}"
echo -e "${YELLOW}用户名: admin${NC}"
echo -e "${YELLOW}密码: $STATS_PWD${NC}"

echo -e "\n${GREEN}HAProxy 已成功安装并配置!${NC}"
```

## 运维命令

配置检查命令
``` bash
# 检查配置文件语法
haproxy -c -f /etc/haproxy/haproxy.cfg

# 测试配置并显示完整解析
haproxy -f /etc/haproxy/haproxy.cfg -d
```

基本状态查看
``` bash
# 查看 HAProxy 基本信息
echo "show info" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看所有统计信息（CSV格式）
echo "show stat" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看前端状态
echo "show frontend" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看后端状态
echo "show backend" | socat /var/lib/haproxy/haproxy.sock stdio
```

服务器管理
``` bash
# 禁用后端服务器
echo "disable server <backend>/<server>" | socat /var/lib/haproxy/haproxy.sock stdio
# 示例：禁用 http_backend 中的 web1
echo "disable server http_backend/web1" | socat /var/lib/haproxy/haproxy.sock stdio

# 启用后端服务器
echo "enable server <backend>/<server>" | socat /var/lib/haproxy/haproxy.sock stdio

# 设置服务器进入维护模式
echo "set server <backend>/<server> state maint" | socat /var/lib/haproxy/haproxy.sock stdio

# 设置服务器为就绪状态
echo "set server <backend>/<server> state ready" | socat /var/lib/haproxy/haproxy.sock stdio
```

连接与会话管理
``` bash
# 查看当前活动会话
echo "show sess" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看指定会话详细信息
echo "show sess <session_id>" | socat /var/lib/haproxy/haproxy.sock stdio

# 关闭指定会话
echo "shutdown session <session_id>" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看当前连接
echo "show conn" | socat /var/lib/haproxy/haproxy.sock stdio
```

诊断与调试
``` bash
# 清除计数器
echo "clear counters" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看错误信息
echo "show errors" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看当前活动请求
echo "show activity" | socat /var/lib/haproxy/haproxy.sock stdio

# 查看映射表内容
echo "show map" | socat /var/lib/haproxy/haproxy.sock stdio
```

实时日志监控
``` bash
# 使用 journalctl 查看实时日志
journalctl -u haproxy -f

# 直接查看日志文件
tail -f /var/log/haproxy.log

# 过滤错误日志
grep -i error /var/log/haproxy.log

# 查看特定后端的日志
grep ' http_backend/' /var/log/haproxy.log
```

高级日志分析
``` bash
# 统计状态码分布
awk '{print $11}' /var/log/haproxy.log | sort | uniq -c | sort -rn

# 统计客户端IP访问量
awk '{print $6}' /var/log/haproxy.log | cut -d: -f1 | sort | uniq -c | sort -rn | head -20

# 找出响应时间最长的请求
awk '{print $NF,$6,$7,$8,$9,$10,$11}' /var/log/haproxy.log | sort -rn | head -20

# 统计后端服务器响应时间
awk '/ http_backend\/web/ {gsub(/ms$/,"",$NF); sum[$8]+=$NF; count[$8]++} END {for (s in sum) print s, sum[s]/count[s]"ms"}' /var/log/haproxy.log
```

实时性能查看
``` bash
# 查看进程资源使用
top -p $(pgrep haproxy)

# 监控网络连接
iftop -P -f "port 80 or port 443 or port 10000"

# 查看连接状态统计
ss -ant | awk '{print $1}' | sort | uniq -c
```

使用统计页面
``` bash
# 命令行获取统计信息
curl -u admin:password http://localhost:9000/haproxy?stats;csv

# 格式化查看统计信息
curl -s -u admin:password http://localhost:9000/haproxy?stats;csv | \
column -s, -t
```

热更新配置
``` bash
# 创建新的配置文件
vi /etc/haproxy/haproxy_new.cfg

# 检查新配置
haproxy -c -f /etc/haproxy/haproxy_new.cfg

# 执行热更新（保留现有连接）
haproxy -f /etc/haproxy/haproxy_new.cfg -p /var/run/haproxy.pid -sf $(cat /var/run/haproxy.pid)
```

调试模式
``` bash
# 前台运行调试模式
haproxy -f /etc/haproxy/haproxy.cfg -d

# 调试特定连接
haproxy -f /etc/haproxy/haproxy.cfg -d -p /var/run/haproxy_debug.pid
```

快速查看后端状态
``` bash
echo "show stat" | socat /var/lib/haproxy/haproxy.sock stdio | \
awk -F ',' 'BEGIN {print "Backend\t\tServer\t\tStatus\tCheck"} 
{ if($1=="http_backend" || $1=="hive_backend") print $1"\t"$2"\t"$18"\t"$37 }'
```

监控错误率
``` bash
watch -n 5 'echo "show stat" | socat /var/lib/haproxy/haproxy.sock stdio | \
awk -F "," "/http_backend/,/^$/ {sum+=$49; count++} END {print \"HTTP错误率: \" sum/count \"%\"}"'
```

会话分布查看
``` bash
echo "show stat" | socat /var/lib/haproxy/haproxy.sock stdio | \
awk -F ',' '/BACKEND/ {print $2","$9}' | column -s, -t
```