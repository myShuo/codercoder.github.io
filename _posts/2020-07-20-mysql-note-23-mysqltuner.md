---
id: 1163
title: 'MySQL手记23 &#8212; MySQL运行情况统计小工具mysqltuner'
date: '2020-07-20T23:01:27+08:00'
author: Shuo
excerpt: mysqltuner，用以统计MySQL实例运行时候的基本情况，会给出基础的一些建议，DBA可进行参考，优化实例配置。
layout: post
guid: 'http://codercoder.cn/?p=1163'
image: http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2020.png
permalink: /index.php/2020/07/mysql-note-23-mysqltuner/
views:
    - '29193'
    - '29193'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

无意中发现一个小工具，perl语言开发的mysqltuner.pl，可以用来展示MySQL实例的状态：

Github地址：https://github.com/major/MySQLTuner-perl

# 运行结果

如下图所示，运行结果将会按照分类进行分块展示，后文将进行分析讨论：  
![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2020.png)

# 测试情况

下面展示我在测试环境MySQL8.0的情况：

```
>>  MySQLTuner 1.7.19 - Major Hayden <major@mhtx.net>
 >>  Bug reports, feature requests, and downloads at http://mysqltuner.com/
 >>  Run with '--help' for additional options and output filtering
​
[--] Skipped version check for MySQLTuner script
[--] Performing tests on 127.0.0.1:4406
[OK] Logged in using credentials passed on the command line
Argument "" isn't numeric in numeric ge (>=) at mysqltuner.pl line 302 (#1)
    (W numeric) The indicated string was fed as an argument to an operator
    that expected a numeric value instead.  If you're fortunate the message
    will identify which operator was so unfortunate.
​
[OK] Currently running supported MySQL version 8.0.17
[OK] Operating on 64-bit architecture
​
-------- Log file Recommendations ------------------------------------------------------------------
[!!] Log file /tmp/4406-error.log doesn't exist
​
-------- Storage Engine Statistics -----------------------------------------------------------------
[--] Status: +ARCHIVE +BLACKHOLE +CSV -FEDERATED +InnoDB +MEMORY +MRG_MYISAM +MyISAM +PERFORMANCE_SCHEMA
[--] Data in InnoDB tables: 146.6M (Tables: 1889)
[OK] Total fragmented tables: 0
​
-------- Analysis Performance Metrics --------------------------------------------------------------
[--] innodb_stats_on_metadata: OFF
[OK] No stat updates during querying INFORMATION_SCHEMA.
​
-------- Security Recommendations ------------------------------------------------------------------
[--] Skipped due to unsupported feature for MySQL 8
​
-------- CVE Security Recommendations --------------------------------------------------------------
[--] Skipped due to --cvefile option undefined
​
-------- Performance Metrics -----------------------------------------------------------------------
[--] Up for: 2d 19h 43m 16s (62K q [0.257 qps], 276 conn, TX: 23M, RX: 10M)
[--] Reads / Writes: 98% / 2%
[--] Binary logging is enabled (GTID MODE: ON)
[--] Physical Memory     : 8.0G
[--] Max MySQL memory    : 12.8G
[--] Other process memory: 0B
[--] Total buffers: 784.0M global + 24.6M per thread (500 max threads)
[--] P_S Max memory usage: 72B
[--] Galera GCache Max memory usage: 0B
[OK] Maximum reached memory usage: 857.9M (10.47% of installed RAM)
[!!] Maximum possible memory usage: 12.8G (160.01% of installed RAM)
[!!] Overall possible memory usage with other process exceeded memory
[OK] Slow queries: 2% (1K/62K)
[OK] Highest usage of available connections: 0% (3/500)
[!!] Aborted connections: 14.86%  (41/276)
[--] Query cache have been removed in MySQL 8
[OK] Sorts requiring temporary tables: 0% (0 temp sorts / 1K sorts)
[!!] Joins performed without indexes: 3013
[OK] Temporary tables created on disk: 0% (0 on disk / 4K total)
[OK] Thread cache hit rate: 98% (3 created / 276 connections)
[OK] Table cache hit rate: 61% (2K open / 3K opened)
[!!] table_definition_cache(2000) is lower than number of tables(2194) 
[OK] Open file limit used: 0% (39/1M)
[OK] Table locks acquired immediately: 100% (480 immediate / 480 locks)
[OK] Binlog cache memory access: 100.00% (4255 Memory / 4255 Total)
​
-------- Performance schema ------------------------------------------------------------------------
[--] Memory used by P_S: 72B
[--] Sys schema is installed.
​
-------- ThreadPool Metrics ------------------------------------------------------------------------
[--] ThreadPool stat is disabled.
​
-------- MyISAM Metrics ----------------------------------------------------------------------------
[--] MyISAM Metrics are disabled on last MySQL versions.
​
-------- InnoDB Metrics ----------------------------------------------------------------------------
[--] InnoDB is enabled.
[--] InnoDB Thread Concurrency: 0
[OK] InnoDB File per table is activated
[OK] InnoDB buffer pool / data size: 512.0M/146.6M
[!!] Ratio InnoDB log file size / InnoDB Buffer pool size (18.75 %): 48.0M * 2/512.0M should be equal to 25%
[OK] InnoDB buffer pool instances: 1
[--] Number of InnoDB Buffer Pool Chunk : 4 for 1 Buffer Pool Instance(s)
[OK] Innodb_buffer_pool_size aligned with Innodb_buffer_pool_chunk_size & Innodb_buffer_pool_instances
[OK] InnoDB Read buffer efficiency: 100.00% (123128483 hits/ 123129674 total)
[OK] InnoDB Write log efficiency: 92.19% (2024330 hits/ 2195892 total)
[OK] InnoDB log waits: 0.00% (0 waits / 171562 writes)
​
-------- AriaDB Metrics ----------------------------------------------------------------------------
[--] AriaDB is disabled.
​
-------- TokuDB Metrics ----------------------------------------------------------------------------
[--] TokuDB is disabled.
​
-------- XtraDB Metrics ----------------------------------------------------------------------------
[--] XtraDB is disabled.
​
-------- Galera Metrics ----------------------------------------------------------------------------
[--] Galera is disabled.
​
-------- Replication Metrics -----------------------------------------------------------------------
[--] Galera Synchronous replication: NO
[--] No replication slave(s) for this server.
[--] Binlog format: ROW
[--] XA support enabled: ON
[--] Semi synchronous replication Master: Not Activated
[--] Semi synchronous replication Slave: Not Activated
[--] This is a standalone server
​
-------- Recommendations ---------------------------------------------------------------------------
General recommendations:
Reduce your overall MySQL memory footprint for system stability
Dedicate this server to your database for highest performance.
Reduce or eliminate unclosed connections and network issues
We will suggest raising the 'join_buffer_size' until JOINs not using indexes are found.
See https://dev.mysql.com/doc/internals/en/join-buffer-size.html
             (specially the conclusions at the bottom of the page).
Before changing innodb_log_file_size and/or innodb_log_files_in_group read this: https://bit.ly/2TcGgtU
Variables to adjust:
  *** MySQL's maximum memory usage is dangerously high ***
  *** Add RAM before increasing MySQL buffer variables ***
    join_buffer_size (> 8.0M, or always use indexes with JOINs)
    table_definition_cache(2000) > 2194 or -1 (autosizing if supported)
    innodb_log_file_size should be (=64M) if possible, so InnoDB total log files size equals to 25% of buffer pool size.

```

# 结果分析

分析结果分为如下几个部分：

## 1. 实例基本信息

版本、实例系统配置简介等

## 2. Storage Engine Statistics

存储引擎统计信息  
 Innodb表的数目、总大小

## 3. Analysis Performance Metrics

innodb\_stats\_on\_metadata开启情况

## 4. Performance Metrics

 性能相关的统计信息：  
 uptime、QPS、读写操作情况  
 binlog是否开启？  
 内存情况、buffer缓存情况  
 慢查比例  
 连接情况  
 临时表创建情况  
 open files  
 获取锁等待的情况  
***（乍眼一看，指标和我们平时看的MySQL监控大同小异，PMM提供的面板中，也是这些个指标：[MySQL手记9 — Percona Monitoring Management（PMM监控）](http://codercoder.cn/index.php/2020/04/mysql-note-9-percona-monitoring-management/)\]***

## 5. Performance schema

P\_S使用内存的情况

## 6. ThreadPool Metrics

是否打开线程池

## 7. MyISAM Metrics、AriaDB Metrics、TokuDB Metrics、XtraDB Metrics、Galera Metrics

各种存储引擎情况

## 8. InnoDB Metrics

**InnoDB情况**  
 是否开启InnoDB File per table  
 InnoDB buffer pool / data size  
 InnoDB buffer pool instances数目  
 InnoDB读写效率

## 9. Replication Metrics：

**复制情况：**  
 Binlog format: ROW  
 Slave情况  
 是否半同步

## 10. Recommendation:

 **Log file Recommendations**  
 例如本例中提示：Log file /tmp/4406-error.log doesn’t exist  
 **Security Recommendations**  
 由于我使用的是MySQL8.0版本，所以本项不支持，跳过  
 **CVE Security Recommendations**  
 Skipped due to –cvefile option undefined

## 汇总的Recommendation：

 降低MySQL内存使用率，以提高系统稳定性  
 由于我的实例为单实例，所以建议组建高可用集群  
 减少或消除未封闭的连接和网络问题

## 建议调整的参数：

Variables to adjust:  
 \*\*\* MySQL’s maximum memory usage is dangerously high \*\*\*  
MySQL实例可用的内存，配置太高，可能会发生OOM  
 \*\*\* Add RAM before increasing MySQL buffer variables \*\*\*  
（在调高buffer相关配置前，先升级内存）  
 **join\_buffer\_size** (> 8.0M, or always use indexes with JOINs)  
 **table\_definition\_cache**(2000) > 2194 or -1 (autosizing if supported)  
 **innodb\_log\_file\_size** should be (=64M) if possible, so InnoDB total log files size equals to 25% of buffer pool size.  
​

ps.  
这个项目github上的一个FAQ:)  
![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2047.png)

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)