---
id: 297
title: MySQL中MVCC是否也能防止幻读
date: '2019-09-03T14:43:10+08:00'
author: Shuo
excerpt: 本文主要描述MVCC和GAP-lock与幻读之间的关系，分别解决了那种情况下的幻读。
layout: post
guid: 'http://codercoder.cn/?p=297'
permalink: /index.php/2019/09/mysql-mvcc-phantom-read/
bigfa_ding:
    - '2'
views:
    - '3279'
    - '3279'
    - '3279'
    - '3279'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# 前言

 前段时间，小伙伴问了我一个问题：*在RR级别下，MVCC是否也能防止幻读的产生？* 本篇文章主要分析一下这个问题，不重点介绍MVCC（InnoDB Multi-Versioning, 多版本并发控制）和幻读的概念。  
![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-115.jpg)

# MVCC与幻读概念

 [MVCC](https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html)：将变更的数据行保存为不同的版本，以应对不同的并发线程。  
 [幻读](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)：表述在同一个事务中的连续多次查询，返回不同的结果。例如第二次查询中读取到了第一次查询没有的rows。  
 [Gap锁](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-gap-locks)：A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record.

# MVCC防止幻读

## 对于select

 在RR（Repeatable-Read）隔离级别下，第一个线程的select语句，可以通过MVCC，不读取在事务begin后的数据行version，即**只有select查询时，可以通过MVCC防止读取到其它线程插入的数据**。

## 对于数据变更DML

 同样在RR隔离级别下，由于DML需要读取到最新的数据版本（也称为：**当前读**），MVCC只能根据数据版本进行是否能够读取，DML在操作数据时候，则会把其它线程更改的数据也视为符合条件，导致出现DML操作了不该操作的数据，所以不能符合要求。  
 此时，就需要一个锁来锁住这个间隙，防止出现幻读的现象，这个锁在MySQL中为**Gap锁**。

## Gap锁防止幻读

在事务中执行DML时，为了防止符合where条件的结果集被改变，所以需要进行加锁，需要根据锁类型的不同（主键索引，非主键索引（又包括唯一索引和普通索引）），加上不同的区间：  
**通用：** 符合条件的index需要加next-key，左开右闭区间  
**主键/唯一索引：**  
 **等值查询：** 只加在必要的行上  
 **区间查询：** 向右遍历到不符合条件的index为止  
**普通索引：**  
 **等值查询：** 按照 **通用** 的规则，加next-key锁  
 **区间查询：** 向右遍历到第一个不满足条件的index，此时next-key退化为gap

通过以上的加锁方式，线程访问的时候，防止其它线程对结果集进行修改，从而达到防止幻读的情况。所以，**通常我们说：在RR隔离级别下，采用Gap锁防止幻读的产生。**