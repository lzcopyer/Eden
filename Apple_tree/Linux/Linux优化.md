## 内核参数优化

**临时修改内核参数**
`sysctl -w $NEW_PARAMETER`

**永久修改内核参数**

``` bash
#!/bin/bash
OLD_PARAMETER=""
NEW_PARAMETER=""
sed "/^$OLD_PARAMETER/ {
    a$NEW_PARAMETER"
    s/&/#&/
}" /etc/sysctl.conf

if [ $? != 0 ]; then
    echo "$NEW_PARAMETER" >> /etc/sysctl.conf
fi
```

**刷新参数**

`sysctl -p`

**禁用swap**

``` bash
swapoff -a
sed -i '/swap/ s/^/#/g' /etc/fstab
sysctl -w vm.swappiness=0
```

**文件缓存管理**
`vm.vfs_cache_pressure=50`

**大内存支持**
`vm.nr_hugepages=1024`

### **文件描述符扩容**

**文件句柄数优化**

``` Bash
#   将原有文件底部“nofile”、“nproc”相关内容注释掉，然后将下述内容追加至文件末尾
* soft nofile 655350
* hard nofile 655350
* soft nproc 655350
* hard nproc 655350

# nproc: 操作系统级别对每个用户创建的进程数的限制
# nofile: 每个进程可以打开的文件数的限制

# nofile不能大于“/proc/sys/fs/nr_open”，否则需要先调整“/proc/sys/fs/nr_open”，对应/etc/sysctl.conf中的fs.nr_open参数
```

优化*/etc/sysctl.conf*

``` Bash
# 将下述内容追加至文件末尾
fs.file-max = 1024000

fs.nr_open = 2000000
net.ipv4.ip_local_port_range = 1024 65535
net.core.somaxconn = 32768
net.ipv4.tcp_max_syn_backlog = 16384
net.core.netdev_max_backlog = 16384

net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.optmem_max = 16777216

net.ipv4.tcp_rmem = 1024 4096 16777216
net.ipv4.tcp_wmem = 1024 4096 16777216

net.netfilter.nf_conntrack_max = 1000000
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30

net.ipv4.tcp_max_tw_buckets = 1048576

net.ipv4.tcp_fin_timeout = 5
net.ipv4.tcp_tw_timeout = 5
```


**数据库服务器优化**

``` Bash
cat << EOF > /etc/security/limits.conf
mysql   soft    nofile  655350
mysql   hard    nofile  655350
mysql   soft    memlock unlimited
mysql   hard    memlock unlimited
EOF
```

**Java 应用优化**

``` Bash
# 在启动脚本中添加
ulimit -n 100000
ulimit -l unlimited

# JVM 参数示例
java -XX:+UseLargePages -Xmx16g ...
```

**关闭透明大页**
``` bash
# 临时关闭
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# 永久关闭
vim /etc/default/grub

# 在GRUB_CMDLINE_LINUX_DEFAULT中添加参数
transparent_hugepage=never numa=off

# 更新并重启
update-grub                 
grub2-mkconfig -o /boot/grub2/grub.cfg  
reboot
```

### 关闭 SELinux

---

#### ⚡ 一、临时关闭 SELinux（无需重启）
适用于临时调试或测试场景，重启后恢复原状态。
1. **切换为宽容模式（Permissive）**  
   此模式下 SELinux 记录违规但不拦截操作：
   ```bash
   sudo setenforce 0
   # 或
   sudo setenforce Permissive
   ```
2. **验证状态**  
   ```bash
   getenforce
   ```
   输出应为 `Permissive` 。

---

#### 🔒 二、永久关闭 SELinux（需重启生效）

##### **方法 1：修改配置文件（推荐）**

1. **编辑配置文件**  
   ```bash
   sudo vi /etc/selinux/config
   ```
2. **修改参数**  
   找到 `SELINUX=` 行，修改为：
   ```bash
   SELINUX=disabled  # 完全禁用，不加载策略
   # 或
   SELINUX=permissive # 仅记录违规，不拦截（适合调试）
   ```
     
3. **重启系统**  
   ```bash
   sudo reboot
   ```
4. **验证状态**  
   ```bash
   sestatus  # 输出 "SELinux status: disabled"
   getenforce # 输出 "Disabled"
   ```

##### **方法 2：通过内核参数禁用（适用于无 `/etc/selinux/config` 的场景）**
1. **修改 GRUB 配置** 
	
   ```bash
   sudo grubby --update-kernel ALL --args selinux=0
   ```
     
2. **重启生效**  
	
   ```bash
   sudo reboot
   ```

---

#### ⚠️ 三、注意事项与替代方案
1. **安全风险**  
   - 禁用 SELinux 会显著降低系统安全性，可能暴露于提权攻击或服务漏洞。
   - **生产环境建议**：优先尝试调整策略（如 `audit2allow` 生成规则）而非完全禁用。

2. **常见问题**  
   - **配置文件路径差异**：  
     - CentOS/RHEL 6~7：`/etc/sysconfig/selinux`（符号链接到 `/etc/selinux/config`）。
     - CentOS 8+/RHEL 8+：直接修改 `/etc/selinux/config` 。
   - **状态验证命令**：  
     ```bash
     sestatus  # 查看详细状态
     getenforce # 快速检查（Enforcing/Permissive/Disabled）
     ```

3. **替代方案**  
   - **保持宽容模式**：  
     ```Bash
     SELINUX=permissive  # 记录日志但不拦截，便于调试
     ```
   - **针对性调整策略**：  
     - 使用 `semanage` 修改文件上下文：  
       ```Bash
       sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/html(/.*)?"
       ```
     - 用 `setsebool` 调整布尔值：
       ```Bash
       sudo setsebool -P httpd_can_network_connect on
       ```

## 网络优化

**TCP缓冲区调优**

``` bash
net.ipv4.tcp_rmem = 8192 16777216 33554432  # 接收缓冲区
net.ipv4.tcp_wmem = 8192 65536 33554432     # 发送缓冲区
```

## 安全优化

**内核级防护**

``` bash
kernel.kptr_restrict=1      # 隐藏内核地址信息
net.ipv4.icmp_echo_ignore_broadcasts=1  # 防Ping洪水
```

**修改用户登陆和操作历史记录**

``` bash
cat << EOF > /etc/profile
HISTSIZE=5000 
export HISTTIMEFORMAT="%F %T " 
user=`whoami` 
ip=`who -u am i | awk '{print $NF}' | sed 's/[()]//g'` 
dt=`who -u am i | awk '{print $3" "$4}'` date=`date "+%Y-%m-%d"` user_date=/tmp/.history/$user/$date history_file=$user_date/${user}_history_$date.txt login_file=$user_date/${user}_login_$date.txt 
mkdir -p $user_date 
echo "$user\t$dt\t$ip\n" >> $login_file 
chmod 600 $login_file 
touch $history_file 
export HISTFILE="$history_file" 
chmod 600 $history_file
EOF

source /etc/profile

# 下次登陆即可以在/tmp/.history目录下看到历史登陆记录
```