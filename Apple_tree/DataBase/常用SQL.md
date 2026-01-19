
## mysql 锁表操作

**全局锁** `flush tables with read lock;`
**锁表** `lock table table_name read;`
**检查被锁的表** `show processlist;`
**解锁表** `unlock tables;`

## mysql 用户操作

1、`create schema [数据库名称] default character set utf8 collate utf8_general_ci;`--创建数据库
采用`create schema`和`create database`创建数据库的效果一样。

2、`create user '[用户名称]'@'%' identified by '[用户密码]';`--创建用户
密码8位以上，包括：大写字母、小写字母、数字、特殊字符
%：匹配所有主机，该地方还可以设置成‘localhost’，代表只能本地访问，例如root账户默认为‘localhost‘

3、`grant select,insert,update,delete,create on [数据库名称].* to [用户名称];` --用户授权数据库
*代表整个数据库

4、`flush  privileges ;` --立即启用修改
5、`revoke all on *.* from tester;` --取消用户所有数据库（表）的所有权限
6、`delete from mysql.user where user='tester';` --删除用户
7、`drop database [schema名称|数据库名称];` --删除数据库

## 查看mysql性能

启用Performance Schema：在MySQL配置文件中添加以下行以启用Performance Schema：`set global performance_schema=ON;`
配置Performance Schema：在MySQL配置文件中添加以下行以配置Performance Schema：`set global performance_schema_instrument='memory/%=COUNTED';`
查看性能数据：在MySQL客户端中输入以下命令以查看性能数据：`SELECT * FROM performance_schema.global_status WHERE variable_name LIKE 'Com_%';`

``` sql
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_total';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_data';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages_dirty';
```
  
|概念|定义|计算方法|  
|-|:-:|-:|
|InnoDB缓存读命中率|InnoDB引擎从缓存中读取数据的比率|(1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) *100%|
|InnoDB缓存使用率|InnoDB缓存中已经使用的内存比例|(Innodb_buffer_pool_pages_data / Innodb_buffer_pool_pages_total)* 100%|
|InnoDB脏块率|InnoDB缓存中已修改但尚未写回磁盘的数据块比例|(Innodb_buffer_pool_pages_dirty / Innodb_buffer_pool_pages_total) * 100%|
