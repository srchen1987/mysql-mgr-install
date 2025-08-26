# 基于mgr搭建高可用方案

## mysql版本
注意:这里采用的是mysql8.0.x  如果是 8.4.x或9.x请参考官方文档调整参数.

例如:
```shell
#注释掉此配置
#transaction_write_set_extraction=XXHASH64
```

## 简介

MGR是mysql官方推荐的高可用集群方案,支持单主或多主模式，可随时切换，最少3台机器才能实现高可用方案,少于3台无法实现,会导致脑裂，无法提供服务.

允许故障数量

3台允许挂1台,5台允许挂2台,7台允许挂3台以此类推.

## 准备工作

#### 准备3台主机,并安装MySQL

1、四台机器 192.168.80.110,192.168.80.111,192.168.80.112,192.168.80.113

分别更改 host文件 mysql1 mysql2 mysql3,proxysql  对应上面ip

| hosts    | ip             | member   |
| -------- | -------------- | -------- |
| mysql1   | 192.168.80.110 | master   |
| mysql2   | 192.168.80.111 | slave    |
| mysql3   | 192.168.80.112 | slave    |
| proxysql | 192.168.80.113 | proxysql |



2、下载并安装mysql 参考 https://github.com/srchen1987/mysql_install

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

transaction_write_set_extraction=XXHASH64
loose-group_replication_bootstrap_group=OFF
loose-group_replication_start_on_boot=OFF
loose-group_replication_group_name="b0e3a840-0c38-4cb3-aefb-82f1a2b9b32d"
loose-group_replication_local_address= "192.168.80.110:33061"
loose-group_replication_group_seeds= "192.168.80.110:33061,192.168.80.111:33061,192.168.80.112:33061"
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

transaction_write_set_extraction=XXHASH64
loose-group_replication_bootstrap_group=OFF
loose-group_replication_start_on_boot=OFF
loose-group_replication_group_name="b0e3a840-0c38-4cb3-aefb-82f1a2b9b32d"
loose-group_replication_local_address= "192.168.80.111:33061"
loose-group_replication_group_seeds= "192.168.80.110:33061,192.168.80.111:33061,192.168.80.112:33061"
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

transaction_write_set_extraction=XXHASH64
loose-group_replication_bootstrap_group=OFF
loose-group_replication_start_on_boot=OFF
loose-group_replication_group_name="b0e3a840-0c38-4cb3-aefb-82f1a2b9b32d"
loose-group_replication_local_address= "192.168.80.111:33061"
loose-group_replication_group_seeds= "192.168.80.110:33061,192.168.80.111:33061,192.168.80.112:33061"
loose-group_group_replication_single_primary_mode = ON;
[mysqld_safe]
log-error=/usr/local/mysql8/log/mysql_err.log
pid-file=/usr/local/mysql8/mysql8.pid

```

更改完配置文件记得service mysqld restart 重启mysql

####    主库开通复制账户主从都安装plugins并启动

```sql
--110 机器上执行

CREATE USER 'replication_user'@'192.168.80.%' IDENTIFIED BY 'replication_pass';

GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'192.168.80.%';

flush privileges;

install plugin group_replication soname 'group_replication.so';
show plugins;

set sql_log_bin=0;
change master to master_user='replication_user',master_password='replication_pass' for channel 'group_replication_recovery';
set sql_log_bin=1;

set global group_replication_bootstrap_group=on;
start group_replication;
set global group_replication_bootstrap_group=off;

select * from performance_schema.replication_group_members;

--111/112分别执行
install plugin group_replication soname 'group_replication.so';

change master to master_user='replication_user',master_password='replication_pass' for channel 'group_replication_recovery';

set global group_replication_allow_local_disjoint_gtids_join=ON;

start group_replication;

--查看主库名称
show status like 'group_replication_primary_member';
--查看只读情况
show variables like '%read_on%';
```



**注意群组全部启停的情况需要注意顺序, 顺序为 停止顺序: 先停从后停主, 启动顺序先启动主后启动从.**



## MGR读写分离与故障转移

借助proxysql来实现读写分离更充分的利用数据库读库资源, 当然中间件来做读写分离稍有些性能损耗(网络传输可以忽略),还有单点问题 这个需要做高可用vip漂移.推荐参考 dawdler的实现方式(https://github.com/srchen1987/dawdler-series)

#### proxysql 安装 



在113上用root用户 安装 yum localinstall -y proxy-2.0.16-1-centos7.x86_64.rpm

如果没有 rpm 请去https://github.com/sysown/proxysql/releases/tag/v2.0.16 官方下载

启动 systemctl start proxysql

#6032 proxsql的管理端口
#6033 proxsql对外服务端口

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

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$

DELIMITER ;

```



#### 创建监控与proxysql帐号

在master(111)中执行以下sql

 创建监控帐号

```sql
grant select on sys.* to 'monitor'@'%' identified by 'monitor';
```

 创建proxysql帐号

```sql
grant all on *.* to 'proxysql'@'%' identified by 'proxysql';
```



#### 验证视图与帐号是否创建

分别在 110 111 112 上执行 看看视图同步创建没有
SELECT * FROM sys.gr_member_routing_candidate_status;

另外查询下mysql的用户表看看创建的帐号信息是否同步



#### 设置proxysql

```sql
--proxysql节点(proxysql 113)上登录proxysql admin管理接口 proxysql的配置在 /etc/proxysql.cnf中
mysql -uadmin -padmin --prompt='proxysql>' -P6032 -h192.168.80.113
```

添加节点

```sql
insert into mysql_servers(hostgroup_id,hostname,port,weight,max_connections,max_replication_lag,comment) values (10,'192.168.80.110',3306,1,3000,10,'mysql1');

insert into mysql_servers(hostgroup_id,hostname,port,weight,max_connections,max_replication_lag,comment) values (10,'192.168.80.111',3306,1,3000,10,'mysql2');

insert into mysql_servers(hostgroup_id,hostname,port,weight,max_connections,max_replication_lag,comment) values (10,'192.168.80.112',3306,1,3000,10,'mysql3');
```

设置监控者帐号密码(上面在mysql的master上有过创建的)

```sql
set mysql-monitor_username='monitor';
set mysql-monitor_password='monitor';
```

添加访问帐号

```sql
insert into mysql_users(username,password,active,default_hostgroup,transaction_persistent)values('proxysql','proxysql',1,10,1);
```



热加载到runtime并将配置持久化存储到硬盘

```sql
load mysql servers to runtime;
save mysql servers to disk;
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
select * from mysql_group_replication_hostgroups\G
```



确认链接信息和ping信息,如果以上最后一列为NULL 则正常 否则需要处理 具体看 mysql错误日志和proxysql的错误日志

```sql
SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us ;

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
for i in `seq 1 10`; do mysql -uproxysql -pproxysql -h192.168.80.113 -P6033 -e 'select * from performance_schema.global_variables where variable_name="server_id";' ; done  | grep server

```

写测试(模拟写 因为用了 for update 也会走写规则)

```bash
for i in `seq 1 10`; do mysql -uproxysql -pproxysql -h192.168.80.113 -P6033 -e 'select * from performance_schema.global_variables where variable_name="server_id" for update;' ; done  | grep server
```



##### 故障转移测试

主库停机(110) service mysqld stop

创建数据库测试

```bash
mysql -uproxysql -pproxysql -h192.168.80.113 -P6033 -e 'create database mydemo;'
```

#slave1,slave2查看结果；

在从库中执行 (111,112)

show database; 

验证下 数据库是否创建了



查看当前运行服务器列状态

```bash
mysql -uadmin -padmin -P6032 -h192.168.80.113 -e 'select hostgroup_id,hostname,port,status from runtime_mysql_servers;'
```

重新启动主库(110)

```bash
/etc/init.d/mysqld start
```



登录主库

执行以下脚本

```bash
stop group_replication;

change master to master_user='replication_user',master_password='replication_pass' for channel 'group_replication_recovery';

start group_replication;


```

查看具体目前服务器运行状况

```bash
mysql -uadmin -padmin -P6032 -h192.168.80.113 -e 'select hostgroup_id,hostname,port,status from runtime_mysql_servers;'
```



友情提醒 生产环境下要把proxysql的密码设置复杂些,具体在/etc/proxysql.cnf中,另外各种网络权限也要设置好同步和管理的端口不要暴露在公网.

