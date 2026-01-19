# 磁盘挂载

挂载脚本

``` bash
unmounted_disks=$(lsblk -o NAME,MOUNTPOINT,SIZE -n -p -l | awk '$3 == "" {print $1,$2}' | grep T|sort -k2,2nr|head -n 1|awk '{print $1}')
pvcreate -y $unmounted_disks
vgcreate -y datavg $unmounted_disks
lvcreate -y -n database -l 34%FREE datavg
lvcreate -y -n backup -l 50%FREE datavg
lvcreate -y -n archive -l 100%FREE datavg
mkdir -p /{backup,database,archive}
mkfs.xfs /dev/datavg/backup
mkfs.xfs /dev/datavg/database
mkfs.xfs /dev/datavg/archive
mount  -t xfs /dev/datavg/backup /backup
mount  -t xfs /dev/datavg/database /database
mount  -t xfs /dev/datavg/archive /archive
mount -va
uuid=`blkid /dev/datavg/database -o list | awk 'END { print $NF  }'` && echo "UUID=$uuid /database xfs defaults 0 1" >> /etc/fstab
uuid=`blkid /dev/datavg/backup -o list | awk 'END { print $NF  }'` && echo "UUID=$uuid /backup xfs defaults 0 1" >> /etc/fstab
uuid=`blkid /dev/datavg/archive -o list | awk 'END { print $NF  }'` && echo "UUID=$uuid /archive xfs defaults 0 1" >> /etc/fstab
```

## 磁盘扩容

``` bash
#创建分区
fdisk $devname
n
p
\n
\n
w
#刷新分区并创建物理卷
partprobe $devname
pvcreate $newdevname
#查看卷组名称，以及卷组使用情况
vgdisplay
#将物理卷扩展到卷组
vgextend $vggroup $newdevname
#查看当前逻辑卷的空间状态
lvdisplay
#将卷组中的空闲空间扩展到根分区逻辑卷
lvextend -l +100%FREE $addname
lvextend -L +10G $addname
#刷新分区
xfs_growfs $addname
```

## 磁盘缩容

``` bash
# 取消挂载
umount $disk

# 磁盘检测
e2fsck -f $disk

# 重设大小
resize2fs $disk $size

# 减小磁盘容量
lvresize -l -$size $disk

# 将物理卷从卷组中移除
vgreduce $disk $diskgroup
```

磁盘自动扩容脚本  

``` bash
devname=$(lsblk -o NAME,MOUNTPOINT,SIZE -n -p -l  | grep -w -B1 /boot|head -n 1|awk '{print $1}')
num=$(lsblk -o NAME,MOUNTPOINT,SIZE -n -p -l  | grep $devname|wc -l)
vgname=$(lvs|grep -w root|awk '{print $2}')
addname="/var"
adddisk=$(grep -w "$addname" /etc/fstab|awk '{print $1}')
size=""

echo -e "n\n$num\n\n\nw\n" | fdisk $devname
partprobe $devname
pvcreate $devname$num
vgextend $vgname $devname$num
lvextend -l +$size $adddisk
xfs_growfs $adddisk
```

## 其他问题

删除卷  
`lvremove /dev/datavg/datalv`  
查看`vg`、`pv`卷  
`lvdisplay` `vgdisplay` `fdisk -l`  
删除磁盘分区及重做磁盘分区表  
`fdisk /dev/sda`交互输入`d 1 w`，d代表删除，1代表sda1，w为保存操作。即删除/dev/sda1  
`parted`交互输入`mklabel msdos` `yes` `quit`  
重读分区表信息`partprobe  /dev/sdb`
其他删除`/dev/datavg`方式  
`dmsetup remove /dev/mapper/datavg-data`
