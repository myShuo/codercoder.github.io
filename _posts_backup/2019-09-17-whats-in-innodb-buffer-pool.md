---
id: 626
title: InnoDB缓冲池中的内容
date: '2019-09-17T11:40:38+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=626'
permalink: /index.php/2019/09/whats-in-innodb-buffer-pool/
views:
    - '1073'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# 缓冲池信息相关的表

查看缓冲池中的信息，主要是通过information\_schema库下的三张表进行查看：  
\+ INNODB\_BUFFER\_PAGE：INNODB缓冲池中page的情况  
\+ INNODB\_BUFFER\_PAGE\_LRU：INNODB缓冲池中page的LRU(Least Recently Used)情况  
\+ INNODB\_BUFFER\_POOL\_STATS：INNODB缓冲池状态总览

```
root:information_schema> SHOW TABLES FROM INFORMATION_SCHEMA LIKE 'INNODB_BUFFER%';
+-----------------------------------------------+
| Tables_in_information_schema (INNODB_BUFFER%) |
+-----------------------------------------------+
| INNODB_BUFFER_PAGE                            |
| INNODB_BUFFER_PAGE_LRU                        |
| INNODB_BUFFER_POOL_STATS                      |
+-----------------------------------------------+

```

详情查看：\[InnoDB INFORMATION\_SCHEMA Buffer Pool Tables\]  
注意事项：

> 查询 INNODB\_BUFFER\_PAGE 或者 INNODB\_BUFFER\_PAGE\_LRU 会影响数据库的性能！  
>  [![](http://121.40.246.76/wp-content/uploads/2019/09/2019-09-1155-300x67.png)](http://121.40.246.76/wp-content/uploads/2019/09/2019-09-1155.png)  
>  (https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-buffer-pool-tables.html)

# 从INNODB\_BUFFER\_PAGE表查询系统数据

系统数据记为MySQL用以存储数据库实例相关状态或者配置的数据，这部分数据同样会部分缓存在缓冲池中。

```
root:information_schema> SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE        WHERE TABLE_NAME IS NULL OR (INSTR(TABLE_NAME, '/') = 0 AND INSTR(TABLE_NAME, '.') = 0);
+----------+
| COUNT(*) |
+----------+
|    29573 |
+----------+
1 row in set (0.25 sec)

```

## 系统数据在缓冲池中的分布

查看系统数据在整个缓冲池中的分布情况：

```
mysql> SELECT
       (SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE TABLE_NAME IS NULL OR (INSTR(TABLE_NAME, '/') = 0 AND INSTR(TABLE_NAME, '.') = 0)
       ) AS system_pages,
       (
       SELECT COUNT(*)
       FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       ) AS total_pages,
       (
       SELECT ROUND((system_pages/total_pages) * 100)
       ) AS system_page_percentage;
+--------------+-------------+------------------------+
| system_pages | total_pages | system_page_percentage |
+--------------+-------------+------------------------+
|        29573 |       32768 |                     90 |
+--------------+-------------+------------------------+

```

## 缓存中系统数据的属性

PAGE\_TYPE字段可以查看系统数据的属性：

```
mysql> SELECT DISTINCT PAGE_TYPE FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE TABLE_NAME IS NULL OR (INSTR(TABLE_NAME, '/') = 0 AND INSTR(TABLE_NAME, '.') = 0);
+-------------------+
| PAGE_TYPE         |
+-------------------+
| SYSTEM            |
| INODE             |
| IBUF_INDEX        |
| TRX_SYSTEM        |
| RSEG_ARRAY        |
| UNDO_LOG          |
| FILE_SPACE_HEADER |
| IBUF_BITMAP       |
| LOB_FIRST         |
| LOB_DATA          |
| LOB_INDEX         |
| UNKNOWN           |
| INDEX             |
+-------------------+
13 rows in set (0.37 sec)

```

# 从INNODB\_BUFFER\_PAGE表查询用户数据

```
mysql> SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE TABLE_NAME IS NOT NULL AND TABLE_NAME NOT LIKE '%INNODB_TABLES%';
+----------+
| COUNT(*) |
+----------+
|     3195 |
+----------+

```

## 用户数据在缓冲池中的分布情况

同理，查看用户数据在缓冲池中的分布情况：

```
mysql> SELECT
       (SELECT COUNT(*) FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE TABLE_NAME IS NOT NULL AND (INSTR(TABLE_NAME, '/') > 0 OR INSTR(TABLE_NAME, '.') > 0)
       ) AS user_pages,
       (
       SELECT COUNT(*)
       FROM information_schema.INNODB_BUFFER_PAGE
       ) AS total_pages,
       (
       SELECT ROUND((user_pages/total_pages) * 100)
       ) AS user_page_percentage;
+------------+-------------+----------------------+
| user_pages | total_pages | user_page_percentage |
+------------+-------------+----------------------+
|       3195 |       32768 |                   10 |
+------------+-------------+----------------------+

```

查看用户数据对应的表

```
mysql&gt; SELECT DISTINCT TABLE_NAME FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE TABLE_NAME IS NOT NULL AND (INSTR(TABLE_NAME, '/') &gt; 0 OR INSTR(TABLE_NAME, '.') &gt; 0)
       AND TABLE_NAME NOT LIKE '`mysql`.`innodb_%';
+----------------------+
| TABLE_NAME           |
+----------------------+
| `wstestdb`.`sbtest2` |
| `wstestdb`.`sbtest1` |
+----------------------+
2 rows in set (0.15 sec)

```

## 从INNODB\_BUFFER\_PAGE表查看索引数据

当然，也可以使用该表查看索引被缓存的情况情况。索引缓存情况可以对比另一篇文章查看[InnoDB索引在缓存中的情况](http://codercoder.cn/index.php/2019/09/mysql-innodb_cached_indexes/)。

```
mysql&gt; SELECT INDEX_NAME, COUNT(*) AS Pages,
ROUND(SUM(IF(COMPRESSED_SIZE = 0, @@GLOBAL.innodb_page_size, COMPRESSED_SIZE))/1024/1024)
AS 'Total Data (MB)'
FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
WHERE INDEX_NAME='k_2' AND TABLE_NAME = '`wstestdb`.`sbtest2`';
+------------+-------+-----------------+
| INDEX_NAME | Pages | Total Data (MB) |
+------------+-------+-----------------+
| k_2        |   167 |               3 |
+------------+-------+-----------------+
1 row in set (0.19 sec)

```

**查看某个表的所有索引缓存情况：**

```
mysql&gt; SELECT INDEX_NAME, COUNT(*) AS Pages,
       ROUND(SUM(IF(COMPRESSED_SIZE = 0, @@GLOBAL.innodb_page_size, COMPRESSED_SIZE))/1024/1024)
       AS 'Total Data (MB)'
       FROM INFORMATION_SCHEMA.INNODB_BUFFER_PAGE
       WHERE TABLE_NAME = '`wstestdb`.`sbtest2`'
       GROUP BY INDEX_NAME;
+------------+-------+-----------------+
| INDEX_NAME | Pages | Total Data (MB) |
+------------+-------+-----------------+
| PRIMARY    |  1767 |              28 |
| k_2        |   167 |               3 |
+------------+-------+-----------------+
2 rows in set (0.21 sec)

```

# 查询LRU\_POSITION信息

LRU可以反映表中热数据的情况，可以参考这个状态，判断是否会影响sql的查询，或者是否需要刷数据，使数据变“热”。

```
root:information_schema&gt; select table_name,INDEX_NAME,min(LRU_POSITION),max(LRU_POSITION)  from   INFORMATION_SCHEMA.INNODB_BUFFER_PAGE_LRU where table_name='`wstestdb`.`sbtest2`' group by INDEX_NAME;
+----------------------+------------+-------------------+-------------------+
| table_name           | INDEX_NAME | min(LRU_POSITION) | max(LRU_POSITION) |
+----------------------+------------+-------------------+-------------------+
| `wstestdb`.`sbtest2` | PRIMARY    |               485 |              4273 |
| `wstestdb`.`sbtest2` | k_2        |               527 |              2910 |
+----------------------+------------+-------------------+-------------------+
2 rows in set (0.03 sec)

```

# 查询INNODB\_BUFFER\_POOL\_STATS表

查询INNODB\_BUFFER\_POOL\_STATS表和直接执行***show engine innodb status \\G*** 和 ***查看Innodb\_buffer的状态***的结果相似：

```
root:information_schema&gt; SELECT * FROM information_schema.INNODB_BUFFER_POOL_STATS \G
*************************** 1. row ***************************
                         POOL_ID: 0
                       POOL_SIZE: 32768
                    FREE_BUFFERS: 27417
                  DATABASE_PAGES: 5169
              OLD_DATABASE_PAGES: 1888
         MODIFIED_DATABASE_PAGES: 0
              PENDING_DECOMPRESS: 0
                   PENDING_READS: 0
               PENDING_FLUSH_LRU: 0
              PENDING_FLUSH_LIST: 0
                PAGES_MADE_YOUNG: 0
            PAGES_NOT_MADE_YOUNG: 0
           PAGES_MADE_YOUNG_RATE: 0
       PAGES_MADE_NOT_YOUNG_RATE: 0
               NUMBER_PAGES_READ: 3590
            NUMBER_PAGES_CREATED: 1579
            NUMBER_PAGES_WRITTEN: 68456
                 PAGES_READ_RATE: 0
               PAGES_CREATE_RATE: 0
              PAGES_WRITTEN_RATE: 8.973448067156127
                NUMBER_PAGES_GET: 25963207
                        HIT_RATE: 1000
    YOUNG_MAKE_PER_THOUSAND_GETS: 0
NOT_YOUNG_MAKE_PER_THOUSAND_GETS: 0
         NUMBER_PAGES_READ_AHEAD: 0
       NUMBER_READ_AHEAD_EVICTED: 0
                 READ_AHEAD_RATE: 0
         READ_AHEAD_EVICTED_RATE: 0
                    LRU_IO_TOTAL: 0
                  LRU_IO_CURRENT: 0
                UNCOMPRESS_TOTAL: 0
              UNCOMPRESS_CURRENT: 0
1 row in set (0.00 sec)

```

```
mysql&gt; SHOW ENGINE INNODB STATUS \G
...
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 549453824
Dictionary memory allocated 494195
Buffer pool size   32768
Free buffers       27417
Database pages     5169
Old database pages 1888
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 3590, created 1579, written 68456
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/sLRU len: 5169, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]

root:information_schema&gt; SHOW STATUS LIKE 'Innodb_buffer%';
+---------------------------------------+--------------------------------------------------+
| Variable_name                         | Value                                            |
+---------------------------------------+--------------------------------------------------+
| Innodb_buffer_pool_dump_status        | Dumping of buffer pool not started               |
| Innodb_buffer_pool_load_status        | Buffer pool(s) load completed at 190911  6:50:47 |
| Innodb_buffer_pool_resize_status      |                                                  |
| Innodb_buffer_pool_pages_data         | 5169                                             |
| Innodb_buffer_pool_bytes_data         | 84688896                                         |
| Innodb_buffer_pool_pages_dirty        | 0                                                |
| Innodb_buffer_pool_bytes_dirty        | 0                                                |
| Innodb_buffer_pool_pages_flushed      | 68446                                            |
| Innodb_buffer_pool_pages_free         | 27417                                 -           |
| Innodb_buffer_pool_pages_misc         | 182                                              |
| Innodb_buffer_pool_pages_total        | 32768                                            |
| Innodb_buffer_pool_read_ahead_rnd     | 0                                                |
| Innodb_buffer_pool_read_ahead         | 0                                                |
| Innodb_buffer_pool_read_ahead_evicted | 0                                                |
| Innodb_buffer_pool_read_requests      | 25963233                                         |
| Innodb_buffer_pool_reads              | 3384                                             |
| Innodb_buffer_pool_wait_free          | 0                                                |
| Innodb_buffer_pool_write_requests     | 4119405                                          |
+---------------------------------------+--------------------------------------------------+
18 rows in set (0.01 sec)

```