---
id: 825
title: 'Tips: MySQL8.0版本DEFAULT_GENERATED的问题'
date: '2019-11-28T18:00:49+08:00'
author: Shuo
excerpt: 'MySQL8.0, DEFAULT_GENERATED for columns that have an expression default value.'
layout: post
guid: 'http://codercoder.cn/?p=825'
image: http://codercoder.cn/wp-content/uploads/2019/11/7c92af75372793c7b8ebddadd31d3e61.png
permalink: /index.php/2019/11/tips-mysql8-default_generated/
views:
    - '6326'
bigfa_ding:
    - '1'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Tool
    - Fell-in-pit
---

# 背景

最近使用mysqldiff工具进行多源数据表结构对比时（一个数据源为8.0版本，另一个为5.7版本），发现mysqldiff得到的结果中，出现了如下语句：

```
ALTER TABLE  tb1
CHANGE COLUMN create_time create_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP DEFAULT_GENERATED COMMENT '创建时间';

```

把mysqldiff生成的sql文件在5.7版本中回放mysqldiff的结果时候，报错：  
\*\*ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ‘DEFAULT\_GENERATED COMMENT ‘创建时间’, \*\*

> 所以推测为8.0新增的表结构说明，而5.7版本不兼容

# 解决

8.0文档对于DEFAULT\_GENERATED的说明：  
**DEFAULT\_GENERATED for columns that have an expression default value.**  
即，在column上若有表达式类型的默认值时，会在该行的Extra列打上DEFAULT\_GENERATED的tag

![](http://codercoder.cn/wp-content/uploads/2019/11/7c92af75372793c7b8ebddadd31d3e61.png)

**The latest version of mysql is now adding a DEFAULT\_GENERATED tag when a default is explicitly set**

[https://dev.mysql.com/doc/refman/8.0/en/show-columns.html](https://dev.mysql.com/doc/refman/8.0/en/show-columns.html "https://dev.mysql.com/doc/refman/8.0/en/show-columns.html")

登录到数据库中，查看该表的columns信息：  
![](http://codercoder.cn/wp-content/uploads/2019/11/8fc132f300744dcc09ebb870598a37fb.png)

其对于数据无影响，但是在使用mysqldiff工具时候，会把Extra列同样作为一个指标，出现不一致的情况。

# 解决

可以编辑mysqldiff的结果，删除DEFAULT\_GENERATED关键字即可。

附加：  
mysqldiff检查项说明：  
![](http://codercoder.cn/wp-content/uploads/2019/11/124229c96998c54231b7021345eba362.png)

[https://www.percona.com/blog/2015/04/15/checking-table-definition-consistency-mysqldiff/](https://www.percona.com/blog/2015/04/15/checking-table-definition-consistency-mysqldiff/ "https://www.percona.com/blog/2015/04/15/checking-table-definition-consistency-mysqldiff/")