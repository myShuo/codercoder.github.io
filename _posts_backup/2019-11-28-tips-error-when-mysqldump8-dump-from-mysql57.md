---
id: 822
title: 'Tips: mysqldump8.0导出MySQL5.7版本的数据时报错'
date: '2019-11-28T17:36:25+08:00'
author: Shuo
excerpt: 'MySQL8.0版本中，mysqldump加入了参数 --column-statistics，默认为打开，在导出时，需要在information_schema.column_statistics表中检查导出表的信息。'
layout: post
guid: 'http://codercoder.cn/?p=822'
permalink: /index.php/2019/11/tips-error-when-mysqldump8-dump-from-mysql57/
views:
    - '19027'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 现象

今日在使用mysqldump导出MySQL5.7版本的数据时，报错：  
mysqldump版本：8.0.17

```
mysqldump -h172.16.0.100 -P3306 -B testdb --compact > testdb_20191123.sql 
**Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events. **

**mysqldump: Couldn't execute 'SELECT COLUMN_NAME,                       JSON_EXTRACT(HISTOGRAM, '$."number-of-buckets-specified"')                FROM information_schema.COLUMN_STATISTICS                WHERE SCHEMA_NAME = 'testdb' AND TABLE_NAME = 'st_acc';': Unknown table 'column_statistics' in information_schema (1109)**

```

备份失败，按照提示，是因为找不到**information\_schema.column\_statistics**这个元信息表。

# 解决

按照8.0版本的文档提示，mysqldump加入了该参数 –column-statistics，该参数默认为打开，在dump时，会在information\_schema.column\_statistics表中检查导出表的信息。

![](http://codercoder.cn/wp-content/uploads/2019/11/15570b2374ec9581a3968fd417b954f7.png)

[https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html#option\_mysqldump\_column-statistics](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html#option_mysqldump_column-statistics "https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html#option_mysqldump_column-statistics")

所以若需要dump出MySQL5.7的数据，需要设置其为false

```
mysqldump -h172.16.0.100 -P3306 -B testdb --compact  --column-statistics=0  > testdb_20191123.sql 

```

参考：  
[https://serverfault.com/questions/912162/mysqldump-throws-unknown-table-column-statistics-in-information-schema-1109](https://serverfault.com/questions/912162/mysqldump-throws-unknown-table-column-statistics-in-information-schema-1109 "https://serverfault.com/questions/912162/mysqldump-throws-unknown-table-column-statistics-in-information-schema-1109")