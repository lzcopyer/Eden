
## 服务端部署

[frp官网](https://github.com/fatedier/frp/releases)

``` bash
tar -zxf  frp_0.37.0_linux_amd64.tar.gz
vim frps.ini
[common]
# 与客户端建立连接端口
bind_port = 7000
# 与客户端校验的token
token = my_token
# 提供web界面端口
dashboard_port = 7500
# web界面登录用户名
dashboard_user = admin
# web界面登录密码
dashboard_pwd = password
enable_prometheus=true
#日志存储
log_file=/data/frp/logs/frps.logs/frps
#日志级别
log_level=info
#日志记录时间
log_max_days=5

##客户端配置
vim frpc.ini
[common]
# 连接frps的IP地址
server_addr = 192.168.1.100
# 连接frps的端口
server_port = 7000
# 和frps服务端校验的token
token = my_token

# 这里取名随意,一般有意义就行
[tcp_ssh]
# 看官网有这些类型: TCP,UDP,HTTP,HTTPS,STCP,SUDP
# 去这里开始看实例: https://gofrp.org/docs/examples/ssh/
type = tcp
# 本地访问的IP地址
local_ip = 127.0.0.1
# 本地访问的端口
local_port = 22
# 在frps服务器上对应的端口
remote_port = 6600

[tcp_http]
type = tcp
local_ip = 127.0.0.1
local_port = 80
remote_port = 6001

#服务端启动
./frps -c ./frps.toml

#客户端启动
./frpc -c ./frpc.toml
```
