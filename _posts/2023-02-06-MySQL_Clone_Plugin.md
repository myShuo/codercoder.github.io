---
id: 1301
title: 'MySQL Clone Plugin'
date: '2023-02-06T18:34:40+08:00'
author: Shuo
excerpt: '介绍MySQL clone plugin的用法及使用场景'
layout: post
guid: 'http://codercoder.cn/?p=1301'
image: http://codercoder.cn/wp-content/uploads/2023/02/2023-02-06-MySQL_Clone_Plugin.png
permalink: /index.php/2023/02/2023-02-06-MySQL_Clone_Plugin
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# 目的
介绍clone plugin的用法及使用场景。
参考：[MySQL-Doc-The Clone Plugin](https://dev.mysql.com/doc/refman/8.0/en/clone-plugin.html)
# 用法
## 注意事项
### （1）版本要求：MySQL 8.0.17及之后
`An instance cannot be cloned from a different MySQL server version or release. The donor and recipient must have exactly the same MySQL server version and release. For example, you cannot clone between MySQL 5.7 and MySQL 8.0, or between MySQL 8.0.19 and MySQL 8.0.20. The clone plugin is only supported in MySQL 8.0.17 and higher.`
源端和目标端的MySQL版本必须完全一致，并且大于等于8.0.17

### （2）源端：Donor；目标端：Recipient，需满足：
· `MySQL版本完全一致`
· `操作系统和平台版本也需要一致`
· `字符集、排序规则需一致`
· `innodb_page_size、innodb_data_file_path需要一致`
    `均需要安装所需的所有plugin`

### （3）磁盘空间需足够

### （4）数据目录的权限、InnoDB的表空间文件目录的权限

### （5）一次只能有一个clone任务在运行（clone_status查看）


## 安装plugin
```
[mysqld]
plugin-load-add=mysql_clone.so

INSTALL PLUGIN clone SONAME 'mysql_clone.so';

mysql> SELECT PLUGIN_NAME, PLUGIN_STATUS
       FROM INFORMATION_SCHEMA.PLUGINS
       WHERE PLUGIN_NAME = 'clone';
+------------------------+---------------+
| PLUGIN_NAME            | PLUGIN_STATUS |
+------------------------+---------------+
| clone                  | ACTIVE        |
+------------------------+---------------+

```
## 配置
### （1）max_allowed_packet需要大于2MB
```
SHOW VARIABLES LIKE 'max_allowed_packet';
```
### （2）UNDO files
UNDO文件名必须唯一，否则会被覆盖（8.0.18以前）或者报错（8.0.18及以后）：
```
mysql> SELECT TABLESPACE_NAME, FILE_NAME FROM INFORMATION_SCHEMA.FILES
       WHERE FILE_TYPE LIKE 'UNDO LOG';
```
## 权限
```
grant SYSTEM_VARIABLES_ADMIN ,CLONE_ADMIN ,BACKUP_ADMIN , SHUTDOWN on *.* to "dba"@"%";
```
在clone结束后，会尝试restart实例，报错信息为：
**ERROR 3707 (HY000): Restart server failed (mysqld is not managed by supervisor process).**



## Clone远程实例
There is no difference with respect to data that is cloned by a local cloning operation as compared to a remote cloning operation. Both operations clone the same set of data.


![](http://codercoder.cn/wp-content/uploads/2023/02/2023-02-06-MySQL_Clone_Plugin.png)


### （1）添加白名单
```
set global clone_valid_donor_list="172.25.1.69:3306,172.25.1.70:3306,172.25.1.71:3306,172.25.1.72:3306";
```
### （2）开始clone远端数据
**（1）直接覆盖当前文件**
```
CLONE INSTANCE FROM 'dba'@'172.16.232.69':3306 identified by "xjDMHJg8";
```
**需要注意端口号的写法！！**

**（2）clone到指定文件**
```
CLONE INSTANCE FROM 'dba'@'172.16.232.69':3306 identified by "dba" DATA DIRECTORY '/data/mysql/mysql3306-2';
```

**注意：**
> 指定目录需要存在，并且需要有写入权限。

## Clone本地实例
适用场景：需要在一台机器上启动多个数据库实例。
 ```
CLONE LOCAL DATA DIRECTORY = '/path/to/clone_dir';
```
![](http://codercoder.cn/wp-content/uploads/2023/02/2023-02-06-MySQL_Clone_Plugin-2.png)

## 停止clone
**（1）通过processlist找到对应的processlist_id**
**（2）KILL QUERY processlist_id**
 
# 使用场景
**延迟较大
文件损坏
实例恢复**

## 更好地支持group replication集群

> `The clone plugin supports replication. In addition to cloning data, a cloning operation extracts and transfers replication coordinates from the donor and applies them on the recipient, which enables using the clone plugin for provisioning Group Replication members and replicas.`
**除了clone data外，还会从源端获取复制信息，并在目标端回放。所以可以用clone进行Group replication集群的支持。**

> `Using the clone plugin for provisioning is considerably faster and more efficient than replicating a large number of transactions (see Section 5.6.7.7, “Cloning for Replication”). `
**在有大量事务需要回放时，clone plugin更快更有效。**


> Group Replication members can also be configured to use the clone plugin as an alternative method of recovery, so that members automatically choose the most efficient way to retrieve group data from seed members. For more information, see Section 18.5.4.2, “Cloning for Distributed Recovery”.·

> The clone plugin supports cloning of encrypted and page-compressed data. See Section 5.6.7.5, “Cloning Encrypted Data”, and Section 5.6.7.6, “Cloning Compressed Data”.·
支持clone加密和压缩后的数据



# 监控状态
**在目标端的performance_schema中查看：**
* `The clone_status table provides the state of the current or last executed cloning operation. A clone operation has four possible states: Not Started, In Progress, Completed, and Failed.`
**最近一次clone的情况：Not Started, In Progress, Completed, and Failed。**

* `The clone_progress table provides progress information for the current or last executed clone operation, by stage. The stages of a cloning operation include DROP DATA, FILE COPY, PAGE_COPY, REDO_COPY, FILE_SYNC, RESTART, and RECOVERY.`
**最近一次clone的进度：DROP DATA, FILE COPY, PAGE_COPY, REDO_COPY, FILE_SYNC, RESTART, and RECOVERY.**

## 查看状态
```
mysql> SELECT * FROM performance_schema.clone_status\G
*************************** 1. row ***************************
             ID: 1
            PID: 8
          STATE: In Progress
     BEGIN_TIME: 2019-07-15 11:58:36.767
       END_TIME: NULL
         SOURCE: LOCAL INSTANCE
    DESTINATION: /path/to/clone_dir/
       ERROR_NO: 0
  ERROR_MESSAGE:
    BINLOG_FILE:
BINLOG_POSITION: 0
  GTID_EXECUTED:
```

## 查看进度
```
mysql> SELECT STAGE, STATE, END_TIME FROM performance_schema.clone_progress;
+-----------+-----------+----------------------------+
| stage     | state     | end_time                   |
+-----------+-----------+----------------------------+
| DROP DATA | Completed | 2019-01-27 22:45:43.141261 |
| FILE COPY | Completed | 2019-01-27 22:45:44.457572 |
| PAGE COPY | Completed | 2019-01-27 22:45:44.577330 |
| REDO COPY | Completed | 2019-01-27 22:45:44.679570 |
| FILE SYNC | Completed | 2019-01-27 22:45:44.918547 |
| RESTART   | Completed | 2019-01-27 22:45:48.583565 |
| RECOVERY  | Completed | 2019-01-27 22:45:49.626595 |
+-----------+-----------+----------------------------+
```
![](http://codercoder.cn/wp-content/uploads/2023/02/2023-02-06-MySQL_Clone_Plugin-3.png)

## 使用Stage Events
（1）stage/innodb/clone (file copy)

（2）stage/innodb/clone (page copy)

（3）stage/innodb/clone (redo copy)


```
mysql> UPDATE performance_schema.setup_instruments SET ENABLED = 'YES' WHERE NAME LIKE 'stage/innodb/clone%';
       
       
mysql> UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE NAME LIKE '%stages%';

mysql> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED FROM performance_schema.events_stages_current
       WHERE EVENT_NAME LIKE 'stage/innodb/clone%';
+--------------------------------+----------------+----------------+
| EVENT_NAME                     | WORK_COMPLETED | WORK_ESTIMATED |
+--------------------------------+----------------+----------------+
| stage/innodb/clone (redo copy) |              1 |              1 |
+--------------------------------+----------------+----------------+


mysql> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED FROM events_stages_history
       WHERE EVENT_NAME LIKE 'stage/innodb/clone%';
+--------------------------------+----------------+----------------+
| EVENT_NAME                     | WORK_COMPLETED | WORK_ESTIMATED |
+--------------------------------+----------------+----------------+
| stage/innodb/clone (file copy) |            301 |            301 |
| stage/innodb/clone (page copy) |              0 |              0 |
| stage/innodb/clone (redo copy) |              1 |              1 |
+--------------------------------+----------------+----------------+

```

## 使用Clone Instrumentation
参考：[Monitoring Cloning Operations](https://dev.mysql.com/doc/refman/8.0/en/clone-plugin-monitoring.html)
```
SELECT * FROM performance_schema.setup_instruments
       WHERE NAME LIKE '%clone%';
```

# 配置
## 相关变量
![](http://codercoder.cn/wp-content/uploads/2023/02/2023-02-06-MySQL_Clone_Plugin-4.png)

* **clone_autotune_concurrency**：自动优化并发，默认打开
    The autotuning process does not support decreasing the number of threads.

* **clone_max_concurrency**：在目标端设置，最大线程数，默认16。

* **clone_max_data_bandwidth**：在目标端配置，限制每个线程每秒传输的速度，单位MiB，默认为不限制。仅需要在源端的I/O带宽达到上限时配置。

* **clone_max_network_bandwidth**：在目标端配置，限制每个线程每秒传输的速度，单位MiB，默认为不限制。仅需要在源端的网络带宽达到上限时配置。

* **clone_valid_donor_list**：在目标端配置，定义可用的源端地址，格式为：“HOST1:PORT1,HOST2:PORT2,HOST3:PORT3”。

* **clone_buffer_size**：在local cloning中使用，默认为4MB，较大的中间buffer能够改善clone性能。

* **clone_block_ddl**：默认关闭，即是否允许在源端加上backup排他锁，以阻止clone期间的DDL执行。

* **clone_ddl_timeout**：clone操作等待backup锁的超时时间。克隆操作等待当前的DDL操作完成。一旦获得了备份锁，DDL操作必须等待克隆操作完成。
8.0.27之前，备份锁阻塞源端和目标端的DDL，同样克隆操作需等待当前的DDL操作完成。
8.0.27及之后，若clone_block_ddl为OFF，则支持在clone期间源端可进行DDL。
Bug：[Document Bug-clone_ddl_timeout](https://bugs.mysql.com/bug.php?id=107842)

* **clone_delay_after_data_drop**：删除数据后等待N秒后开始复制远程的数据。（部分文件系统例如VxFS，是后台逐渐释放空间，删除data后不是瞬间可用）

* **clone_donor_timeout_after_network_failure**：在网络故障时，源端允许目标端重试clone的时间，单位分钟，在源端设置。

* **clone_enable_compression**：在目标端设置，在网络传输时，压缩数据，会消耗CPU。



# 限制
## DDL阻塞
`Prior to MySQL 8.0.27, DDL operations on the donor and recipient MySQL Server instances, including TRUNCATE TABLE, are not permitted during a cloning operation.`
8.0.27前，阻塞所有源端和目标端的DDL，包括truncate

`To prevent concurrent DDL during a cloning operation, an exclusive backup lock is acquired on the donor and recipient. The clone_ddl_timeout variable defines the time in seconds on the donor and recipient that a cloning operation waits for a backup lock. The default setting is 300 seconds. If a backup lock is not obtained with the specified time limit, the cloning operation fails with an error.`
在clone期间，会在源端和目标端加上排他的backup lock。clone_ddl_timeout用来定于锁超时时间，默认300秒。

`From MySQL 8.0.27, concurrent DDL is permitted on the donor by default. Concurrent DDL support on the donor is controlled by the clone_block_ddl variable. Concurrent DDL support can be enabled and disabled dynamically using a SET statement.`
从MySQL 8.0.27开始，默认在clone期间支持DDL，可通过clone_block_ddl进行动态调整。
```
SET GLOBAL clone_block_ddl={OFF|ON}
```
`The default setting is clone_block_ddl=OFF, which permits concurrent DDL on the donor.`

`Whether the effect of a concurrent DDL operation is cloned or not depends on whether the DDL operation finishes before the dynamic snapshot is taken by the cloning operation.`
DDL语句是否被clone，取决于DDL的操作是否在clone的动态快照之前完成



## 其它
* 一次仅能clone一个实例
* 目标端的系统变量不会被覆盖
* clong文件不包括binlog
* 仅克隆InnoDB的数据，其它存储引擎仅复制表结构
* 不能通过代理进行clone操作

# 性能测试
参考：
[MySQL Clone Plugin Speed Test](https://mydbops.wordpress.com/2019/11/14/mysql-clone-plugin-speed-test/)
[The MySQL Clone Wars: Plugin vs. Percona XtraBackup](https://www.percona.com/blog/2021/01/19/the-mysql-clone-wars-plugin-vs-percona-xtrabackup/)
