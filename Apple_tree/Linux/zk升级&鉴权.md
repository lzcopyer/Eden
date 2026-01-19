## zookeeper升级脚本

``` bash
#!/bin/bash

# 设置环境变量
ZK_HOME=/data/zookeeper
BACKUP_DIR=/data/zkbackup
serverip="10.252.33.11 10.252.33.12"
ZK_VERSION=3.6.3

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份数据和配置文件
mv $ZK_HOME/bin $BACKUP_DIR/bin_$(date +%Y%m%d%H%M%S)

tar -xvf apache-zookeeper-$ZK_VERSION-bin.tar.gz -C /data/
mv /data/apache-zookeeper-$ZK_VERSION-bin/bin $ZK_HOME/

# 更新配置文件
#diff $ZK_HOME/conf/zoo.cfg $ZK_HOME-$ZK_VERSION/conf/zoo.cfg
#if [ $? -ne 0 ]
#then
#    cp $ZK_HOME/conf/zoo.cfg $ZK_HOME-$ZK_VERSION/conf/zoo.cfg
#fi

# 逐个重启节点
for ip in serverip
do
    echo "Restarting zookeeper on $ip..."
    ssh $ip "$ZK_HOME/bin/zkServer.sh stop"
    sleep 5
    scp -r $ZK_HOME/bin $ip:$ZK_HOME/bin
    ssh $ip "$ZK_HOME/bin/zkServer.sh start"
done
```

## zookeeper鉴权

>Zookeeper的五种ACL

| 权限说明              | 权限简称      | 权限详情|
| --------------------- | ------------- | ---------------------------------------------------- |
| 创建权限              | create(c)     | 可以在当前znode下创建子znode                         |
| 删除权限              | delete(d)     | 删除当前的znode                                      |
| 读权限                | read(r)       | 获取当前znode的数据，可以列出当前znode所有的子znodes |
| 写权限                | write(w)      | 向当前znode写数据，写入子znode                       |
| 管理权限              | admin(a)      | 设置当前znode的权限                                  |

|方案 |说明 |  
|--- |--- |
|word|只有一个用户：anyone，代表所有人（默认）|  
|ip|使用IP地址认证|
|auth|使用已添加认证的用户认证|
|digest|使用“用户名:密码”方式认证|

### zookeeper登录客户端方式鉴权

>前期准备，需要root登录ZooKeeper客户端的服务器，需要zookeeper管理用户的信息和系统域名信息(例如<userA@HADOOP.COM>)

``` bash
#使用root进入zookeeper文件加
cd /data/zookeeper
source bigdata_env
kinit userA
#查看znode
ls
#查看znode ACL权限
getAcl /znodename
#auth方式，添加用户
addauth digest yoonper:123456
#为用户增加znode ACL权限
setAcl /znodename auth:yoonper:cdrwa
#Digest方式
echo -n yoonper:123456 | openssl dgst -binary -sha1 | openssl base64
setAcl / digest:yoonper:UvJWhBril5yzpEiA2eV7bwwhfLs=:cdrwa
#用户鉴权之后认证
addauth digest yoonper:123456

#ip鉴权
bin/zkCli.sh -server 127.0.0.1:2181
setAcl / ip:192.168.1.112:cdrwa,ip:192.168.1.113:cdrwa,ip:127.0.0.1:cdrwa 
```

## 关闭zookeeper webUI

zookeeper webUI端口为8080，可能会暴露jetty版本过低漏洞，建议禁用zookeeper管理服务。

``` bash
echo "admin.enableServer=false" >> zoo.cfg
```

## zookeeper Kerberos 鉴权认证

创建用户并生成keytab鉴权文件

``` bash
#服务端
kadmin.local -q "addprinc -randkey zookeeper/hadoop-node1@HADOOP.COM"
kadmin.local -q "addprinc -randkey zookeeper/hadoop-node2@HADOOP.COM"
kadmin.local -q "addprinc -randkey zookeeper/hadoop-node3@HADOOP.COM"

# 导出keytab文件
kadmin.local -q "xst  -k /root/zookeeper.keytab  zookeeper/hadoop-node1@HADOOP.COM"
# 先定义其它名字，当使用之前得改回zookeeper-server.keytab
kadmin.local -q "xst  -k /root/zookeeper-node2.keytab  zookeeper/hadoop-node2@HADOOP.COM"
kadmin.local -q "xst  -k /root/zookeeper-node3.keytab  zookeeper/hadoop-node3@HADOOP.COM"

#客户端
kadmin.local -q "addprinc -randkey zkcli@HADOOP.COM"

# 导出keytab文件
kadmin.local -q "xst  -k /root/zkcli.keytab  zkcli@HADOOP.COM"
```

修改配置zoo.cfg

``` bash
$ cd $ZOOKEEPER_HOME
# 将上面生成的keytab 放到zk目录下
$ mkdir conf/kerberos
$ mv /root/zookeeper.keytab /root/zookeeper-ndoe2.keytab /root/zookeeper-node3.keytab /root/zkcli.keytab conf/kerberos/

$ vi conf/zoo.cfg

# 在conf/zoo-kerberos.cfg配置文件中添加如下内容：
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
jaasLoginRenew=3600000
#将principal对应的主机名去掉，防止hbase等服务访问zookeeper时报错，如GSS initiate failed时就有可能是该项没配置
kerberos.removeHostFromPrincipal=true
kerberos.removeRealmFromPrincipal=true
```

配置jaas.conf

``` bash
$ cat >  $ZOOKEEPER_HOME/conf/kerberos/jaas.conf <<EOF
Server {
 com.sun.security.auth.module.Krb5LoginModule required
 useKeyTab=true
 keyTab="/opt/bigdata/hadoop/server/apache-zookeeper-3.8.0-bin/conf/kerberos/zookeeper.keytab"
 storeKey=true
 useTicketCache=false
 principal="zookeeper/hadoop-node1@HADOOP.COM";
}; 
Client {
 com.sun.security.auth.module.Krb5LoginModule required
 useKeyTab=true
 keyTab="/opt/bigdata/hadoop/server/apache-zookeeper-3.8.0-bin/conf/kerberos/zkcli.keytab"
 storeKey=true
 useTicketCache=false
 principal="zkcli@HADOOP.COM";
};
EOF
```

`JAAS`配置文件定义用于身份验证的属性，如服务主体和 keytab 文件的位置等。其中的属性意义如下：  

`useKeyTab`：这个布尔属性定义了我们是否应该使用一个keytab文件（在这种情况下是true）。  
`keyTab`：JAAS配置文件的此部分用于主体的keytab文件的位置和名称。路径应该用双引号括起来。
`storeKey`：这个布尔属性允许密钥存储在用户的私人凭证中。  
`useTicketCache`：该布尔属性允许从票证缓存中获取票证。  
`debug`：此布尔属性将输出调试消息，以帮助进行疑难解答。  
`principal`：要使用的服务主体的名称。  

配置java.env

``` bash
$ cat >  $ZOOKEEPER_HOME/conf/java.env <<EOF
export JVMFLAGS="-Djava.security.auth.login.config=/opt/bigdata/hadoop/server/apache-zookeeper-3.8.0-bin/conf/kerberos/jaas.conf"
EOF
```

将配置copy到其它节点

``` bash
# copy kerberos认证文件
$ scp -r $ZOOKEEPER_HOME/conf/kerberos hadoop-node2:/$ZOOKEEPER_HOME/conf/
$ scp -r $ZOOKEEPER_HOME/conf/kerberos hadoop-node3:/$ZOOKEEPER_HOME/conf/

# copy zoo.cfg
$ scp $ZOOKEEPER_HOME/conf/zoo.cfg hadoop-node2:/$ZOOKEEPER_HOME/conf/
$ scp $ZOOKEEPER_HOME/conf/zoo.cfg hadoop-node3:/$ZOOKEEPER_HOME/conf/

# copy java.env
$ scp $ZOOKEEPER_HOME/conf/java.env hadoop-node2:/$ZOOKEEPER_HOME/conf/
$ scp $ZOOKEEPER_HOME/conf/java.env hadoop-node3:/$ZOOKEEPER_HOME/conf/
```

重启zk

``` bash
$ cd $ZOOKEEPER_HOME
$ ./bin/zkServer.sh  start
# 查看状态
$ ./bin/zkServer.sh  status
```
