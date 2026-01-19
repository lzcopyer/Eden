> vsftpd 是“very secure FTP daemon”的缩写，安全性是它的一个最大的特点。vsftpd 是一个 UNIX 类操作系统上运行的服务器的名字，它可以运行在诸如 Linux、BSD、Solaris、 HP-UNIX等系统上面，是一个完全免费的、开放源代码的ftp服务器软件，支持很多其他的 FTP 服务器所不支持的特征。比如：非常高的安全性需求、带宽限制、良好的可伸缩性、可创建虚拟用户、支持IPv6、速率高等。

# 安装

## 包管理器安装

**CentOS:**
``` Bash
yum -y install vsftpd
```

`Ubuntu/Debian：`

``` Bash
apt-get install vsftpd
```

## 编译安装

**1、下载源码：**

``` Bash
wget https://security.appspot.com/downloads/vsftpd-3.0.2.tar.gz
```

**2、编译vsftpd源码：**

``` Bash
# 64位操作系统需执行
cp /lib64/libcap.so.1 /lib/libcap.so.1

tar xzvf vsftpd-3.0.2.tar.gz
cd vsftpd-2.3.4
./configure
make
make install --perfix=$vsftp_home

# 编译安装的默认路径：/usr/local/sbin/vsftpd
# 默认配置文件路径/etc/vsftpd/vsftpd.conf或/usr/local/etc/vsftpd.conf
```

**3、配置systemd：**

``` Bash
cat << EOF > /etc/systemd/system/vsftpd.service
[Unit]
Description=vsftpd FTP server
After=network.target

[Service]
Type=forking
# ！！注意：请将这里的路径修改为你的实际路径！！
ExecStart=$vsftp_home /etc/vsftpd.conf

# 推荐使用 /run/vsftpd/vsftpd.pid
PIDFile=/run/vsftpd/vsftpd.pid
RuntimeDirectory=vsftpd
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 重新加载systemd配置
systemctl daemon-reload
```

## 启动管理

``` Bash
# 启动
systemctl start vsftpd
# 设置开机启动
systemctl enable vsftpd
# 停止
systemctl stop vsftpd
```

# 配置

## 配置文件说明

**主要配置文件：**

- /etc/vsftpd/vsftpd.conf    Vsftpd服务器主配置文件
- /etc/vsftpd/ftpusers         禁止访问Vsftp服务器的用户名单
- /etc/vsftpd/user_list         指定用户能否访问FTP服务器取决于userlist_deny选项的设置
- /etc/vsftpd/chroot_list     目录访问控制文件

**vsftopd.conf**

``` TOML
anonymous_enable=NO      #禁用匿名登录。
local_enable=YES         #允许本地登录。
write_enable=YES         #启用更改文件系统的FTP命令。
local_umask=022          #本地用户创建文件的权限值。
dirmessage_enable=YES    #在用户首次进入新目录时启用消息显示。
xferlog_enable=YES       #把操作记录写入到日志文件。
connect_from_port_20=YES #使用 20 端口传输。
xferlog_std_format=YES   #保持标准日志文件格式。
listen=NO                #防止 vsftpd 在独立模式下运行。
listen_ipv6=YES          #使用 IPv6，如果没有 IPv6 可以设置为 `NO` 。
pam_service_name=vsftpd  #PAM 服务的名称。
userlist_enable=YES      #启用 vsftpd 加载用户名列表。
#userlist_deny=YES       #启用 vsftpd 禁用用户名列表。
userlistfile=/etc/vsftpd/user_list #用户列表路径
tcp_wrappers=YES         #使用 tcp_wrappers 限制访问。
chroot_local_user=YES    #禁止用户离开自己的主目录
chroot_list_enable=YES   #是否开启目录权限控制
chroot_list_file=/etc/vsftpd/file_list  #授权路径列表
```

**user_list**

``` Text
# 禁止/允许访问ftp的用户
toto
```

**file_list**

``` Text
# 允许访问的路径
/data
```

## 用户配置

vsftpd 的用户可以分为：匿名用户、系统用户、虚拟用户。

### 匿名用户

匿名用户就是不需要用户名和密码，在启动 vsftpd 服务后直接使用 IP + 端口号 就能登录。匿名用户比较适合用于公开的文件分享，只需要在浏览器或资源管理器的地址栏输入：`IP:21` 就能直接访问文件。

匿名用户的文件目录是 `/var/ftp/pub` ，在默认情况下匿名用户只能访问和下载文件，不能创建和修改文件。

如果要允许匿名用户上传文件可以在 vsftpd 的配置文件，也就是 `/etc/vsftpd/vsftpd.conf` 设置 `anon_upload_enable=YES` 如果前面有 `#` 的就删除 `#` 。

如果要禁止匿名用户登录可以在配置文件中设置 `anonymous_enable=NO` 。

### 系统用户

系统用户就是直接使用 Linux 的用户登录，用户名和密码就是 Linux 的用户名和密码。

创建用户：

``` Bash
useradd userName
```

其中的 `userName` 就是用户名。

给新创建的用户设置密码：

``` Bash
passwd userName
```

其中的 `userName` 就是要设置密码的用户，回车后需要输入两次密码。

用 VI 之类的编辑器打开 `/etc/vsftpd/user_list` ：

``` Bash
vi /etc/vsftpd/user_list
```

把创建的新用户的用户名添加到最后一行，然后保存。

现在就可以使用刚创建的用户名和密码登录 FTP 服务器了。登录后的 FTP 目录就是当前用户的目录，比如我创建了一个名为 `mark` 的目录，那我的目录就是 `/home/mark` 。

### 虚拟用户

虚拟用户就是在 vsftpd 中配置用户，然后把配置的 vsftpd 用户映射到 Linux 的系统用户。

下面创建一个 www 用户作为虚拟用户的宿主用户：

``` Bash
useradd www
```

打开 vsftpd 配置文件 `/etc/vsftpd/vsftpd.conf`，添加下面几行配置：

``` Bash
guest_enable=YES
guest_username=www
virtual_use_local_privs=YES
user_config_dir=/etc/vsftpd/vu
```

上面的配置信息在默认的 vsftpd 配置文件中是没有的，需要手动添加。

下面是配置信息说明：

- `guest_enable`：启用虚拟用户功能。
- `guest_username`：设置虚拟用户的宿主用户，上面设置的就是刚创建的 www 用户。
- `virtual_use_local_privs`：设置虚拟用户的权限和宿主用户一致。
- `user_config_dir`：存放虚拟用户的配置文件的位置，上面设置的是 `/etc/vsftpd/vu`。

启用虚拟用户，`listen` 也需要设置为 `YES`，否则启动 FTP 服务时可能会出现错误。

接下来就是配置虚拟用户的用户名和密码，我下面会创建一个 `/home/vu-cfg` 来存放虚拟用户的用户名和密码：

``` Bash
touch /home/vu-cfg
```

这个存放虚拟用户的文件对命名和位置没有要求，只要你能找到就可以。

用 vi 之类的编辑器打开 `/home/vu-cfg` 添加用户名和密码：

``` Text
user1
123456
user2
123456
```

用户名和密码的配置文件格式是一行用户名、一行密码，基数行为用户名、偶数行为密码。上面配置了两个虚拟用户分别是 `user1` 和 `user2`，两个用户的密码都是一样的。最后需要留一行空行，否则生成数据文件的时候可能会报错。

生成用户数据文件：

``` Bash
db_load -T -t hash -f /home/vu-cfg /etc/vsftpd/vu-cfg.db
```

上面使用 `/home/vu-cfg` 生成了一个数据文件，数据文件存放在 `/etc/vsftpd/vu-cfg.db`。数据文件需要用 `db` 后缀。

以后如果需要增加或删除虚拟用户，可以更改 `/home/vu-cfg`，更改完成后也需要重新生成数据文件。

配置 `pam` 文件，`vsftpd` 的 `pam` 文件默认在 `/etc/pam.d/vsftpd`。

在配置之前可以先备份一下：

``` Bash
cp /etc/pam.d/vsftpd /etc/pam.d/vsftpd-backup
```

用 `vi` 之类的编辑器打开 `/etc/pam.d/vsftpd`，删除默认的内容，加入下面的内容：

``` Bash
#%PAM-1.0
auth sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vu-cfg
account sufficient /lib64/security/pam_userdb.so db=/etc/vsftpd/vu-cfg
```

其中的 `db=/etc/vsftpd/vu-cfg` 中的 `/etc/vsftpd/vu-cfg` 就是数据文件，这里的数据文件不需要加 `db` 后缀。

接下来配置单独的虚拟用户，在 `vsftpd.conf` 中设置的虚拟用户配置文件的存放位置是 `/etc/vsftpd/vu`。

如果设置的目录不存在的话可以创建一个：

``` Bash
mkdir /etc/vsftpd/vu
```

创建虚拟用户配置文件：

``` Bash
touch /etc/vsftpd/vu/user1
```

虚拟用户配置文件的文件名需要和虚拟用户名相同，例如上面配置的虚拟用户名是 `user1` 和 `user2` ，虚拟用户的配置文件的文件名也需要是 `user1` 和 `user2`。

在虚拟用户的配置文件中加入：

``` Bash
local_root=/home/www/user1
```

上面配置 `user1` 用户的目录为 `/home/www/user1`。

不同的虚拟用户可以配置不同的目录，虚拟用户的目录需要在宿主用户目录之下。例如上面设置的宿主用户是 `www`，宿主用户目录是 `home/www`，虚拟用户目录就需要在 `home/www` 之下。

虚拟用户目录不是必须的，如果不配置的话，虚拟用户目录就是宿主用户目录。

## 主被动模式切换

vsftp默认是被动模式  
在`/etc/vsftpd/vsftpd.conf`中添加`pasv_enable=NO`参数可以切换为主动模式  

主动模式配置  

``` bash
pasv_enable=NO
connect_from_port_20=YES
```

被动模式配置  

``` bash
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=30999
```

# 客户端使用教程

## 连接vsftp服务器

``` Bash
ftp hostnanme -P port
```

## 常用命令

``` Bash
# 启动时没有指定地址，可以使用此命令连接服务器
open [hostname]
# 断开ftp服务连接
bye
quit
# 帮助命令
help [command]
# 切换登录用户
user [username]
# 列出目录和文件
ls [path]
dir [path]
# 切换工作目录
cd [path]
# 显示当前目录
pwd
# 创建目录
mkdir [directory_name]
# 删除空目录
rmdir [directory_name]
# 浏览本地机器
! [shell_command]
! ls -l
! pwd
# 切换本地机器上的工作目录（Local Change Directory）。下载的文件将默认保存到这个目录。
lcd [directory]
# 文件下载
get [remote_file] [local_file]
# 批量下载
mget [files]
# 文件上传
put [local_file] [remote_file]
# 批量上传
mput [files]
# 删除vsftp服务端上的文件
delete [remote_file]
# 批量删除
mdelete [files]
# 重命名vsftp服务器上的文件
rename [from_name] [to_name]
```

## 模式设置

- `**binary` 或 `bin**`
    
    - **用法：** 将文件传输模式设置为**二进制模式 (Binary)**。
        
    - **!! 关键 !!** 在传输非纯文本文件（如：图片、Zip压缩包、可执行文件、视频）时，**必须**使用此模式，否则文件会损坏。
        
- `**ascii**`
    
    - **用法：** 将文件传输模式设置为**ASCII模式**。仅适用于传输纯文本文件（.txt, .html, .conf 等）。
        
- `**passive**`
    
    - **用法：** 切换**被动模式 (Passive Mode)**。大多数现代 FTP 连接（尤其是当客户端在防火墙后时）都需要被动模式。如果连接后 `ls` 或 `get` 卡住，尝试输入 `passive`。
        
- `**prompt**`
    
    - **用法：** 切换交互提示。在使用 `mget` 或 `mput` 时，默认会逐个询问您是否要传输。输入 `prompt` 命令可以关闭这个提示，自动传输所有匹配的文件。