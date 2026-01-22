## 安装nfs

``` bash
#安装nfs
yum -y install nfs-utils rpcbind

#编辑exports文件
vim /etc/exports
/data 192.168.168.0/24(rw,sync,no_root_squash)

#配置生效
exportfs -r

#查看生效
exportfs

#启动rpcbind、nfs服务
systemctl restart rpcbind && systemctl enable rpcbind
systemctl restart nfs-server && systemctl enable nfs-server

#客户端挂载
mount -t nfs 192.168.168.201:/data /haha

#取消挂载
umount /haha

#禁用v4以下版本
sed -i 's/^#vers2=y/^vers2=n/g' /etc/nfs.conf
sed -i 's/^#vers3=y/^vers3=n/g' /etc/nfs.conf
systemctl restart nfs-server
```

## 安装部署Rsync+Inofity

``` bash
#安装rsync、inofity
yum -y install epel-release
yum -y install rsync inotify-tools

#配置rsync
vim /etc/rsyncd.conf
uid = root
gid = root
use chroot = 0
port = 873
#允许ip访问设置，可以指定ip或ip段
hosts allow = 192.168.168.0/24
max connections = 0
timeout = 300
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.lock
log file = /var/log/rsyncd.log
log format = %t %a %m %f %b
transfer logging = yes
syslog facility = local3

[slave_web]
path = /data/k8s
comment = slave_web
ignore errors
#是否允许客户端上传文件
read only = no
list = no
#指定由空格或逗号分隔的用户名列表，只有这些用户才允许连接该模块
auth users = rsync
#保存密码和用户名文件，需要自己生成
secrets file = /etc/rsyncd.passwd

#编辑本节点用户和密码
vim /etc/rsyncd.passwd
rsync:123456

#编辑同步节点密码
vim /opt/rsyncd/rsyncd.passwd
123456

#设置文件权限
chmod 600 /etc/rsyncd.passwd
chmod 600 /opt/rsyncd/rsyncd.passwd

#启动rsyncd
systemctl enable rsyncd && systemctl restart rsyncd

#编写自动同步脚本
cat <<EOF > /opt/rsyncd/rsync_inotify.sh
#!/bin/bash
host=192.168.168.202
src=/data
des=slave_web
password=/opt/rsyncd/rsyncd.passwd
user=rsync
inotifywait=/usr/bin/inotifywait

$inotifywait -mrq --timefmt '%Y%m%d %H:%M' --format '%T %w%f%e' -e modify,delete,create,attrib $src \
| while read files ;do
 rsync -avzP --delete --timeout=100 --password-file=${password} $src $user@$host::$des
 echo "${files} was rsynced" >>/tmp/rsync.log 2>&1
done
EOF

#编写VIP监控脚本
cat <<EOF > /opt/rsyncd/vip_monitor.sh
#!/bin/bash
VIP_NUM=`ip addr|grep 192.168.168.199|wc -l`
RSYNC_INOTIRY_NUM=`ps -ef|grep /usr/bin/inotifywait|grep -v grep|wc -l`
if [ ${VIP_NUM} -ne 0 ];then
   echo "VIP在当前NFS节点服务器上" >/dev/null 2>&1
   if [ ${RSYNC_INOTIRY_NUM} -ne 0 ];then
      echo "rsync_inotify.sh脚本已经在后台执行中" >/dev/null 2>&1
   else
      echo "需要在后台执行rsync_inotify.sh脚本" >/dev/null 2>&1
      nohup sh /opt/rsyncd/rsync_inotify.sh &
  fi
else
   echo "VIP不在当前NFS节点服务器上" >/dev/null 2>&1
   if [ ${RSYNC_INOTIRY_NUM} -ne 0 ];then
      echo "需要关闭后台执行的rsync_inotify.sh脚本" >/dev/null 2>&1
      ps -ef|grep rsync_inotify.sh|grep -v grep|awk '{print $2}'|xargs kill -9
      ps -ef|grep inotifywait|grep -v grep|awk '{print $2}'|xargs kill -9
   else
      echo "rsync_inotify.sh脚本当前未执行" >/dev/null 2>&1
   fi
fi
EOF

#编写持续执行脚本
cat <<EOF > /opt/rsyncd/rsync_monit.sh
#!/bin/bash
while [ "1" = "1" ]
do
  /bin/bash -x /opt/rsyncd/vip_monitor.sh >/dev/null 2>&1
done
EOF

#脚本授权
chmod +x /opt/rsyncd/*.sh

#后台执行脚本
nohup sh /opt/rsyncd/rsync_monit.sh &

#添加开机启动脚本
chmod +x /etc/rc.d/rc.local
echo "nohup sh /opt/rsyncd/rsync_monit.sh & " >> /etc/rc.d/rc.local
```
