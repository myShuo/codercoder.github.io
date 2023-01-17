---
id: 662
title: MySQL中加字段设置默认值的问题
date: '2019-09-18T10:45:44+08:00'
author: Shuo
excerpt: 在一个MySQL数据库多活的场景里，执行DDL新增字段需要进行更加准确的控制，本文主要介绍在加字段的时候，新字段默认值的问题及控制。
layout: post
guid: 'http://codercoder.cn/?p=662'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1874.png
permalink: /index.php/2019/09/mysql-attention-default-value-when-adding-column/
views:
    - '20339'
    - '20339'
bigfa_ding:
    - '1'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 背景

在一个MySQL数据库多活的场景里，执行DDL新增字段需要进行更加准确的控制：  
1\. 在alter语句中，不能指定默认值，因为这回导致先加上默认值的一端同步到另一端的数据中心含有默认值，导致同步报错。（**不指定默认值时即为default NULL**）  
2\. 若先加上default NULL的字段，那么数据库中该字段则全为NULL，往往业务需要新增字段的值为非NULL，此时，就需要对该字段批量更新

若需要变更的表为大表，那么进行批量的更新就会非常消耗资源，产生大量binlog，还会导致同步延迟升高。  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1874.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1874.png)

# Repeat & Fix

## 环境：

MySQL version: 5.7.22  
sql\_mode=””

```
CREATE TABLE if not exists `test1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(10) DEFAULT NULL COMMENT 'name',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=20250319 DEFAULT CHARSET=utf8

mysql&gt; select count(1) from test1;
+----------+
| count(1) |
+----------+
| 16777219 |
+----------+


```

**需求：** 对这样的一个表进行添加字段，并且设置新加字段的默认值为1。

## 情况一：

**ALTER语句中直接设置默认值：**

```
mysql&gt; **alter table test1 add column status tinyint default 1;**

mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+
| id       | name    | status |
+----------+---------+--------+
| 20250318 | aaqqwww |      1 |
| 20250317 | aaqq    |      1 |
| 20250316 | aaqq    |      1 |
| 20184908 | qq      |      1 |
| 20184907 | qq      |      1 |
| 20184906 | qq      |      1 |
| 20184905 | qq      |      1 |
| 20184904 | qq      |      1 |
| 20184903 | qq      |      1 |
| 20184902 | qq      |      1 |
+----------+---------+--------+

mysql&gt; insert into test1(id,name) values (null,'qwe');
Query OK, 1 row affected (0.00 sec)

mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+
| id       | name    | status |
+----------+---------+--------+
| 20250319 | qwe     |      1 |
| 20250318 | aaqqwww |      1 |
| 20250317 | aaqq    |      1 |
| 20250316 | aaqq    |      1 |
| 20184908 | qq      |      1 |
| 20184907 | qq      |      1 |
| 20184906 | qq      |      1 |
| 20184905 | qq      |      1 |
| 20184904 | qq      |      1 |
| 20184903 | qq      |      1 |
+----------+---------+--------+


```

可见：在ALTER语句中直接设置默认值，则表中的历史数据对应的字段也会被更新为默认值1。

## 情况二：

**先加字段，再设置默认值：**

```
mysql&gt; **alter table test1 add column addr tinyint ;**

mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+------+
| id       | name    | status | addr |
+----------+---------+--------+------+
| 20250319 | qwe     |      1 | NULL |
| 20250318 | aaqqwww |      1 | NULL |
| 20250317 | aaqq    |      1 | NULL |
| 20250316 | aaqq    |      1 | NULL |
| 20184908 | qq      |      1 | NULL |
| 20184907 | qq      |      1 | NULL |
| 20184906 | qq      |      1 | NULL |
| 20184905 | qq      |      1 | NULL |
| 20184904 | qq      |      1 | NULL |
| 20184903 | qq      |      1 | NULL |
+----------+---------+--------+------+


----设置新增字段addr的默认值为1：
mysql&gt; **alter table test1 alter addr set default 1;**
Query OK, 0 rows affected (0.00 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+------+
| id       | name    | status | addr |
+----------+---------+--------+------+
| 20250319 | qwe     |      1 | NULL |
| 20250318 | aaqqwww |      1 | NULL |
| 20250317 | aaqq    |      1 | NULL |
| 20250316 | aaqq    |      1 | NULL |
| 20184908 | qq      |      1 | NULL |
| 20184907 | qq      |      1 | NULL |
| 20184906 | qq      |      1 | NULL |
| 20184905 | qq      |      1 | NULL |
| 20184904 | qq      |      1 | NULL |
| 20184903 | qq      |      1 | NULL |
+----------+---------+--------+------+
10 rows in set (0.00 sec)


mysql&gt; insert into test1(id,name) values (null,'qweasd');
Query OK, 1 row affected (0.00 sec)

mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+------+
| id       | name    | status | addr |
+----------+---------+--------+------+
| 20250320 | qweasd  |      1 |    1 |
| 20250319 | qwe     |      1 | NULL |
| 20250318 | aaqqwww |      1 | NULL |
| 20250317 | aaqq    |      1 | NULL |
| 20250316 | aaqq    |      1 | NULL |
| 20184908 | qq      |      1 | NULL |
| 20184907 | qq      |      1 | NULL |
| 20184906 | qq      |      1 | NULL |
| 20184905 | qq      |      1 | NULL |
| 20184904 | qq      |      1 | NULL |
+----------+---------+--------+------+
10 rows in set (0.00 sec)

```

**现象：** 新增的列的历史数据值全为NULL，只有新增的row才能有默认值1，若业务需要该字段历史数据为默认值1，则需要进行批量更新（**更新注意事项见附录**）。  
**原因：** 根据MySQL OnlineDDL，对于列设置默认值，只会更改元信息，不在原表中进行更改，所以历史数据为NULL<sup id="fnref-662-1">[1](#fn-662-1)</sup>。

## 情况二（补充）

在**情况二**的基础上，若需要设置历史数据为非NULL，需要再进行一次结构变更：

```
mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+------+
| id       | name    | status | addr |
+----------+---------+--------+------+
| 20250320 | qweasd  |      1 |    1 |
| 20250319 | qwe     |      1 | NULL |
| 20250318 | aaqqwww |      1 | NULL |
| 20250317 | aaqq    |      1 | NULL |
| 20250316 | aaqq    |      1 | NULL |
| 20184908 | qq      |      1 | NULL |
| 20184907 | qq      |      1 | NULL |
| 20184906 | qq      |      1 | NULL |
| 20184905 | qq      |      1 | NULL |
| 20184904 | qq      |      1 | NULL |
+----------+---------+--------+------+

mysql&gt; **alter table test1 modify addr int not null default 1;**

mysql&gt; select * from test1 order by id desc limit 10;
+----------+---------+--------+------+
| id       | name    | status | addr |
+----------+---------+--------+------+
| 20250320 | qweasd  |      1 |    1 |
| 20250319 | qwe     |      1 |    0 |
| 20250318 | aaqqwww |      1 |    0 |
| 20250317 | aaqq    |      1 |    0 |
| 20250316 | aaqq    |      1 |    0 |
| 20184908 | qq      |      1 |    0 |
| 20184907 | qq      |      1 |    0 |
| 20184906 | qq      |      1 |    0 |
| 20184905 | qq      |      1 |    0 |
| 20184904 | qq      |      1 |    0 |
+----------+---------+--------+------+
10 rows in set (0.00 sec)

```

**现象：**  
. . **若使用ALTER命令修改该列为非NULL，则会将该列全部更新为0<sup id="fnref-662-2">[2](#fn-662-2)</sup>。**

.  
.  
.

# 附：

若采取批量更新的方式刷新老数据：  
1\. 需要放在业务低峰进行执行  
2\. 分批次进行update，例如一次更新1000条，每次停顿1秒（按照后端binlog消费能力和数据库负载进行调整）

<div class="footnotes" role="doc-endnotes">- - - - - -

1. [Online DDL Operations](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html) [↩︎](#fnref-662-1)
2. [Data Type Default Values](https://dev.mysql.com/doc/refman/5.7/en/data-type-defaults.html) [↩︎](#fnref-662-2)

</div>