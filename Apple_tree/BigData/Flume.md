# 简介

>Apache Flume 是一个分布式、高可靠、高可用的用来收集、聚合、转移不同来源的大量日志数据到中央数据仓库的工具

# 体系结构

## 数据流模型

Event是Flume定义的一个数据流传输的最小单元。Agent就是一个Flume的实例，本质是一个JVM进程，该JVM进程控制Event数据流从外部日志生产者那里传输到目的地（或者是下一个Agent）。

> [!tip] 提示
学习Flume必须明白这几个概念，Event英文直译是事件，但是在Flume里表示数据传输的一个最小单位（被Flume收集的一条条日志又或者一个个的二进制文件，不管你在外面叫什么，进入Flume之后它就叫event）。参照下图可以看得出Agent就是Flume的一个部署实例， 一个完整的Agent中包含了必须的三个组件Source、Channel和Sink，Source是指数据的来源和方式，Channel是一个数据的缓冲池，Sink定义了数据输出的方式和目的地（这三个组件是必须有的，另外还有很多可选的组件interceptor、channel selector、sink processor等后面会介绍）。

Source消耗由外部（如Web服务器）传递给它的Event。外部以Flume Source识别的格式向Flume发送Event。例如，[Avro Source](https://flume.liyifeng.org/#avro-source) 可接收从Avro客户端（或其他FlumeSink）接收Avro Event。用 [Thrift Source](https://flume.liyifeng.org/#thrift-source) 也可以实现类似的流程，接收的Event数据可以是任何语言编写的只要符合Thrift协议即可。

当Source接收Event时，它将其存储到一个或多个channel。该channel是一个被动存储器（或者说叫存储池），可以存储Event直到它被Sink消耗。『文件channel』就是一个例子 - 它由本地文件系统支持。sink从channel中移除Event并将其放入外部存储库（如HDFS，通过 Flume的 [HDFS Sink](https://flume.liyifeng.org/#hdfs-sink) 实现）或将其转发到流中下一个Flume Agent（下一跳）的Flume Source。

Agent中的source和sink与channel存取Event是异步的。

Flume的Source负责消费外部传递给它的数据（比如web服务器的日志）。外部的数据生产方以Flume Source识别的格式向Flume发送Event。

> [!tip] 提示
“Source消耗由外部传递给它的Event”，这句话听起来好像Flume只能被动接收Event，实际上Flume也有Source是主动收集Event的，比如：[Spooling Directory Source](https://flume.liyifeng.org/#spooling-directory-source) 、[Taildir Source](https://flume.liyifeng.org/#taildir-source) 。

![Flume架构图](res/flume_image01.png)

## 复杂流

Flume可以设置多级Agent连接的方式传输Event数据。也支持扇入和扇出的部署方式，类似于负载均衡方式或多点同时备份的方式。

![复杂流架构图](res/flume_image02.png)

> [!tip] 提示
这里必须解释一下，第一句的意思是可以部署多个Agent组成一个数据流的传输链。第二句要知道扇入（多对一）和扇出（一对多）的概念，就是说Agent可以将数据流发到多个下级Agent，也可以从多个Agent发到一个Agent中。
>
>其实他们就是想告诉你，你可以根据自己的业务需求来任意组合传输日志的Agent流，引用一张后面章节的图，这就是一个扇入方式的Flume部署方式，前三个Agent的数据都汇总到一个Agent4上，最后由Agent4统一存储到HDFS。

## 可靠性

Event会在每个Agent的Channel上进行缓存，随后Event将会传递到流中的下一个Agent或目的地（比如HDFS）。只有成功地发送到下一个Agent或目的地后Event才会从Channel中删除。这一步保证了Event数据流在Flume Agent中传输时端到端的可靠性。

> [!tip] 提示
同学们这里是知识点啊（敲黑板）！Flume的这个channel最重要的功能是用来保证数据的可靠传输的。其实另外一个重要的功能也不可忽视，就是实现了数据流入和流出的异步执行。

Flume使用事务来保证Event的 **可靠传输**。Source和Sink对Channel提供的每个Event数据分别封装一个事务用于存储和恢复，这样就保证了数据流的集合在点对点之间的可靠传输。在多层架构的情况下，来自前一层的sink和来自下一层的Source 都会有事务在运行以确保数据安全地传输到下一层的Channel中。

## 可恢复性

Event数据会缓存在Channel中用来在失败的时候恢复出来。Flume支持保存在本地文件系统中的『文件channel』，也支持保存在内存中的『内存Channel』，『内存Channel』显然速度会更快，缺点是万一Agent挂掉『内存Channel』中缓存的数据也就丢失了。

# 部署第一个Agent

## 依赖准备

Flume 环境依赖只有 java1.8（或以上版本），额外的功能依赖于相应的jar包或者插件。

### 安装java

这里推荐安装openjdk（并不是我十分讨厌Oracle）

``` Bash
#ubuntu/debian 安装openjdk1.8
sudo apt install openjdk-8-jre
#centos/fedora
sudo yum install java-1.8.0-openjdk
#高版本安装前往https://jdk.java.net下载对应版本的安装包解压即可
wget https://download.java.net/openjdk/jdk8u44/ri/openjdk-8u44-linux-x64.tar.gz -o ~/
tar -zxf openjdk-8u44-linux-x64.tar.gz -C ~/
```

### 安装插件

#### jar包安装

连接 `hdfs` 或 `mysql` 所需要的jar包通常放在 $FLUME_HOME/lib/ 下

#### 第三方插件安装

Flume有完整的插件架构。尽管Flume已经提供了很多现成的source、channel、sink、serializer可用。

然而通过把自定义组件的jar包添加到flume-env.sh文件的FLUME_CLASSPATH 变量中使用自定义的组件也是常有的事。现在Flume支持在一个特定的文件夹自动获取组件，这个文件夹就是pluguins.d。这样使得插件的包管理、调试、错误定位更加容易方便，尤其是依赖包的冲突处理。

#### plugins.d文件夹

`plugins.d` 文件夹的所在位置是 $FLUME_HOME/plugins.d ，在启动时 flume-ng 会启动脚本检查这个文件夹把符合格式的插件添加到系统中。

#### 插件的目录结构

每个插件（也就是 `plugins.d` 下的子文件夹）都可以有三个子文件夹：

1. lib - 插件自己的jar包
    
2. libext - 插件依赖的其他所有jar包
    
3. native - 依赖的一些本地库文件，比如 _.so_ 文件
    
下面是两个插件的目录结构例子：
	
``` Bash
	plugins.d/
	plugins.d/custom-source-1/
	plugins.d/custom-source-1/lib/my-source.jar
	plugins.d/custom-source-1/libext/spring-core-2.5.6.jar
	plugins.d/custom-source-2/
	plugins.d/custom-source-2/lib/custom.jar
	plugins.d/custom-source-2/native/gettext.so
```

## 配置

### 环境配置

flume-env.sh 主要加载flume的java环境和启动参数

``` Bash
#java安装路径
export JAVA_HOME=/usr/lib/jvm/java-8-oracle
#启动参数
export JAVA_OPTS="-Xms100m -Xmx2000m -Dcom.sun.management.jmxremote"
export JAVA_OPTS="$JAVA_OPTS -Dorg.apache.flume.log.rawdata=true -Dorg.apache.flume.log.printconfig=true "
#加载插件（推荐插件安装在plugins.d中自动加载）
FLUME_CLASSPATH=""
```

### 日志配置

log4j2.xml 定义日志保存路径和日志级别

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Properties>
    <Property name="FLUME_HOME">/home/malou/BigData/flume</Property>
    <Property name="LOG_DIR">${sys:FLUME_HOME}/logs</Property>
  </Properties>

  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} [%p - %c{1.}] %m%n" />
    </Console>

    <RollingFile name="LogFile" 
                 fileName="${LOG_DIR}/flume.log" 
                 filePattern="${LOG_DIR}/archive/flume.log.%d{yyyy-MM-dd}.%i.gz">
      
      <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p [%t] (%c{1}:%L) - %m%n" />
      
      <Policies>
        <SizeBasedTriggeringPolicy size="100 MB"/>
        <TimeBasedTriggeringPolicy interval="1" modulate="true" />
      </Policies>

      <DefaultRolloverStrategy max="20">
        <Delete basePath="${LOG_DIR}/archive" maxDepth="1">
          <IfFileName glob="flume.log.*.gz" />
          <IfLastModified age="7d" />
          <IfAccumulatedFileSize exceeds="1 GB"/>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>

  <Loggers>
    <Logger name="org.apache.flume.lifecycle" level="INFO"/>
    <Logger name="org.jboss" level="WARN"/>
    <Logger name="org.apache.avro.ipc.NettyTransceiver" level="WARN"/>
    <Logger name="org.apache.hadoop" level="INFO"/>
    <Logger name="org.apache.hadoop.hive" level="ERROR"/>
    
    <Root level="INFO">
      <AppenderRef ref="Console" />
      <AppenderRef ref="LogFile" />
    </Root>
  </Loggers>
</Configuration>
```

### 主配置

> [!tip] 提示
flume-conf.properties 定义了 flume 实例的类型和功能。主要是source、sink、channel三大组件和其他几个组件channel selector、sink processor、serializer、interceptor的配置、使用方法和各自的适用范围。 如果硬要翻译这些组件的话，三大组件分别是数据源（source）、数据目的地（sink）和缓冲池（channel）。其他几个分别是Event多路复用的channel选择器（channel selector）， Sink组逻辑处理器（sink processor）、序列化器（serializer）、拦截器（interceptor）。
>
>一个agent实例里面可以有多条独立的数据流，一个数据流里必须有且只能有一个Source，但是可以有多个Channel和Sink，Source和Channel是一对多的关系，Channel和Sink也是一对多的关系。

``` properties
# 列出所有的组件, agent_foo 就是agent名, 启动agent时需指定的参数
agent_foo.sources = avro-AppSrv-source
agent_foo.sinks = hdfs-Cluster1-sink
agent_foo.channels = mem-channel-1

# 将source和sink与channel相连接
agent_foo.sources.avro-appserver-src-1.channels = mem-channel-1
agent_foo.sinks.hdfs-sink-1.channel = mem-channel-1

# 配置avro-AppSrv-source的属性
agent_foo.sources.avro-AppSrv-source.type = avro         # avro-AppSrv-source 的类型是Avro Source
agent_foo.sources.avro-AppSrv-source.bind = localhost    # 监听的hostname或者ip是localhost
agent_foo.sources.avro-AppSrv-source.port = 10000        # 监听的端口是10000

# 配置mem-channel-1的属性
agent_foo.channels.mem-channel-1.type = memory                # channel的类型是内存channel
agent_foo.channels.mem-channel-1.capacity = 1000              # channel的最大容量是1000
agent_foo.channels.mem-channel-1.transactionCapacity = 100    # source和sink每次事务从channel写入和读取的Event数量

# 配置hdfs-Cluster1-sink的属性
agent_foo.sinks.hdfs-Cluster1-sink.type = hdfs                                   # sink的类型是HDFS Sink
agent_foo.sinks.hdfs-Cluster1-sink.hdfs.path = hdfs://namenode/flume/webdata     # 写入的HDFS目录路径
```

### 拦截器

>Flume支持在运行时对Event进行修改或丢弃，可以通过拦截器来实现。Flume里面的拦截器是实现了 _org.apache.flume.interceptor.Interceptor_ 接口的类。拦截器可以根据开发者的意图随意修改甚至丢弃Event， Flume也支持链式的拦截器执行方式，在配置文件里面配置多个拦截器就可以了。拦截器的顺序取决于它们被初始化的顺序（实际也就是配置的顺序），Event就这样按照顺序经过每一个拦截器，如果想在拦截器里面丢弃Event， 在传递给下一级拦截器的list里面把它移除就行了。如果想丢弃所有的Event，返回一个空集合就行了。拦截器也是通过命名配置的组件，下面就是通过配置文件来创建拦截器的例子。

**拦截器配置示例**

``` properties
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = regex_extractor

# 匹配 RFC5424 格式：时间(s1) 主机(s2) 进程[PID](s3): 内容(s4)
a1.sources.r1.interceptors.i1.regex = ^(\\d{4}-\\d{2}-\\d{2}T[\\d:.]+.*?)\\s+(\\S+)\\s+([^:]+):\\s+(.*)$

a1.sources.r1.interceptors.i1.serializers = s1 s2 s3 s4
a1.sources.r1.interceptors.i1.serializers.s1.name = log_time
a1.sources.r1.interceptors.i1.serializers.s2.name = hostname
a1.sources.r1.interceptors.i1.serializers.s3.name = process
a1.sources.r1.interceptors.i1.serializers.s4.name = message
```

> [!tip] 提示
> flume有多种属性的拦截器（例如：查找替换拦截器、正则拦截器等等）。推荐查看[拦截器配置](https://github.com/apachecn/apachecn-bigdata-zh-pt2/blob/master/docs/flume-dist-log-col-hadoop/6.md)，这里就不多赘述了。

### SSL配置

SSL相关配置项

|System property|Environment variable|描述|
|---|---|---|
|javax.net.ssl.keyStore|FLUME_SSL_KEYSTORE_PATH|Keystore 路径|
|javax.net.ssl.keyStorePassword|FLUME_SSL_KEYSTORE_PASSWORD|Keystore 密码|
|javax.net.ssl.keyStoreType|FLUME_SSL_KEYSTORE_TYPE|Keystore 类型 (默认是JKS)|
|javax.net.ssl.trustStore|FLUME_SSL_TRUSTSTORE_PATH|Truststore 路径|
|javax.net.ssl.trustStorePassword|FLUME_SSL_TRUSTSTORE_PASSWORD|Truststore 密码|
|javax.net.ssl.trustStoreType|FLUME_SSL_TRUSTSTORE_TYPE|Truststore 类型 (默认是 JKS)|
|flume.ssl.include.protocols|FLUME_SSL_INCLUDE_PROTOCOLS|将要使用的SSL/TLS协议版本，多个用逗号分隔，如果一个协议在下面的exclude.protocols中也配置了的话，那么这个协议会被排除，也就是exclude.protocols的优先级更高一些|
|flume.ssl.exclude.protocols|FLUME_SSL_EXCLUDE_PROTOCOLS|不使用的SSL/TLS协议版本，多个用逗号分隔|
|flume.ssl.include.cipherSuites|FLUME_SSL_INCLUDE_CIPHERSUITES|将要使用的密码套件，多个用逗号分隔，如果一个套件在下面的exclude.cipherSuites中也配置了的话，那么这个套件会被排除，也就是exclude.cipherSuites的优先级更高一些|
|flume.ssl.exclude.cipherSuites|FLUME_SSL_EXCLUDE_CIPHERSUITES|不使用的密码套件，多个用逗号分隔|

建议在conf/flume-env.sh 中添加以下配置，示例如下：

``` Bash
export FLUME_SSL_KEYSTORE_PATH=/path/to/keystore.jks
export FLUME_SSL_KEYSTORE_PASSWORD=password
```

## 启动

### Flume 启动命令

``` Bash
bin/flume-ng agent --conf conf --conf-file ${flume_conf} --name ${agent_name}
```

> [!tip] 提示
在完整的部署中通常会包含 –conf=\<conf-dir>这个参数，\<conf-dir>目录里面包含了flume-env.sh和一个log4j properties文件。额外参数"-Dflume.root.logger=INFO,console" 可以强制Flume把日志输出到了控制台，运行的时候没有任何自定义的环境脚本。

### 基于Zookeeper的启动方式

Flume支持使用Zookeeper配置Agent。**这是个实验性的功能**。配置文件需要上传到zookeeper中，在一个可配置前缀下。配置文件存储在Zookeeper节点数据里。下面是a1 和 a2 Agent在Zookeeper节点树的配置情况。

``` bash
- /flume
 |- /a1 [Agent config file]
 |- /a2 [Agent config file]
```

启动命令

``` bash
bin/flume-ng agent --conf conf -z zkhost:2181,zkhost1:2181 -p /flume --name a1 -Dflume.root.logger=INFO,console
```

# 参考链接

[flume中文文档](https://flume.liyifeng.org/#)