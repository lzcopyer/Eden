
# 先决条件

## 基线配置

1. 关闭防火墙 or 设置白名单
2. 配置时间同步
3. 配置_SSH_免密登录
4. 配置主机名映射 or 配置内网_DNS_

## 所需依赖

Linux和Windows所需软件包括:

	1. Java 必须安装，建议选择Sun公司发行的Java版本。
	2. **ssh** 必须安装并且保证 **sshd**一直运行，以便用Hadoop 脚本管理远端Hadoop守护进程。

## 依赖安装

``` Bash
apt-get install ssh rsync
```

# 伪分布式部署

## 准备凭证

**创建 Principal：** 为 HDFS 和 HTTP 服务创建主体。

- `kadmin.local -q "addprinc -randkey nn/localhost@EXAMPLE.COM"`
    
- `kadmin.local -q "addprinc -randkey dn/localhost@EXAMPLE.COM"`
    
- `kadmin.local -q "addprinc -randkey HTTP/localhost@EXAMPLE.COM"`
    
- `kadmin.local -q "addprinc -randkey hive/localhost@EXAMPLE.COM"`

## 准备配置文件

- core-site.xml

``` XML
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
    <property>
        <name>hadoop.security.authentication</name>
        <value>kerberos</value>
    </property>
    <property>
        <name>hadoop.security.authorization</name>
        <value>true</value>
    </property>
</configuration>
```

- hdfs-site.xml

``` XML
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.kerberos.principal</name>
        <value>nn/_HOST@EXAMPLE.COM</value>
    </property>
    <property>
        <name>dfs.namenode.keytab.file</name>
        <value>/etc/security/keytabs/hdfs.keytab</value>
    </property>
    <property>
        <name>dfs.web.authentication.kerberos.principal</name>
        <value>HTTP/_HOST@EXAMPLE.COM</value>
    </property>
    <property>
        <name>dfs.web.authentication.kerberos.keytab</name>
        <value>/etc/security/keytabs/hdfs.keytab</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir.perm</name>
        <value>700</value>
    </property>
    <property>
        <name>dfs.data.transfer.protection</name>
        <value>authentication</value>
    </property>
</configuration>
```

- yarn-site.xml

``` XML
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>localhost</value>
    </property>
    <property>
        <name>yarn.resourcemanager.principal</name>
        <value>rm/_HOST@EXAMPLE.COM</value>
    </property>
    <property>
        <name>yarn.resourcemanager.keytab</name>
        <value>/etc/security/keytabs/yarn.keytab</value>
    </property>
    <property>
        <name>yarn.nodemanager.principal</name>
        <value>nm/_HOST@EXAMPLE.COM</value>
    </property>
    <property>
        <name>yarn.nodemanager.keytab</name>
        <value>/etc/security/keytabs/yarn.keytab</value>
    </property>
    <property>
        <name>yarn.nodemanager.container-executor.class</name>
        <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
    </property>
</configuration>
```

- hive-site.xml

``` XML
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://localhost:3306/metastore?createDatabaseIfNotExist=true</value>
    </property>
    <property>
        <name>hive.server2.authentication</name>
        <value>KERBEROS</value>
    </property>
    <property>
        <name>hive.server2.authentication.kerberos.principal</name>
        <value>hive/_HOST@EXAMPLE.COM</value>
    </property>
    <property>
        <name>hive.server2.authentication.kerberos.keytab</name>
        <value>/etc/security/keytabs/hive.keytab</value>
    </property>
    <property>
        <name>hive.metastore.sasl.enabled</name>
        <value>true</value>
    </property>
    <property>
        <name>hive.metastore.kerberos.principal</name>
        <value>hive/_HOST@EXAMPLE.COM</value>
    </property>
    <property>
        <name>hive.metastore.kerberos.keytab.file</name>
        <value>/etc/security/keytabs/hive.keytab</value>
    </property>
</configuration>
```

- mapred-site.xml

``` XML
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

> [!tip] 补充
> 除了以上的配置文件，还需要修改`hadoop-env.sh` `hive-env.sh` `yarn-env.sh`等环境配置文件。如有需要调整日志路径和级别可以修改对应组件的`log4j.properties`。