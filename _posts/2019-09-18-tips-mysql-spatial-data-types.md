---
id: 641
title: MySQL中使用空间位置需注意的问题
date: '2019-09-18T09:49:17+08:00'
author: Shuo
excerpt: 使用MySQL空间数据类型时的注意事项。
layout: post
guid: 'http://codercoder.cn/?p=641'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1822.png
permalink: /index.php/2019/09/tips-mysql-spatial-data-types/
views:
    - '4873'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 空间位置数据类型

<dl><dt>MySQL支持的空间数据类型<sup id="fnref-641-1">[1](#fn-641-1)</sup>：</dt><dd>**GEOMETRY**</dd><dd>**POINT**</dd><dd>**LINESTRING**</dd><dd>**POLYGON**</dd></dl>最常用的为**GEOMETRY，POLYGON，POINT**。因为目前很多应用都是判断是否某个点在所画范围内，或者是两个多边形范围的交叉情况。

[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1822.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1822.png)

## GEOMETRY

GEOMETRY是一种强大的空间数据类型，可以用来表示**POINT, LINESTRING, and POLYGON**，即另外的三种空间数据类型均可以使用GEOMETRY来表示（线上也推荐使用此种模式，避免出现**兼容性问题** ）

.  
.

# 常见问题

## 空间索引

所有的空间字段**不能为NULL**，不然在创建空间索引的时候会报错：

```
mysql&gt; create table spatest(id int auto_increment primary key, userspace geometry default null,  SPATIAL KEY `idx_userspace` (`userspace`));
ERROR 1252 (42000): All parts of a SPATIAL index must be NOT NULL

```

## 关于使用polygon数据类型

polygon中的打点，必须为首尾相连，形成闭合图形（**在应用中容易打点时忽略最后一个点**）

```
mysql&gt; select st_astext(ST_geomfromtext('POLYGON(( 119.41667 24.078513,115.416761 26.078578,115.41725 24.078327,113.4177 24.078027,115.418154 25.077749, 119.41667 26.078513))')); 
ERROR 3037 (22023): Invalid GIS data provided to function st_geomfromtext.


mysql&gt; select st_astext(ST_geomfromtext('POLYGON(( 119.41667 24.078513,115.416761 26.078578,115.41725 24.078327,113.4177 24.078027,115.418154 25.077749, 119.41667 24.078513))')); 
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| st_astext(ST_geomfromtext('POLYGON(( 119.41667 24.078513,115.416761 26.078578,115.41725 24.078327,113.4177 24.078027,115.418154 25.077749, 119.41667 24.078513))')) |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| POLYGON((119.41667 24.078513,115.416761 26.078578,115.41725 24.078327,113.4177 24.078027,115.418154 25.077749,119.41667 24.078513))                                 |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)



```

**注意**

> MySQL8.0将astext函数转换为st\_astext。

# 空间相关函数

常用的空间相关函数<sup id="fnref-641-2">[2](#fn-641-2)</sup>：  
**ST\_Intersects(g1, g2):**  
Returns 1 or 0 to indicate whether g1 spatially intersects g2.

**ST\_Contains(g1, g2):**  
Returns 1 or 0 to indicate whether g1 completely contains g2. This tests the opposite relationship as ST\_Within().

<div class="footnotes" role="doc-endnotes">- - - - - -

1. [MySQL空间数据类型](https://dev.mysql.com/doc/refman/5.6/en/spatial-type-overview.html) [↩︎](#fnref-641-1)
2. [MySQL支持的空间函数](https://dev.mysql.com/doc/refman/5.6/en/spatial-relation-functions-object-shapes.html#function_st-intersects) [↩︎](#fnref-641-2)

</div>