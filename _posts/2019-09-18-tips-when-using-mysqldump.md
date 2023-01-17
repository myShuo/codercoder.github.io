---
id: 648
title: 'Tips: MySQL数据库使用mysqldump备份恢复时的注意事项'
date: '2019-09-18T10:10:24+08:00'
author: Shuo
excerpt: mysqldump作为MySQL数据库逻辑备份的常用工具，对于其备份出来的文件，应该进行确认，防止在恢复时误删数据。MySQL主从和双机为MySQL复制的常见架构，在此类集群中，对于数据的恢复，更需要小心。
layout: post
guid: 'http://codercoder.cn/?p=648'
permalink: /index.php/2019/09/tips-when-using-mysqldump/
views:
    - '14628'
    - '14628'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 背景

- mysqldump作为MySQL数据库逻辑备份的常用工具，对于其备份出来的文件，应该进行确认，防止在恢复时误删数据。
- mysqldump提供了很多选项，使用时候需确认默认选项及所需选项配置。
- MySQL主从和双机为MySQL复制的常见架构，在此类集群中，对于数据的恢复，更需要小心，确认无误后再进行操作。  
  [![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1858.jpg)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1858.jpg)

# mysqldump采坑点

## 1. -E, -R, –triggers

常用的mysqldump的格式为（例如备份wstest.t1）：

```
mysqldump -uroot -p -P 8001 -h 192.168.101.185  -E -R --triggers wstest t1 > wstest_t1.sql

```

这样既可备份wstest.t1及表上的EVENTS，ROUTINES，TRIGGERS，下文默认加上三者，但是在备份时需要注意，**因为若未备份，则在恢复时候虽然数据正确，但是实例运行起来后，会出现问题。**

## 2. drop table

默认情况下，mysqldump备份出来的文件中，在恢复表时候，会先**drop table**。  
按照第一点的命令，备份得到的文件内容

```
-- MySQL dump 10.13  Distrib 8.0.12, for linux-glibc2.12 (x86_64)
--
-- Host: 192.168.101.185    Database: wstest
-- ------------------------------------------------------
-- Server version       8.0.12

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
 SET NAMES utf8mb4 ;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;

--
-- GTID state at the beginning of the backup 
--

SET @@GLOBAL.GTID_PURGED=/*!80000 '+'*/ 'b3e550a7-b009-11e8-aca0-000c299263cc:1-284814';

--
-- Table structure for table `t1`
--
------注意在create表之前，先进行了drop--------
**DROP TABLE IF EXISTS `t1`;**
/*!40101 SET @saved_cs_client     = @@character_set_client */;
 SET character_set_client = utf8mb4 ;
CREATE TABLE if not exists `t1` (
  `a` int(11) NOT NULL AUTO_INCREMENT,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`a`),
  UNIQUE KEY `uniq_b` (`b`)
) ENGINE=InnoDB AUTO_INCREMENT=111112 DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `t1`
--

LOCK TABLES `t1` WRITE;
/*!40000 ALTER TABLE `t1` DISABLE KEYS */;
INSERT INTO `t1` VALUES (3,10004,100),(11,10016,30000),(431,12,99),(436,18,33),(111111,23231313,12313131);
/*!40000 ALTER TABLE `t1` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Dumping events for database 'wstest'
--

--
-- Dumping routines for database 'wstest'
--
SET @@SESSION.SQL_LOG_BIN = @MYSQLDUMP_TEMP_LOG_BIN;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2019-06-11 10:23:09

```

若环境中已经存在新的数据，则这种方式将会删除表后重建，导致数据丢失。所以，可以在命令中加上 **–skip-add-drop-table**：

```
mysqldump --skip-add-drop-table  wstest t1  &gt; wstest_t1.sql 

```

从备份文件可以看出，在新建表之后，会使用**insert into**语句进行数据的插入。为了防止有重复数据中断恢复（恢复大表的时候，花费时间比较长，若中断后排错，又需要重新恢复，耗时较长），还可以使用 **–replace**：

```
mysqldump --skip-add-drop-table --replace wstest t1  &gt; wstest_t1.sql 

```

得到的备份文件的插入数据部分为：

```
LOCK TABLES `t1` WRITE;
/*!40000 ALTER TABLE `t1` DISABLE KEYS */;
REPLACE INTO `t1` VALUES (3,10004,100),(11,10016,30000),(431,12,99),(436,18,33),(111111,23231313,12313131);
/*!40000 ALTER TABLE `t1` ENABLE KEYS */;
UNLOCK TABLES;

```

## 3. –set-gtid-purged

执行第一点的mysqldump命令时，会有一个warning：

> Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don’t want to restore GTIDs, pass –set-gtid-purged=OFF. To make a complete dump, pass –all-databases –triggers –routines –events.

需要关注**–set-gtid-purged**选项，在上文中备份出来的文件中，有如下两行：

> SET @MYSQLDUMP\_TEMP\_LOG\_BIN = @@SESSION.SQL\_LOG\_BIN;  
>  SET @@SESSION.SQL\_LOG\_BIN= 0;

这两行的意思是：不把恢复产生的sql记录在binlog中。这点尤其重要，因为在MySQL主从，或者MySQL双机的环境，若进行恢复，则会同步到对端，这样就存在一个问题：  
**若一端正常在数据写入，需要回复另一端的时候，此时，有这两行的话，相对来说就会使恢复过程更安全。** 否则，在另一端也会执行备份文件中的内容（若备份文件中有类似drop table的选项，那将是一个删库的结果~）。

> 若是在一个全新的一个集群环境做恢复，那么可以加上**–set-gtid-purged=OFF**，这样，可在在从库中也收到binlog信息，得到一个数据一致的集群。

## 4. –single-transaction

备份时创建一致的快照，在单个事务中转储所有表。**强烈建议开启，并配合–master-data使用**  
由于备份是一个阶段式的dump，所以可能出现：A表已经备份，B表关联了A的数据，那么再备份B表时，A表没有相关的记录。就会造成恢复的时候，出现数据逻辑不完整的情况。

## 5. –master-data

**–master-data** 决定是否将 **binary log position和filename** 信息记录在备份文件中。

- =1：直接记录在文件中，恢复时会执行change master to….:

```
--
-- Position to start replication or point-in-time recovery from
--

CHANGE MASTER TO MASTER_LOG_FILE='mysql8001.000025', MASTER_LOG_POS=159500640;

```

- =2：以注释的方式记录在备份文件：

```
--
-- Position to start replication or point-in-time recovery from
--

-- CHANGE MASTER TO MASTER_LOG_FILE='mysql8001.000025', MASTER_LOG_POS=159500640;

```

## 6. -e, –extended-insert, –skip-extended-insert

加上 **-e** 或者 **–extended-insert**，可以使insert语句为一个insert插入多条数据，如上文所示。  
加上**–skip-extended-insert**， 则为一条insert插入一条数据：

```
LOCK TABLES `t1` WRITE;
/*!40000 ALTER TABLE `t1` DISABLE KEYS */;
INSERT INTO `t1` VALUES (3,10004,100);
INSERT INTO `t1` VALUES (11,10016,30000);
INSERT INTO `t1` VALUES (431,12,99);
INSERT INTO `t1` VALUES (436,18,33);
INSERT INTO `t1` VALUES (111111,23231313,12313131);
/*!40000 ALTER TABLE `t1` ENABLE KEYS */;
UNLOCK TABLES;

```

## 7. -F, –flush-logs

在进行逻辑备份的全量+增量的恢复时，往往需要找到对应的binlog的position信息，所以在备份时，建议加上**-F**，这样，在dump之前会先flush logs，产生新的binlog文件，便于恢复。**建议和–master-data一起使用**

> Note that if you dump many databases at once (using the option  
>  –databases= or –all-databases), the logs will be  
>  flushed for each database dumped. The exception is when  
>  using –lock-all-tables or –master-data: in this case  
>  the logs will be flushed only once, corresponding to the  
>  moment all tables are locked. So if you want your dump  
>  and the log flush to happen at the same exact moment you  
>  should use –lock-all-tables or –master-data with  
>  –flush-logs.

## 8. 字符集

可以看到备份出的文件中有**SET NAMES utf8mb4 ;** 等关于字符集的字样，同样在辈分时候应该注意，防止字符集不一致出现的数据不可用。

## 9. 注释

对于mysqldump中的注释部分，同样需要进行确认，例如上文中的**SQL\_MODE**，**TIME\_ZONE**等等。

## 10. 其它

(1) LOCK TABLE  
对于表的回复操作，会有**LOCK TABLE …. WRITE**操作，这个在使用时一定要注意，必要时（前提是已经重复确认数据不会产生加锁等相互影响），可以加上\*\*–skip-add-locks \*\* 选项取消加表锁。

(2) — Dump completed on  
备份结束后，可以查看最后一行是否为：**— Dump completed on …..**