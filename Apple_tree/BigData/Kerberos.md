
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