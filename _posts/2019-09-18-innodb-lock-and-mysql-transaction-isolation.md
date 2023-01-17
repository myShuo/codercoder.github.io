---
id: 656
title: InnoDB锁及MySQL事务隔离级别
date: '2019-09-18T10:39:06+08:00'
author: Shuo
excerpt: InnoDB锁及MySQL事务隔离级别简介。
layout: post
guid: 'http://codercoder.cn/?p=656'
permalink: /index.php/2019/09/innodb-lock-and-mysql-transaction-isolation/
views:
    - '11953'
    - '11953'
    - '11953'
    - '11953'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# Innodb锁机制

参看文档:  
<sup id="fnref-656-1">[1](#fn-656-1)</sup>  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1835-150x150.jpg)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1835.jpg)

## 共享锁和排它锁

**Share-lock**：简单理解为读取某些行  
**Exclusive-lock**：操作某些行

## 意向锁

InnoDB提供了多个粒度的锁，允许行锁和表锁的共同存在。例如在lock table ..write 时，允许其他事务加上意向锁。意向锁为在需要加上读锁或者写锁之前，需要首先申请意向锁，意向锁的分类：  
1\. 意向共享锁intention shared lock (IS)：  
2\. 意向排它锁intention exclusive lock (IX)：  
例如：select … lock in share mode为IS，select … for update为IX。

|  | X | IX | S | IS |
|:-:|---|---|---|---|
| X | Conflict | Conflict | Conflict | Conflict |
| IX | Conflict | Compatible | Conflict | Compatible |
| S | Conflict | Conflict | Compatible | Compatible |
| IS | Conflict | Compatible | Compatible | Compatible |

**意向锁不会和任何类型的锁冲突，除了LOCK TABLES … WRITE**

## 行锁

锁在索引列，

```
SHOW ENGINE INNODB STATUS：

RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;

```

## Gap锁

锁在两个索引行之间，或者无穷小到最小索引值，最大索引值到无穷大。例如：

```
SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE; 

```

对于使用唯一索引锁定行以搜索唯一行的语句，不需要gap锁，（搜索条件只包含多列惟一索引的某些列，还是可能发生gap）  
\*\*\*\*READ COMMITTED隔离级别下，gap锁仅用作外键检查和冲突检查\*\*

> It is also worth noting here that conflicting locks can be held on a  
>  gap by different transactions. For example, transaction A can hold a  
>  shared gap lock (gap S-lock) on a gap while transaction B holds an  
>  exclusive gap lock (gap X-lock) on the same gap. The reason  
>  conflicting gap locks are allowed is that if a record is purged from  
>  an index, the gap locks held on the record by different transactions  
>  must be merged.
> 
>  Gap locks in InnoDB are “purely inhibitive”, which means that their  
>  only purpose is to prevent other transactions from inserting to the  
>  gap. Gap locks can co-exist. A gap lock taken by one transaction does  
>  not prevent another transaction from taking a gap lock on the same  
>  gap. There is no difference between shared and exclusive gap locks.  
>  They do not conflict with each other, and they perform the same  
>  function.

## Next-Key Locks

索引值及其前面的gap，假设一个索引列包好以下值：10, 11, 13, and 20

```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)

```

**在RR隔离级别下，MySQL使用Next-Key Locks防止幻读<sup id="fnref-656-2">[2](#fn-656-2)</sup>。**  
**幻读：**  
 同一个查询在不同的时间得到不同的结果集。  
例如一个表的索引字段有90,102两个值，当执行：

```
SELECT * FROM child WHERE id &gt; 100 FOR UPDATE;

```

另一个会话此时插入value=101，则再次查询时，出现value=101的值，违背了事务的ACID规则。

## 插入意向锁Insert Intention Locks

插入意向锁为在插入之前需要加上的gap锁，**作用**：在多个事务需要插入的在时候，如果需要插入的gap位置不一样，则不用进行等待。例如两个索引值为4和7，现有两个事务需要分别插入5和6的值，则在获取到对应行的排它锁之前，会加上插入意向锁，但由于是插入不同的值，索引事务不会相互阻塞。

```
mysql&gt; CREATE TABLE if not exists child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql&gt; INSERT INTO child (id) values (90),(102);

mysql&gt; START TRANSACTION;
mysql&gt; SELECT * FROM child WHERE id &gt; 100 FOR UPDATE;

B：
mysql&gt; START TRANSACTION;
mysql&gt; INSERT INTO child (id) VALUES (101);

```

此时查看锁的状态：

```
show engine innodb status:
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...

```

## AUTO-INC Locks自增锁

**自增锁是一种表级锁**，在insert一条自增字段的值时，需要加上自增锁。简单来说，当一个事务插入自增列值时，另外一个需要插入该列的insert事务需要等待，以得到连续递增的值。  
当然，我们可以通过控制innodb\_autoinc\_lock\_mode参数进行自增的控制<sup id="fnref-656-3">[3](#fn-656-3)</sup>。

在MySQL 5.1.22之前，innodb使用一个表锁解决自增字段的一致性问题，但是如果大量的并发插入就会引起SQL堵塞，不但影响效率，而且可能会瞬间达到max\_connections而崩溃。

*在 5.1.22之后，innodb使用新的方式解决自增字段一致性问题，对于可以预判行数的insert语句，innodb使用一个轻量级的互斥量。如：某一insert语句1执行前，表的AUTO\_INCREMENT=1，语句A的插入行数已知为3，innodb在语句1的实际插入操作执行前就预分配给该语句三个自增值，当有一个新的insert语句2要执行时，读取的AUTO\_INCREMENT=4，这样虽然语句1可能还没有执行完，语句2就可直接执行无需等待语句2。*

这种方式对于可预判插入行数的插入语句有效，如：insert和replace。对于无法提前获知插入行数的语句，如：insert…select…、replace…select…和load data则innodb还是使用表锁。

```
  innodb_autoinc_lock_mode = 0 (“traditional” lock mode：全部使用表锁) 
  innodb_autoinc_lock_mode = 1 (默认)(“consecutive” lock mode：可预判行数时使用新方式，不可时使用表锁)  
  innodb_autoinc_lock_mode = 2 (“interleaved” lock mode：全部使用新方式，不安全，不适合replication) 

```

# MySQL事务隔离级别

## 事务隔离级别简介

事务隔离级别（Transaction Isolation Levels）是数据库的基础，ISOLATION即为ACID属性中的I项。隔离级别用以控制在多个事务同时进行更改和执行查询时，对性能和结果的可靠性、一致性和重现性之间的平衡进行设置。

## 一致性非锁定读（Consistent Nonlocking Reads）

InnoDB使用多版本控制，以显示数据库在某一时刻的快照数据。查询查看在此时间点之前提交的事务所做的更改，而稍后或未提交的事务则不做任何更改<sup id="fnref-656-4">[4](#fn-656-4)</sup>。

## REPEATABLE READ

是InnoDB默认的隔离级别，可以做到一致非锁定读 <sup id="fnref2:656-4">[4](#fn-656-4)</sup>。在RR( REPEATABLE READ)级别下，事务中的读取总是读的是第一次读取时创建的快照数据，若需要获取最新的状态，需要提交事务。

| Session A | Session B |
|:-:|---|
| SET autocommit=0; | SET autocommit=0; |
| SELECT \* FROM t; —-empty set |  |
|  | INSERT INTO t VALUES (1, 2); |
| SELECT \* FROM t; —-empty set |  |
|  | COMMIT; |
| SELECT \* FROM t; —-empty set |  |
| COMMIT; |  |
| SELECT \* FROM t; —–(1,2) |  |

## READ COMMITTED

对于RC(READ COMMITTED)级别，一致性读每次读到的都是最新的快照，即每次读取到的数据均为最新版本。RC级别由于没有了gap锁，所以会出现幻读。binlog\_format必须为ROW。  
**RC级别事务的影响：**  
1\. 对于UPDATE或DELETE语句，InnoDB只对它更新或删除的行持有锁。非匹配行的记录锁在MySQL评估WHERE条件之后立即释放。这大大降低了死锁的可能性，但它们仍然可能发生。  
2\. 对于UPDATE语句，如果一行已经被锁定，InnoDB执行“semi-consistent”读取，将最新提交的版本返回给MySQL，以便MySQL可以确定该行是否匹配更新的WHERE条件。如果行匹配(必须更新)，MySQL将再次读取该行，这次InnoDB要么锁定该行，要么等待锁。

举例：

```
CREATE TABLE if not exists t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
COMMIT;

--Session A
START TRANSACTION;
UPDATE t SET b = 5 WHERE b = 3;

--Session B
UPDATE t SET b = 4 WHERE b = 2;

```

**如果事务中不修改该行，则马上会释放锁，如果修改该行，则会在事务结束后进行释放。**

对于RR级别：

```
事务A
x-lock(1,2); retain x-lock
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); retain x-lock
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); retain x-lock

```

在符合条件的行接上X锁，并且不进行释放，所以事务B在更新时候，需要等待事务A的提交或者释放：

```
x-lock(1,2); block and wait for first UPDATE to commit or roll back

```

**对于RC级别：**

```
事务A：
x-lock(1,2); unlock(1,2)
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); unlock(3,2)
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); unlock(5,2)

```

事务B：

```
x-lock(1,2); update(1,2) to (1,4); retain x-lock
x-lock(2,3); unlock(2,3)
x-lock(3,2); update(3,2) to (3,4); retain x-lock
x-lock(4,3); unlock(4,3)
x-lock(5,2); update(5,2) to (5,4); retain x-lock

```

\*\*但是，如果WHERE条件包含索引列，并且InnoDB使用索引，那么在获取和保留记录锁时只考虑索引列。((

```
CREATE TABLE if not exists t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2,3),(2,2,4);
COMMIT;

--Session A
START TRANSACTION;
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

--Session B
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;

```

<div class="footnotes" role="doc-endnotes">- - - - - -

1. [InnoDB locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html) [↩︎](#fnref-656-1)
2. [Phantom Rows](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html) [↩︎](#fnref-656-2)
3. [AUTO\_INCREMENT Handling in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html) [↩︎](#fnref-656-3)
4. [Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html) [↩︎](#fnref-656-4) [↩︎](656-4)

</div>