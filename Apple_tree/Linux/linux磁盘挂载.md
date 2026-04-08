# Linux LVM 磁盘管理实战指南

本手册涵盖了从新硬盘挂载、动态扩容到风险缩容的全流程操作。

## 一、 初始化挂载（LVM 自动配置脚本）

**场景**：系统新增一块硬盘，需要将其划分为 `database`、`backup` 和 `archive` 三个逻辑卷。

### 脚本实现
```bash
#!/bin/bash
# 1. 自动获取未挂载且容量包含 "T" 的最大磁盘名称 (例如 /dev/sdb)
target_disk=$(lsblk -o NAME,MOUNTPOINT,SIZE -n -p -l | awk '$2 == "" && $3 ~ /T/ {print $1}' | sort -k3,3nr | head -n 1)

if [ -z "$target_disk" ]; then
    echo "未找到符合要求的空闲磁盘！"
    exit 1
fi

# 2. 创建物理卷与卷组
pvcreate -y "$target_disk"
vgcreate -y datavg "$target_disk"

# 3. 创建逻辑卷 (分配比例：34% 数据库, 33% 备份, 其余全部给归档)
lvcreate -y -n database -l 34%FREE datavg
lvcreate -y -n backup -l 50%FREE datavg  # 基于剩余空间的 50%
lvcreate -y -n archive -l 100%FREE datavg

# 4. 创建挂载点与格式化
mkdir -p /{backup,database,archive}
mkfs.xfs /dev/datavg/backup
mkfs.xfs /dev/datavg/database
mkfs.xfs /dev/datavg/archive

# 5. 配置持久化挂载 (使用 UUID 避免路径漂移)
for part in database backup archive; do
    uuid=$(blkid -s UUID -o value /dev/datavg/$part)
    echo "UUID=$uuid /$part xfs defaults 0 0" >> /etc/fstab
done

# 6. 挂载测试
mount -a
df -h | grep datavg
```

---

## 二、 磁盘扩容（支持 MBR 与 GPT 分区）

**场景**：底层磁盘空间增加，需要扩展逻辑卷及文件系统。根据磁盘大小（是否大于 2TB），分区工具的选择会有所不同。

### 步骤 1：准备新分区/物理卷

**场景 A：使用 fdisk (适用于 <2TB 的 MBR 磁盘)**
```bash
fdisk /dev/sdb  
# 交互输入: n (新分区) -> p (主分区) -> 回车 -> 回车 -> w (保存)
partprobe /dev/sdb
pvcreate /dev/sdb1
```

**场景 B：使用 parted (适用于 >2TB 的 GPT 磁盘)【补充场景】**
```bash
# 使用新磁盘
parted /dev/sdb
# 进入交互模式
(parted) mklabel gpt             # 如果是全新磁盘，先创建 GPT 分区表 (警告: 会清除数据)
(parted) mkpart primary 0% 100%  # 将整盘 100% 空间划分为一个主分区
(parted) print                   # 查看分区确认
(parted) quit                    # 退出
# 在原磁盘上对分区扩容
parted /dev/sdb
# 进入交互模式
(parted) unit s print free                                                
Model: NVMe Device (nvme)
Disk /dev/sdb: 8001573552s
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start        End          Size         File system  Name                  标志
        34s          2047s        2014s        Free Space
 1      2048s        411647s      409600s      fat16        EFI System Partition  启动
 2      411648s      2508799s     2097152s     xfs
 3      2508800s     3907028991s  3904520192s                                     lvm
(parted) mkpart primary 3907028991s 100%  # 将整盘 100% 空间划分为一个主分区
(parted) print                   # 查看分区确认
(parted) quit                    # 退出
# 刷新并创建物理卷
partprobe /dev/sdb
pvcreate /dev/sdb1
```

### 步骤 2：扩展卷组 (VG) 与逻辑卷 (LV)
```bash
vgextend datavg /dev/sdb1

# 扩展指定容量
lvextend -L +10G /dev/datavg/database
# 或将所有剩余空间分配
lvextend -l +100%FREE /dev/datavg/database
```

### 步骤 3：刷新文件系统
```bash
# 如果是 XFS 文件系统 (挂载点)
xfs_growfs /database         

# 如果是 Ext4 文件系统 (设备路径)
resize2fs /dev/datavg/database 
```

---

## 三、 磁盘缩容（高风险操作）

**🚨 重要警告：**
1. **XFS 文件系统原生不支持缩容！** 必须通过备份重建来实现。
2. **Ext4 缩容必须先取消挂载 (umount)**。
3. 任何缩容操作前，**必须备份重要数据**！

### 场景 A：XFS 格式缩容方案（备份与重建）【补充场景】
由于 XFS 无法直接 `resize`，我们需要推倒重来。

```bash
# 1. 备份数据至其他充足的空间 (例如 /tmp 或其他挂载点)
# 可使用 xfsdump 或简单的 rsync/tar
xfsdump -f /tmp/backup.dump /backup  

# 2. 取消挂载
umount /backup

# 3. 移除原有的逻辑卷 (释放空间)
lvremove /dev/datavg/backup

# 4. 重新创建较小的逻辑卷 (例如缩减到 50G)
lvcreate -L 50G -n backup datavg

# 5. 重新格式化为 XFS
mkfs.xfs /dev/datavg/backup

# 6. 重新挂载
mount -a

# 7. 恢复数据
xfsrestore -f /tmp/backup.dump /backup
```

### 场景 B：Ext4 格式直接缩容
```bash
# 1. 取消挂载
umount /backup

# 2. 强制磁盘检测 (必做)
e2fsck -f /dev/datavg/backup

# 3. 缩减文件系统 (缩减到 50G)
resize2fs /dev/datavg/backup 50G

# 4. 缩减逻辑卷
lvreduce -L 50G /dev/datavg/backup

# 5. 重新挂载
mount -a
```

---

## 四、 磁盘自动扩容脚本（针对根分区/var）

此脚本适用于虚拟机磁盘整体增大后，自动识别并扩展到指定逻辑卷。

```bash
#!/bin/bash
# 寻找根分区所在的物理磁盘 (例如 /dev/sda)
devname=$(lsblk -no PKNAME $(findmnt -nvo SOURCE /) | head -n 1)
# 获取当前分区数量并加 1，作为新分区编号
num=$(lsblk -n "$devname" | wc -l)
vgname=$(vgs --noheadings -o vg_name | tr -d ' ')
addname="/var"
target_lv=$(findmnt -nvo SOURCE "$addname")

# 1. 自动在空闲空间创建新分区 (默认 MBR 模式)
echo -e "n\np\n$num\n\n\nw" | fdisk "$devname"
partprobe "$devname"

# 2. 加入 LVM 链条
pvcreate "${devname}${num}"
vgextend "$vgname" "${devname}${num}"

# 3. 扩容 LV 并自动刷新文件系统 (使用 -r 参数自动判断并刷新 XFS/Ext4)
lvextend -r -l +100%FREE "$target_lv"
```

---

## 五、 常用维护指令汇总

| 操作类型 | 命令示例 | 说明 |
| :--- | :--- | :--- |
| **查看状态** | `lsblk`, `lvs`, `vgs`, `pvs` | 简要查看磁盘、LV、VG、PV 状态 |
| **详细查看** | `pvdisplay`, `vgdisplay` | 查看 PE/LE 详细分配情况及空闲空间 |
| **移除 VG** | `vgreduce datavg /dev/sdb1` | 将故障或多余的物理卷从卷组移除 |
| **强制清理** | `dmsetup remove /dev/mapper/datavg-data` | 当逻辑卷被占用无法正常删除时的强制解除手段 |
| **分区重写** | `parted /dev/sdb mklabel gpt` | 彻底重置磁盘分区表 (会导致该磁盘数据全丢) |
