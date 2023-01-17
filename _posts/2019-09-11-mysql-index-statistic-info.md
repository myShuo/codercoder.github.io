---
id: 570
title: MySQL索引统计信息
date: '2019-09-11T11:24:30+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=570'
permalink: /index.php/2019/09/mysql-index-statistic-info/
views:
    - '5598'
    - '5598'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1159.jpg)

# 背景

上一篇文章中([MySQL索引统计信息information\_schema.INDEX\_STATISTICS](http://codercoder.cn/index.php/2019/09/11/mysql-index-statistics-table))提到了在MySQL的社区版本中，都提供了对于索引统计信息的表，但是，在官方的MySQL版本中，是怎么维护索引（或表）的统计信息的呢？

官方MySQL中，有performance\_schema数据库，主要功能即为监控MySQL server的性能指标，不同的指标对应不同表（表的存储引擎为PERFORMANCE\_SCHEMA），在MySQL5.7开始，有sys数据库，其中包含了多个视图，展示了PERFORMANCE\_SCHEMA中的数据集合，使其更具可读性。

本片文章主要阐述使用索引统计信息的过程，对于performance\_schema的说明，可以查阅官方文档，有更详细的说明：[Chapter 26 MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)

在 MySQL 中，有两种存储索引统计的方式，可以通过设置参数 innodb\_stats\_persistent 的值来选择：  
 设置为 on 的时候，表示统计信息会持久化存储。这时，默认的 N 是 20，M 是 10。  
 设置为 off 的时候，表示统计信息只存储在内存中。这时，默认的 N 是 8，M 是 16。

.

# 在performance\_schema中查看统计信息

使用performance\_schema进行统计信息的手机，需要在数据库启动时加入参数：**performance\_schema=ON**，不可以在运行时进行更改。

```
mysql> set global performance_schema=on;
ERROR 1238 (HY000): Variable 'performance_schema' is a read only variable

```

打开之后，可以查看统计信息表：

```
mysql&gt; show create table performance_schema.table_io_waits_summary_by_index_usage \G
*************************** 1. row ***************************
       Table: table_io_waits_summary_by_index_usage
Create Table: CREATE TABLE if not exists `table_io_waits_summary_by_index_usage` (
  `OBJECT_TYPE` varchar(64) DEFAULT NULL,
  `OBJECT_SCHEMA` varchar(64) DEFAULT NULL,
  `OBJECT_NAME` varchar(64) DEFAULT NULL,
  `INDEX_NAME` varchar(64) DEFAULT NULL,
  `COUNT_STAR` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_WAIT` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_WAIT` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_WAIT` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_WAIT` bigint(20) unsigned NOT NULL,
  `COUNT_READ` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_READ` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_READ` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_READ` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_READ` bigint(20) unsigned NOT NULL,
  `COUNT_WRITE` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_WRITE` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_WRITE` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_WRITE` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_WRITE` bigint(20) unsigned NOT NULL,
  `COUNT_FETCH` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_FETCH` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_FETCH` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_FETCH` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_FETCH` bigint(20) unsigned NOT NULL,
  `COUNT_INSERT` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_INSERT` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_INSERT` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_INSERT` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_INSERT` bigint(20) unsigned NOT NULL,
  `COUNT_UPDATE` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_UPDATE` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_UPDATE` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_UPDATE` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_UPDATE` bigint(20) unsigned NOT NULL,
  `COUNT_DELETE` bigint(20) unsigned NOT NULL,
  `SUM_TIMER_DELETE` bigint(20) unsigned NOT NULL,
  `MIN_TIMER_DELETE` bigint(20) unsigned NOT NULL,
  `AVG_TIMER_DELETE` bigint(20) unsigned NOT NULL,
  `MAX_TIMER_DELETE` bigint(20) unsigned NOT NULL,
  UNIQUE KEY `OBJECT` (`OBJECT_TYPE`,`OBJECT_SCHEMA`,`OBJECT_NAME`,`INDEX_NAME`)
) ENGINE=PERFORMANCE_SCHEMA DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)

```

查看文档中对于该表的介绍：  
[26.12.16.8.2 The table\_io\_waits\_summary\_by\_index\_usage Table](https://dev.mysql.com/doc/refman/8.0/en/table-waits-summary-tables.html#table-io-waits-summary-by-index-usage-table)

table\_io\_waits\_summary\_by\_index\_usage表整合了所有索引的I/O操作信息该表的列分类（同table\_io\_waits\_summary\_by\_table 相同），按照(OBJECT\_TYPE, OBJECT\_SCHEMA, OBJECT\_NAME)这三个字段进行group。

## 指标分类：

**COUNT\_STAR, SUM\_TIMER\_WAIT, MIN\_TIMER\_WAIT, AVG\_TIMER\_WAIT, MAX\_TIMER\_WAIT**  
These columns aggregate all I/O operations. They are the same as the sum of the corresponding xxx\_READ and xxx\_WRITE columns.  
对应xxx\_READ 和 xxx\_WRITE 列的和

**COUNT\_READ, SUM\_TIMER\_READ, MIN\_TIMER\_READ, AVG\_TIMER\_READ, MAX\_TIMER\_READ**  
These columns aggregate all read operations. They are the same as the sum of the corresponding xxx\_FETCH columns.  
对应xxx\_FETCH 列的和

**COUNT\_WRITE, SUM\_TIMER\_WRITE, MIN\_TIMER\_WRITE, AVG\_TIMER\_WRITE, MAX\_TIMER\_WRITE**  
These columns aggregate all write operations. They are the same as the sum of the corresponding xxx\_INSERT, xxx\_UPDATE, and xxx\_DELETE columns.  
对应 xxx\_INSERT，xxx\_UPDATE，xxx\_DELETE三个值的和

**COUNT\_FETCH, SUM\_TIMER\_FETCH, MIN\_TIMER\_FETCH, AVG\_TIMER\_FETCH, MAX\_TIMER\_FETCH**  
These columns aggregate all fetch operations.  
所有的select操作次数

**COUNT\_INSERT, SUM\_TIMER\_INSERT, MIN\_TIMER\_INSERT, AVG\_TIMER\_INSERT, MAX\_TIMER\_INSERT**  
These columns aggregate all insert operations.  
所有的insert操作次数

**COUNT\_UPDATE, SUM\_TIMER\_UPDATE, MIN\_TIMER\_UPDATE, AVG\_TIMER\_UPDATE, MAX\_TIMER\_UPDATE**  
These columns aggregate all update operations.  
所有的update 操作次数

**COUNT\_DELETE, SUM\_TIMER\_DELETE, MIN\_TIMER\_DELETE, AVG\_TIMER\_DELETE, MAX\_TIMER\_DELETE**  
These columns aggregate all delete operations.  
所有的delete 操作次数  
.

对于INDEX\_NAME列：  
1\. A value of PRIMARY indicates that table I/O used the primary index.  
2\. A value of NULL means that table I/O used no index.  
3\. Inserts are counted against INDEX\_NAME = NULL

## 举例

```
<br></br>mysql&gt; select * from performance_schema.table_io_waits_summary_by_index_usage where object_schema='wstest' and object_name='test2' limit 10 ;
+-------------+---------------+-------------+------------+------------+----------------+----------------+----------------+----------------+------------+----------------+----------------+----------------+----------------+-------------+-----------------+-----------------+-----------------+-----------------+-------------+-----------------+-----------------+-----------------+-----------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+
| OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | COUNT_STAR | SUM_TIMER_WAIT | MIN_TIMER_WAIT | AVG_TIMER_WAIT | MAX_TIMER_WAIT | COUNT_READ | SUM_TIMER_READ | MIN_TIMER_READ | AVG_TIMER_READ | MAX_TIMER_READ | COUNT_WRITE | SUM_TIMER_WRITE | MIN_TIMER_WRITE | AVG_TIMER_WRITE | MAX_TIMER_WRITE | COUNT_FETCH | SUM_TIMER_FETCH | MIN_TIMER_FETCH | AVG_TIMER_FETCH | MAX_TIMER_FETCH | COUNT_INSERT | SUM_TIMER_INSERT | MIN_TIMER_INSERT | AVG_TIMER_INSERT | MAX_TIMER_INSERT | COUNT_UPDATE | SUM_TIMER_UPDATE | MIN_TIMER_UPDATE | AVG_TIMER_UPDATE | MAX_TIMER_UPDATE | COUNT_DELETE | SUM_TIMER_DELETE | MIN_TIMER_DELETE | AVG_TIMER_DELETE | MAX_TIMER_DELETE |
+-------------+---------------+-------------+------------+------------+----------------+----------------+----------------+----------------+------------+----------------+----------------+----------------+----------------+-------------+-----------------+-----------------+-----------------+-----------------+-------------+-----------------+-----------------+-----------------+-----------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+
| TABLE       | wstest        | test2       | PRIMARY    |          6 |      139617054 |        8265600 |       23269509 |       42271164 |          0 |              0 |              0 |              0 |              0 |           6 |       139617054 |         8265600 |        23269509 |        42271164 |           0 |               0 |               0 |               0 |               0 |            0 |                0 |                0 |                0 |                0 |            6 |        139617054 |          8265600 |         23269509 |         42271164 |            0 |                0 |                0 |                0 |                0 |
| TABLE       | wstest        | test2       | idx_name   |          7 |       69869781 |        6575211 |        9981081 |       23699394 |          7 |       69869781 |        6575211 |        9981081 |       23699394 |           0 |               0 |               0 |               0 |               0 |           7 |        69869781 |         6575211 |         9981081 |        23699394 |            0 |                0 |                0 |                0 |                0 |            0 |                0 |                0 |                0 |                0 |            0 |                0 |                0 |                0 |                0 |
| TABLE       | wstest        | test2       | NULL       |         25 |      249500088 |         836154 |        9979974 |       72511083 |         19 |       76699233 |         836154 |        4036491 |       28997865 |           6 |       172800855 |        10806903 |        28800081 |        72511083 |          19 |        76699233 |          836154 |         4036491 |        28997865 |            6 |        172800855 |         10806903 |         28800081 |         72511083 |            0 |                0 |                0 |                0 |                0 |            0 |                0 |                0 |                0 |                0 |
+-------------+---------------+-------------+------------+------------+----------------+----------------+----------------+----------------+------------+----------------+----------------+----------------+----------------+-------------+-----------------+-----------------+-----------------+-----------------+-------------+-----------------+-----------------+-----------------+-----------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+
3 rows in set (0.00 sec)

```

可以看出，对于test2表，PRIMARY主键共使用了6次（COUNT\_WRITE=6，即按照主键顺序写入了6行数据），idx\_name供使用了7次（COUNT\_READ=7，读了7行），再次读取后查看状态：

```
mysql&gt; explain select * from wstest.test2 where name='111xyz';
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | test2 | NULL       | ref  | idx_name      | idx_name | 93      | const |    6 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)


mysql&gt; select * from wstest.test2 where name='111xyz';   
+----+--------+--------+------+-------+
| id | name   | status | addr | addr2 |
+----+--------+--------+------+-------+
|  1 | 111xyz |    110 |      |     0 |
|  2 | 111xyz |    110 |      |     0 |
|  3 | 111xyz |    110 |      |     0 |
|  4 | 111xyz |    110 |      |     0 |
|  5 | 111xyz |      5 |      |     0 |
|  6 | 111xyz |    110 |      |     0 |
+----+--------+--------+------+-------+
6 rows in set (0.00 sec)

mysql&gt; select * from performance_schema.table_io_waits_summary_by_index_usage where object_schema='wstest' and object_name='test2' limit 10  ;
+-------------+---------------+-------------+------------+------------+----------------+----------------+----------------+----------------+------------+----------------+----------------+----------------+----------------+-------------+-----------------+-----------------+-----------------+-----------------+-------------+-----------------+-----------------+-----------------+-----------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+
| OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | INDEX_NAME | COUNT_STAR | SUM_TIMER_WAIT | MIN_TIMER_WAIT | AVG_TIMER_WAIT | MAX_TIMER_WAIT | COUNT_READ | SUM_TIMER_READ | MIN_TIMER_READ | AVG_TIMER_READ | MAX_TIMER_READ | COUNT_WRITE | SUM_TIMER_WRITE | MIN_TIMER_WRITE | AVG_TIMER_WRITE | MAX_TIMER_WRITE | COUNT_FETCH | SUM_TIMER_FETCH | MIN_TIMER_FETCH | AVG_TIMER_FETCH | MAX_TIMER_FETCH | COUNT_INSERT | SUM_TIMER_INSERT | MIN_TIMER_INSERT | AVG_TIMER_INSERT | MAX_TIMER_INSERT | COUNT_UPDATE | SUM_TIMER_UPDATE | MIN_TIMER_UPDATE | AVG_TIMER_UPDATE | MAX_TIMER_UPDATE | COUNT_DELETE | SUM_TIMER_DELETE | MIN_TIMER_DELETE | AVG_TIMER_DELETE | MAX_TIMER_DELETE |
+-------------+---------------+-------------+------------+------------+----------------+----------------+----------------+----------------+------------+----------------+----------------+----------------+----------------+-------------+-----------------+-----------------+-----------------+-----------------+-------------+-----------------+-----------------+-----------------+-----------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+
| TABLE       | wstest        | test2       | PRIMARY    |          6 |      139617054 |        8265600 |       23269509 |       42271164 |          0 |              0 |              0 |              0 |              0 |           6 |       139617054 |         8265600 |        23269509 |        42271164 |           0 |               0 |               0 |               0 |               0 |            0 |                0 |                0 |                0 |                0 |            6 |        139617054 |          8265600 |         23269509 |         42271164 |            0 |                0 |                0 |                0 |                0 |
| TABLE       | wstest        | test2       | idx_name   |         13 |      164544849 |        6575211 |       12657069 |       74654973 |         13 |      164544849 |        6575211 |       12657069 |       74654973 |           0 |               0 |               0 |               0 |               0 |          13 |       164544849 |         6575211 |        12657069 |        74654973 |            0 |                0 |                0 |                0 |                0 |            0 |                0 |                0 |                0 |                0 |            0 |                0 |                0 |                0 |                0 |
| TABLE       | wstest        | test2       | NULL       |         25 |      249500088 |         836154 |        9979974 |       72511083 |         19 |       76699233 |         836154 |        4036491 |       28997865 |           6 |       172800855 |        10806903 |        28800081 |        72511083 |          19 |        76699233 |          836154 |         4036491 |        28997865 |            6 |        172800855 |         10806903 |         28800081 |         72511083 |            0 |                0 |                0 |                0 |                0 |            0 |                0 |                0 |                0 |                0 |
+-------------+---------------+-------------+------------+------------+----------------+----------------+----------------+----------------+------------+----------------+----------------+----------------+----------------+-------------+-----------------+-----------------+-----------------+-----------------+-------------+-----------------+-----------------+-----------------+-----------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+--------------+------------------+------------------+------------------+------------------+
3 rows in set (0.00 sec)


```

得到，**再次使用idx\_name查询数据时，返回了6条数据，在统计结果中，idx\_name的COUNT\_STAR从7增加到了13（均为COUNT\_READ，COUNT\_READ从7变为13）**，据此，可使用该统计信息判断索引使用的频次。

# sys库中的统计信息

sys数据库中整合了performance\_schema数据库的指标，例如：

```
mysql&gt; show create table sys.schema_index_statistics \G    
*************************** 1. row ***************************
                View: schema_index_statistics
         Create View: CREATE ALGORITHM=MERGE DEFINER=`mysql.sys`@`localhost` SQL SECURITY INVOKER VIEW `sys`.`schema_index_statistics` (`table_schema`,`table_name`,`index_name`,`rows_selected`,`select_latency`,`rows_inserted`,`insert_latency`,`rows_updated`,`update_latency`,`rows_deleted`,`delete_latency`) AS select `performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_SCHEMA` AS `table_schema`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`OBJECT_NAME` AS `table_name`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` AS `index_name`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_FETCH` AS `rows_selected`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_FETCH`) AS `select_latency`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_INSERT` AS `rows_inserted`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_INSERT`) AS `insert_latency`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_UPDATE` AS `rows_updated`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_UPDATE`) AS `update_latency`,`performance_schema`.`table_io_waits_summary_by_index_usage`.`COUNT_DELETE` AS `rows_deleted`,`sys`.`format_time`(`performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_DELETE`) AS `delete_latency` from `performance_schema`.`table_io_waits_summary_by_index_usage` where (`performance_schema`.`table_io_waits_summary_by_index_usage`.`INDEX_NAME` is not null) order by `performance_schema`.`table_io_waits_summary_by_index_usage`.`SUM_TIMER_WAIT` desc
character_set_client: utf8mb4
collation_connection: utf8mb4_0900_ai_ci
1 row in set (0.00 sec)

```

与之对应的sys.x$schema\_index\_statistics则提供了更为精确的value，方便工具进行调用。

## 举例

查询：

```
mysql&gt; select * from sys.schema_index_statistics where table_schema='wstest' and table_name='test2';
+--------------+------------+------------+---------------+----------------+---------------+----------------+--------------+----------------+--------------+----------------+
| table_schema | table_name | index_name | rows_selected | select_latency | rows_inserted | insert_latency | rows_updated | update_latency | rows_deleted | delete_latency |
+--------------+------------+------------+---------------+----------------+---------------+----------------+--------------+----------------+--------------+----------------+
| wstest       | test2      | PRIMARY    |             0 | 0 ps           |             0 | 0 ps           |            6 | 139.62 us      |            0 | 0 ps           |
| wstest       | test2      | idx_name   |             7 | **69.87 us**       |             0 | 0 ps           |            0 | 0 ps           |            0 | 0 ps           |
+--------------+------------+------------+---------------+----------------+---------------+----------------+--------------+----------------+--------------+----------------+
2 rows in set (0.00 sec)


```

对应之前的再次按照idx\_name查询，再次查询sys的统计信息：

```
mysql&gt; select * from sys.schema_index_statistics where table_schema='wstest' and table_name='test2';
+--------------+------------+------------+---------------+----------------+---------------+----------------+--------------+----------------+--------------+----------------+
| table_schema | table_name | index_name | rows_selected | select_latency | rows_inserted | insert_latency | rows_updated | update_latency | rows_deleted | delete_latency |
+--------------+------------+------------+---------------+----------------+---------------+----------------+--------------+----------------+--------------+----------------+
| wstest       | test2      | idx_name   |            13 | **164.54 us**      |             0 | 0 ps           |            0 | 0 ps           |            0 | 0 ps           |
| wstest       | test2      | PRIMARY    |             0 | 0 ps           |             0 | 0 ps           |            6 | 139.62 us      |            0 | 0 ps           |
+--------------+------------+------------+---------------+----------------+---------------+----------------+--------------+----------------+--------------+----------------+
2 rows in set (0.00 sec)

```