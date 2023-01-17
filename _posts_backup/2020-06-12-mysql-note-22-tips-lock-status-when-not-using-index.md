---
id: 1155
title: 'MySQL手记22 &#8212; Tips：不走索引就锁全表数据吗？'
date: '2020-06-12T19:07:19+08:00'
author: Shuo
excerpt: 在RC隔离级别下，没走索引时，可以更新不同的行；RR级别下，没走索引时，不可以更新不同的行。RC级别只锁定加锁的一行，但是提交之后，再进行查询时，可能会获取到其它事务更新的结果，所以为不可重复读；RR级别通过GAP锁，防止其它的session更新所有的行与间隙，从而得到了一个可重复读取的结果。
layout: post
guid: 'http://codercoder.cn/?p=1155'
permalink: /index.php/2020/06/mysql-note-22-tips-lock-status-when-not-using-index/
views:
    - '16813'
    - '16813'
    - '16813'
    - '16813'
    - '16813'
    - '16813'
    - '16813'
    - '16813'
    - '16813'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

今天和小伙伴讨论到：  
 如果MySQL的加锁，没有走索引，走全表扫描的话，那么加锁是把所有的数据行都锁住，还是只锁住符合where条件的数据？  
 确实，我们在工作中经常提醒开发人员，让SQL都能走索引，以使得加锁的范围越小越好。所以这个问题，还需要看一个方面：**事务的隔离级别**。  
![](http://codercoder.cn/wp-content/uploads/2020/06/2020-06-1223.png)

# Repeatable\_Read隔离级别

表结构：name字段没有索引，在name字段上进行update操作：

```
CREATE TABLE if not exists `test3` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8
​
mysql&gt; select * from test3;
+----+-----------+
| id | name      |
+----+-----------+
|  1 | aaa       |
|  2 | bbb       |
|  3 | ccc333ccc |
|  4 | ddd       |
|  5 | eee       |
+----+-----------+
5 rows in set (0.00 sec)

```

   
开启一个事务，执行update进行加锁：

```
mysql&gt; begin;
Query OK, 0 rows affected (0.01 sec)
​
​
mysql&gt; update test3 set name='abc' where name='ccc333ccc';
Query OK, 1 row affected (0.02 sec)

```

   
查看当前加锁的情况：

```
root:information_schema&gt; select * from INNODB_TRX \G
*************************** 1. row ***************************
                    trx_id: 458026
                 trx_state: RUNNING
               trx_started: 2020-06-12 05:00:40
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 972
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 6
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0

```

   
或者可使用\*\*show engine innodb status \*\*查看：

```
------------
TRANSACTIONS
------------
Trx id counter 458027
Purge done for trx's n:o &lt; 458010 undo n:o  begin;
Query OK, 0 rows affected (0.00 sec)
​
root:wstestdb&gt; update test3 set name
----等待

```

   
第二个事务执行后，处于等待状态，此时查看加锁的情况：

```
mysql&gt; select * from information_schema.INNODB_TRX \G
*************************** 1. row ***************************
                    trx_id: 458027
                 trx_state: LOCK WAIT
               trx_started: 2020-06-12 05:10:27
     trx_requested_lock_id: 458027:77:3:2
          trx_wait_started: 2020-06-12 05:10:27
                trx_weight: 2
       trx_mysql_thread_id: 993
                 trx_query: update test3 set name='abc' where name='eee'
       trx_operation_state: starting index read
         trx_tables_in_use: 1
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 0
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 458026
                 trx_state: RUNNING
               trx_started: 2020-06-12 05:00:40
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 972
                 trx_query: select * from information_schema.INNODB_TRX
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 6
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
2 rows in set (0.01 sec)

```

第二个事务的状态显示：**trx\_state: LOCK WAIT**，即由于不能马上获得锁，所以需要等待。

## 结果二：

 即使第二个事务中，update的是不同的行，但是由于name字段上没有索引，所以InnoDB需要对所有的行及间隙上锁，所以会出现“LOCK WAIT”的状态。

# Read-Committed隔离级别

 由于RC隔离级别是没有GAP锁的，所以在进行加锁的时候，只会对于符合条件的数据进行加锁：

```
mysql&gt; begin;
Query OK, 0 rows affected (0.00 sec)
​
​
mysql&gt; update test3 set name='abc' where name='ccc333ccc';
Query OK, 1 row affected (0.00 sec)

```

   
查看此时的加锁情况：

```
mysql&gt; select * from information_schema.INNODB_TRX \G
*************************** 1. row ***************************
                    trx_id: 458034
                 trx_state: RUNNING
               trx_started: 2020-06-12 06:15:58
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 998
                 trx_query: select * from information_schema.INNODB_TRX
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: READ COMMITTED
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
1 row in set (0.01 sec)

```

   
第二个session执行：

```
root:wstestdb&gt; begin;
Query OK, 0 rows affected (0.00 sec)
​
​
root:wstestdb&gt; update test3 set name='abc' where name='eee';
Query OK, 1 row affected (0.01 sec)
​

```

## 结果一：

可以看出，第二个session也是能够成功执行的，因为更改的是不同的行。

查看加锁的情况：

```
mysql&gt; select * from information_schema.INNODB_TRX \G
*************************** 1. row ***************************
                    trx_id: 458035
                 trx_state: RUNNING
               trx_started: 2020-06-12 06:16:22
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 997
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 2
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: READ COMMITTED
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
                    trx_id: 458034
                 trx_state: RUNNING
               trx_started: 2020-06-12 06:15:58
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 3
       trx_mysql_thread_id: 998
                 trx_query: select * from information_schema.INNODB_TRX
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 2
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 1
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: READ COMMITTED
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
trx_adaptive_hash_latched: 0
trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
2 rows in set (0.01 sec)

```

# 小结

这个例子也是“不可重复读”的一个体现：  
 RC：没走索引时，可以更新不同的行  
 RR：没走索引时，不可以更新不同的行

 在RC级别下，虽然只更新了一行数据update test3 set name=’abc’ where name=’ccc333ccc’;，但是提交之后，再进行查询时，得到name=‘eee’的这行数据也被更新了。  
 而RR级别通过GAP锁，防止其它的session更新所有的行与间隙，从而得到了一个可重复读取的结果。