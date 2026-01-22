
# 先决条件

## 基线配置

1. 关闭防火墙 or 设置白名单
2. 配置时间同步
3. 配置_SSH_免密登录
4. 配置主机名映射 or 配置内网_DNS_

## 所需依赖

Linux和Windows所需软件包括:

	1. Java 必须安装，建议选择Sun公司发行的Java版本。
	2. ssh 必须安装并且保证 sshd 一直运行，以便用 Hadoop 脚本管理远端Hadoop守护进程。

## 依赖安装

``` Bash
apt-get install ssh rsync
```

# 伪分布式部署

## 准备凭证

**创建 Principal：** 为 HDFS 和 YARM 服务创建主体。

``` Bash
#!/bin/bash

HOSTNAMES='hadoop'
PRINCNAMES='nn dn snn HTTP rm nm hive'

for PRINCNAME in ${PRINCNAMES} do
    for HOSTNAME in ${HOSTNAMES} do
        kadmin.local -q "addprinc -randkey nn/localhost@EXAMPLE.COM"
    done
done
```

## 准备配置文件

### core-site.xml

``` XML
<?xml version="1.0"?>
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://_HOST:9000</value>
  </property>
  <property>
    <name>hadoop.security.authentication</name>
    <value>kerberos</value>
  </property>
  <property>
    <name>hadoop.security.authorization</name>
    <value>true</value>
  </property>
  <property>
    <name>hadoop.rpc.protection</name>
    <value>authentication</value>
  </property>
  <property>
    <name>hadoop.http.authentication.type</name>
    <value>kerberos</value>
  </property>
</configuration>
```

###  hdfs-site.xml

``` XML
<?xml version="1.0"?>
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <!-- 配置集群namenode的kerberos认证 -->
  <property>
    <name>dfs.block.access.token.enable</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.data.transfer.protection</name>
    <value>authentication</value>
  </property>
  <property>
    <name>dfs.namenode.kerberos.principal</name>
    <value>nn/_HOST@BIGDATA.COM</value>
  </property>
  <property>
    <name>dfs.namenode.keytab.file</name>
    <value>/etc/security/keytabs/hdfs.keytab</value>
  </property>
  <property>
    <name>dfs.web.authentication.kerberos.principal</name>
    <value>HTTP/_HOST@BIGDATA.COM</value>
  </property>
  <property>
    <name>dfs.web.authentication.kerberos.keytab</name>
    <value>/etc/security/keytabs/hdfs.keytab</value>
  </property>
  <!-- 配置对NameNode Web UI的SSL访问 -->
  <property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.http.policy</name>
    <value>HTTPS_ONLY</value>
  </property>
  <property>
    <name>dfs.namenode.https-address</name>
    <value>0.0.0.0:50070</value>
  </property>
  <property>
    <name>dfs.permissions.supergroup</name>
    <value>hadoop</value>
    <description>The name of the group of super-users.</description>
  </property>
  <!-- 配置集群datanode的kerberos认证 -->
  <property>
    <name>dfs.datanode.kerberos.principal</name>
    <value>dn/_HOST@BIGDATA.COM</value>
  </property>
  <property>
    <name>dfs.datanode.keytab.file</name>
    <value>/etc/security/keytabs/hdfs.keytab</value>
  </property>
  <!-- 配置datanode SASL配置 -->
  <property>
    <name>dfs.datanode.data.dir.perm</name>
    <value>700</value>
  </property>
  <property>
    <name>dfs.datanode.address</name>
    <value>0.0.0.0:50010</value>
  </property>
  <property>
    <name>dfs.datanode.http.address</name>
    <value>0.0.0.0:50075</value>
  </property>
  <property>
    <name>dfs.data.transfer.protection</name>
    <value>integrity</value>
  </property>
  <!-- 配置集群secondarynamenode的kerberos认证 -->
  <property>
    <name>dfs.secondary.namenode.kerberos.principal</name>
    <value>snn/_HOST@BIGDATA.COM</value>
  </property>
  <property>
    <name>dfs.secondary.namenode.keytab.file</name>
    <value>/etc/security/keytabs/hdfs.keytab</value>
  </property>
  <!-- 配置集群journalnode的kerberos认证 -->
  <property>
    <name>dfs.journalnode.keytab.file</name>
    <value>/etc/security/keytab/hadoop.keytab</value>
  </property>
  <property>
    <name>dfs.journalnode.kerberos.principal</name>
    <value>hadoop/_HOST@HADOOP.COM</value>
  </property>
  <property>
    <name>dfs.journalnode.kerberos.internal.spnego.principal</name>
    <value>${dfs.web.authentication.kerberos.principal}</value>
  </property>
  <property>
    <name>dfs.journalnode.http-address</name>
    <value>0.0.0.0:8480</value>
  </property>
</configuration>
```

### yarn-site.xml

``` XML
<?xml version="1.0"?>
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>_HOST</value>
  </property>
  <property>
    <name>yarn.nodemanager.container-executor.class</name>
    <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
  </property>
  <!-- 配置yarn的web ui 访问https -->
  <property>
    <name>yarn.http.policy</name>
    <value>HTTPS_ONLY</value>
  </property>
  <!-- 指定RM1的Web端访问地址 -->
  <property>
    <name>yarn.resourcemanager.webapp.address.rm1</name>
    <value>ha01:23188</value>
  </property>
  <!-- RM1 HTTP访问地址,查看集群信息 -->
  <property>
    <name>yarn.resourcemanager.webapp.https.address.rm1</name>
    <value>ha01:23188</value>
  </property>
  <!-- 指定RM2的Web端访问地址 -->
  <property>
    <name>yarn.resourcemanager.webapp.address.rm2</name>
    <value>ha02:23188</value>
  </property>
  <!-- RM2 HTTP访问地址,查看集群信息 -->
  <property>
    <name>yarn.resourcemanager.webapp.https.address.rm2</name>
    <value>ha02:23188</value>
  </property>
  <!-- 开启 YARN 集群的日志聚合功能 -->
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>
  <!-- YARN 集群的聚合日志最长保留时长 -->
  <property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <!--7days:604800-->
    <value>86400</value>
  </property>
  <!-- 配置yarn提交的app程序在hdfs上的日志存储路径 -->
  <property>
    <description>Where to aggregate logs to.</description>
    <name>yarn.nodemanager.remote-app-log-dir</name>
    <value>/tmp/logs/yarn-nodemanager</value>
  </property>
  <!--YARN kerberos security-->
  <property>
    <name>yarn.resourcemanager.keytab</name>
    <value>/etc/security/keytab/hadoop.keytab</value>
  </property>
  <property>
    <name>yarn.resourcemanager.principal</name>
    <value>hadoop/_HOST@HADOOP.COM</value>
  </property>
  <property>
    <name>yarn.nodemanager.keytab</name>
    <value>/etc/security/keytab/hadoop.keytab</value>
  </property>
  <property>
    <name>yarn.nodemanager.principal</name>
    <value>hadoop/_HOST@HADOOP.COM</value>
  </property>
  <property>
    <name>yarn.nodemanager.container-executor.class</name>
    <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
  </property>
  <!--此处的group为nodemanager用户所属组-->
  <property>
    <name>yarn.nodemanager.linux-container-executor.group</name>
    <value>hadoop</value>
  </property>
</configuration>
```

### mapred-site.xml

``` XML
<?xml version="1.0"?>
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <!--mapred kerberos security-->
  <property>
    <name>mapreduce.jobhistory.keytab</name>
    <value>/etc/security/keytab/hadoop.keytab</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.principal</name>
    <value>hadoop/_HOST@HADOOP.COM</value>
  </property>
</configuration>
```

### container-executor.cfg

``` INI
#configured value of yarn.nodemanager.linux-container-executor.group
yarn.nodemanager.linux-container-executor.group=hadoop
#comma separated list of users who can not run applications
banned.users=root
#Prevent other super-users
min.user.id=500
#comma separated list of system users who CAN run applications
allowed.system.users=hadoop
```

> [!tip]
> 除了以上的配置文件，还需要修改`hadoop-env.sh` `hive-env.sh` `yarn-env.sh`等环境配置文件。如有需要调整日志路径和级别可以修改对应组件的`log4j.properties`。

## 初始化

``` Bash
$HADOOP_HOME/bin/hdfs namenode -format
```

## 启动服务

``` Bash
$HADOOP_HOME/sbin/start-all.sh
```

> [!tip]
> 如果启动有异常可以开启*DEBUG*模式`export HADOOP_ROOT_LOGGER="DEBUG,console"`

# 分布式安装

> 分布式HA集群需要Zookeeper集群配合，Zookeeper集群的部署这里就不讲述🤣

## Zookeeper 配置Kerberos认证

### zoo.cfg

在 $ZOOKEEPER_HOME/conf/zoo.cfg 中追加以下内容

``` INI
kerberos.removeHostFromPrincipal=true
kerberos.removeRealmFromPrincipal=true
 
authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
jaasLoginRenew=3600000
```

### jaas.conf

新增配置文件 $ZOOKEEPER_HOME/conf/jaas.conf

``` INI
Server {
 
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/etc/security/keytab/hadoop.keytab" #keytab证书的位置
  storeKey=true
  useTicketCache=false
  principal="zookeeper/host_name@HADOOP.COM"; #这里必须是zookeeper，否则zk的客户端后面启动报错
};
 
Client {
 
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/etc/security/keytab/hadoop.keytab"
  storeKey=true
  useTicketCache=false
  principal="hadoop/host_name@HADOOP.COM";
};
```

### java.env

新增配置文件 $ZOOKEEPER_HOME/conf/java.env

``` INI
export JVMFLAGS="-Djava.security.auth.login.config=$ZOOKEEPER_HOME/conf/jaas.conf"
```

> [!tip] 
> 修改配置之后重启 Zookeeper 集群即可

# 参考链接

[apache-hadoop](https://hadoop.apache.org/)