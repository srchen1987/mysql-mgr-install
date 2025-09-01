# 基于mgr和proxysql搭建mysql高可用方案

## 简介

MGR是mysql官方推荐的高可用集群方案,通过一个分布式共识协议（如Paxos 的变种）,将集群内的节点组成一个复制组,事务提交前，需要经过多数节点进行决议和验证，通过后才能提交，实现了全局的数据一致性.

MGR的组复制对比同步、半同步、增强半同步、异步四种模式的优势请自己脑补.

MGR是mysql官方推荐的高可用方案,早期mha很多大公司都在用也不错后来停止更新了(我猜测是mysql被收购之后增新料太快,作者跟不上节奏,再加上推出了MGR这种MHA存在的价值就不大了).

MGR支持单主或多主模式，可随时切换，最少3台机器才能实现高可用方案,少于3台无法实现,会导致脑裂，无法提供服务.

允许故障数量

3台允许挂1台,5台允许挂2台,7台允许挂3台以此类推.

头脑风暴下: mysql被oracle收购之后mysql作者又去搞mariadb. oracle真该多出点钱留住他,他也不该走舍弃自己的孩子. 不如一起合作重构下mysql,必定mysql代码有很多瑕疵和bug.比如拿比较火的rust语言开发一套全新的mysql.改下开源协议像redis那种不被云厂商白嫖收些抽层也好.针对个人和企业本地化部署的完全开源免费.

## 准备软件

### mysql版本

注意:这里采用的是mysql8.4.x 如果是8.0.x或9.x请参考官方文档调整参数.

例如:

```shell
#注释掉此配置
#transaction_write_set_extraction=XXHASH64
```

### proxysql版本

目前最新版是v3.0.2

## 准备工作

### 准备主机,并安装MySQL

1、四台机器(我本机是fedora42 太新了 通过kvm安装了4台 fedora41-server版)ip分别为 192.168.80.110,192.168.80.111,192.168.80.112,192.168.80.113

设置hosts

```shell
192.168.80.110 mgr-node1
192.168.80.111 mgr-node2
192.168.80.112 mgr-node3
192.168.80.113 proxysql
```

分别更改 host文件 mysql1 mysql2 mysql3,proxysql  对应上面ip

| hosts    | ip             | member   |
| -------- | -------------- | -------- |
| mysql1   | 192.168.80.110 | master   |
| mysql2   | 192.168.80.111 | slave    |
| mysql3   | 192.168.80.112 | slave    |
| proxysql | 192.168.80.113 | proxysql |

2、下载并安装mysql 参考 <https://github.com/srchen1987/mysql_install>

3、mysql配置

mysql1的my.cnf

```bash
[mysqld]
user=mysql
basedir=/usr/local/mysql8
datadir=/usr/local/mysql8/data
server_id=110
port=3306
socket=/usr/local/mysql8/data/mysql.sock
gtid-mode=on
enforce-gtid-consistency=true
log_bin=/usr/local/mysql8/data/log-bin
binlog_format=row
binlog_checksum=NONE
skip-name-resolve
#master_info_repository=TABLE
#relay_log_info_repository=TABLE
log_slave_updates=ON

#transaction_write_set_extraction=XXHASH64
loose-group_replication_bootstrap_group=OFF
loose-group_replication_start_on_boot=ON
loose-group_replication_group_name="b0e3a840-0c38-4cb3-aefb-82f1a2b9b32d"
group_replication_local_address= "192.168.80.110:33061"
group_replication_group_seeds= "192.168.80.110:33061,192.168.80.111:33061,192.168.80.112:33061"
loose-group_group_replication_single_primary_mode = ON;
[mysqld_safe]
log-error=/usr/local/mysql8/log/mysql_err.log
pid-file=/usr/local/mysql8/mysql8.pid

```

Mysql2的my.cnf

```bash
[mysqld]
user=mysql
basedir=/usr/local/mysql8
datadir=/usr/local/mysql8/data
server_id=111
port=3306
socket=/usr/local/mysql8/data/mysql.sock
gtid-mode=on
enforce-gtid-consistency=true
log_bin=/usr/local/mysql8/data/log-bin
binlog_format=row
binlog_checksum=NONE
skip-name-resolve
#master_info_repository=TABLE
#relay_log_info_repository=TABLE
log_slave_updates=ON

#transaction_write_set_extraction=XXHASH64
loose-group_replication_bootstrap_group=OFF
loose-group_replication_start_on_boot=ON
loose-group_replication_group_name="b0e3a840-0c38-4cb3-aefb-82f1a2b9b32d"
group_replication_local_address= "192.168.80.111:33061"
group_replication_group_seeds= "192.168.80.110:33061,192.168.80.111:33061,192.168.80.112:33061"
loose-group_group_replication_single_primary_mode = ON;
[mysqld_safe]
log-error=/usr/local/mysql8/log/mysql_err.log
pid-file=/usr/local/mysql8/mysql8.pid

```

Mysql3的my.cnf

```bash
[mysqld]
user=mysql
basedir=/usr/local/mysql8
datadir=/usr/local/mysql8/data
server_id=112
port=3306
socket=/usr/local/mysql8/data/mysql.sock
gtid-mode=on
enforce-gtid-consistency=true
log_bin=/usr/local/mysql8/data/log-bin
binlog_format=row
binlog_checksum=NONE
skip-name-resolve
#master_info_repository=TABLE
#relay_log_info_repository=TABLE
log_slave_updates=ON

#transaction_write_set_extraction=XXHASH64
loose-group_replication_bootstrap_group=OFF
loose-group_replication_start_on_boot=ON
loose-group_replication_group_name="b0e3a840-0c38-4cb3-aefb-82f1a2b9b32d"
group_replication_local_address= "192.168.80.111:33061"
group_replication_group_seeds= "192.168.80.110:33061,192.168.80.111:33061,192.168.80.112:33061"
loose-group_group_replication_single_primary_mode = ON;
[mysqld_safe]
log-error=/usr/local/mysql8/log/mysql_err.log
pid-file=/usr/local/mysql8/mysql8.pid

```

更改完配置文件记得service mysqld restart 重启mysql

### 主库开通复制账户并安装plugins并启动

```sql
--110 机器上执行

./mysql -uroot --socket=/usr/local/mysql8/data/mysql.sock -p

CREATE USER 'replication_user'@'192.168.80.%' IDENTIFIED BY 'replication_pass';

GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'192.168.80.%';

flush privileges;

install plugin group_replication soname 'group_replication.so';
show plugins;

set sql_log_bin=0;
CHANGE REPLICATION SOURCE TO SOURCE_USER='replication_user', SOURCE_PASSWORD='replication_pass' FOR CHANNEL 'group_replication_recovery';
set sql_log_bin=1;

set global group_replication_bootstrap_group=on;
start group_replication;
set global group_replication_bootstrap_group=off;


--111/112分别执行
install plugin group_replication soname 'group_replication.so';

CHANGE REPLICATION SOURCE TO SOURCE_USER='replication_user', SOURCE_PASSWORD='replication_pass' FOR CHANNEL 'group_replication_recovery';

start group_replication;

如果从库状态不正确请执行以下命令:

-- 1. 停止 Group Replication
STOP GROUP_REPLICATION;

-- 2. 重置复制状态
RESET REPLICA ALL;

-- 3. 清理二进制日志
PURGE BINARY LOGS BEFORE NOW();

-- 4. 检查当前 GTID 状态
SELECT @@global.gtid_executed;
SELECT @@global.gtid_purged;

-- 5. 如果需要，清空 GTID_PURGED
SET GLOBAL gtid_purged='';

-- 6. 设置正确的 GTID 集合（需要从主节点获取正确的值）
SET GLOBAL gtid_purged='[从主节点获取的完整GTID集合]';  #在节点执行 SELECT @@global.gtid_executed;

-- 7. 重新配置复制源
CHANGE REPLICATION SOURCE TO SOURCE_USER='replication_user', SOURCE_PASSWORD='replication_pass' FOR CHANNEL 'group_replication_recovery';

-- 8. 启动 Group Replication
START GROUP_REPLICATION;

如果以上解决不了问题 建议删除data目录 直接从主库搞个过来. 如果生产环境可以用tx工具 你懂得.  记得删除 data下的auto.cnf   rm -r auto.cnf


```

**注意群组全部启停的情况需要注意顺序, 顺序为 停止顺序: 先停从后停主, 启动顺序先启动主后启动从.**

## MGR读写分离与故障转移

借助proxysql来实现读写分离更充分的利用数据库读库资源, 当然中间件来做读写分离稍有些性能损耗(网络传输可以忽略),还有单点问题 这个需要做高可用vip漂移.推荐参考 dawdler的实现方式[https://github.com/srchen1987/dawdler-series/tree/master/dawdler/dawdler-db-plug/dawdler-db-core#4-%E6%95%B0%E6%8D%AE%E6%BA%90%E8%A7%84%E5%88%99%E9%85%8D%E7%BD%AE](https://github.com/srchen1987/dawdler-series/tree/master/dawdler/dawdler-db-plug/dawdler-db-core#4-%E6%95%B0%E6%8D%AE%E6%BA%90%E8%A7%84%E5%88%99%E9%85%8D%E7%BD%AE)

### proxysql 安装

在113上用root用户 安装 yum localinstall -y proxysql-3.0.2-1-fedora41.x86_64.rpm

如果没有 rpm 请去<[https://github.com/sysown/proxysql/releases/tag/v3.0.2](https://github.com/sysown/proxysql/releases/tag/v3.0.2)> 官方下载

启动 systemctl start proxysql

端口说明:

6032 proxsql的管理端口

6033 proxsql对外服务端口

#### master(110)中为proxysql创建视图

```sql
--为proxysql提供判断节点状态的视图
USE sys;

DELIMITER $$

CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$

CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$

CREATE VIEW gr_member_routing_candidate_status AS 
SELECT
    -- 替换 sys.gr_member_in_primary_partition()
    IF(
        rgm.MEMBER_STATE = 'ONLINE' 
        AND (SELECT COUNT(*) FROM performance_schema.replication_group_members 
             WHERE MEMBER_STATE != 'ONLINE') = 0,
        'YES', 
        'NO'
    ) AS viable_candidate,
     
    -- 保持原有的 read_only 检查
    IF(
        (SELECT GROUP_CONCAT(variable_value) 
         FROM performance_schema.global_variables 
         WHERE variable_name IN ('read_only', 'super_read_only')) != 'OFF,OFF', 
        'YES', 
        'NO'
    ) AS read_only,
               
    -- 替换 sys.gr_applier_queue_length()
    (SELECT COUNT(*) 
     FROM performance_schema.replication_applier_status_by_worker 
     WHERE SERVICE_STATE = 'ON') 
    AS transactions_behind,
    
    -- 保持原有的字段
    rgms.Count_Transactions_in_queue AS transactions_to_cert
    
FROM performance_schema.replication_group_member_stats rgms
JOIN performance_schema.replication_group_members rgm 
  ON rgms.MEMBER_ID = rgm.MEMBER_ID$$

DELIMITER ;

```

#### 创建监控与proxysql账号

在master(110)中执行以下sql

 创建监控账号

```sql
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';

-- 授予必要的监控权限
GRANT SELECT ON performance_schema.* TO 'monitor'@'%';
GRANT SELECT ON sys.* TO 'monitor'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'monitor'@'%';

-- 刷新权限
FLUSH PRIVILEGES;
```

 创建proxysql账号

```sql
CREATE USER 'proxysql'@'%' IDENTIFIED BY 'proxysql';

--为方便操作授权了select和update全部的库和表 根据实际业务调整
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON *.* TO 'proxysql'@'%';

-- 刷新权限
FLUSH PRIVILEGES;
```

#### 验证视图与账号是否创建

分别在 110 111 112 上执行 看看视图同步创建没有
SELECT * FROM sys.gr_member_routing_candidate_status;

另外查询下mysql的用户表看看创建的账号信息是否同步

#### 设置proxysql

proxysql节点(proxysql 113)上安装mysql客户端

```shell
yum install -y mysql.x86_64
```

登录proxysql admin管理接口,proxysql的配置在 /etc/proxysql.cnf

```sql
mysql -uadmin -padmin --prompt='proxysql>' -P6032 -h127.0.0.1
```

添加节点

```sql
insert into mysql_servers(hostgroup_id,hostname,port,weight,max_connections,max_replication_lag,comment) values (10,'192.168.80.110',3306,1,3000,10,'mysql1');

insert into mysql_servers(hostgroup_id,hostname,port,weight,max_connections,max_replication_lag,comment) values (10,'192.168.80.111',3306,1,3000,10,'mysql2');

insert into mysql_servers(hostgroup_id,hostname,port,weight,max_connections,max_replication_lag,comment) values (10,'192.168.80.112',3306,1,3000,10,'mysql3');
```

设置监控者账号密码(上面在mysql的master上有过创建的)

```sql
set mysql-monitor_username='monitor';
set mysql-monitor_password='monitor';
```

添加访问账号

```sql
insert into mysql_users(username,password,active,default_hostgroup,transaction_persistent)values('proxysql','proxysql',1,10,1);
```

热加载到runtime并将配置持久化存储到硬盘

```sql
-- 刷新所有配置
LOAD MYSQL USERS TO RUNTIME;
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
LOAD MYSQL VARIABLES TO RUNTIME;
LOAD ADMIN VARIABLES TO RUNTIME;

-- 保存到磁盘
SAVE MYSQL USERS TO DISK;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;
SAVE MYSQL VARIABLES TO DISK;
SAVE ADMIN VARIABLES TO DISK;
```

更改mysql版本 默认是5.5.30

```sql
update global_variables set variable_value="8.4.3" where variable_name="mysql-server_version";
LOAD MYSQL VARIABLES TO RUN;
SAVE MYSQL VARIABLES TO DISK;
```

定义组信息

```sql
insert into mysql_group_replication_hostgroups(writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind)  values(10,20,30,40,1,1,0,0);
```

查看链接情况

```sql
select * from monitor.mysql_server_connect_log;
select * from monitor.mysql_server_ping_log;
```

查看服务器目前分组情况

```sql
select hostgroup_id,hostname,port,status,weight from mysql_servers;
```

#### proxysql设置读写分离规则

```sql
insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)values(1,1,'^SELECT.*FOR UPDATE$',10,1); --10为写

insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)values(2,1,'^SELECT',30,1);-- 30为读
```

查看MGR配置信息

```sql
select * from mysql_group_replication_hostgroups\G；
```

确认链接信息和ping信息,如果以上最后一列为NULL 则正常 否则需要处理 具体看 mysql错误日志和proxysql的错误日志

```sql
SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us;

SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us;
```

查看各节点分组的信息

```sql
select hostgroup_id, hostname, port,status from runtime_mysql_servers;
```

sql测试读写分离

通过循环10次调用看看是否负载均衡 是否走到从库(也称为读库),通过server_id来确认不同机器

读测试

```bash
for i in `seq 1 10`; do mysql -uproxysql -pproxysql -h192.168.80.113  -P6033 -e 'SELECT @@server_id'; done
```

写测试(模拟写 因为用了 for update 也会走写规则)

```bash
for i in `seq 1 10`; do mysql -uproxysql -pproxysql -h192.168.80.113 -P6033 -e 'SELECT @@server_id for update'; done
```

### 故障转移测试

在主节点(110)执行以下命令

```sql
SELECT * FROM performance_schema.replication_group_members;
```

可以看到返回的结果如下：

```shell
mysql> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 754d27be-8258-11f0-a5e6-525400b26eb7 | mgr-node1   |        3306 | ONLINE       | PRIMARY   | 8.4.3          | XCom                       |
| group_replication_applier | d0a51766-8272-11f0-8b37-52540069f9d5 | mgr-node3   |        3306 | ONLINE       | SECONDARY   | 8.4.3          | XCom                       |
| group_replication_applier | e11c57cb-826e-11f0-867b-52540097b806 | mgr-node2   |        3306 | ONLINE       | SECONDARY   | 8.4.3          | XCom                       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+

```

主库停机(110) service mysqld stop

通过任意一台mysql节点的client连接到proxysql节点(113)创建数据库测试

```bash
# mysql -uproxysql -pproxysql -h192.168.80.113 -P6033 -e 'create database mydemo;'
```

经过以上步骤,依旧可以创建数据库.

在主库停机后,从库(111,112)上执行以下命令查看当前运行服务器列状态.

```sql
SELECT * FROM performance_schema.replication_group_members;
```

可以看到从库(111,112)的运行状态为ONLINE,mgr-node1(110)不见了,同时mgr-node3的MEMBER_ROLE为PRIMARY.

```shell
mysql> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | d0a51766-8272-11f0-8b37-52540069f9d5 | mgr-node3   |        3306 | ONLINE       | PRIMARY     | 8.4.3          | XCom                       |
| group_replication_applier | e11c57cb-826e-11f0-867b-52540097b806 | mgr-node2   |        3306 | ONLINE       | SECONDARY   | 8.4.3          | XCom                       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+

```

#### slave1,slave2查看结果

在从库中执行 (111,112)

show database;

验证下 数据库已经创建成功

查看当前运行服务器列状态

这个需要在proxysql上执行,因为没有admin授权其他ip可访问,如果需要授权其他ip可访问,则需要授权.

```bash
mysql -uadmin -padmin -P6032 -h127.0.0.1 -e 'select hostgroup_id,hostname,port,status from runtime_mysql_servers;'
```

重新启动主库(110)

```bash
/etc/init.d/mysqld start
```

查看具体目前服务器运行状况

```bash
mysql -uadmin -padmin -P6032 -h127.0.0.1 -e 'select hostgroup_id,hostname,port,status from runtime_mysql_servers;'
```

在主库(110)上执行以下命令查看当前运行服务器列状态.

```sql
SELECT * FROM performance_schema.replication_group_members;
```

```shell
mysql> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 754d27be-8258-11f0-a5e6-525400b26eb7 | mgr-node1   |        3306 | ONLINE       | SECONDARY   | 8.4.3          | XCom                       |
| group_replication_applier | d0a51766-8272-11f0-8b37-52540069f9d5 | mgr-node3   |        3306 | ONLINE       | PRIMARY     | 8.4.3          | XCom                       |
| group_replication_applier | e11c57cb-826e-11f0-867b-52540097b806 | mgr-node2   |        3306 | ONLINE       | SECONDARY   | 8.4.3          | XCom                       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+

```

#### 一些常用命令

系统模式中存在哪些函数

SHOW FUNCTION STATUS WHERE Db='sys';

proxysql的一些统计

查看查询统计

```sql
-- 查看查询摘要统计
SELECT * FROM stats_mysql_query_digest ORDER BY sum_time DESC LIMIT 10;

-- 查看查询规则匹配情况
SELECT * FROM stats_mysql_query_rules;
```

查看连接统计

```sql
-- 查看连接池统计
SELECT * FROM stats_mysql_connection_pool;

-- 查看连接统计
SELECT * FROM stats_mysql_processlist;

-- 查看命令统计
SELECT * FROM stats_mysql_commands_counters;
```

实时监控查询

```sql
-- 实时查看当前查询
SELECT * FROM stats_mysql_processlist WHERE Info IS NOT NULL;

-- 查看最近的查询（需要启用查询日志）
SELECT * FROM stats_history_mysql_query_digest ORDER BY timestamp DESC LIMIT 20;
```

查看所有代理的mysql服务器

```sql
SELECT * FROM mysql_servers;
```

查看主库名称

```sql
show status like 'group_replication_primary_member';
```

查看只读情况

```sql
show variables like '%read_on%';
```

查看mysql的GTID状态

```sql
SHOW VARIABLES LIKE '%gtid%';
```

查看mysql的MGR状态

```sql
SHOW VARIABLES LIKE '%group%replication%';
```

查看二进制日志状态（替代 SHOW MASTER STATUS）

```sql
SHOW BINARY LOG STATUS;
```

查看所有二进制日志

```sql
SHOW BINARY LOGS;
```

查看作为主库的复制状态

```sql
SHOW REPLICAS;
```

查看作为从库的复制状态

```sql
SHOW REPLICA STATUS\G;
```

查看组成员状态

```sql
SELECT * FROM performance_schema.replication_group_members\G;
```

查看当前机器的server_uuid

```sql
SELECT @@global.server_uuid;
```

查看组成员统计信息

```sql
SELECT * FROM performance_schema.replication_group_member_stats\G;
```

查看连接状态

```sql
SELECT * FROM performance_schema.replication_connection_status\G;
```

友情提醒 生产环境下要把proxysql的密码设置复杂些,具体在/etc/proxysql.cnf中,另外各种网络权限也要设置好同步和管理的端口不要暴露在公网.
