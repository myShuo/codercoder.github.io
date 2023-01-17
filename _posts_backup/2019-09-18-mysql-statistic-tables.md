---
id: 644
title: MySQL数据库信息统计表
date: '2019-09-18T09:55:49+08:00'
author: Shuo
excerpt: MySQL在执行sql时，会使用统计信息进行判断，采用最优（cost花费最低）的执行计划，而这些统计信息是怎么进行的，在用户角度如何去调整或者理解统计信息呢？
layout: post
guid: 'http://codercoder.cn/?p=644'
permalink: /index.php/2019/09/mysql-statistic-tables/
views:
    - '12172'
    - '12172'
    - '12172'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# 前言

 聊聊MySQL数据库中的统计信息，众所周知，MySQL在执行sql时，会使用统计信息进行判断，采用最优（cost花费最低）的执行计划，而这些统计信息是怎么进行的，在用户角度如何去调整或者理解统计信息呢？  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-181.jpg)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-181.jpg)

> 本次演示使用的MySQL版本：**8.0.12**

# innodb\_table\_stats 和 innodb\_index\_stats

innodb\_table\_stats和innodb\_index\_stats都是在MySQL实例中的mysql数据库下的表，分别存储表和索引级别的统计信息。  
**innodb\_table\_stats的字段说明**：

| Column name | Description | 说明 |
|:-:|---|---|
| database\_name | Database name | 数据库名 |
| table\_name | Table name, partition name, or subpartition name | 表名 |
| last\_update | A timestamp indicating the last time that InnoDB updated this row | 最后更新时间 |
| n\_rows | The number of rows in the table | 表的行数(估值) |
| clustered\_index\_size | The size of the primary index, in pages | 聚集索引页数 |
| sum\_of\_other\_index\_sizes | The total size of other (non-primary) indexes, in pages | 二级索引页数 |

参考MySQL官方文档

## innodb\_table\_stats示例：

```
mysql> select * from mysql.innodb_table_stats where database_name='wstest';
+---------------+-------------+---------------------+----------+----------------------+--------------------------+
| database_name | table_name  | last_update         | n_rows   | clustered_index_size | sum_of_other_index_sizes |
+---------------+-------------+---------------------+----------+----------------------+--------------------------+
| wstest        | mul_replace | 2018-11-13 22:01:58 |        2 |                    1 |                        1 |
| wstest        | rdslist     | 2018-11-16 10:30:52 |      324 |                    5 |                        0 |
| wstest        | t1          | 2019-03-12 14:13:33 |        8 |                    1 |                        1 |
| wstest        | t2          | 2018-11-27 11:39:08 |       10 |                    1 |                        1 |
| wstest        | t3          | 2018-12-12 14:52:23 |   302472 |                  865 |                        0 |
| wstest        | t4          | 2018-12-12 15:07:34 |  4155228 |                21043 |                        0 |
| wstest        | t5          | 2019-03-05 19:15:28 | 11301308 |                58688 |                        0 |
| wstest        | test        | 2018-10-30 16:29:22 |        6 |                    1 |                        0 |
| wstest        | testgroupby | 2019-01-02 16:36:04 |        6 |                    1 |                        0 |
| wstest        | tmstamp     | 2019-01-07 17:02:43 |        2 |                    1 |                        0 |
+---------------+-------------+---------------------+----------+----------------------+--------------------------+
10 rows in set (0.00 sec)

```

从上述内容可以看出表的基本状态，包括表名，聚集索引和非聚集索引（二级索引）大小等等，此外，还能看出部分表在一段时间内没有更新，或者更新的行数没有超过recalculate的阈值（10% rows）

**而想要获取到更为详细的信息，需要查看innodb\_index\_stats**：

| Column name | Description | 说明 |
|:-:|---|---|
| database\_name | Database name | 数据库名 |
| table\_name | Table name, partition name, or subpartition name | 表名 |
| index\_name | Index name | 索引名 |
| last\_update | A timestamp indicating the last time that InnoDB updated this row | 最后更新时间 |
| stat\_name | The name of the statistic, whose value is reported in the stat\_value column | 统计线程的名称 |
| stat\_value | The value of the statistic that is named in stat\_name column | 统计线程的值 |
| sample\_size | The number of pages sampled for the estimate provided in the stat\_value column | 采样页数 |
| stat\_description | Description of the statistic that is named in the stat\_name column | 统计的说明 |

## innodb\_index\_stats示例：

```
<br></br>mysql> select * from mysql.innodb_index_stats where database_name='wstest' and table_name='t4';
+---------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
| database_name | table_name | index_name | last_update         | stat_name    | stat_value | sample_size | stat_description                  |
+---------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
| wstest        | t4         | PRIMARY    | 2019-03-21 18:57:41 | n_diff_pfx01 |    4155228 |          20 | id                                |
| wstest        | t4         | PRIMARY    | 2019-03-21 18:57:41 | n_leaf_pages |      20988 |        NULL | Number of leaf pages in the index |
| wstest        | t4         | PRIMARY    | 2019-03-21 18:57:41 | size         |      21043 |        NULL | Number of pages in the index      |
| wstest        | t4         | idx_name   | 2019-03-21 18:57:41 | n_diff_pfx01 |          1 |           2 | name                              |
| wstest        | t4         | idx_name   | 2019-03-21 18:57:41 | n_diff_pfx02 |    4189010 |          20 | name,id                           |
| wstest        | t4         | idx_name   | 2019-03-21 18:57:41 | n_leaf_pages |       5810 |        NULL | Number of leaf pages in the index |
| wstest        | t4         | idx_name   | 2019-03-21 18:57:41 | size         |       6699 |        NULL | Number of pages in the index      |
+---------------+------------+------------+---------------------+--------------+------------+-------------+-----------------------------------+
7 rows in set (0.00 sec)

```

首先，对于stat\_name这列的解释：  
– size：该索引总的页数  
– n\_leaf\_pages：索引中叶子节点的页数  
– n\_diff\_pfx01：前01个字段在索引中的唯一值数目（估值）  
– n\_diff\_pfx02：前02个字段在索引中的唯一值数目（估值）

```
mysql> select count(distinct name) from wstest.t4;  ----n_diff_pfx01对应的value
+----------------------+
| count(distinct name) |
+----------------------+
|                    2 |
+----------------------+
1 row in set (0.00 sec)


mysql> select count(distinct name,id) from wstest.t4;  ----n_diff_pfx02对应的value
+-------------------------+
| count(distinct name,id) |
+-------------------------+
|                 4194304 |
+-------------------------+
1 row in set (2.99 sec)

```

## 其他信息：

使用information\_schema.tables查看统计信息：

```
mysql> select table_name,table_rows,data_length,index_length from information_schema.tables where table_name='t4';
+------------+------------+-------------+--------------+
| TABLE_NAME | TABLE_ROWS | DATA_LENGTH | INDEX_LENGTH |
+------------+------------+-------------+--------------+
| t4         |    4155228 |   344768512 |            0 |
+------------+------------+-------------+--------------+
1 row in set (0.07 sec)

```

发现INDEX\_LENGTH 列为0，这是由于idx\_name这列，是我为了展示二级索引时加上的，还没有能刷新在information\_schema.tables里。此时，使用analyze table t4;进行统计信息的重新收集  
（**注：若是必须analyze，一定要放在低峰进行操作**”）

```
mysql> analyze table wstest.t4;
+-----------+---------+----------+----------+
| Table     | Op      | Msg_type | Msg_text |
+-----------+---------+----------+----------+
| wstest.t4 | analyze | status   | OK       |
+-----------+---------+----------+----------+
1 row in set (0.11 sec)

```

重新收集统计信息后，查看information\_schema.tables中的信息，与mysql.innodb\_index\_stats中的结果进行对比：

```
mysql> SELECT SUM(stat_value) pages, index_name, SUM(stat_value)*@@innodb_page_size size FROM mysql.innodb_index_stats WHERE table_name='t4' AND stat_name = 'size' GROUP BY index_name; 
+-------+------------+-----------+
| pages | index_name | size      |
+-------+------------+-----------+
| 21108 | PRIMARY    | 345833472 |
|  6699 | idx_name   | 109756416 |
+-------+------------+-----------+
2 rows in set (0.00 sec)

mysql> select table_name,table_rows,data_length,index_length from information_schema.tables where table_name='t4';                                                    
 +------------+------------+-------------+--------------+
| TABLE_NAME | TABLE_ROWS | DATA_LENGTH | INDEX_LENGTH |
+------------+------------+-------------+--------------+
| t4         |    4173444 |   345833472 |    109756416 |
+------------+------------+-------------+--------------+
1 row in set (0.00 sec)

```

可以看出：  
1\. information\_schema.tables中的DATA\_LENGTH 值 = PRIMARY页数 \* 页的大小，即对于InnoDB，聚集索引的大小即为数据大小。

2. information\_schema.tables中的INDEX\_LENGTH值 = 二级索引页数 \* 页的大小

再在t4上添加一个索引：

```
mysql> analyze table wstest.t4;                                                                                                                                        +-----------+---------+----------+----------+
| Table     | Op      | Msg_type | Msg_text |
+-----------+---------+----------+----------+
| wstest.t4 | analyze | status   | OK       |
+-----------+---------+----------+----------+
1 row in set (0.06 sec)

mysql> select table_name,table_rows,data_length,index_length from information_schema.tables where table_name='t4';
+------------+------------+-------------+--------------+
| TABLE_NAME | TABLE_ROWS | DATA_LENGTH | INDEX_LENGTH |
+------------+------------+-------------+--------------+
| t4         |    4173444 |   345833472 |    356155392 |
+------------+------------+-------------+--------------+
1 row in set (0.00 sec)

mysql> SELECT SUM(stat_value) pages, index_name, SUM(stat_value)*@@innodb_page_size size FROM mysql.innodb_index_stats WHERE table_name='t4' AND stat_name = 'size' GROUP BY index_name;
+-------+------------+-----------+
| pages | index_name | size      |
+-------+------------+-----------+
| 21108 | PRIMARY    | 345833472 |
| 15039 | idx_addr   | 246398976 |
|  6699 | idx_name   | 109756416 |
+-------+------------+-----------+
3 rows in set (0.01 sec)

```

可以看出：INDEX\_LENGTH = size( idx\_addr + idx\_name )

# 相关配置信息

为了使统计信息更加准确，需要关注：  
**innodb\_stats\_transient\_sample\_pages**（innodb\_stats\_persistent=OFF时）：在收集统计信息时的采样索引pages页数，默认为8。

**innodb\_stats\_persistent\_sample\_pages**（innodb\_stats\_persistent=ON时）：在收集统计信息时的采样索引pages页数，默认为20。

**innodb\_stats\_persistent**：是否将InnoDB的索引统计信息同步到磁盘，若设置为0，则在产生变更时，会自动重新收集信息，得到不同的执行计划，默认为ON。  
**innodb\_stats\_auto\_recalc**（innodb\_stats\_persistent=ON时）：在表的数据被更改后（阈值：10%），自动地重新估算统计信息