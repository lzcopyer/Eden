### 部署

``` bash
yum -y install gcc openssl openssl-devel libnsl libnl-devel
/data/keepalived/configure --prefix=/usr/local/keepalived
make -C /data/keepalived
make -C /data/keepalived install
mkdir -p /etc/keepalived/script
cp keepalived/etc/init.d/keepalived /etc/init.d/
cp keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived
ln -s /usr/local/keepalived/sbin/keepalived /usr/bin/keepalived
sed -i 's/KillMode=process/#&/' /usr/lib/systemd/system/keepalived.service
sed -i 's/KEEPALIVED_OPTIONS/#&/' /usr/local/keepalived/etc/sysconfig/keepalived
sed '/^#KEEPALIVED_OPTIONS/a \KEEPALIVED_OPTIONS="-f /etc/keepalived/keepalived.conf -D"' /usr/local/keepalived/etc/sysconfig/keepalived

#启停命令
systemctl enable keepalived
systemctl start keepalived
```

### 配置

```  Bash
cat <<EOF > /etc/keepalived/keepalived.conf
global_defs 
{
   script_user root      # 检测脚本运行的用户
   enable_script_security 
}
vrrp_script check 
{
    script "/etc/keepalived/script/check.sh"
    interval 30
    fall 2 # 检测两次失败将实例定义为故障，移除vip
    rise 1 # 检测一次成功后将状态改为正常（设置不抢占后不会抢占vip）
    #weight 10
}
vrrp_instance OPN_PAAS_OUT {
    state BACKUP
    interface eth1
    virtual_router_id 48 # 同一个网络不能和其它vip重复，同一组vip必须相同
    priority 100
    nopreempt # 设置不抢占模式
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1npaas6e-2cdc-48bf-83b2-01a96d1593e4
    }

    track_script {
        check
    }

    virtual_ipaddress {
        10.253.153.248
    }
}
EOF
```

### 编写监控脚本

``` bash
mkdir -p /etc/keepalived/script
cat << EOF > /etc/keepalived/script/check.sh
#!/bin/bash

# --- 脚本配置 ---
max_retries=5     # 最大重试次数
retry_delay=10    # 重试间隔（秒）
current_retries=0 # 当前重试次数

# 命令定义
# 检查 mysqld 进程是否运行。注意：pgrep -x mysqld >/dev/null 返回 0 表示找到，即运行中。
# 在 bash 脚本中，0 是“成功”的意思。
check_mysql_status="pgrep -x mysqld"
mysql_start="systemctl restart mysqld"


# 初始检查：如果 MySQL 已经在运行，直接退出。
if \$check_mysql_status >/dev/null; then
    echo "MySQL is already running."
    exit 0
fi

echo "MySQL process not found. Attempting to start/restart."

# --- 使用 while 循环尝试启动 MySQL ---
# 循环条件：MySQL 进程未找到 且 重试次数未达到上限
# 注意：感叹号 '!' 用来对 check_mysql_status 的返回状态进行取反
while ! \$check_mysql_status >/dev/null && [ "\$current_retries" -lt "\$max_retries" ]; do
    
    echo "Attempt $((current_retries + 1)): Starting mysqld..."
    
    # 尝试执行启动/重启命令
    if \$mysql_start; then
        echo "Service command executed successfully. Waiting \${retry_delay}s for process check..."
        sleep "\$retry_delay"
        
        # 再次检查进程是否运行
        if \$check_mysql_status >/dev/null; then
            echo "MySQL started successfully!"
            exit 0 # 成功，退出脚本
        else
            echo "Service started, but process check failed. Retrying..." >&2
        fi
    else
        echo "Service command failed. Retrying in \${retry_delay}s..." >&2
        sleep "\$retry_delay"
    fi
    
    # 增加重试计数
    current_retries=\$((current_retries + 1))
done


# --- 循环结束后判断结果 ---

# 如果循环是因为达到最大重试次数而终止
if [ "\$current_retries" -eq "\$max_retries" ]; then
    echo "Critical: Failed to start MySQL after \$max_retries attempts." >&2
    exit 1
fi

# 如果循环是因为 MySQL 成功启动而终止（这个分支通常在 while 内部的 exit 0 覆盖，但作为安全措施保留）
if \$check_mysql_status >/dev/null; then
    echo "MySQL started successfully."
    exit 0
fi

# 最后的失败退出（一般由上面的 Critical 捕获）
exit 1
EOF
```

### 编写keepalived监控脚本

``` bash
cat << EOF > /etc/keepalived/script/check_keepalived.sh
#!/bin/bash
keepalived_counter=$(ps -C keepalived --no-heading|wc -l)
if [ "\${keepalived_counter}" = "0" ]; then
    /usr/sbin/keepalived
fi
EOF
echo '* * * * * sleep 10;/etc/keepalived/check_keepalived.sh' >> /var/spool/cron/root
```
