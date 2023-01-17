---
id: 1123
title: 'MySQL手记20 &#8212; MySQL Group Replication(MGR组复制)'
date: '2020-05-27T18:55:03+08:00'
author: Shuo
excerpt: 'MGR（MySQL Group Replication）是MySQL原生的数据库集群架构，底层使用Paxos协议实现多写、选举等过程，MySQL官方也在不断添加相应的内容，使MGR更加可控稳定。可配合Router、ProxySQL进行使用。'
layout: post
guid: 'http://codercoder.cn/?p=1123'
permalink: /index.php/2020/05/mysql-note-20-mysql-group-replication/
views:
    - '5423'
    - '5423'
    - '5423'
enclosure:
    - "http://codercoder.cn/wp-content/uploads/2020/05/2020-06-0154.mp4\n5614967\nvideo/mp4\n"
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、简介

 MGR（MySQL Group Replication）是MySQL原生的数据库集群架构，底层使用Paxos协议实现多写、选举等过程，目前最大支持9个节点。可分为单写（Single-Primary）、多写（Multiple-Primary）两类集群，虽然能够多写，但是还是建议单写，因为多写需要有选举的过程，在节点较多、或者网络环境较差的情况下，会严重影响性能。官方work有相关的测试报告：http://mysqlhighavailability.com/zooming-in-on-group-replication-performance/  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-2789.png)

# 二、安装部署

## （1）数据库配置

在启动前，需要修改启动的配置文件：

```
#gtid
gtid_mode=on
enforce_gtid_consistency=on
binlog_checksum=none
transaction_write_set_extraction    = XXHASH64
loose-group_replication_group_name  = "8cb61fd9-8931-11ea-ad6f-0242ac110003"
loose_group_replication_single_primary_mode = 0    #0:single-primary; 1:multiple-primary
loose-group_replication_start_on_boot = OFF
loose_group_replication_compression_threshold = 100
loose_group_replication_flow_control_mode = 0
loose-group_replication_bootstrap_group = OFF
loose_group_replication_enforce_update_everywhere_checks =true   #single-primary需设为关闭，multiple-primary则必须打开，主要是使（serialized隔离级别的事务，或者是使事务中有外键关系的表的）事务失败
loose-group_replication_transaction_size_limit = 10485760     #事务的大小，若太大，则会影响集群的性能
loose_group_replication_unreachable_majority_timeout = 120
loose-group_replication_auto_increment_increment = 7   #自增值的跨度大小
loose-group_replication_local_address="172.21.0.2:15501"    #当前主机的ip，可动态修改
loose-group_replication_group_seeds='172.21.0.2:15501,172.21.0.4:15503,172.21.0.3:15502'   ##集群中所有节点的信息，可动态修改
loose_group_replication_ip_whitelist='172.21.0.4,172.21.0.2,172.21.0.3'  #节点的ip白名单，可动态修改

```

 其中loose开头的为group replication安装插件时候启动的配置。MGR不支持binlog\_checksum，切gtid必须打开。

## （2）添加插件

Group Replication是以插件的形式加入到集群中的，所以需要进行插件的安装：

```
mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
​
mysql> SHOW PLUGINS;
+---------------------------------+----------+--------------------+----------------------+---------+
| Name                            | Status   | Type               | Library              | License |
+---------------------------------+----------+--------------------+----------------------+---------+
...
| group_replication               | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+---------------------------------+----------+--------------------+----------------------+---------+

```

### a. 启动第一个节点

第一个节点启动时，需要先设置：

```
SET GLOBAL group_replication_bootstrap_group=ON;

```

即表示该节点为运行集群中的第一个节点，启动Group Replication：

```
START GROUP_REPLICATION;

```

日志中打印的信息为：

```
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_applier' executed'. Previous state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''.

```

### b .启动第二个节点

在第二个节点执行：

```
START GROUP_REPLICATION;

```

 由于集群中已经有第一个节点，所以不需要再设置group\_replication\_bootstrap\_group=on  
日志：

```
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_applier' executed'. Previous state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''.
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_recovery' executed'. Previous state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='70b353672cdc', master_port= 5501, master_log_file='', master_log_pos= 4, master_bind=''.
[Warning] [MY-010897] [Repl] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
2020-05-22T07:34:16.119089-00:00 27 [System] [MY-010562] [Repl] Slave I/O thread for channel 'group_replication_recovery': connected to master 'root@70b353672cdc:5501',replication started in log 'FIRST' at position 4
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_recovery' executed'. Previous state master_host='70b353672cdc', master_port= 5501, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''.

```

查看第二个节点的添加情况：

```
mysql> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| group_replication_applier | e3f0850e-8936-11ea-98ba-0242ac150004 | 2bd38a52527e |        5502 | ONLINE       | PRIMARY     | 8.0.20         |
| group_replication_applier | e4a14f46-8936-11ea-b379-0242ac150005 | 70b353672cdc |        5501 | ONLINE       | PRIMARY     | 8.0.20         |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
2 rows in set (0.00 sec)

```

重启group replication时，状态为RECOVERY:

```
mysql> select * from performance_schema.replication_group_members;
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| group_replication_applier | e3f0850e-8936-11ea-98ba-0242ac150004 | 2bd38a52527e |        5502 | RECOVERING   | PRIMARY     | 8.0.20         |
| group_replication_applier | e4a14f46-8936-11ea-b379-0242ac150005 | 70b353672cdc |        5501 | ONLINE       | PRIMARY     | 8.0.20         |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
2 rows in set (0.03 sec)

```

### c. 添加第三个节点

直接执行start group\_replication;，打印日志为：

```
[Warning] [MY-011735] [Repl] Plugin group_replication reported: '[GCS] Automatically adding IPv4 localhost address to the whitelist. It is mandatory that it is added.'
[Warning] [MY-011735] [Repl] Plugin group_replication reported: '[GCS] Automatically adding IPv6 localhost address to the whitelist. It is mandatory that it is added.'
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_applier' executed'. Previous state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''.
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_recovery' executed'. Previous state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='70b353672cdc', master_port= 5501, master_log_file='', master_log_pos= 4, master_bind=''.
[Warning] [MY-010897] [Repl] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
[System] [MY-010562] [Repl] Slave I/O thread for channel 'group_replication_recovery': connected to master 'root@70b353672cdc:5501',replication started in log 'FIRST' at position 4
[System] [MY-010597] [Repl] 'CHANGE MASTER TO FOR CHANNEL 'group_replication_recovery' executed'. Previous state master_host='70b353672cdc', master_port= 5501, master_log_file='', master_log_pos= 4, master_bind=''. New state master_host='<NULL>', master_port= 0, master_log_file='', master_log_pos= 4, master_bind=''.

```

 通过以上的日志可以看出，在各个节点启动的时候，均会有CHANGE MASTER TO FOR CHANNEL …的操作。

完成后，查看集群中节点的状态，查看集群的状态：

```
mysql> SELECT * FROM performance_schema.replication_group_members;
​
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| group_replication_applier | e0dd6e3d-8936-11ea-9176-0242ac150003 | 7bcecb1807d5 |        5503 | ONLINE       | PRIMARY     | 8.0.20         |
| group_replication_applier | e3f0850e-8936-11ea-98ba-0242ac150004 | 2bd38a52527e |        5502 | ONLINE       | PRIMARY     | 8.0.20         |
| group_replication_applier | e4a14f46-8936-11ea-b379-0242ac150005 | 70b353672cdc |        5501 | ONLINE       | PRIMARY     | 8.0.20         |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
3 rows in set (0.02 sec)

```

# 三、MGR相关参数

 MGR为了保证数据的一致性以及提升集群的性能，用户可进行相关的配置，当前版本8.0.20：

```
mysql> show variables like '%group_repl%';
+-----------------------------------------------------+----------------------------------------------------+
| Variable_name                                       | Value                                              |
+-----------------------------------------------------+----------------------------------------------------+
| group_replication_allow_local_lower_version_join    | OFF                                                |
| group_replication_auto_increment_increment          | 7                                                  |
| group_replication_autorejoin_tries                  | 0                                                  |
| group_replication_bootstrap_group                   | OFF                                                |
| group_replication_clone_threshold                   | 9223372036854775807                                |
| group_replication_communication_debug_options       | GCS_DEBUG_NONE                                     |
| group_replication_communication_max_message_size    | 10485760                                           |
| group_replication_components_stop_timeout           | 31536000                                           |
| group_replication_compression_threshold             | 100                                                |
| group_replication_consistency                       | EVENTUAL                                           |
| group_replication_enforce_update_everywhere_checks  | ON                                                 |
| group_replication_exit_state_action                 | READ_ONLY                                          |
| group_replication_flow_control_applier_threshold    | 25000                                              |
| group_replication_flow_control_certifier_threshold  | 25000                                              |
| group_replication_flow_control_hold_percent         | 10                                                 |
| group_replication_flow_control_max_quota            | 0                                                  |
| group_replication_flow_control_member_quota_percent | 0                                                  |
| group_replication_flow_control_min_quota            | 0                                                  |
| group_replication_flow_control_min_recovery_quota   | 0                                                  |
| group_replication_flow_control_mode                 | DISABLED                                           |
| group_replication_flow_control_period               | 1                                                  |
| group_replication_flow_control_release_percent      | 50                                                 |
| group_replication_force_members                     |                                                    |
| group_replication_group_name                        | 8cb61fd9-8931-11ea-ad6f-0242ac110003               |
| group_replication_group_seeds                       | 172.21.0.2:15501,172.21.0.4:15503,172.21.0.3:15502 |
| group_replication_gtid_assignment_block_size        | 1000000                                            |
| group_replication_ip_whitelist                      | 172.21.0.4,172.21.0.2,172.21.0.3,172.21.0.5        |
| group_replication_local_address                     | 172.21.0.3:15502                                   |
| group_replication_member_expel_timeout              | 0                                                  |
| group_replication_member_weight                     | 50                                                 |
| group_replication_message_cache_size                | 1073741824                                         |
| group_replication_poll_spin_loops                   | 0                                                  |
| group_replication_recovery_complete_at              | TRANSACTIONS_APPLIED                               |
| group_replication_recovery_compression_algorithms   | uncompressed                                       |
| group_replication_recovery_get_public_key           | OFF                                                |
| group_replication_recovery_public_key_path          |                                                    |
| group_replication_recovery_reconnect_interval       | 60                                                 |
| group_replication_recovery_retry_count              | 10                                                 |
| group_replication_recovery_ssl_ca                   |                                                    |
| group_replication_recovery_ssl_capath               |                                                    |
| group_replication_recovery_ssl_cert                 |                                                    |
| group_replication_recovery_ssl_cipher               |                                                    |
| group_replication_recovery_ssl_crl                  |                                                    |
| group_replication_recovery_ssl_crlpath              |                                                    |
| group_replication_recovery_ssl_key                  |                                                    |
| group_replication_recovery_ssl_verify_server_cert   | OFF                                                |
| group_replication_recovery_tls_ciphersuites         |                                                    |
| group_replication_recovery_tls_version              | TLSv1,TLSv1.1,TLSv1.2,TLSv1.3                      |
| group_replication_recovery_use_ssl                  | OFF                                                |
| group_replication_recovery_zstd_compression_level   | 3                                                  |
| group_replication_single_primary_mode               | OFF                                                |
| group_replication_ssl_mode                          | DISABLED                                           |
| group_replication_start_on_boot                     | OFF                                                |
| group_replication_transaction_size_limit            | 10485760                                           |
| group_replication_unreachable_majority_timeout      | 120                                                |
+-----------------------------------------------------+----------------------------------------------------+
55 rows in set (0.12 sec)

```

其中：  
**group\_replication\_enforce\_update\_everywhere\_checks=ON**  
 该MGR集群为multiple-primary，即多主的集群，其中较为重要的参数配置：  
**group\_replication\_consistency=EVENTUAL**  
 事务一致性等级配置，本实例的配置为：在执行”读”或者”写”事务之前，不用等待之前的事务完成。即看到的是其它事务开始前的快照。  
 使用EVENTUAL，由于不需要进行等待其它事务的完成，所以可以获得较高的性能。当然，为了得到更高的事务一致性，还有其它配置可供选择，并且支持session级别的修改。  
http://codercoder.cn/index.php/2019/09/innodb-cluster-mgr-group\_replication\_consistency/  
**group\_replication\_exit\_state\_action=READ\_ONLY**  
 从8.0.12开始加入该参数，节点非正常退出集群后记性的操作，非正常退出，包括：异常退出、由于网络等其它问题重试group\_replication\_autorejoin\_tries次数之后，还是未能重新加入集群。为了防止异常的节点还能被访问，或者重新加入集群后影响正常的数据。所以提供此参数进行配置。

集群的基本连接信息，也可在实例启动后通过**set global xxx**进行配置：  
**group\_replication\_start\_on\_boot**  
 是否自动启动group\_replication  
**group\_replication\_group\_name**  
 当前集群的名称  
**group\_replication\_group\_seeds**  
 集群中的节点信息  
**group\_replication\_ip\_whitelist**  
 节点所在机器的ip白名单  
**group\_replication\_local\_address**  
 当前节点的地址

# 四、MGR相关表

MGR相关的表，是存储在performance\_schema库下的，两张表：  
**replication\_group\_members**：用以存储MGR集群的节点信息  
**replication\_group\_member\_stats**：用以存储节点的状态信息，执行的事务数等

```
mysql> use performance_schema
Database changed
​
mysql> show tables like '%group%';
+----------------------------------------+
| Tables_in_performance_schema (%group%) |
+----------------------------------------+
| replication_group_member_stats         |
| replication_group_members              |
+----------------------------------------+
2 rows in set (0.00 sec)

```

## （1）replication\_group\_members表

```
mysql> select * from replication_group_members ;
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
| group_replication_applier | e0dd6e3d-8936-11ea-9176-0242ac150003 | 7bcecb1807d5 |        5503 | ONLINE       | PRIMARY     | 8.0.20         |
| group_replication_applier | e3f0850e-8936-11ea-98ba-0242ac150004 | 2bd38a52527e |        5502 | ONLINE       | PRIMARY     | 8.0.20         |
| group_replication_applier | e4a14f46-8936-11ea-b379-0242ac150005 | 70b353672cdc |        5501 | ONLINE       | PRIMARY     | 8.0.20         |
+---------------------------+--------------------------------------+--------------+-------------+--------------+-------------+----------------+
3 rows in set (0.01 sec)

```

 可以看出，该MGR集群一共三个节点，并且均为PRIMARY，MEMBER\_STATE均为ONLINE状态，即三个节点均在正常运行。其中：MEMBER\_HOST显示了三个节点的hostname，MEMBER\_PORT为对应的端口。

## （2）replication\_group\_member\_stats表

```
mysql&gt; show create table replication_group_member_stats \G
*************************** 1. row ***************************
       Table: replication_group_member_stats
Create Table: CREATE TABLE if not exists `replication_group_member_stats` (
  `CHANNEL_NAME` char(64) NOT NULL,
  `VIEW_ID` char(60) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  `MEMBER_ID` char(36) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL,
  `COUNT_TRANSACTIONS_IN_QUEUE` bigint unsigned NOT NULL,
  `COUNT_TRANSACTIONS_CHECKED` bigint unsigned NOT NULL,
  `COUNT_CONFLICTS_DETECTED` bigint unsigned NOT NULL,
  `COUNT_TRANSACTIONS_ROWS_VALIDATING` bigint unsigned NOT NULL,
  `TRANSACTIONS_COMMITTED_ALL_MEMBERS` longtext NOT NULL,
  `LAST_CONFLICT_FREE_TRANSACTION` text NOT NULL,
  `COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE` bigint unsigned NOT NULL,
  `COUNT_TRANSACTIONS_REMOTE_APPLIED` bigint unsigned NOT NULL,
  `COUNT_TRANSACTIONS_LOCAL_PROPOSED` bigint unsigned NOT NULL,
  `COUNT_TRANSACTIONS_LOCAL_ROLLBACK` bigint unsigned NOT NULL
) ENGINE=PERFORMANCE_SCHEMA DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

```

各个字段的介绍：  
 **CHANNEL\_NAME**：Name of the Group Replication channel  
 各个节点的名称  
 **VIEW\_ID**：Current view identifier for this group.  
 当前MGR集群的view id  
 **MEMBER\_ID**：The member server UUID. This has a different value for each member in the group. This also serves as a key because it is unique to each member.  
 不同成员的UUID，每个集群中的成员UUID唯一  
 **COUNT\_TRANSACTIONS\_IN\_QUEUE**：The number of transactions in the queue pending conflict detection checks. Once the transactions have been checked for conflicts, if they pass the check, they are queued to be applied as well.  
 队列中等待冲突检测检查的事务数。一旦检查了事务是否存在冲突，待通过检查，则将它们排队等待应用。  
 **COUNT\_TRANSACTIONS\_CHECKED**：The number of transactions that have been checked for conflicts.  
 已检查有冲突的事务数。  
 **COUNT\_CONFLICTS\_DETECTED**：The number of transactions that have not passed the conflict detection check.  
 尚未通过冲突检测检查的事务数。  
 **COUNT\_TRANSACTIONS\_ROWS\_VALIDATING**： Number of transaction rows which can be used for certification, but have not been garbage collected. Can be thought of as the current size of the conflict detection database against which each transaction is certified.  
 可以用于认证但尚未被垃圾回收的交易行数。可以认为是每个事务都针对其进行认证的冲突检测数据库的当前大小。  
 **TRANSACTIONS\_COMMITTED\_ALL\_MEMBERS**：The transactions that have been successfully committed on all members of the replication group, shown as GTID Sets. This is updated at a fixed time interval.  
 在复制组的所有成员上已成功提交的事务，显示为 GTID Sets。这将以固定的时间间隔进行更新。  
 **LAST\_CONFLICT\_FREE\_TRANSACTION**：The transaction identifier of the last conflict free transaction which was checked.  
 最后检查的无冲突交易的交易标识符。  
 **COUNT\_TRANSACTIONS\_REMOTE\_IN\_APPLIER\_QUEUE**：The number of transactions that this member has received from the replication group which are waiting to be applied.  
 该成员已从复制组收到的等待应用的事务数。  
 **COUNT\_TRANSACTIONS\_REMOTE\_APPLIED**：Number of transactions this member has received from the group and applied.  
 该成员已从该组收到并应用的交易数。  
 **COUNT\_TRANSACTIONS\_LOCAL\_PROPOSED**：Number of transactions which originated on this member and were sent to the group.  
 起源于此成员并发送到该组的交易数量。  
 **COUNT\_TRANSACTIONS\_LOCAL\_ROLLBACK**：Number of transactions which originated on this member and were rolled back by the group.  
 该成员发起并被该组回滚的事务数。

ps.注意：不能在replication\_group\_member\_stats表上执行truncate table 命令。

```
mysql&gt; select * from replication_group_member_stats;
+---------------------------+---------------------+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+---------------------------------------------------------------------------------------------------------------------+-----------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| CHANNEL_NAME              | VIEW_ID             | MEMBER_ID                            | COUNT_TRANSACTIONS_IN_QUEUE | COUNT_TRANSACTIONS_CHECKED | COUNT_CONFLICTS_DETECTED | COUNT_TRANSACTIONS_ROWS_VALIDATING | TRANSACTIONS_COMMITTED_ALL_MEMBERS                                                                                  | LAST_CONFLICT_FREE_TRANSACTION          | COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE | COUNT_TRANSACTIONS_REMOTE_APPLIED | COUNT_TRANSACTIONS_LOCAL_PROPOSED | COUNT_TRANSACTIONS_LOCAL_ROLLBACK |
+---------------------------+---------------------+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+---------------------------------------------------------------------------------------------------------------------+-----------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| group_replication_applier | 15903747614426007:3 | e0dd6e3d-8936-11ea-9176-0242ac150003 |                           0 |                        167 |                        0 |                                  1 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:1-43:1000020-1000167:2000020-2000037,
e4a14f46-8936-11ea-b379-0242ac150005:1-5 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43 |                                          0 |                                21 |                               146 |                                 0 |
| group_replication_applier | 15903747614426007:3 | e3f0850e-8936-11ea-98ba-0242ac150004 |                           0 |                        167 |                        0 |                                  1 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:1-43:1000020-1000167:2000020-2000037,
e4a14f46-8936-11ea-b379-0242ac150005:1-5 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43 |                                          0 |                               151 |                                17 |                                 0 |
| group_replication_applier | 15903747614426007:3 | e4a14f46-8936-11ea-b379-0242ac150005 |                           0 |                        167 |                        0 |                                  1 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:1-43:1000020-1000167:2000020-2000037,
e4a14f46-8936-11ea-b379-0242ac150005:1-5 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43 |                                          0 |                               166 |                                 4 |                                 0 |
+---------------------------+---------------------+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+---------------------------------------------------------------------------------------------------------------------+-----------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
3 rows in set (0.00 sec)

```

## （3）replication\_group\_member\_stats表字段分类

可以简单把replication\_group\_member\_stats表的字段分为两类：

### a. 基础信息

查看replication\_group\_member\_stats表的数据情况：

```
mysql&gt; select CHANNEL_NAME,VIEW_ID,MEMBER_ID,TRANSACTIONS_COMMITTED_ALL_MEMBERS from replication_group_member_stats;
+---------------------------+---------------------+--------------------------------------+-------------------------------------------------------------------------------------------------------------+
| CHANNEL_NAME              | VIEW_ID             | MEMBER_ID                            | TRANSACTIONS_COMMITTED_ALL_MEMBERS                                                                          |
+---------------------------+---------------------+--------------------------------------+-------------------------------------------------------------------------------------------------------------+
| group_replication_applier | 15903747614426007:3 | e0dd6e3d-8936-11ea-9176-0242ac150003 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:1-39:1000020-1000021:2000020,
e4a14f46-8936-11ea-b379-0242ac150005:1-5 |
| group_replication_applier | 15903747614426007:3 | e3f0850e-8936-11ea-98ba-0242ac150004 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:1-39:1000020-1000021:2000020,
e4a14f46-8936-11ea-b379-0242ac150005:1-5 |
| group_replication_applier | 15903747614426007:3 | e4a14f46-8936-11ea-b379-0242ac150005 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:1-39:1000020-1000021:2000020,
e4a14f46-8936-11ea-b379-0242ac150005:1-5 |
+---------------------------+---------------------+--------------------------------------+-------------------------------------------------------------------------------------------------------------+
3 rows in set (0.00 sec)

```

 这几个字段展示了CHANNEL\_NAME、成员信息、当前已提交的事务id。均属于基础信息类。

## b.计数类

查看当前状态：

```
mysql&gt; select MEMBER_ID,COUNT_TRANSACTIONS_IN_QUEUE,COUNT_TRANSACTIONS_CHECKED,COUNT_CONFLICTS_DETECTED,COUNT_TRANSACTIONS_ROWS_VALIDATING,LAST_CONFLICT_FREE_TRANSACTION,COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE,COUNT_TRANSACTIONS_REMOTE_APPLIED,COUNT_TRANSACTIONS_LOCAL_PROPOSED,COUNT_TRANSACTIONS_LOCAL_ROLLBACK from replication_group_member_stats;
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+----------------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| MEMBER_ID                            | COUNT_TRANSACTIONS_IN_QUEUE | COUNT_TRANSACTIONS_CHECKED | COUNT_CONFLICTS_DETECTED | COUNT_TRANSACTIONS_ROWS_VALIDATING | LAST_CONFLICT_FREE_TRANSACTION               | COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE | COUNT_TRANSACTIONS_REMOTE_APPLIED | COUNT_TRANSACTIONS_LOCAL_PROPOSED | COUNT_TRANSACTIONS_LOCAL_ROLLBACK |
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+----------------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| e0dd6e3d-8936-11ea-9176-0242ac150003 |                           0 |                        166 |                        0 |                                 18 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:2000037 |                                          0 |                                20 |                               146 |                                 0 |
| e3f0850e-8936-11ea-98ba-0242ac150004 |                           0 |                        166 |                        0 |                                 18 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:2000037 |                                          0 |                               150 |                                17 |                                 0 |
| e4a14f46-8936-11ea-b379-0242ac150005 |                           0 |                        166 |                        0 |                                 18 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:2000037 |                                          0 |                               166 |                                 3 |                                 0 |
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+----------------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
3 rows in set (0.01 sec)

```

**在5501节点（member\_id=e4a14f46-8936-11ea-b379-0242ac150005）插入一条数据**，再次查看：

```
mysql&gt; insert into t2 values(6);
Query OK, 1 row affected (0.04 sec)
​
mysql&gt; select MEMBER_ID,COUNT_TRANSACTIONS_IN_QUEUE,COUNT_TRANSACTIONS_CHECKED,COUNT_CONFLICTS_DETECTED,COUNT_TRANSACTIONS_ROWS_VALIDATING,LAST_CONFLICT_FREE_TRANSACTION,COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE,COUNT_TRANSACTIONS_REMOTE_APPLIED,COUNT_TRANSACTIONS_LOCAL_PROPOSED,COUNT_TRANSACTIONS_LOCAL_ROLLBACK from replication_group_member_stats;
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+----------------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| MEMBER_ID                            | COUNT_TRANSACTIONS_IN_QUEUE | COUNT_TRANSACTIONS_CHECKED | COUNT_CONFLICTS_DETECTED | COUNT_TRANSACTIONS_ROWS_VALIDATING | LAST_CONFLICT_FREE_TRANSACTION               | COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE | COUNT_TRANSACTIONS_REMOTE_APPLIED | COUNT_TRANSACTIONS_LOCAL_PROPOSED | COUNT_TRANSACTIONS_LOCAL_ROLLBACK |
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+----------------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| e0dd6e3d-8936-11ea-9176-0242ac150003 |                           0 |                        167 |                        0 |                                 19 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:2000037 |                                          0 |                                21 |                               146 |                                 0 |
| e3f0850e-8936-11ea-98ba-0242ac150004 |                           0 |                        167 |                        0 |                                 19 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43      |                                          0 |                               151 |                                17 |                                 0 |
| e4a14f46-8936-11ea-b379-0242ac150005 |                           0 |                        167 |                        0 |                                 19 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:2000037 |                                          0 |                               166 |                                 4 |                                 0 |
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+----------------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
3 rows in set (0.01 sec)

```

过一段时间，再次查询：

```
mysql&gt; select MEMBER_ID,COUNT_TRANSACTIONS_IN_QUEUE,COUNT_TRANSACTIONS_CHECKED,COUNT_CONFLICTS_DETECTED,COUNT_TRANSACTIONS_ROWS_VALIDATING,LAST_CONFLICT_FREE_TRANSACTION,COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE,COUNT_TRANSACTIONS_REMOTE_APPLIED,COUNT_TRANSACTIONS_LOCAL_PROPOSED,COUNT_TRANSACTIONS_LOCAL_ROLLBACK from replication_group_member_stats;
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+-----------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| MEMBER_ID                            | COUNT_TRANSACTIONS_IN_QUEUE | COUNT_TRANSACTIONS_CHECKED | COUNT_CONFLICTS_DETECTED | COUNT_TRANSACTIONS_ROWS_VALIDATING | LAST_CONFLICT_FREE_TRANSACTION          | COUNT_TRANSACTIONS_REMOTE_IN_APPLIER_QUEUE | COUNT_TRANSACTIONS_REMOTE_APPLIED | COUNT_TRANSACTIONS_LOCAL_PROPOSED | COUNT_TRANSACTIONS_LOCAL_ROLLBACK |
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+-----------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+
| e0dd6e3d-8936-11ea-9176-0242ac150003 |                           0 |                        167 |                        0 |                                  1 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43 |                                          0 |                                21 |                               146 |                                 0 |
|  e3f0850e-8936-11ea-98ba-0242ac150004 |                           0 |                        167 |                        0 |                                  1 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43 |                                          0 |                               151 |                                17 |                                 0 |
| e4a14f46-8936-11ea-b379-0242ac150005 |                           0 |                        167 |                        0 |                                  1 | 8cb61fd9-8931-11ea-ad6f-0242ac110003:43 |                                          0 |                               166 |                                 4 |                                 0 |
+--------------------------------------+-----------------------------+----------------------------+--------------------------+------------------------------------+-----------------------------------------+--------------------------------------------+-----------------------------------+-----------------------------------+-----------------------------------+

```

可以看到：  
 **COUNT\_TRANSACTIONS\_IN\_QUEUE**： 三个节点+1  
 **COUNT\_TRANSACTIONS\_LOCAL\_PROPOSED**：5501加1（由于事务从5501发出），其它节点不变  
 **COUNT\_TRANSACTIONS\_REMOTE\_APPLIED**：5502、5503均加1，两个节点均为接收5501节点的事务  
 **COUNT\_TRANSACTIONS\_ROWS\_VALIDATING**：三节点均+1  
 Number of transaction rows which can be used for certification, but have not been garbage collected. Can be thought of as the current size of the conflict detection database against which each transaction is certified.  
 未被GC的用以冲突检测的事务行数，该值会按照一定频率清零（已执行完成的事务，writeset存储在certification\_info的数据结构中，而每当事务结束后，各个节点会每隔（在MySQL社区版中broadcast\_gtid\_executed\_period\_var硬编码为）60秒广播一次自己节点的gtid\_executed。延续阅读：https://zhuanlan.zhihu.com/p/55323854）。  
\\

<div class="wp-video" style="width: 1280px;"><video class="wp-video-shortcode" controls="controls" height="720" id="video-1123-3" preload="metadata" width="1280"><source src="http://codercoder.cn/wp-content/uploads/2020/05/2020-06-0154.mp4?_=3" type="video/mp4"></source><http://codercoder.cn/wp-content/uploads/2020/05/2020-06-0154.mp4></video></div>\[/video\] # 三、MGR\_ProxySQL

 了解了MGR的集群后，还需要在该集群上进行代理的搭建，以便于读写及宕机等情况的调度，完成高可用。常用的解决方案有：ProxySQL和MySQL Router。现简单介绍下ProxySQL+MGR：  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-2481.png)  
 即：ProxySQL作为数据库集群的代理，Client通过连接ProxySQL访问数据库，ProxySQL则按照配置的规则，将请求分发到不同的MySQL实例，而ProxySQL后端的MySQL实例，为原生的MGR集群。ProxySQL可查阅：[MySQL手记19 — MySQL代理工具ProxySQL](http://codercoder.cn/index.php/2020/05/mysql-note19-mysql-proxy-tool-proxysql/)  
效果为：  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-06-0143.png)

# 小结

 对于MGR高可用的架构，5.7后的讨论声越来越多，值得我们逐渐在生产环境中使用。MySQL官方也在不断添加相应的内容，使MGR更加可控稳定，例如group\_replication\_exit\_state\_action为了让事务的控制更为精细，group\_replication\_consistency让集群更稳定…再配合着代理，能够让宕机、网络差、读写切换等运维操作更加方便。(http://codercoder.cn/index.php/2020/05/mysql-note19-mysql-proxy-tool-proxysql/)）。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)