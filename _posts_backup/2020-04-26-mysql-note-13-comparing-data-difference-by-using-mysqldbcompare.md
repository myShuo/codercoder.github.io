---
id: 1030
title: 'MySQL手记13 &#8212; 使用mysqldbcompare对比数据一致性'
date: '2020-04-26T14:32:38+08:00'
author: Shuo
excerpt: mysqldbcompare可以说是非常常用的一个数据对比工具，用以检验数据一致性，在我们测试数据同步、数据迁移的时候，经常会用到，用来判断是否会出现数据不一致的情况。
layout: post
guid: 'http://codercoder.cn/?p=1030'
permalink: /index.php/2020/04/mysql-note-13-comparing-data-difference-by-using-mysqldbcompare/
views:
    - '3488'
    - '3488'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、简介

 前面说到使用mysqldiff工具进行不同数据源的结构差异的对比，本篇将介绍对比不同数据源中的数据差异。在生产环境中非常有用，可以用来对比迁移的数据是否一致，又或是用来对比数据间差异，进行数据的补齐。

# 二、使用说明

 同样通过–help查看相关的帮助信息，选择合适的对比策略：

```
# mysqldbcompare --help

MySQL Utilities mysqldbcompare version 1.6.5
License type: GPLv2
Usage: mysqldbcompare --server1=user:pass@host:port:socket --server2=user:pass@host:port:socket db1:db2
mysqldbcompare - compare databases for consistency
...

```

比较有意思的是，可以选择多种不同的跳过类型：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2622.png)

## 2.1 对比不同数据源的数据

 使用mysqldbcompare对比两个数据库的结构及数据差异：

### （1）–run-all-test

 若不加run-all-test，则当遇到第一个差异时候，就会退出：

```
mysqldbcompare --server1=root:root@172.16.3.3:4407 --server2=root:root@172.16.3.3:4408 wstestdb1:wstestdb2 --difftype=SQL --show-reverse -vvv --changes-for=server1

```

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-26100.png)  
   
 截图即找到第一个差异时，就直接退出，不再进行其它对象的比较。

### （2）–skip-diff

 mysqldbcompare –server1=root:root@172.16.3.3:4407 –server2=root:root@172.16.3.3:4408 wstestdb1:wstestdb2 –difftype=SQL –show-reverse -vvv –changes-for=server1 –skip-diff  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2627.png)

 跳过对象差异的步骤，例如上部分的defination，在对比时可以忽略，并在注释中提示：Definination Diff — Skip

但是–skip-diff不会跳过数据比较时的差异！

### （3）加上–run-all-test，

 会把库中的对象全部进行对比：  
*mysqldbcompare –server1=root:root@172.16.3.3:4407 –server2=root:root@172.16.3.3:4408 wstestdb1:wstestdb2 –difftype=SQL –show-reverse -vvv –changes-for=server1 –skip-table-options –run-all-test*

```
$ mysqldbcompare --server1=root:root@172.16.3.3:4407 --server2=root:root@172.16.3.3:4408 wstestdb1:wstestdb2 --difftype=SQL --show-reverse -vvv --changes-for=server1 --skip-table-options --run-all-test
​
# WARNING: Using a password on the command line interface can be insecure.
# server1 on 172.16.3.3: ... connected.
# server2 on 172.16.3.3: ... connected.
# Checking databases wstestdb1 on server1 and wstestdb2 on server2
#
Looking for object types table, view, trigger, procedure, function, and event.
Object types found common to both databases:
     FUNCTION : 0
      TRIGGER : 0
        TABLE : 3
        EVENT : 0
    PROCEDURE : 0
         VIEW : 0
#                                                   Defn    Row     Data
# Type      Object Name                             Diff    Count   Check
# -------------------------------------------------------------------------
# TABLE     wm_order                                FAIL    pass    -
#           - Compare table checksum                                FAIL
#           - Find row differences                                  pass
# Definition for object wstestdb1.wm_order:
....
#
# INFO: for table wm_order the index PRIMARY is used to compare.
#
# Transformation for --changes-for=server1:
#
ALTER TABLE `wstestdb1`.`wm_order`
  DROP COLUMN company;
#
# Transformation for reverse changes (--changes-for=server2):
#
# ALTER TABLE `wstestdb2`.`wm_order`
#   ADD COLUMN company varchar(100) NOT NULL DEFAULT 'Tuya' COMMENT 'company' AFTER alias_name;
#
# TABLE     ws_test_0                               pass    pass    -
#           - Compare table checksum                                FAIL
#           - Find row differences                                  FAIL
# Definition for object wstestdb1.ws_test_0:
....
#
# INFO: for table ws_test_0 the index PRIMARY is used to compare.
#
# Transformation for --changes-for=server1:
#
DELETE FROM `wstestdb1`.`ws_test_0` WHERE `id` = '10';
INSERT INTO `wstestdb1`.`ws_test_0` (`id`, `name`, `device_id`, `addr`, `alias_name`, `company`) VALUES('12', 'e11', '11', 'D11', 'd11-d', 'd-d-d4');
#
# Transformation for reverse changes (--changes-for=server2):
#
# DELETE FROM `wstestdb2`.`ws_test_0` WHERE `id` = '12';
# INSERT INTO `wstestdb2`.`ws_test_0` (`id`, `name`, `device_id`, `addr`, `alias_name`, `company`) VALUES('10', 'e10', '10', 'D10', 'd10-d', 'd-d-d');
#
# TABLE     ws_test_1                               pass    FAIL    -
#           - Compare table checksum                                FAIL
#           - Find row differences                                  FAIL
# Definition for object wstestdb1.ws_test_1:
...
......
# Database consistency check failed.
#
# ...done

```

 只要有任何的不一致，结果均为：Database consistency check failed.

**查看对比的情况：**  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2694.png)  
例如：  
 一共需要对比两个库的3张TABLES，其中：  
 wm\_order表：  
 —- Defination Diff：失败  
 —- Row count：通过  
 —-compare checksum：失败

## 2.2 注意事项：

 （1）由于数据的对比需要抽取源端和目标端的数据进行逐行比对，所以会消耗大量的内存资源，在使用时应该注意  
 （2）尽量阶段性对比数据，不要一次全量对比很大的数据量，防止对数据源产生影响

# 三、小结

 mysqldbcompare可以说是非常常用的一个数据对比工具，在我们测试数据同步、数据迁移的时候，经常会用到，用来判断是否会出现数据不一致的情况，DBA只需使用工具，就可以对于环境中的差异，使得效率大大提高。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)