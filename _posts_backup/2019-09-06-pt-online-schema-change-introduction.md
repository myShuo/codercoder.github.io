---
id: 314
title: 'MySQL手记10 &#8212; pt-online-schema-change使用简介'
date: '2019-09-06T16:16:42+08:00'
author: Shuo
excerpt: pt-online-schema-change使用简介(pt-osc)、注意事项及举例：在对大表进行结构变更时，报错退出；变更主键；变更时候的负载控制。
layout: post
guid: 'http://codercoder.cn/?p=314'
permalink: /index.php/2019/09/pt-online-schema-change-introduction/
bigfa_ding:
    - '2'
views:
    - '8838'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

![](http://121.40.246.76/wp-content/uploads/2019/09/2019-09-0641.jpg)

# 1. 功能介绍：

 pt-online-schema-change，即pt-osc，目的为在alter操作更改表结构的时候不用长时间锁定表，执行alter的时候不会阻塞写和读取操作。

```
注意：执行这个工具的时候 **必须做好备份** ，操作之前详细读一下官方文档[pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html "pt-online-schema-change")

```

![](http://codercoder.cn/wp-content/uploads/2019/09/2020-04-2273.png)

# 2. 工作原理

根据pt-osc在执行时打印的日志，可以大致了解到其工作原理：  
1\. 创建新表，并在新表上变更  
2\. 从原表中copy原始数据到表结构修改后的表

```
在copy数据的过程中，任何在原表的更新操作都会更新到新表，因为这个工具在会在原表上创建触发器，触发器会将在原表上更新的内容更新到新表。**如果表中已经定义了其它触发器，这个工具就不能工作了！**

```

3\. 当数据copy完成以后就会将原表移走，用新表代替原表，默认动作是将原表drop掉，**drop操作很危险，需要按照实际情况评估**。  
 在rename的过程，会短暂的锁表，为了得到全局的数据一致。  
 drop表会消耗大量的IO，可以通过如下方式解决：  
 （1）[MySQL安全删除大表drop table](http://codercoder.cn/index.php/2019/09/mysql-drop-large-table/)  
 （2）先把老的表数据进行归档，再进行drop删除老的表

# 3. 用法及注意事项

## 3.1 用法

```
pt-online-schema-change [OPTIONS] DSN

```

options可以自行查看help，DNS为你要操作的数据库和表。  
![](http://codercoder.cn/wp-content/uploads/2019/09/2020-04-2220.png)

### （1）通常用法介绍

在线更改表的的引擎，这个尤其在整理innodb表的时候非常有用，示例如下：

```
pt-online-schema-change --user=root --password=wang123 --host=localhost --lock-wait-time=120 --alter="ENGINE=InnoDB" D=test,t=tb1 --execute 

```

从下面的日志中可以看出它的执行过程：

```
      Altering `test`.`tb1`... 

# 创建新表，并变更
      Creating new table... 
      Created new table test._tb1_new OK. 

      Altering new table... 
      Altered `test`.`_tb1_new` OK. 

#创建触发器
      Creating triggers... 
      Created triggers OK. 

#copy数据到新表
      Copying approximately 995696 rows... 

      Copied rows OK.

#切换新表和老表
      Swapping tables... 
      Swapped original and new tables OK. 

#删除老表
      Dropping old table... 
      Dropped old table `test`.`_tb1_old` OK. 

#删除触发器
      Dropping triggers... 
      Dropped triggers OK. 

Successfully altered `test`.`tb1`. 


```

## 3.2 注意事项

### 3.2.1 删除原表

在参数配置时，**注意是否需要–drop-new-table、–drop-old-table**  
 所以在使用pt-osc工具前，一定要仔细阅读官方文档，并进行测试。

### 3.2.2 变更时候的负载控制

**（1）根据主从延时**  
在添加了–check-slave-lag时候，配置–max-lag，延迟低于–max-lag的时候才继续，多个slave可以使用–recursion-method 参数：  
 指定–recursion-method=”dsn=D=database,t=dsns”

```
CREATE TABLE if not exists `dsns`(
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `parent_id` int(11) DEFAULT NULL,
    `dsn` varchar(255) NOT NULL,
    PRIMARY KEY(`id`)  );

INSERT INTO dsns(parent_id , dsn) VALUES (1, 'h=10.10.1.16,u=*,p=*,P=3306');
INSERT INTO dsns( parent_id , dsn) VALUES (1, 'h=10.10.1.17,u=*,p=*,P=3306');

```

**（2）根据活跃连接数**  
–max-load=：  
 Examine SHOW GLOBAL STATUS after every chunk, and pause if any status variables are higher than their thresholds (default Threads\_running=25)  
 根据 SHOW GLOBAL STATUS去判断Threads\_running的大小，超过阈值，则pt-osc进程暂停。

### 3.2.3 变更主键

 例如需要更高联合主键为一个id自增主键时，应在一个alter中进行如下操作：  
 a. 删除复合主键定义  
 b. 添加新的自增主键  
 c. 原复合主键字段，修改成唯一索引  
在c中，修改成唯一索引的原因：  
 percona手册里有两个地方对修改主键进行了特殊注解：

–alter 和 –\[no\]check-alter  
 A notable exception is when a PRIMARY KEY or UNIQUE INDEX is being created from existing columns as part of the ALTER clause; in that case it will use these column(s) for the DELETE trigger.

```
DROP PRIMARY KEY
If –alter contain DROP PRIMARY KEY (case- and space-insensitive), a warning is printed and the tool exits unless –dry-run is specified. Altering the primary key can be dangerous, but the tool can handle it. The tool’s triggers, particularly the DELETE trigger, are most affected by altering the primary key because the tool prefers to use the primary key for its triggers. You should first run the tool with –dry-run and –print and verify that the triggers are correct.

```

pt-online-schema-change会在原表t1上创建 AFTER DELETE/UPDATE/INSERT 三个触发器，而**对于主键的操作，会影响delete触发器**，因为触发器依赖于主键。

```sql
CREATE TRIGGER `pt_osc_confluence_sbtest3_del` AFTER DELETE ON `confluence`.`sbtest3` FOR EACH ROW DELETE IGNORE FROM `confluence`.`_sbtest3_new` 
WHERE `confluence`.`_sbtest3_new`.`id` = OLD.`id` AND `confluence`.`_sbtest3_new`.`k` = OLD.`k`

```

注：sbtest3表上以(id,k)作为复合主键

所以，如果id或k列上没有索引，**这个删除的代价非常高，所以一定要同时添加复合（唯一）索引 (id,k)** 。如果使用pt-osc去修改删除主键，务必同时添加原主键为 UNIQUE KEY，否则很有可能导致性能问题。

```
而对于INSERT,UPDATE的触发器，依然是  **REPLACE INTO** 语法，因为它采用的是先插入，如果违反主键或唯一约束，则根据主键或意义约束删除这条数据，再执行插入。
    (**但是注意不能依赖于新表的主键递增，因为如果原表有update，新表就会先插入这一条，导致id与原表记录所在顺序不一样**）

```

### 3.2.4 变更主键介绍

#### (1)dry-run

```
$ pt-online-schema-change --user=user--password=userpwd --host=10.10.10.34  --alter "DROP PRIMARY KEY,add column pk int auto_increment primary key,add unique key uk_id_k(id,k)"  D=confluence,t=sbtest3--print --dry-run

```

**部分日志如下所示**

```
--alter contains 'DROP PRIMARY KEY'.  Dropping and altering the primary key can be dangerous,


 ** especially if the original table does not have other unique indexes. **  
 ==>注意 dry-run的输出

ALTER TABLE `confluence`.`_sbtest3_new` DRO PPRIMARY KEY, add column pk int auto_increment primary key, add unique key uk_id_k(id,k)

Altered `confluence`.`_sbtest3_new` OK.

Using original table index PRIMARY for the DELETE trigger instead of new table index PRIMARY because 
==>使用原表主键值判断

the new table index uses column pk which does not exist in the original table.

CREATE TRIGGER `pt_osc_confluence_sbtest3_del` AFTER DELETEON `confluence`.`sbtest3` FOR EACH ROW DELETE IGNORE FROM `confluence`.`_sbtest3_new`
WHERE `confluence`.`_sbtest3_new`.`id` = OLD.`id` AND `confluence`.`_sbtest3_new`.`k` = OLD.`k`

```

#### (2)主键有0值

主键有0值时，需要注意 **pt-online-schema-change会设置 SQL\_MODE中NO\_AUTO\_VALUE\_ON\_ZERO！！！**  
[LP #1709650: pt-online-schema-change eating portion of a table](https://jira.percona.com/browse/PT-1439 "LP #1709650: pt-online-schema-change eating portion of a table")

## 3.4 相关报错及解决方案

## 3.4.1 大表变更

在对大表进行结构变更时，报错退出：  
 pt-online-schema-change 3.0.4  
 MySQL 5.7.17

执行语句为：

```
pt-online-schema-change --alter "add column addr varchar(200)" --print --charset utf8  --chunk-size=50000 --max-load Threads_running=100 --recurse=1 --alter-foreign-keys-method=none --force --execute --statistics --max-lag 3.000000 --noversion-check --recursion-method=processlist --progress percentage,1 D=osc,t=test

```

报错信息如下：

```
......

Copying `osc`.`test`: 34% 02:05:40 remain

**Exiting on SIGHUP.**

Not dropping triggers because the tool was interrupted.  To drop the triggers, execute:

DROP TRIGGER IF EXISTS `osc`.`pt_osc_osc_test_del`

DROP TRIGGER IF EXISTS `osc`.` pt_osc_osc_test_upd`

DROP TRIGGER IF EXISTS `osc`.` pt_osc_osc_test_ins`

Not dropping the new table `osc`.`_test_new` because the tool was interrupted.  To drop the new table, execute:

DROP TABLE IF EXISTS `osc`.`_test_new`;

```

由于没有具体的报错信息，只有一个：\*\* Exiting on SIGHUP.\*\*

**解决**  
猜测可能是由于资源不够导致的，因为毕竟都已经执行到了：**Copying `osc`.`test`: 34% 02:05:40 remain**

—->所以调整**Threads\_running=50**，再次进行测试，没有再出现这个问题，done!

**即如果遇到此类问题，可能是由于资源不够了，可以降低变更时候的工作负载。**

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)