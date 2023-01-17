---
id: 997
title: 'MySQL手记12 &#8212; 表结构对比工具mysqldiff'
date: '2020-04-26T13:28:59+08:00'
author: Shuo
excerpt: mysqldiff可以用来比较两个指定数据源中的结构差异。
layout: post
guid: 'http://codercoder.cn/?p=997'
permalink: /index.php/2020/04/mysql-note-12-comparing-table-difference-by-using-mysqldiff/
views:
    - '3657'
    - '3657'
    - '3657'
    - '3657'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、功能

 mysqldiff可以用来比较两个指定数据源中的结构差异，类似于Linux操作系统中的diff命令。

# 二、使用介绍

 对于工具的使用，直接使用–help进行查看，对其功能的介绍也及其详细：

```
mysqldiff --help

MySQL Utilities mysqldiff version 1.6.5
License type: GPLv2
Usage: mysqldiff --server1=user:pass@host:port:socket --server2=user:pass@host:port:socket db1.object1:db2.object1 db3:db4

mysqldiff - compare object definitions among objects where the difference is
how db1.obj1 differs from db2.obj2

...

```

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2626.png)

 可以灵活选用提供的选项进行结构的比对，下文将选取常见的集中情况介绍。

## 2.1 使用介绍

**（1）不加上–force**  
 说明：若不加上–force选项，则在第一个差异产生的时候，进程就会退出。这样的情况适用于只有少量差异，并逐一对比修改的情况，但是效率较低，通常，我们会把所有的差异查找出来，进行修改。

**（2）加上–force进行完整对比**  
 使用mysqldiff对比两个数据库的结构差异

```
mysqldiff --server1=root:root@172.16.3.3:4407 --server2=root:root@172.16.3.3:4408 wstestdb1:wstestdb2 --difftype=SQL --show-reverse -vvv --changes-for=server1 --force

# WARNING: Using a password on the command line interface can be insecure.
# server1 on 172.16.3.3: ... connected.
# server2 on 172.16.3.3: ... connected.

# Definition for object wstestdb1:
CREATE DATABASE `wstestdb1` /*!40100 DEFAULT CHARACTER SET utf8 */

# Definition for object wstestdb2:
CREATE DATABASE `wstestdb2` /*!40100 DEFAULT CHARACTER SET utf8 */
# Comparing `wstestdb1` to `wstestdb2`                             [PASS]
**###### 第一部分，对比数据库建库的属性是否一致  --【pass】**

# Definition for object wstestdb1.wm_order:
....

# Definition for object wstestdb2.wm_order:
....
# Comparing `wstestdb1`.`wm_order` to `wstestdb2`.`wm_order`       [PASS]
**###### 第二部分，对比wstestdb1.wm_order表结构是否一致  --【pass】**

# Definition for object wstestdb1.ws_test_0:
....

# Definition for object wstestdb2.ws_test_0:
....
# Comparing `wstestdb1`.`ws_test_0` to `wstestdb2`.`ws_test_0`     [FAIL]
**###### 第二部分，对比wstestdb1.ws_test_0表结构是否一致  --【fail】**

###由于命令中指定--difftype=SQL --show-reverse --changes-for=server1，所以对比后会提示根据可以在server1上执行  ALTER TABLE `wstestdb1`.`ws_test_0` AUTO_INCREMENT=13, COMMENT='wstest';   即可使结构一致。
# Transformation for --changes-for=server1:
#

ALTER TABLE `wstestdb1`.`ws_test_0`
AUTO_INCREMENT=13, COMMENT='wstest';

**######由于命令中有-vvv选项，所以会冗余的提示若需要修改server2，则可以执行：ALTER TABLE `wstestdb2`.`ws_test_0` AUTO_INCREMENT=11, COMMENT='wstest'; **
# Transformation for reverse changes (--changes-for=server2):
#
# ALTER TABLE `wstestdb2`.`ws_test_0`
# AUTO_INCREMENT=11, COMMENT='wstest';
#


# Definition for object wstestdb1.ws_test_1:
...
......

**######显示最终的结果：**
# Compare failed. One or more differences found.

```

**（1）可以看出，mysqldiff的对比步骤为：**  
– 对比数据库定义 —-下图part(1)  
– 对比表结构 —-下图part(2)  
 —-表的定义、字段定义、字段的数量  
– 汇总结果：  
 并且在每个步骤结束后，提示reverse的信息 下图part(3)  
 并且会在每个步骤打印出是否一致（下图箭头指向的部分）  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2660.png)

**（2）提示信息**  
 可以查看上述的加粗部分，每个步骤的提示信息均与命令的选项有关：

**（3）其它选项**  
 可以看出，两个库（wstestdb1和wstestdb2）中的主要的结构不同，在于对于表的定义，而我们在乎的更多的，可能是表字段的差异，所以，可以选择过滤掉表的定义信息的比较，使用选项：

```
 --skip-table-options  skip check of all table options (e.g., AUTO_INCREMENT, ENGINE, CHARSET, etc.).

```

示范不对比表的定义：

```
mysqldiff --server1=root:root@172.16.3.3:4407 --server2=root:root@172.16.3.3:4408 wstestdb1:wstestdb2 --difftype=SQL --show-reverse -vvv --changes-for=server1 --skip-table-options --force
# WARNING: Using a password on the command line interface can be insecure.
# server1 on 172.16.3.3: ... connected.
# server2 on 172.16.3.3: ... connected.

# Definition for object wstestdb1:
CREATE DATABASE `wstestdb1` /*!40100 DEFAULT CHARACTER SET utf8 */

# Definition for object wstestdb2:
CREATE DATABASE `wstestdb2` /*!40100 DEFAULT CHARACTER SET utf8 */
# Comparing `wstestdb1` to `wstestdb2`                             [PASS]

# Definition for object wstestdb1.wm_order:
....

# Definition for object wstestdb2.wm_order:
....
# Comparing `wstestdb1`.`wm_order` to `wstestdb2`.`wm_order`       [PASS]

# Definition for object wstestdb1.ws_test_0:
....

# Definition for object wstestdb2.ws_test_0:
....

# Comparing `wstestdb1`.`ws_test_0` to `wstestdb2`.`ws_test_0`     [PASS]
# WARNING: Table options are ignored and differences were found:
# --- `wstestdb1`.`ws_test_0`
# +++ `wstestdb2`.`ws_test_0`
# @@ -1,5 +1,5 @@
#  ENGINE=InnoDB
# -AUTO_INCREMENT=11
# +AUTO_INCREMENT=13
#  DEFAULT
#  CHARSET=utf8mb4
#  COMMENT='wstest'

# Definition for object wstestdb1.ws_test_1:
....

# Definition for object wstestdb2.ws_test_1:
....

# Comparing `wstestdb1`.`ws_test_1` to `wstestdb2`.`ws_test_1`     [PASS]
# WARNING: Table options are ignored and differences were found:
# --- `wstestdb1`.`ws_test_1`
# +++ `wstestdb2`.`ws_test_1`
# @@ -1,4 +1,5 @@
#  ENGINE=InnoDB
# +AUTO_INCREMENT=12
#  DEFAULT
#  CHARSET=utf8mb4
#  COMMENT='wstest'
# Success. All objects are the same.

```

可以看到：  
 在出现定义不一致时候，会在注释中打上WARNING，并提示差异。  
 在忽略了表的定义后，两个库之间结构对比成功，提示：Success. All objects are the same.

# 三、小结

 mysqldiff可以方便的对比出两个数据源之间的结构差异，在生产环境中，往往用以比较新、老环境，或者是测试、开发环境之间的差异，非常方便的可以解决这个繁琐的表结构对比步骤，解放DBA的双手。

*ps.若想要对比数据差异，mysqldiff不能做到，此时，就需要另一个工具：**mysqldbcompare**将在后续介绍。*

欢迎关注公众号：**朔的话**  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)