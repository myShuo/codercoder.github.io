---
id: 573
title: MySQL索引统计信息INDEX_STATISTICS
date: '2019-09-11T11:21:56+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=573'
permalink: /index.php/2019/09/mysql-index-statistics-table/
views:
    - '9116'
    - '9116'
    - '9116'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0899-1-300x208.jpg)

# 背景

MySQL的开源版本MariaDB、Percona MySQL Server和AliSQL 5.6版本支持统计索引的信息，即可以统计出使用某个索引扫描的行数。依照此，可以找出未被使用的，或者使用频率较低的索引，从而进行下线。本文主要介绍AliSQL 5.6版本的使用方式，使用阿里云RDS环境。  
**RDS MySQL5.7开始不再具有该表，可以使用performance\_schema中的统计信息table\_io\_waits\_summary\_by\_index\_usage进行查看**

# 环境

阿里云RDS 5.6版本

# 现象

测试表表结构：

```
mysql&gt; show create table wstest.test2 \G
*************************** 1. row ***************************
       Table: test2
Create Table: CREATE TABLE if not exists `test2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(30) DEFAULT '1',
  `status` int(11) DEFAULT '1',
  `addr` varchar(10) NOT NULL,
  `addr2` tinyint(4) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8


mysql&gt; select * from wstest.test2;
+----+--------+--------+------+-------+
| id | name   | status | addr | addr2 |
+----+--------+--------+------+-------+
|  1 | aaaxyz |    110 |      |     0 |
|  2 | aaaxyz |    110 |      |     0 |
|  3 | aaaxyz |    110 |      |     0 |
|  4 | aaaxyz |    110 |      |     0 |
|  5 | aaax   |      5 |      |     0 |
|  6 | aaaxyz |    110 |      |     0 |
+----+--------+--------+------+-------+
6 rows in set (0.01 sec)



```

.  
**注意：**  
最开始查询 information\_schema.INDEX\_STATISTICS 表发现结果为空，需要打开参数才会进行统计。  
打开参数：  
**loose\_rds\_indexstat=1**

使用字段*name*上的索引进行查询数据：

```
mysql&gt;  select * from wstest.test2 where name='aaax';
+----+------+--------+------+-------+
| id | name | status | addr | addr2 |
+----+------+--------+------+-------+
|  5 | aaax |      5 |      |     0 |
+----+------+--------+------+-------+
1 row in set (0.00 sec)


```

查看统计信息：

```
select * from information_schema.INDEX_STATISTICS where table_schema='wstest' and table_name='test2';
+--------------+------------+------------+-----------+
| TABLE_SCHEMA | TABLE_NAME | INDEX_NAME | ROWS_READ |
+--------------+------------+------------+-----------+
| wstest       | test2      | idx_name   |         1 |
| wstest       | test2      | PRIMARY    |   6314081 |
+--------------+------------+------------+-----------+
2 rows in set (0.02 sec)


```

**说明：**  
由于在使用idx\_name查询数据时，扫描行数为1，所以在information\_schema.INDEX\_STATISTICS表的ROWS\_READ字段对应的值为1；若再次使用idx\_name查询，则ROWS\_READ会再次加1。  
至此，索引的使用情况得以统计。

.  
.  
.

# 索引下线

**继而，结合业务的使用情况，找出使用频率较低的索引进行下线**

## 设置索引不可见

可以使用：

```
alter table test2 alter index idx_name invisible;

```

使索引不可见，避免突然下线导致某些应用出现慢查。

## 删除索引

设置索引不可见之后，可以过一段时间，选择低峰期将索引进行删除：

```
alter table test2 drop index idx_name;

```