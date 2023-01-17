---
id: 548
title: MySQL重复索引检查工具：pt-duplicate-key-checker
date: '2019-09-10T17:39:23+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=548'
permalink: /index.php/2019/09/mysql-duplicate-check-tool-pt-duplicate-key-checker/
views:
    - '15812'
    - '15812'
    - '15812'
    - '15812'
    - '15812'
categories:
    - Tech
tags:
    - MySQL
    - 工具
    - Tech
---

# 功能介绍：

在MySQL数据库表中找出重复的索引和外键，这个工具会将重复的索引和外键都列出来，并生成删除重复索引的语句。  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1027.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1027.png)

# 用法介绍：

pt-duplicate-key-checker \[OPTION…\]

\[DSN\]包含比较多的选项，具体的可以通过命令***pt-duplicate-key-checker –help***来查看具体支持那些选项。

# 使用示例：

 查看test数据库的重复索引和外键使用情况：

```
pt-duplicate-key-checker --host=localhost --user=root --password=aaa123 --databases=test

```

得到结果如下：

```
......

Key idx_Create_id ends with a prefix of the clustered index

Key definitions:
    KEY `idx_Create_id` (`CREATE`,`ID`)
    PRIMARY KEY (`ID`),

Column types:
    `create` timestamp not null default '0000-00-00 00:00:00'
     `id` bigint(20) not null auto_increment

To shorten this duplicate clustered index, execute:

     **ALTER TABLE `test`.`OUTPUT_STAT` DROP INDEX `idx_GmtCreate_id`, ADD INDEX `idx_GmtCreate_id` (`GMT_CREATE`);**

########################################################################

Summary of indexes                                                     

########################################################################

Size Duplicate Indexes  252

Total Duplicate Indexes  8

Total Indexes            149

```

**根据日志可以看出：**  
1.发现了重复索引 ***`idx_Create_id` (`CREATE`,`ID`)*** ，可以使用以下语句进行优化：

> ALTER TABLE `test`.`OUTPUT_STAT` DROP INDEX `idx_GmtCreate_id`, ADD INDEX `idx_GmtCreate_id` (`GMT_CREATE`);

2.索引情况的汇总信息