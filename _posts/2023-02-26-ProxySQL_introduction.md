---
id: 1302
title: 'ProxySQL简介'
date: '2023-02-26T18:34:40+08:00'
author: Shuo
excerpt: 'ProxySQL简介,ProxySQL被用作数据库代理，完成读写分离、routing等操作'
layout: post
guid: 'http://codercoder.cn/?p=1302'
image: http://codercoder.cn/wp-content/uploads/2023/02/2023-02-26-ProxySQL_introduction-1.png
permalink: /index.php/2023/02/2023-02-26-ProxySQL_introduction
categories:
    - Tech
tags:
    - MySQL
    - Tech
---
# 一、用途
ProxySQL被用作数据库代理，完成读写分离、routing等操作。目前使用版本：2.3.2。


> **注意：**
>
> （1）通常不建议使用数据库代理，因为在流量高时，proxy容易成为性能瓶颈。
> （2）在开发client端的过渡期使用，例如目前jdbc需要支持group replication等高可用架构。

对于ProxySQL对于group replication的支持，参考：
[MySQL Group Replication: native support in ProxySQL](https://lefred.be/content/mysql-group-replication-native-support-in-proxysql/)
[ProxySQL 2.3.0: Enhanced Support for MySQL Group Replication](https://www.percona.com/blog/proxysql-2-3-0-enhanced-support-for-mysql-group-replication/)


# 二、性能
## 性能开支：
**（1）Network Overhead**

**（2）Processing Overhead：**

* Query size
* Result set size
* ProxySQL configuration
* Query rules
* Mysql-threads: `limits use no more than 4 CPU cores.`
* Socket/TCP
* Query Cache：对内存的消耗


## 优势
### 更稳定
* **相比MaxScale**

参考：[ProxySQL versus MaxScale for OLTP RO workloads](https://www.percona.com/blog/proxysql-versus-maxscale-for-oltp-ro-workloads/)

* **Query Cache**

参考：[ProxySQL Query Cache: What It Is, How It Works](https://www.percona.com/blog/2018/02/07/proxysql-query-cache/)

### 多路复用
[Mysql Proxysql 多路复用到底有多大作用](https://cloud.tencent.com/developer/article/1722148)

即：

PROXYSQL将连接到自身的访问，来复用MYSQL数据库本身的连接，达到分时复用的效果，让少开连接，让一个连接承接多个访问。


# 三、架构
## 多层配置系统
文档：[Multi layer configuration system](https://proxysql.com/Documentation/configuring-proxySQL/)

**三层分层结构：**
### （1）RUNTIME
处理请求的工作线程使用的ProxySQL内存数据结构。

包含：
* **(1) the actual values defined in the global variables**
全局的真实变量

* **(2) the list of backend servers grouped into hostgroups**
hostgroup中的后端servers

* **(3) the list of MySQL users that can connect to the proxy**
能够连接proxy的用户

Note:

Operators can never modify the contents of the RUNTIME configuration section directly.

不能直接修改RUNTIME阶段的参数

### MEMORY
(also referred to as main) 

layer represents an in-memory database which is exposed via a MySQL-compatible interface. 
Users can connect with a MySQL client to this interface and view / edit the various ProxySQL configuration tables.

同main，表示与MySQL接口兼容的内存数据库。用户可进行查看和编辑。

包括以下的表：
* mysql_servers
后端连接的MySQL的主机列表

* mysql_users 
需要连接到proxySQL的所有用户信息

* mysql_query_rules 
路由规则

* global_variables
参数配置

* mysql_collations


### DISK
将配置信息，持久化到磁盘


**架构图：**
![](http://codercoder.cn/wp-content/uploads/2023/02/2023-02-26-ProxySQL_introduction-1.png)

This is achieved through a multi-layer configuration system where settings are moved from runtime to memory, and persisted to disk as desired.

配置信息，从RUNTIME移动到MEMORY，并根据需要持久化到磁盘。

## 优势
**（1）Easy-to-use dynamic runtime configuration system to ensure zero-downtime changes**
变更配置不需要停服，可直接修改RUMTIME配置

**（2）Effortless rollback of configuration to its previous state**
可无影响地回滚配置到之前的状态

**（3）A MySQL-compatible administrative interface**
与MySQL兼容的管理接口


**注意事项：**
* Note that all the content of the in-memory tables (main database) are lost when ProxySQL is restarted if their content wasn’t saved on disk database.
  内存的配置会在proxysql重启时清空。

* The “disk” database has exactly the same tables as the “main” database

# 四、使用
## 部署
直接使用对应的rpm包进行安装。

（集群版本）
下载对应版本的rpm包：[https://www.percona.com/download-proxysql](https://www.percona.com/download-proxysql)

```
rpm -ivh proxysql-2.3.2-1-centos7.x86_64.rpm
```


## 配置

配置项说明：[ProxySQL Global Variables](https://proxysql.com/documentation/global-variables/)

**配置文件：**

```
cat /etc/proxysql.cnf 

datadir="${data_disk}/proxysql"
errorlog="${data_disk}/proxysql/proxysql.log"
admin_variables=
{
	admin_credentials="admin:$proxy_pwd"
	mysql_ifaces="0.0.0.0:6032"
}
mysql_variables=
{
	threads=4
	max_connections=5000
	default_query_delay=0
	default_query_timeout=36000000
	have_compress=true
	poll_timeout=2000
	interfaces="0.0.0.0:$app_port;/tmp/proxysql.sock"
	default_schema="information_schema"
	stacksize=1048576
	server_version="8.0.27"
	connect_timeout_server=3000
	monitor_username="monitor"
	monitor_password="monitor"
	monitor_history=600000
	monitor_connect_interval=60000
	monitor_ping_interval=10000
	monitor_read_only_interval=1500
	monitor_read_only_timeout=500
	ping_interval_server_msec=120000
	ping_timeout_server=500
	commands_stats=true
	sessions_sort=true
	connect_retries_on_failure=10
}
mysql_servers =
(
)
mysql_users:
(
)
mysql_query_rules:
(
)
scheduler=
(
)
mysql_replication_hostgroups=
(
)

```

**（1）重要配置项：**

* **服务端口：** 可指定
* **管理端口：** 默认6032，只能在本地登录
* **查看表：**
`show tables from stats;`
注意：不能使用use database，然后再show tables;会得到不准确的结果！！

**（2）常用表：**
* mysql_users

存储连接proxysql的用户信息

* mysql_servers

存储mysql实例列表

* mysql_group_replication_hostgroups

存储group replication的分组信息（Read/Write）

* mysql_query_rules

SQL的路由规则

**注意事项：**

（1）在添加mysql server时，备份节点不要加入到proxysql中。


## 管理
变量：[ProxySQL-MySQL Variables](https://proxysql.com/documentation/global-variables/mysql-variables/)

统计信息：[ProxySQL-STATS (statistics)](https://proxysql.com/documentation/stats-statistics/#stats_mysql_global)

### 查看连接情况
各个节点之间的连接情况：

```
MySQL [(none)]> select * from stats_mysql_connection_pool;
+-----------+---------------+----------+--------------+----------+----------+--------+---------+-------------+---------+-------------------+-----------------+-----------------+------------+
| hostgroup | srv_host      | srv_port | status       | ConnUsed | ConnFree | ConnOK | ConnERR | MaxConnUsed | Queries | Queries_GTID_sync | Bytes_data_sent | Bytes_data_recv | Latency_us |
+-----------+---------------+----------+--------------+----------+----------+--------+---------+-------------+---------+-------------------+-----------------+-----------------+------------+
| 10        | 172.16.236.81 | 3306     | ONLINE       | 0        | 1        | 1      | 0       | 1           | 2       | 0                 | 58              | 306             | 450        |
| 30        | 172.16.236.83 | 3306     | ONLINE       | 0        | 1        | 1      | 0       | 1           | 1151    | 0                 | 9364            | 12150           | 204        |
| 30        | 172.16.236.81 | 3306     | ONLINE       | 0        | 1        | 1      | 0       | 1           | 2161    | 0                 | 17349           | 18222           | 450        |
| 30        | 172.16.236.84 | 3306     | ONLINE       | 0        | 1        | 1      | 0       | 1           | 156     | 0                 | 1309            | 6192            | 428        |
| 30        | 172.16.236.82 | 3306     | OFFLINE_HARD | 0        | 0        | 1      | 0       | 1           | 1       | 0                 | 8               | 6               | 535        |
| 40        | 172.16.236.81 | 3306     | OFFLINE_HARD | 0        | 0        | 0      | 0       | 0           | 0       | 0                 | 0               | 0               | 450        |
| 40        | 172.16.236.83 | 3306     | OFFLINE_HARD | 0        | 0        | 0      | 0       | 0           | 0       | 0                 | 0               | 0               | 204        |
| 40        | 172.16.236.84 | 3306     | OFFLINE_HARD | 0        | 0        | 0      | 0       | 0           | 0       | 0                 | 0               | 0               | 428        |
| 40        | 172.16.236.82 | 3306     | ONLINE       | 0        | 0        | 0      | 0       | 0           | 0       | 0                 | 0               | 0               | 535        |
+-----------+---------------+----------+--------------+----------+----------+--------+---------+-------------+---------+-------------------+-----------------+-----------------+------------+
9 rows in set (0.01 sec)
```

**Latency_us: ** 

the current ping time in microseconds, as reported from Monitor

**status：**

当前MySQL实例的状态，ONLINE为正常，若出现其它状态，需要查看错误日志，根据报错信息修复集群。

### 连接数使用情况
最大连接数，已使用连接数等信息：

```
MySQL [(none)]> select * from stats.stats_mysql_global where variable_name like '%connect%';
+--------------------------------------+----------------+
| Variable_Name                        | Variable_Value |
+--------------------------------------+----------------+
| Client_Connections_aborted           | 0              |
| Client_Connections_connected         | 0              |
| Client_Connections_created           | 24388          |
| Server_Connections_aborted           | 1              |
| Server_Connections_connected         | 5              |
| Server_Connections_created           | 28             |
| Server_Connections_delayed           | 0              |
| Client_Connections_non_idle          | 0              |
| mysql_killed_backend_connections     | 0              |
| max_connect_timeouts                 | 0              |
| client_host_error_killed_connections | 0              |
| Client_Connections_hostgroup_locked  | 0              |
| Access_Denied_Max_Connections        | 0              |
| Access_Denied_Max_User_Connections   | 0              |
| MySQL_Monitor_connect_check_OK       | 45135          |
| MySQL_Monitor_connect_check_ERR      | 1              |
+--------------------------------------+----------------+
16 rows in set (0.01 sec)
```

**用户的最大连接数**：
```
MySQL [(none)]> select username, active, transaction_persistent, max_connections from mysql_users;
+----------+--------+------------------------+-----------------+
| username | active | transaction_persistent | max_connections |
+----------+--------+------------------------+-----------------+
| dba | 1      | 1                      | 10000           |
| app      | 1      | 1                      | 10000           |
+----------+--------+------------------------+-----------------+
2 rows in set (0.00 sec)
```

**各个server的最大连接数**：
```
MySQL [(none)]> select * from mysql_servers;
+--------------+---------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname      | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+---------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | 172.16.236.81 | 3306 | 0         | ONLINE | 1      | 0           | 3000            | 0                   | 0       | 0              |         |
| 10           | 172.16.236.82 | 3306 | 0         | ONLINE | 1      | 0           | 3000            | 0                   | 0       | 0              |         |
| 10           | 172.16.236.83 | 3306 | 0         | ONLINE | 1      | 0           | 3000            | 0                   | 0       | 0              |         |
| 10           | 172.16.236.84 | 3306 | 0         | ONLINE | 1      | 0           | 3000            | 0                   | 0       | 0              |         |
+--------------+---------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
4 rows in set (0.00 sec)
```

### Query相关状态监控
可使用以下的状态信息，定位部分常见问题，例如：慢查询、最消耗内存的SQL类型、SQL分布等。


**注意：**
需要做好状态监控信息的限制，避免监控使用过多的资源，倒是proxysql成为性能瓶颈。


**查询解析相关**
```
mysql> show variables like "%query%";
+---------------------------------------------------+----------+
| Variable_name                                     | Value    |
+---------------------------------------------------+----------+
| admin-stats_mysql_query_cache                     | 60       |
| admin-stats_mysql_query_digest_to_disk            | 0        |
| admin-cluster_mysql_query_rules_diffs_before_sync | 3        |
| admin-cluster_mysql_query_rules_save_to_disk      | true     |
| admin-checksum_mysql_query_rules                  | true     |
| mysql-query_retries_on_failure                    | 1        |
| mysql-monitor_query_interval                      | 60000    |
| mysql-monitor_query_timeout                       | 100      |
| mysql-verbose_query_error                         | false    |
| mysql-threshold_query_length                      | 524288   |
| mysql-query_digests_max_digest_length             | 2048     |
| mysql-query_digests_max_query_length              | 65000    |
| mysql-query_digests_grouping_limit                | 3        |
| mysql-default_query_delay                         | 0        |
| mysql-default_query_timeout                       | 36000000 |
| mysql-query_processor_iterations                  | 0        |
| mysql-query_processor_regex                       | 1        |
| mysql-set_query_lock_on_hostgroup                 | 1        |
| mysql-long_query_time                             | 1000     |
| mysql-query_cache_size_MB                         | 256      |
| mysql-query_digests                               | true     |
| mysql-query_digests_lowercase                     | false    |
| mysql-query_digests_replace_null                  | false    |
| mysql-query_digests_no_digits                     | false    |
| mysql-query_digests_normalize_digest_text         | false    |
| mysql-query_digests_track_hostname                | false    |
| mysql-stats_time_backend_query                    | false    |
| mysql-stats_time_query_processor                  | false    |
| mysql-query_cache_stores_empty_result             | true     |
+---------------------------------------------------+----------+
29 rows in set (0.01 sec)
```

**慢查阈值：**
```
mysql> show variables like "%long_query_time";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| mysql-long_query_time | 1000  |
+-----------------------+-------+
1 row in set (0.00 sec)

```
**慢查次数：**
```
mysql> select * from stats_mysql_global where  variable_name like '%slow%';
+---------------+----------------+
| Variable_Name | Variable_Value |
+---------------+----------------+
| Slow_queries  | 235            |
+---------------+----------------+
1 row in set (0.02 sec)
```

**query明细：**
```
mysql> select count_star,sum_time,(sum_time/count_star)/1000 as average_time_ms,digest_text from stats_mysql_query_digest where count_star > 100 order by average_time_ms desc limit 10;mysql>  select count_star,sum_time,(sum_time/count_star)/1000 as average_time_ms,digest_text from stats_mysql_query_digest where count_star > 100 order by average_time_ms desc limit 10 \G
*************************** 1. row ***************************
     count_star: 473
       sum_time: 62225750
average_time_ms: 131
    digest_text: SELECT id,biz_id,parent_folder_id,name,description,run_type,version,basic_info,configs,script,extra,comment,submit_status,quality_check,owner,tenant_id,workspace_id,create_user,modified_user,env,region,gmt_create,gmt_modified,id FROM datafactory_job_draft WHERE (parent_folder_id IN (?,?,?,...) AND env = ? AND delete_status = ?)
```

**内存的消耗：**
```
mysql> select * from stats_memory_metrics ;
+------------------------------+----------------+
| Variable_Name                | Variable_Value |
+------------------------------+----------------+
| SQLite3_memory_bytes         | 5454816        |
| jemalloc_resident            | 1045889024     |
| jemalloc_active              | 1015087104     |
| jemalloc_allocated           | 920459848      |
| jemalloc_mapped              | 1075642368     |
| jemalloc_metadata            | 21453944       |
| jemalloc_retained            | 322109440      |
| Auth_memory                  | 322831         |
| query_digest_memory          | 2591672        |
| mysql_query_rules_memory     | 9170           |
| mysql_firewall_users_table   | 0              |
| mysql_firewall_users_config  | 0              |
| mysql_firewall_rules_table   | 0              |
| mysql_firewall_rules_config  | 329            |
| stack_memory_mysql_threads   | 67108864       |
| stack_memory_admin_threads   | 8388608        |
| stack_memory_cluster_threads | 0              |
+------------------------------+----------------+
17 rows in set (0.01 sec)
```

**各类SQL执行情况：**
```
mysql> select * from stats_mysql_commands_counters order by Total_cnt desc limit 10;
+-----------------+---------------+-----------+-----------+-----------+---------+----------+----------+----------+-----------+-----------+--------+--------+---------+----------+
| Command         | Total_Time_us | Total_cnt | cnt_100us | cnt_500us | cnt_1ms | cnt_5ms  | cnt_10ms | cnt_50ms | cnt_100ms | cnt_500ms | cnt_1s | cnt_5s | cnt_10s | cnt_INFs |
+-----------------+---------------+-----------+-----------+-----------+---------+----------+----------+----------+-----------+-----------+--------+--------+---------+----------+
| SELECT          | 130008831955  | 47524378  | 332203    | 26309     | 2000277 | 39389302 | 5502284  | 266615   | 4850      | 2395      | 78     | 48     | 1       | 16       |
| SET             | 421597        | 20461892  | 20461825  | 0         | 0       | 11       | 54       | 2        | 0         | 0         | 0      | 0      | 0       | 0        |
| SHOW            | 6262081       | 13450036  | 13449288  | 0         | 1       | 244      | 183      | 319      | 0         | 0         | 0      | 0      | 0       | 1        |
| DELETE          | 10782243027   | 4904396   | 149       | 22403     | 648738  | 3715697  | 469785   | 47282    | 205       | 130       | 2      | 4      | 0       | 1        |
| UPDATE          | 5545025286    | 830182    | 218       | 2         | 8905    | 409273   | 317228   | 93273    | 561       | 565       | 43     | 91     | 7       | 16       |
| INSERT          | 4264675133    | 549151    | 38        | 0         | 296     | 122116   | 359315   | 66385    | 508       | 453       | 19     | 8      | 9       | 4        |
| COMMIT          | 179023175     | 282775    | 255770    | 0         | 5       | 11126    | 13455    | 2358     | 38        | 21        | 1      | 1      | 0       | 0        |
| REPLACE         | 202926277     | 56660     | 1         | 0         | 6       | 49033    | 6995     | 607      | 9         | 8         | 1      | 0      | 0       | 0        |
| CREATE_TABLE    | 378682371     | 56386     | 0         | 0         | 0       | 20801    | 32954    | 2437     | 121       | 71        | 2      | 0      | 0       | 0        |
| CREATE_DATABASE | 524547879     | 56268     | 0         | 0         | 0       | 20366    | 32411    | 3135     | 206       | 97        | 25     | 23     | 2       | 3        |
+-----------------+---------------+-----------+-----------+-----------+---------+----------+----------+----------+-----------+-----------+--------+--------+---------+----------+
10 rows in set (0.01 sec)
```



# 四、其它问题
## （1）mysql servers
1、备份节点不加入管理列表中

2、若有冗余的机器，可作为backup hostgroup

## (2)mysql users
1、用户有默认的hostgroup，需要设置为write

避免用户写入失败的情况

2、mysql_users表，需要重点配置各个用户的属性：
```
mysql> select * from mysql_users limit 1 \G
*************************** 1. row ***************************
              username: dba
              password: ***
                active: 1
               use_ssl: 0
     default_hostgroup: 10
        default_schema:
         schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
               backend: 0
              frontend: 1
       max_connections: 10000
            attributes:
               comment:
1 row in set (0.02 sec)

```
重要配置：
**default_hostgroup：** 默认hostgroup

**transaction_persistent：** 使同一个事务的所有SQL落在同一个节点上。Once a transaction is started, it is possible that some queries are sent to a different hostgroup based on query rules. To prevent this from happening, it is possible to enable transaction_persistent.

**max_connections：** 最大连接数

# 附录
## （1）sys相关functions
**Notice:** 
config mysql group replication need to add view to mysql instance sys
[https://proxysql.com/documentation/main-runtime/](https://gist.github.com/lefred/77ddbde301c72535381ae7af9f968322)

[https://gist.github.com/lefred/77ddbde301c72535381ae7af9f968322](https://gist.github.com/lefred/77ddbde301c72535381ae7af9f968322)
```
USE sys;
drop FUNCTION if exists IFZERO;
drop FUNCTION if exists GTID_COUNT;
drop function if exists LOCATE2;
drop function if exists GTID_NORMALIZE;
drop FUNCTION if exists gr_applier_queue_length;
drop FUNCTION if exists  my_server_uuid;
drop FUNCTION if exists gr_member_in_primary_partition;
drop view if exists gr_member_routing_candidate_status;


DELIMITER $$

CREATE FUNCTION  IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION   LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION my_server_uuid() RETURNS TEXT(36) DETERMINISTIC NO SQL RETURN (SELECT @@global.server_uuid as my_id);$$


CREATE FUNCTION   GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION   GTID_COUNT(gtid_set TEXT(10000))
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

CREATE FUNCTION   gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION   gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id) WHERE MEMBER_ID=@@global.server_uuid);
END$$

CREATE VIEW   gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' 
from performance_schema.replication_group_member_stats where MEMBER_ID=my_server_uuid();$$

DELIMITER ;
```

