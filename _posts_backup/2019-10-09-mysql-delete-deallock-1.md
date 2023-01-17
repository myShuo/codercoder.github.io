---
id: 702
title: MySQL删除数据死锁案例分析
date: '2019-10-09T16:21:20+08:00'
author: Smile
excerpt: MySQL删除数据死锁案例分析一
layout: post
guid: 'http://codercoder.cn/?p=702'
permalink: /index.php/2019/10/mysql-delete-deallock-1/
views:
    - '9480'
bigfa_ding:
    - '2'
categories:
    - Tech
    - Fell-in-pit
tags:
    - MySQL
    - Tech
    - Fell-in-pit
---

![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0899-1-150x150.jpg)

### 表结构

```sql
CREATE TABLE if not exists `queue` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `unit_id` varchar(64) NOT NULL ,
  `queue_id` varchar(32) NOT NULL ,
  `mq_keys` varchar(64) NOT NULL ,
  `mq_tags` varchar(64) NOT NULL ,
  `mq_body` varchar(2048) DEFAULT NULL ,
  `expect_tm` datetime NOT NULL ,
  `ins_tm` datetime NOT NULL ,
  `upd_tm` datetime NOT NULL ,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_keys_tags` (`mq_keys`,`mq_tags`) USING BTREE,
  KEY `idx_queue_id_expect_tm` (`queue_id`,`expect_tm`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8

```

### 死锁业务场景

发生死锁场景是针对于同一条数据，并发情况下，一个请求根据唯一索引去删除该条数据；另一个请求是根据主键id去删除该条数据

### 死锁日志

```sql
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-10-21 16:36:11 7fe396bec700
*** (1) TRANSACTION:
TRANSACTION 6247680825, ACTIVE 0.003 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1184, 2 row lock(s)
LOCK BLOCKING MySQL thread id: 168275072 block 168314813
MySQL thread id 168314813, OS thread handle 0x7fe394a79700, query id 11307015122 110.24.17.162 queue updating
delete from `queue`
     WHERE (  mq_keys = '359843651133825024'
                  and mq_tags = 'test' )
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 28 page no 41578 n bits 160 index `PRIMARY` of table `db`.`queue` trx id 6247680825 lock_mode X locks rec but not gap waiting
Record lock, heap no 37 PHYSICAL RECORD: n_fields 11; compact format; info bits 32


*** (2) TRANSACTION:
TRANSACTION 6247680812, ACTIVE 0.005 sec updating or deleting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 168275072, OS thread handle 0x7fe396bec700, query id 11307015098 110.24.17.162 queue updating
delete from `queue`
    where id = 359848240616589824
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 28 page no 41578 n bits 160 index `PRIMARY` of table `db`.`queue` trx id 6247680812 lock_mode X locks rec but not gap
Record lock, heap no 37 PHYSICAL RECORD: n_fields 11; compact format; info bits 32


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 28 page no 7583 n bits 344 index `uniq_keys_tags` of table `db`.`queue` trx id 6247680812 lock_mode X locks rec but not gap waiting
Record lock, heap no 275 PHYSICAL RECORD: n_fields 3; compact format; info bits 0


*** WE ROLL BACK TRANSACTION (1)

```

### 死锁日志分析

| 步骤 | 事务1 | 事务2 |
|---|---|---|
| 1 | begin: | begin: |
| 2 |  | delete from `queue` where id = ? （事务2获得锁：2 lock struct(s)：2种锁结构，分别为IX和主键的行锁） |
| 3 | delete from `queue` WHERE ( mq\_keys = ? and mq\_tags = ? ) (事务1此时需要获得锁：3 lock struct(s)：3种锁结构，分别为IX，uniq\_keys\_tags（该keys\_tags唯一索引锁）和主键的行锁，此时由于事务2获得了主键的行锁，所以事务1在这里只能获取到：IX，uniq\_keys\_tags两个锁，等待事务2释放主键的行锁) |  |
| 4 |  | 事务2继续获取锁：在找到id=x 的记录，对id=x 记录加上X锁之后，同时，会根据读取到的keys\_tags列，然后将唯一索引上的keys\_tags对应的索引项加X锁此时由于事务一已经拿到了该条记录唯一索引上的锁，需要等待事务1释放，于是死锁 |

### 总结

项目中删除数据最少都是根据主键id去删除，减少死锁的可能

### 参考文档

<https://blog.csdn.net/oyl822/article/details/42297773>  
<https://www.centos.bz/2017/09/mysql-delete-%e5%88%a0%e9%99%a4%e8%af%ad%e5%8f%a5%e5%8a%a0%e9%94%81%e5%88%86%e6%9e%90/>