---
id: 1048
title: 'MySQL手记15 &#8212; 大小写问题'
date: '2020-04-29T20:18:54+08:00'
author: Shuo
excerpt: 在初始化MySQL实例、建表时候，需要注意到大小写的问题，并与开发人员沟通，若需要表结构大小写敏感，则调整lower_case_table_names；若需要数据的大小写敏感，调整utf8mb4_general_ci/utf8mb4_bin，当然常见的还有utf8_general_ci/utf8_bin。
layout: post
guid: 'http://codercoder.cn/?p=1048'
permalink: /index.php/2020/04/mysql-note-15-case-sensitivity-and-case-insensitivity/
views:
    - '1656'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

 最近在测试时候，看到一个字符集校验规则的问题，由于设置了不同的校验规则，获取到的数据结果也就不一致。  
 MySQL的大小写规则，包含：表结构大小写、数据的大小写。结构的大小写在实例初始化的时候就需要进行指定：lower\_case\_table\_names；数据的大小写，则由表或者字段的COLLATE进行指定。

# 一、表结构大小写

 表结构大小写通常在实例初始化时候就行指定，lower\_case\_table\_names，  
https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar\_lower\_case\_table\_names  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-296.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-296.png)

**官方文档中的介绍：**[identifier-case-sensitivity](https://dev.mysql.com/doc/refman/8.0/en/identifier-case-sensitivity.html)  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2915.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2915.png)

## 修改lower\_case\_table\_names

（1）调整为区分大小写，会报错，因为该参数不可以进行动态调整：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2971.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2971.png)

（2）在实例初始化后修改my.cnf也是不能够启动成功的，错误信息：  
*\[ERROR\] \[MY-011087\] \[Server\] Different lower\_case\_table\_names settings for server (‘0’) and data dictionary (‘1’).  
\[ERROR\] \[MY-010020\] \[Server\] Data Dictionary initialization failed.  
\[ERROR\] \[MY-010119\] \[Server\] Aborting*

[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2923.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2923.png)

# 二、数据字符串大小写

 对于数据中字符串的大小写是否做区分，主要是看指定的字符集：  
 字符集\_ci结尾的排序规则，则是不区分大小写的，  
 \_bin结尾，区分大小写

 可以查看select \* from INFORMATION\_SCHEMA.COLLATIONS; 查看支持的字符集校对规则。  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2995.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2995.png)

## 2.1 表级别COLLATIONS

两张表test1、test2

```
CREATE TABLE if not exists `test1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(30) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci



CREATE TABLE if not exists `test2` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(30) COLLATE utf8mb4_bin NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin

```

分别插入相同的数据：

```
mysql&gt; insert into test1 values(1,'aaa');
Query OK, 1 row affected (0.05 sec)

mysql&gt; insert into test2 values(1,'aaa');
Query OK, 1 row affected (0.02 sec)

mysql&gt; select * from test1 where name='AAA';
+----+------+
| id | name |
+----+------+
|  1 | aaa  |
+----+------+
1 row in set (0.00 sec)

mysql&gt; select * from test2 where name='AAA';
Empty set (0.00 sec)

mysql&gt; select * from test2 where name='aaa';
+----+------+
| id | name |
+----+------+
|  1 | aaa  |
+----+------+
1 row in set (0.00 sec)

```

[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2936.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2936.png)

 在test1表中查询select \* from test1 where name=’AAA’; 虽然是大写，但是**由于表的collation为：utf8mb4\_general\_ci，所以是不区分大小写的，可以查询到结果**。

 在test2中执行相同的查询，查询不到结果，**因为test2的collation是utf8mb4\_bin，区分大小写，’AAA’不等于’aaa’，返回空。**

## 2.2 字段级别

 一个表test3，表的collate为utf8mb4\_general\_ci，字段name为**utf8mb4\_general\_ci**，**addr为utf8mb4\_bin**：

```
CREATE TABLE if not exists `test3` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
  `addr` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci

```

插入一行数据：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2950.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2950.png)

按照大写进行查询：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2983.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2983.png)

## 结果

 对于不区分大小写的name为utf8mb4\_general\_ci：使用大写字符可以查询到结果  
 对于区分大小写的addr为utf8mb4\_bin：使用大写字符查询不到结果

# 三、小结

 在初始化MySQL实例、建表时候，需要注意到大小写的问题，并与开发人员沟通，若需要表结构大小写敏感，则调整lower\_case\_table\_names；若需要数据的大小写敏感，调整utf8mb4\_general\_ci/utf8mb4\_bin，当然常见的还有utf8\_general\_ci/utf8\_bin。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)