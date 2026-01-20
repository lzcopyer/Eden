
# 安装依赖

`yum install -y libaio`  

# 准备配置文件

``` bash
cp /usr/local/mariadb/support-files/my-medium.cnf ${mariadbhome/base}/conf/my.cnf
# 编辑配置文件/etc/my.cnf，在[mysqld]块中添加basedir全局目录将默认的数据目录，日志目录，pid文件都放置在basedir目录下，配置如下：
[mysqld]
port        = 3306
socket      = ${mariadbhome}/tmp/mysql.sock
user    = ${user}
basedir = ${mariadbhome}/base
datadir = ${mariadbhome}/data
log_error = ${mariadbhome}/log/mariadb.err
pid-file = ${mariadbhome}/tmp/mariadb.pid
```

# 初始化数据库  

`${mariadbhome/base}scripts/mysql_install_db --defaults-file={mariadbhome/base}/conf/my.cnf`

# 配置service文件

``` bash
cp /usr/local/mariadb/support-files/mysql.server /etc/init.d/mariadb
chmod +x /etc/init.d/mariadb
chkconfig --add ${user}
chkconfig ${user} on
```

> [!tip] 提示
>初始化之后第一次登录不需要输入密码，`mariadb/bin/mysql -uroot -p` 然后直接回车。

# mariadb双主配置

在`${mariadbhome/base}/conf/my.cnf`中添加以下选项：

``` bash
server-id  = 1
log-bin=mysql-bin
relay-log = mysql-relay-bin
replicate-wild-ignore-table=mysql.%
replicate-wild-ignore-table=information_schema.%
log-slave-updates=on
slave-skip-errors=all
auto-increment-offset=1
auto-increment-increment=2
binlog_format=mixed
expire_logs_days = 10
```

`server-id = 1`服务器节点ID，必须唯一。

`log-bin=mysql-bin`开启二进制日志功能，mysql-bin是命名格式，会生成文件名为mysql-bin.000001、mysql-bin.000002等日志文件。

`relay-log = mysql-relay-bin`作为从服务器时的中继日志。

`replicate-wild-ignore-table=mysql.%`是复制过滤选项，可以过滤不需要复制同步的数据库或表，如“mysql.%”指不复制MySQL库下的所有表，依此类推，多个数据库就多写几行。注意不建议在主库或从库上使用`binlog-do-db`或`binlog-ignore-db`，也不要再从库上使用`replicate-do-db`或`replicate-do-db`选项，因为这样可能会产生跨库更新失败的问题。我们推荐使用`replicate-wild-do-table`和`replicate-wild-ignore-table`解决复制过滤问题。

`log-slave-updates=onslave` 将复制事件写进自己的二进制日志。

`slave-skip-errors=all`跳过主从复制中遇到的所有错误或指定类型的错误,避免 slave 端复制中断。

`auto-increment-offset=1`主键自增规则，自增因子（每次加2）。

`auto-increment-increment=2`主键自增规则，自增偏移（从1开始），单数。

`binlog_format=mixed`主从复制的格式(mixed,statement,row,默认格式是 statement)。

`expire_logs_days = 10`日志保存天数：10天。

_修改数据库配置之后重启_

## 创建备份用户

``` bash
MariaDB [(none)]> grant replication slave, replication client on *.* to 'hello32'@'192.168.11.32' identified by '123456x';
mysql> flush privileges;
```

## 检查_binlog_文件

接着马上查看binlog文件的position（偏移）和File（日志文件）的值:

``` bash
MariaDB [(none)]> show master status;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000007 |      358 |              |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.01 sec)
```

## 在从库进行同步

``` bash
MariaDB [(none)]> change master to master_host='192.168.11.32',master_user='hello31', master_password='123456x', master_port=3306, master_log_file='mysql-bin.000007', master_log_pos=358, master_connect_retry=30;
```
