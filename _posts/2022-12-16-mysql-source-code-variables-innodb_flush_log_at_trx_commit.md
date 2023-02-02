---
id: 1202
title: 'MySQL源码-变量-innodb_flush_log_at_trx_commit'
date: '2022-12-16T18:34:40+08:00'
author: Shuo
excerpt: 'mysq source-code-variables-innodb_flush_log_at_trx_commit'
layout: post
guid: 'http://codercoder.cn/?p=1202'
permalink: /index.php/2022/12/2022-12-16-mysql-source-code-variables-innodb_flush_log_at_trx_commit
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---
# 一、innodb_flush_log_at_trx_commit含义
我们在实际使用中常用到的配置项，控制数据的ACID（atomicity, consistency, isolation, durability）属性与性能之间的平衡关系。

对于ACID的要求越高，则性能会降低，反之亦然。

参考：[sysvar_innodb_flush_log_at_trx_commit](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)

*  Controls the balance between strict ACID compliance for commit operations and higher performance that is possible when commit-related I/O operations are rearranged and done in batches. You can achieve better performance by changing the default value but then you can lose transactions in a crash.

# 二、源码解释
本篇文章主要是展示innodb_flush_log_at_trx_commit在源码中的解释：

**（1）innodb_flush_log_at_trx_commit=0**

```
/* innodb_flush_log_at_trx_commit=0
    (write and sync once per second).
    Do not flush the redo log during binlog group commit. */
    
```

**（2）innodb_flush_log_at_trx_commit=1**
```
/* Flush the redo log buffer to the redo log file.
  Sync it to disc if we are in FLUSH LOGS, or if
 innodb_flush_log_at_trx_commit=1
  (write and sync at each commit). */
  
```

**（3）定义**
```
static MYSQL_SYSVAR_ULONG(flush_log_at_trx_commit, srv_flush_log_at_trx_commit,
                          PLUGIN_VAR_OPCMDARG,
                          "Set to 0 (write and flush once per second),"
                          " 1 (write and flush at each commit),"
                          " or 2 (write at commit, flush once per second).",
                          nullptr, nullptr, 1, 0, 2, 0);
```

# 三、说明
MySQL的写盘包括两个步骤，write和flush，其中：

**（1）write:**

将更新的数据写入到system buffer。

参考：[write-wikipedia](https://en.wikipedia.org/wiki/Write_(system_call)

**（2）flush:**

通常使用sync相关命令，将system buffer的数据刷到磁盘上。

参考：[sync-linux](https://linux.die.net/man/8/sync)