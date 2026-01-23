
# Kerberos History

Kerberos 是第三方网络安全认证协议，由MIT研发。Kerberos 源于希腊神话中Hades的地域犬Kerberos（Cerberus）。   

Kerberos 在不安全的网络中通过第三方认证服务实现客户端向服务端证明身份，并以对称加密算法进行通信加密。

---

## 认证原理

Kerberos是第三方的认证机制，通过它，用户和用户希望访问的服务依赖于Kerberos服务器彼此认证。这种机制也支持加密用户和服务之间的所有通信。Kerberos 服务器作为密钥分发中心，简称KDC。在高级别上，它包含三部分：  
  
- 用户和服务的数据库（即principals）以及他们各自的Kerberos 密码  
  
- 认证服务器（AS），执行初始认证并签发授权票据（TGT）  
  
- Ticket授予服务器（TGS）基于初始的TGT签发后续的服务票据（ST）

认证过程可以分为三步：客户端向AS证明身份获取TGT，使用TGT在TGS获取ST，最后使用ST和服务端建立通信。   

### AS Exchange (初始认证)

1. 客户端用户向KDC以明文的方式发起请求。该次请求中携带了自己的用户名，主机IP，和当前时间戳；
2. AS（Authentication Server）接收请求，并在kerberos认证数据库中验证是否存在该用户
3. 如果没有该用户名，认证失败，服务结束；如果存在该用户名，则AS认证中心便认会返回响应给客户端，其中包含两部分内容：
	 - 第一部分内容称为TGT，客户端需要使用TGT去KDC中的TGS（票据授予中心）获取访问网络服务所需的Ticket（服务授予票据），TGT中包含的内容有kerberos数据库中存在的该客户端的Name，IP，当前时间戳，客户端即将访问的TGS的Name，TGT的有效时间以及一把用于客户端和TGS间进行通信的Session_key(CT_SK)。整个TGT使用TGS密钥加密，客户端是解密不了的，由于密钥从没有在网络中传输过，所以也不存在密钥被劫持破解的情况。
	 - 第二部分内容是使用客户端密钥加密的一段内容，其中包括用于客户端和TGS间通信的Session_key1(CT_SK),客户端即将访问的TGS的Name以及TGT的有效时间，和一个当前时间戳。该部分内容使用客户端密钥加密，所以客户端在拿到该部分内容时可以通过自己的密钥解密。如果是一个假的客户端，那么他是不会拥有真正客户端的密钥的，因为该密钥也从没在网络中进行传输过。这也同时认证了客户端的身份，如果是假客户端会由于解密失败从而终端认证流程。

### TGS Exchange (获取服务票据)

此时的客户端收到了来自KDC（其实是AS）的响应，并获取到了其中的两部分内容。此时客户端会用自己的密钥将第二部分内容进行解密，分别获得时间戳，自己将要访问的TGS的信息，和用于与TGS通信时的密钥CT_SK。首先他会根据时间戳判断该时间戳与自己发送请求时的时间之间的差值是否大于5分钟，如果大于五分钟则认为该AS是伪造的，认证至此失败。如果时间戳合理，客户端便准备向TGS发起请求。   

**请求的主要目的是为了获取能够访问目标网络服务的服务授予票据Ticket**，在第二次通信请求中，客户端将携带三部分内容交给KDc中的TGS，第二次通信过程具体如下所述：

**客户端**
1. 客户端使用CT_SK加密将自己的客户端信息发送给KDC，其中包括客户端名，IP，时间戳；
2. 客户端将自己想要访问的Server服务以明文的方式发送给KDC；
3. 客户端将使用TGS密钥加密的TGT也原封不动的也携带给KDC；

**TGS**
1. 此时KDC中的TGS（票据授予服务器）收到了来自客户端的请求。他首先根据客户端明文传输过来的Server服务IP查看当前kerberos系统中是否存在可以被用户访问的该服务。如果不存在，认证失败结束，。如果存在，继续接下来的认证。
2. TGS使用自己的密钥将TGT中的内容进行解密，此时他看到了经过AS认证过后并记录的用户信息，一把Session_KEY即CT_SK，还有时间戳信息，他会现根据时间戳判断此次通信是否真是可靠有无超出时延。
3. 如果时延正常，则TGS会使用CK_SK对客户端的第一部分内容进行解密（使用CT_SK加密的客户端信息），取出其中的用户信息和TGT中的用户信息进行比对，如果全部相同则认为客户端身份正确，方可继续进行下一步。
4. 此时KDC将返回响应给客户端，响应内容包括：
	- 第一部分：用于客户端访问网络服务的使用Server密码加密的ST（Servre Ticket），其中包括客户端的Name，IP，需要访问的网络服务的地址Server IP，ST的有效时间，时间戳以及用于客户端和服务端之间通信的CS_SK2（Session Key）。
	- 第二部分：使用CT_SK加密的内容，其中包括CS_SK2和时间戳，还有ST的有效时间。由于在第一次通信的过程中，AS已将CT_SK通过客户端密码加密交给了客户端，且客户端解密并缓存了CT_SK，所以该部分内容在客户端接收到时是可以自己解密的。

### Client/Server Exchange (访问服务)

此时的客户端收到了来自KDC（TGS）的响应，并使用缓存在本地的CT_SK解密了第二部分内容（第一部分内容中的ST是由Server密码加密的，客户端无法解密），检查时间戳无误后取出其中的CS_SK准备向服务端发起最后的请求。

**客户端**   

客户端使用CK_SK2将自己的主机信息和时间戳进行加密作为交给服务端的第一部分内容，然后将ST（服务授予票据）作为第二部分内容都发送给服务端。

**服务端**   

服务器此时收到了来自客户端的请求，他会使用自己的密钥，即Server密钥将客户端第二部分内容进行解密，核对时间戳之后将其中的CS_SK2取出，使用CS_SK2将客户端发来的第一部分内容进行解密，从而获得经过TGS认证过后的客户端信息，此时他将这部分信息和客户端第二部分内容带来的自己的信息进行比对，最终确认该客户端就是经过了KDC认证的具有真实身份的客户端，是他可以提供服务的客户端。此时服务端返回一段使用CT_SK2加密的表示接收请求的响应给客户端，在客户端收到请求之后，使用缓存在本地的CS_ST2解密之后也确定了服务端的身份（其实服务端在通信的过程中还会使用数字证书证明自己身份）。

### Kerberos认证时序图
   
``` mermaid
sequenceDiagram
    autonumber
    participant C as Client (用户)
    participant AS as KDC - AS (认证服务器)
    participant TGS as KDC - TGS (票据授予服务器)
    participant S as Server (目标服务)

    Note over C, AS: 阶段 1: 初始认证 (AS Exchange)
    C->>AS: 1. 我是 User A (明文请求)
    
    Note right of AS: AS 校验 User A 存在<br/>生成 SessionKey_C_TGS
    
    AS-->>C: 2. 响应包含两部分:<br/>A: {SessionKey_C_TGS}_UserPwdHash<br/>B: TGT {SessionKey_C_TGS, UserInfo}_KrbtgtKey
    
    Note left of C: 用户输密码解密 A 部分，<br/>获得 SessionKey_C_TGS。<br/>(TGT 无法解密，保留)

    Note over C, TGS: 阶段 2: 获取服务票据 (TGS Exchange)
    C->>TGS: 3. 请求服务票据:<br/>A: TGT (原样发送)<br/>B: Authenticator {UserInfo, Timestamp}_SessionKey_C_TGS
    
    Note right of TGS: TGS 用 KrbtgtKey 解密 TGT -> 拿到 SessionKey_C_TGS<br/>用 SessionKey_C_TGS 解密 Authenticator -> 验证身份 & 时间
    
    TGS-->>C: 4. 响应包含两部分:<br/>A: {SessionKey_C_S}_SessionKey_C_TGS<br/>B: ST {SessionKey_C_S, UserInfo}_ServerKey

    Note left of C: 用 SessionKey_C_TGS 解密 A 部分，<br/>获得 SessionKey_C_S。<br/>(ST 无法解密，保留)

    Note over C, S: 阶段 3: 访问服务 (Client/Server Exchange)
    C->>S: 5. 申请访问:<br/>A: ST (原样发送)<br/>B: Authenticator {UserInfo, Timestamp}_SessionKey_C_S
    
    Note right of S: Server 用 ServerKey 解密 ST -> 拿到 SessionKey_C_S<br/>用 SessionKey_C_S 解密 Authenticator -> 验证通过
    
    S-->>C: 6. (可选) 验证响应 {Timestamp}_SessionKey_C_S
    Note left of C: 认证成功，建立安全通道
```

# 运维管理

> [!tip] 提示
> 凭证管理中涉及到和_kadmin_的交互可以使用`kadmin.local`直接登录到_kadmin_控制台，也可以使用`kadmin.local -q 'command'`进行批量操作。

## 增加KDC服务并发

>在大规模集群下，默认的 KDC 配置会导致请求没有响应，会出现诡异的 "UnKnown Hostname" 异常，我们可以通过修改配置文件的方式来增加 KDC 的服务进程。
>
>用 root 账号登陆 KDC 服务器，编辑 _/etc/sysconfig/krb5kdc_ 文件，修改 `KRB5KDC_ARGS` 选项，增加`-w 32`参数，类似如下：

``` Bash
[root@hadoop ~]# cat /etc/sysconfig/krb5kdc 
KRB5KDC_ARGS=-w 32
```

## 凭证管理

### 创建凭证

- 指定密码创建
	`addprinc -pw password123 user/admin`
- 随机密码创建
	`addprinc -randkey hive/hive@BIGDATA.COM`

### 导出凭证

``` Bash
# 使用xst
xst -norandkey -k /etc/security/keytabs/hive.keytab hive/hive@BIGDATA.COM
# 或者ktadd
ktadd -norandkey -k /etc/security/keytabs/hive.keytab hive/hive@BIGDATA.COM
```

### 修改密码

`cpw -pw new_password user/admin`

### 修改加密算法

`modprinc -e aes128-cts-hmac-sha1-96 user/admin`

### 删除凭证

`delprinc user/admin`

### 查询凭证

- 查看指定凭证
	`getprinc user/admin`
- 列出所有凭证
	`listprincs`

### 验证凭证

``` Bash
[root@hadoop ~]# klist -kt hdfs.keytab 
Keytab name: FILE:hdfs.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 01/20/2026 10:25:48 hdfs/hdfs@BIGDATA.COM
   1 01/20/2026 10:25:49 hdfs/hdfs@BIGDATA.COM
[root@hadoop ~]# kinit -kt hdfs.keytab hdfs/hdfs@BIGDATA.COM
```

### 取消验证

``` Bash
[root@hadoop ~]# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: hdfs/hdfs@BIGDATA.COM

Valid starting       Expires              Service principal
01/20/2026 10:32:34  01/21/2026 10:32:34  krbtgt/BIGDATA.COM@BIGDATA.COM
[root@hadoop ~]# kdestroy
[root@hadoop ~]# klist
klist: Credentials cache keyring 'persistent:0:0' not found
```

## 常见问题排查

### 常见错误码对照表

- **`Generic error (status code 60) - Pre-authentication failed`**
    
    - **原因：** 密码错误或 Keytab 与 KDC 中的 kvno（版本号）不一致。
        
    - **解决：** 重新导出 Keytab。
        
- **`Clock skew too great (37)`**
    
    - **原因：** 客户端与 KDC 服务器时间偏差超过 5 分钟（默认值）。
        
    - **解决：** 同步各节点 `ntp` 或 `chrony` 服务。
        
- **`Client not found in Kerberos database (6)`**
    
    - **原因：** Principal 名称写错，或者 Realm（领域）大小写不敏感导致。
        
- **`Ticket expired (32)`**
    
    - **原因：** 票据已过期，且未开启自动续期。

### 开启日志

当无法从报错表面判断问题时，在启动服务或执行命令前开启 Java 调试参数:

``` Bash
export HADOOP_OPTS="-Dsun.security.krb5.debug=true"
# 或者在执行 kinit 时查看详细过程
kinit -V user/admin
```

## 优化建议

### 核心配置优化 (`krb5.conf`)

- **`udp_preference_limit = 1`**：强制使用 TCP 协议。UDP 在处理大数据量（如包含大量 Group 信息的票据）时容易丢包或产生分片错误。
    
- **`ticket_lifetime` & `renew_lifetime`**：
    
    - 建议票据有效期设为 24h，续期时间设为 7 天。
        
    - 避免频繁执行 `kinit` 产生大量的 KDC 请求压力。
        

### 定期清理与审计

- **过期检查：** 定期清理不再使用的 Principal，防止数据库过于臃肿。
    
- **监控：** 重点监控 KDC 所在服务器的 CPU 负载和 88 端口的响应延迟。