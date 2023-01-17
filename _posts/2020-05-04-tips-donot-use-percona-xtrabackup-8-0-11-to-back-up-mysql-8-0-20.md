---
id: 1060
title: 'Tips:升级到MySQL8.0.20后暂不能使用Xtrabackup进行备份'
date: '2020-05-04T18:01:28+08:00'
author: Shuo
excerpt: 'Percona-xtrabackup-8.0.11是基于MySQL 8.0.18进行开发的，所以当前若使用MySQL 8.0.20版本，暂时不要使用Percona Xtrabackup进行备份操作。'
layout: post
guid: 'http://codercoder.cn/?p=1060'
image: http://codercoder.cn/wp-content/uploads/2020/05/2020-05-0447.png
permalink: /index.php/2020/05/tips-donot-use-percona-xtrabackup-8-0-11-to-back-up-mysql-8-0-20/
views:
    - '13240'
    - '13240'
    - '13240'
categories:
    - Tech
    - Fell-in-pit
tags:
    - MySQL手记
    - Tech
    - Fell-in-pit
---

# 序

 前文有说到，在做数据库升级之前，一定要多加小心谨慎，避免兼容问题。这也就要求DBA除了知道新版本特性之外，还需要知道为这些特性，做了哪些修改。[MySQL手记5 — 数据库升级准备](http://codercoder.cn/index.php/2020/04/mysql-5-preparing-for-db-upgrade/)  
 4月27日，MySQL官方发布了8.0.20的GA版本，新增了许多振奋人心的特性，同时，也带了了很多差异，需要进行测试。较为关注的是Percona Xtrabackup在对MySQL8.0.20进行备份恢复的时候，会出现报错：

# 一、环境准备

## 1.1 MySQL 8.0.20

 下载MySQL官方的8.0.20版本进行安装，建库建表：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-0447.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-0447.png)

```
mysql&gt; create database wstestdb;
​
mysql&gt; use wstestdb
mysql&gt; CREATE TABLE if not exists `test1` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
​
mysql&gt; insert into test1 values(1,'aaa');

```

## 1.2 下载安装最新版本Percona Xtrabackup

 截止5月4日，最新版本为percona-xtrabackup-8.0.11，下载解压后，进入到bin目录下，执行：

```
./xtrabackup --defaults-file=/soft/my5501.cnf --user=root --password=abc1abc --host=127.0.0.1 --port=5501 --parallel=4 --databases=wstestdb --backup

```

**其中：**  
 **–defaults-file：**MySQL实例的配置文件  
 **其余项目**为MySQL实例的连接、需要备份的数据库信息，最后使用–backup指明需要进行备份。

# 二、xtrabackup备份报错

 **打印信息：**

```
[root@mysql-01 bin]# ./xtrabackup --defaults-file=/soft/my5501.cnf --user=root --password=abc1abc --host=127.0.0.1 --port=5501 --parallel=4 --databases=wstestdb --backup
xtrabackup: recognized server arguments: --datadir=/data/mysql5501 --innodb_undo_tablespaces=3 --tmpdir=/tmp --log_bin=/data/mysql5501/mysql5501 --server-id=1845501 --innodb_file_per_table=1 --innodb_flush_log_at_trx_commit=1 --innodb_page_size=16k --innodb_buffer_pool_size=256M --innodb_data_file_path=ibdata1:400M:autoextend --innodb_log_files_in_group=2 --innodb_log_buffer_size=200M --innodb_open_files=2000 --innodb_io_capacity=1200 --innodb_read_io_threads=32 --innodb_write_io_threads=32 --innodb_max_dirty_pages_pct=75
xtrabackup: recognized client arguments: --port=5501 --user=root --password=* --host=127.0.0.1 --port=5501 --parallel=4 --databases=wstestdb --backup=1
./xtrabackup version 8.0.11 based on MySQL server 8.0.18 Linux (x86_64) (revision id: 486c270)
200504 09:29:43  version_check Connecting to MySQL server with DSN 'dbi:mysql:;mysql_read_default_group=xtrabackup;host=127.0.0.1;port=5501' as 'root'  (using password: YES).
200504 09:29:43  version_check Connected to MySQL server
200504 09:29:43  version_check Executing a version check against the server...
200504 09:29:43  version_check Done.
200504 09:29:43 Connecting to MySQL server host: 127.0.0.1, user: root, password: set, port: 5501, socket: not set
Using server version 8.0.20
xtrabackup: uses posix_fadvise().
xtrabackup: cd to /data/mysql5501
xtrabackup: open files limit requested 0, set to 1048576
xtrabackup: using the following InnoDB configuration:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:400M:autoextend
xtrabackup:   innodb_log_group_home_dir = ./
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 50331648
Number of pools: 1
200504 09:29:44 Connecting to MySQL server host: 127.0.0.1, user: root, password: set, port: 5501, socket: not set
xtrabackup: Redo Log Archiving is not set up.
Unknown redo log format (4). Please follow the instructions at http://dev.mysql.com/doc/refman/8.0/en/ upgrading-downgrading.html.
xtrabackup: Error: recv_find_max_checkpoint() failed.

```

报错信息为：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-0456.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-0456.png)

 xtrabackup: Redo Log Archiving is not set up.  
 Unknown redo log format (4). Please follow the instructions at http://dev.mysql.com/doc/refman/8.0/en/ upgrading-downgrading.html.  
 xtrabackup: Error: recv\_find\_max\_checkpoint() failed.

# 原因

MySQL8.0.20的Release Notes中有这样的介绍：  
 InnoDB: Redo log records for modifications to undo tablespaces increased in size in MySQL 8.0 due to a change in undo tablespace ID values, which required additional bytes. The change in redo log record size caused a performance regression in workloads with heavy write I/O. To address this issue, the redo log format was modified to reduce redo log record size for modifications to undo tablespaces. (Bug #29536710)

 为了在redo log中记录undo表空间的修改信息（记录undo表空间的ID），所以会增加redo log的大小，导致影响到I/O性能。为了解决这个问题，redo log的格式被修改了，以减小其记录所占的大小。

 Percona-xtrabackup-8.0.11是基于MySQL 8.0.18进行开发的，所以当前若使用MySQL 8.0.20版本，暂时不要使用Percona Xtrabackup进行备份操作。

参考：https://forums.percona.com/discussion/comment/55828  
欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)