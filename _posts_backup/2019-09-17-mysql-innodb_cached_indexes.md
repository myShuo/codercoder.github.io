---
id: 579
title: InnoDB索引在缓存中的情况
date: '2019-09-17T11:41:21+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=579'
permalink: /index.php/2019/09/mysql-innodb_cached_indexes/
views:
    - '4810'
    - '4810'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Tech
---

# innodb的缓存情况

从MySQL8.0开始，可以通过information\_schema中的INNODB\_CACHED\_INDEXES表，查看索引在缓存中的情况。

- - - - - -

**所需权限**  
需要有对于该表的**PROCESS**权限，否则会报错：

```
You must have the PROCESS privilege to query this table.

```

## 查看索引缓存状态

（1）查询INNODB\_CACHED\_INDEXES表

```
root:information_schema> select * from INNODB_CACHED_INDEXES ;
+------------+----------+----------------+
| SPACE_ID   | INDEX_ID | N_CACHED_PAGES |
+------------+----------+----------------+
| 4294967294 |        1 |              1 |
| 4294967294 |        2 |              1 |
| 4294967294 |        3 |              1 |
| 4294967294 |        4 |              1 |
| 4294967294 |        5 |              1 |
| 4294967294 |        7 |              1 |
| 4294967294 |        8 |              1 |
| 4294967294 |        9 |              1 |
| 4294967294 |       10 |              1 |
| 4294967294 |       11 |              1 |
| 4294967294 |       12 |              1 |
| 4294967294 |       13 |              1 |
| 4294967294 |       14 |              1 |
| 4294967294 |       15 |              2 |
...


```

   
（2）获取表名对应关系  
可以根据tables和columns表获取表名：

```
root:information_schema> SELECT
  tables.NAME AS table_name,
  indexes.NAME AS index_name,
  cached.N_CACHED_PAGES AS n_cached_pages
FROM
  INFORMATION_SCHEMA.INNODB_CACHED_INDEXES AS cached,
  INFORMATION_SCHEMA.INNODB_INDEXES AS indexes,
  INFORMATION_SCHEMA.INNODB_TABLES AS tables
WHERE
  cached.INDEX_ID = indexes.INDEX_ID
  AND indexes.TABLE_ID = tables.TABLE_ID;
+------------------+------------+----------------+
| table_name       | index_name | n_cached_pages |
+------------------+------------+----------------+
| wstestdb/sbtest2 | PRIMARY    |            273 |
| wstestdb/sbtest2 | k_2        |            166 |
| wstestdb/sbtest1 | PRIMARY    |            287 |
| wstestdb/sbtest1 | k_1        |            165 |
+------------------+------------+----------------+
4 rows in set (0.01 sec)


```

   
（3）多次查询，观察变化  
再次查询：

```
root:information_schema> SELECT   tables.NAME AS table_name,   indexes.NAME AS index_name,   cached.N_CACHED_PAGES AS n_cached_pages FROM   INFORMATION_SCHEMA.INNODB_CACHED_INDEXES AS cached,   INFORMATION_SCHEMA.INNODB_INDEXES AS indexes,   INFORMATION_SCHEMA.INNODB_TABLES AS tables WHERE   cached.INDEX_ID = indexes.INDEX_ID   AND indexes.TABLE_ID = tables.TABLE_ID;
+------------------+------------+----------------+
| table_name       | index_name | n_cached_pages |
+------------------+------------+----------------+
| wstestdb/sbtest2 | PRIMARY    |           1764 |
| wstestdb/sbtest2 | k_2        |            166 |
| wstestdb/sbtest1 | PRIMARY    |            831 |
| wstestdb/sbtest1 | k_1        |            165 |
+------------------+------------+----------------+
4 rows in set (0.01 sec)

```

（4）表的大小  
环境介绍：

```
root:information_schema&gt; select count(*) from wstestdb.sbtest2;
+----------+
| count(*) |
+----------+
|   116716 |
+----------+

root:information_schema&gt; show create table  wstestdb.sbtest2 \G 
*************************** 1. row ***************************
       Table: sbtest2
Create Table: CREATE TABLE if not exists `sbtest2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `k_2` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=700001 DEFAULT CHARSET=utf8

```

# 结果

## 缓存情况

可以看出，大约有 **5.89%的数据和索引页被缓存在缓冲池中** ：

```
root:information_schema&gt; SELECT COUNT(*) AS total_pages FROM information_schema.INNODB_BUFFER_PAGE;
+----------+
| total_pages |
+----------+
|    32768 |
+----------+
1 row in set (0.15 sec)

root:information_schema&gt; select (1764+166)/32768;
+------------------+
| (1764+166)/32768 |
+------------------+
|           0.0589 |
+------------------+
1 row in set (0.00 sec)

```

## 缓冲池中的信息

可以根据之前的文章[InnoDB缓冲池中的内容](http://codercoder.cn/index.php/2019/09/whats-in-innodb-buffer-pool/)，查看缓冲池中的情况。

# 附加信息

对于索引缓存情况，pmm已经支持对其进行监控：  
使用pmm监控索引缓存情况：[Which Indexes are Cached? Discover with PMM](https://www.percona.com/blog/2019/09/09/which-indexes-are-cached-discover-with-pmm/)