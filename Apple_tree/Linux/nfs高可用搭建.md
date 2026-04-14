# NFS + Rsync + Inotify 部署文档

## 一、NFS 服务部署

### 1. 安装依赖

```bash
yum -y install nfs-utils rpcbind
```

### 2. 配置共享目录

编辑配置文件：

```bash
vim /etc/exports
```

示例：

```bash
/data 192.168.168.0/24(rw,sync,no_root_squash)
```

参数说明：

| 参数             | 含义                          |
| -------------- | --------------------------- |
| rw             | 读写权限                        |
| sync           | 同步写入（安全，性能略低）               |
| no_root_squash | root 用户保持 root 权限（生产环境谨慎使用） |

### 3. 使配置生效

```bash
exportfs -r
exportfs -v
```

### 4. 启动服务

```bash
systemctl enable rpcbind --now
systemctl enable nfs-server --now
```

### 5. 客户端挂载

```bash
mount -t nfs 192.168.168.201:/data /haha
```

建议写入 `/etc/fstab` 实现持久挂载：

```bash
192.168.168.201:/data  /haha  nfs  defaults,_netdev  0  0
```

### 6. 卸载

```bash
umount /haha
```

### 7. 禁用 NFSv2 / NFSv3（仅使用 NFSv4）

> 原文 sed 写法错误，已修正

```bash
sed -i 's/^#vers2=.*/vers2=n/' /etc/nfs.conf
sed -i 's/^#vers3=.*/vers3=n/' /etc/nfs.conf

systemctl restart nfs-server
```

---

## 二、Rsync + Inotify 实时同步

## 1. 安装依赖

```bash
yum -y install epel-release
yum -y install rsync inotify-tools
```

---

## 2. Rsync 服务端配置

编辑配置文件：

```bash
vim /etc/rsyncd.conf
```

```ini
uid = root
gid = root
use chroot = no
port = 873

hosts allow = 192.168.168.0/24

max connections = 0
timeout = 300

pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log

[slave_web]
path = /data/k8s
comment = data sync
ignore errors
read only = no
list = no
auth users = rsync
secrets file = /etc/rsyncd.passwd
```

---

## 3. 配置认证文件

### 服务端：

```bash
vim /etc/rsyncd.passwd
```

```bash
rsync:123456
```

### 客户端：

```bash
mkdir -p /opt/rsyncd
vim /opt/rsyncd/rsyncd.passwd
```

```bash
123456
```

### 权限设置

```bash
chmod 600 /etc/rsyncd.passwd
chmod 600 /opt/rsyncd/rsyncd.passwd
```

---

## 4. 启动 Rsync 服务

```bash
systemctl enable rsyncd --now
```

---

## 5. Inotify 实时同步脚本

```bash
cat <<EOF > /opt/rsyncd/rsync_inotify.sh
#!/bin/bash

host=192.168.168.202
src=/data
des=slave_web
user=rsync
password=/opt/rsyncd/rsyncd.passwd

inotifywait=/usr/bin/inotifywait

$inotifywait -mrq \
--timefmt '%Y-%m-%d %H:%M' \
--format '%T %w%f %e' \
-e modify,delete,create,attrib \
\$src | while read file
do
  rsync -az --delete \
  --timeout=100 \
  --password-file=\${password} \
  \$src \${user}@\$host::\$des

  echo "\$file synced" >> /var/log/rsync_inotify.log 2>&1
done
EOF
```

---

## 6. VIP 漂移监控脚本（主备切换控制）

```bash
cat <<EOF > /opt/rsyncd/vip_monitor.sh
#!/bin/bash

VIP=192.168.168.199

VIP_EXIST=$(ip addr | grep -c \$VIP)
INOTIFY_EXIST=$(pgrep -f inotifywait | wc -l)

if [ \$VIP_EXIST -ne 0 ]; then
    if [ \$INOTIFY_EXIST -eq 0 ]; then
        nohup bash /opt/rsyncd/rsync_inotify.sh >/dev/null 2>&1 &
    fi
else
    if [ \$INOTIFY_EXIST -ne 0 ]; then
        pkill -f rsync_inotify.sh
        pkill -f inotifywait
    fi
fi
EOF
```

---

## 7. 持续监控脚本

> 原脚本存在 CPU 空转问题，已修复（增加 sleep）

```bash
cat <<EOF > /opt/rsyncd/rsync_monit.sh
#!/bin/bash

while true
do
  bash /opt/rsyncd/vip_monitor.sh
  sleep 5
done
EOF
```

---

## 8. 启动与开机自启

```bash
chmod +x /opt/rsyncd/*.sh

nohup bash /opt/rsyncd/rsync_monit.sh >/dev/null 2>&1 &
```

加入开机启动：

```bash
chmod +x /etc/rc.d/rc.local

echo "nohup bash /opt/rsyncd/rsync_monit.sh >/dev/null 2>&1 &" >> /etc/rc.d/rc.local
```

---

# 三、关键优化与注意事项

## 1. 安全建议

* 避免使用 `no_root_squash`（除非明确需要）
* rsync 建议结合防火墙限制访问
* 密码文件必须为 600 权限

---

## 2. 性能建议

| 项目      | 建议                |
| ------- | ----------------- |
| rsync   | 大量小文件建议关闭 `-z` 压缩 |
| inotify | 需调大系统监听限制         |
| NFS     | 高并发建议使用 NFSv4     |

调整 inotify 限制：

```bash
echo "fs.inotify.max_user_watches=1048576" >> /etc/sysctl.conf
sysctl -p
```

---

## 3. 常见问题

| 问题       | 原因             |
| -------- | -------------- |
| 同步延迟     | inotify 队列不足   |
| 权限异常     | root squash 配置 |
| rsync 失败 | 密码文件权限错误       |
