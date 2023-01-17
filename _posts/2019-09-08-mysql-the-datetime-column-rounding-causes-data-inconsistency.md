---
id: 421
title: 'MySQL datetime字段“四舍五入”导致数据不一致'
date: '2019-09-08T16:05:20+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=421'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0890-300x113.png
permalink: /index.php/2019/09/mysql-the-datetime-column-rounding-causes-data-inconsistency/
views:
    - '1824'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 情况介绍

 近日，在两个数据库比对中发现数据不一致的情况，相同的插入语句（由mybatis生成），在查询时，发现其中一个的insert\_time为’2018-07-27 10:59:59’，另一个为’2018-07-27 11:00:00’。想到是插入时候时间精度不一致的原因，进行排查。

 在审计中可以看到两次的插入datetime不一样，一次为’2018-07-27 10:59:59’，另一哥insert为：’2018-07-27 10:59:59.881’，原因找到，可是基本原理是什么呢？

 猜想MySQL进行了“四舍五入”的操作，导致两边数据出现不一致的问题，数据库环境为MySQL 5.6.16，查看官方文档：[Fractional Seconds in Time Values](https://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)

MySQL 5.6.4 and up expands fractional seconds support for TIME, DATETIME, and TIMESTAMP values, with up to microseconds (6 digits) precision:

> 从MySQL 5.6.4起，对于TIME, DATETIME, TIMESTAMP字段，最高可支持到微秒（小数点后6位）的精度

Inserting a TIME, DATE, or TIMESTAMP value with a fractional seconds part into a column of the same type but having fewer fractional digits results in rounding, as shown in this example:  
![inserting while fewer fractional digits](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0890-300x113.png)

No warning or error is given when such rounding occurs. This behavior follows the SQL standard, and is not affected by the server’s sql\_mode setting.

**插入到相同或者精度更短的字段中时，会进行“四舍五入”，并且不会有任何提示。**

此外，JDBC同样也向数据库中提交了毫秒的精度：

- 5.1.6版本：直接舍弃毫秒部分
- 5.1.30版本：保留毫秒部分并发送到server

# 解决思路

1. 通过在程序中截断毫秒的部分：实现com.ibatis.sqlmap.client.extensions.TypeHandlerCallback接口，对ibatis针对java.util.Date类型的赋值前进行拦截，强制丢弃毫秒部分。
2. 若需要存储毫秒精度，则在表定义时进行配置。