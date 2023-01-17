---
id: 426
title: MySQL死锁案例_唯一索引
date: '2019-09-08T16:18:33+08:00'
author: Shuo
excerpt: 间隙锁定可以显式禁用：将事务隔离级别更改为READ-COMMITTED或启用innodb_locks_unsafe_for_binlog系统变量。在这种情况下，对于搜索和索引扫描，间隙锁定将被禁用，此时gap锁仅用于外键约束检查和重复键检查。
layout: post
guid: 'http://codercoder.cn/?p=426'
permalink: /index.php/2019/09/mysql-deadlock-unique-index-example/
views:
    - '5812'
    - '5812'
    - '5812'
bigfa_ding:
    - '1'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 1. 情况描述

 近期在MySQL数据库中产生了死锁的情况，与通常的死锁不同，由于表中有唯一索引，所以加锁方式也比较有趣，本文将对于该例进行阐述(本文将对数据进行脱敏操作)：

环境描述:  
\+ 隔离级别：READ-COMMITTED

表结构：

```
show create table  \G

CREATE TABLE if not exists `uniq` (

`id` int(11) NOT NULL AUTO_INCREMENT,

`aa` int(11) DEFAULT NULL,

`bb` int(11) DEFAULT NULL,

PRIMARY KEY (`id`),

UNIQUE KEY `uniq_ab` (`aa`,`bb`)

) ENGINE=InnoDB DEFAULT CHARSET=utf8

```

**注意其中存在唯一索引：UNIQUE KEY `uniq_ab` (`aa`,`bb`)**

# 2. 分析

SQL执行顺序：  
![SQL执行顺序](http://121.40.246.76/wp-content/uploads/2019/09/2019-09-0839.png)

**问题解析：**  
\+ 对于唯一索引，insert成功后，会加上X lock；

- 对于唯一索引的插入，需要在插入前进行duplicate key的检查，所以需要申请加上S lock，由于S lock与X lock不兼容，所以产生锁等待；
- 由于GAP与INSERT\_INTENTION不兼容，所以产生锁等待，至此死锁产生。

# 3. 基本概念

MySQL Innodb中的锁：  
(参照：https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)：

- S LOCK：共享锁，也称读锁
- X LOCK：排它锁，也称写锁
- IS LOCK：暗示一个事务需要加共享锁
- IX LOCK：暗示一个事务需要加排它锁

对应的兼容关系：  
![锁兼容关系](http://121.40.246.76/wp-content/uploads/2019/09/2019-09-0883.png)

此外，还有一套更为精准的判断逻辑，以符合更多场景，所以MySQL还具有以下几类的锁类型：

- Record lock(记录锁)：加在索引行上的锁。
- GAP lock(间隙锁)：加在record两侧间隙上的锁，但是不包括数据本身。
- Next-key lock：Record lock + GAP lock
- INSERT\_INTENTION lock(插入意向锁)：在insert之前，需要申请INSERT\_INTENTION lock

举例:

```
   Session1:                                   
   mysql> CREATE TABLE if not exists child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;                             
   mysql> INSERT INTO child (id) values (90),(102);                                           
   mysql> START TRANSACTION;                                           
   mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;                     

   Session2: 
   mysql> START TRANSACTION;                                                                
   mysql> INSERT INTO child (id) VALUES (101);           


```

**则Session2会产生INSERT\_INTENTION lock waiting**

> Insert对于唯一索引的加锁方式：  
>  **对于insert的行，加上X lock，在Insert之前，需要加上 INSERT\_INTENTION lock。**

# 4. 问题解决

**（1）方案一(调整后端)：**  
 由于事发的逻辑为：一个事务中进行多次insert。在数据库层面的展示即为多次申请锁资源，并且是并发的事务，所以当插入的数据出现资源抢占时，容易发生死锁。

所以建议：**insert时插入多个值，一次性申请该sql的所有锁资源。这样，则可以避免多次申请锁资源，同时在性能上也能得以提升。**

**（2）方案二(调整前端逻辑)：**  
 规避重复触发的情况，防止不同事务在唯一索引上插入相同数据。

# 5. 附加

[innodb-record-level-locks](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-gap-locks)

`Gap locking can be disabled explicitly. This occurs if you change the transaction isolation level to READ COMMITTED or enable the innodb_locks_unsafe_for_binlog system variable. Under these circumstances, gap locking is disabled for searches and index scans and is used only for foreign-key constraint checking and duplicate-key checking.`  
 间隙锁定可以显式禁用：将事务隔离级别更改为READ-COMMITTED或启用innodb\_locks\_unsafe\_for\_binlog系统变量。在这种情况下，对于搜索和索引扫描，间隙锁定将被禁用，此时gap锁仅用于外键约束检查和重复键检查。