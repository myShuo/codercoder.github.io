---
id: 540
title: 'MySQL数据库Loose Index Scan与Tight Index Scan介绍'
date: '2019-09-10T17:24:15+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=540'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1097.png
permalink: /index.php/2019/09/mysql-loose-index-scan-and-tight-index-scan-introduction/
bigfa_ding:
    - '1'
views:
    - '5737'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# 介绍

本篇文章主要介绍Loose Index Scan与Tight Index Scan，次两种方法均为**group by**语法时候，MySQL进行的优化方法。官方文档：[Group by Optimization](https://dev.mysql.com/doc/refman/8.0/en/group-by-optimization.html)  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1097.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1097.png)

# Group by Optimization

> The most general way to satisfy a GROUP BY clause is to scan the whole table and create a new temporary table where all rows from each group are consecutive, and then use this temporary table to discover groups and apply aggregate functions (if any). In some cases, MySQL is able to do much better than that and to avoid creation of temporary tables by using index access.

**通常情况下，GROUP BY 语法通过扫描全表、创建临时表，并使用这个临时表进行分组和聚合运算。在某些情况下，MySQL可以使用索引来规避创建临时表。**

> There are two ways to execute a GROUP BY query through index access, as detailed in the following sections. In the first method, the grouping operation is applied together with all range predicates (if any). The second method first performs a range scan, and then groups the resulting tuples.

**GROUP BY 有两种方式使用索引。第一：group运算和所有的可能的范围一起应用；第二种：首先执行range扫描，然后再group得到的元祖。**

# Loose Index Scan-松散索引扫描

> The most efficient way to process GROUP BY is when an index is used to directly retrieve the grouping columns. With this access method, MySQL uses the property of some index types that the keys are ordered (for example, BTREE). This property enables use of lookup groups in an index without having to consider all keys in the index that satisfy all WHERE conditions. This access method considers only a fraction of the keys in an index, so it is called a loose index scan. When there is no WHERE clause, a loose index scan reads as many keys as the number of groups, which may be a much smaller number than that of all keys. If the WHERE clause contains range predicates (see the discussion of therangejoin type inSection 8.8.1, “Optimizing Queries with EXPLAIN”), a loose index scan looks up the first key of each group that satisfies the range conditions, and again reads the least possible number of keys.

**GROUP BY 最有效的方式是：group by的字段直接覆盖索引。在此情况下。MySQL使用已按key排序的合适索引类型（例如：BTREE）。这个特性使得对于索引字段group的查找不需要检索WHERE条件的所有索引字段。这种索引方式只扫描一个索引中的部分字段，所以称之为松散索引扫描。**

**当没有WHERE一句时，loose index scan reads需要扫描group by 中的字段，这也比扫描全部的字段更加的轻量。若在WHERE条件中包含range条件，则loose index scan reads查询满足range条件的group by 中的第一个字段，即仅扫描最少数量的字段。**

The query is over a single table.

The GROUP BY names only columns that form a leftmost prefix of the index and no other columns. (If, instead of GROUP BY, the query has a DISTINCT clause, all distinct attributes refer to columns that form a leftmost prefix of the index.) For example, if a table t1 has an index on (c1,c2,c3), loose index scan is applicable if the query has GROUP BY c1, c2,. It is not applicable if the query has GROUP BY c2, c3 (the columns are not a leftmost prefix) or GROUP BY c1, c2, c4 (c4 is not in the index).

The only aggregate functions used in the select list (if any) are MIN() and MAX(), and all of them refer to the same column. The column must be in the index and must immediately follow the columns in the GROUP BY.

Any other parts of the index than those from the GROUP BY referenced in the query must be constants (that is, they must be referenced in equalities with constants), except for the argument of MIN() or MAX() functions.

For columns in the index, full column values must be indexed, not just a prefix. For example, with c1 VARCHAR(20), INDEX (c1(10)), the index cannot be used for loose index scan.

If loose index scan is applicable to a query, the EXPLAIN output shows Using index for group-by in the Extra column.

Assume that there is an index idx(c1,c2,c3) on table t1(c1,c2,c3,c4). The loose index scan access method can be used for the following queries:

```
SELECT c1, c2 FROM t1 GROUP BY c1, c2;

SELECT DISTINCT c1, c2 FROM t1;

SELECT c1, MIN(c2) FROM t1 GROUP BY c1;

SELECT c1, c2 FROM t1 WHERE c1 < const GROUP BY c1, c2;

SELECT MAX(c3), MIN(c3), c1, c2 FROM t1 WHERE c2 > const GROUP BY c1, c2;

SELECT c2 FROM t1 WHERE c1 < const GROUP BY c1, c2;

SELECT c1, c2 FROM t1 WHERE c3 = const GROUP BY c1, c2;

```

The following queries cannot be executed with this quick select method, for the reasons given:

> There are aggregate functions other than MIN() or MAX():

```
SELECT c1, SUM(c2) FROM t1 GROUP BY c1;

```

> The columns in the GROUP BY clause do not form a leftmost prefix of the index:

```
SELECT c1, c2 FROM t1 GROUP BY c2, c3;

```

> The query refers to a part of a key that comes after the GROUP BY part, and for which there is no equality with a constant:

```
SELECT c1, c3 FROM t1 GROUP BY c1, c2;

```

**Were the query to include WHERE c3 = const, loose index scan could be used.**

The loose index scan access method can be applied to other forms of aggregate function references in the select list, in addition to the MIN() and MAX() references already supported:

AVG(DISTINCT), SUM(DISTINCT), and COUNT(DISTINCT) are supported.

AVG(DISTINCT) and SUM(DISTINCT) take a single argument.

COUNT(DISTINCT) can have more than one column argument.

- There must be no GROUP BY or DISTINCT clause in the query.
- The loose scan limitations described earlier still apply.

> Assume that there is an index idx(c1,c2,c3) on table t1(c1,c2,c3,c4). The loose index scan access method can be used for the following queries:

```
SELECT COUNT(DISTINCT c1), SUM(DISTINCT c1) FROM t1;

SELECT COUNT(DISTINCT c1, c2), COUNT(DISTINCT c2, c1) FROM t1;

```

# Tight Index Scan-紧凑索引扫描

> A tight index scan may be either a full index scan or a range index scan, depending on the query conditions.

**紧凑索引扫描即，要么是覆盖索引扫描或者范围索引扫描，依查询条件而定。**

> When the conditions for a loose index scan are not met, it still may be possible to avoid creation of temporary tables for GROUP BY queries. If there are range conditions in the WHERE clause, this method reads only the keys that satisfy these conditions. Otherwise, it performs an index scan. Because this method reads all keys in each range defined by the WHERE clause, or scans the whole index if there are no range conditions, we term it a tight index scan. With a tight index scan, the grouping operation is performed only after all keys that satisfy the range conditions have been found.

**在没有满足loose index scan之前，GROUP BY查询同样是可以避免创建临时表的。如果在WHERE中有range条件，MySQL会扫描这些字段。此外，更趋向于索引扫描。因为这种方法扫描所有符合WHERE条件range中的字段，或者所有字段（没有range条件时），所以称之为tight index scan。**

> Assume that there is an index idx(c1,c2,c3) on table t1(c1,c2,c3,c4). The following queries do not work with the loose index scan access method described earlier, but still work with the tight index scan access method.

**不能用loose index scan，但是能使用tight index scan的情况：**

There is a gap in the GROUP BY, but it is covered by the condition c2 = ‘a’:

```
SELECT c1, c2, c3 FROM t1 WHERE c2 = 'a' GROUP BY c1, c3;

```

The GROUP BY does not begin with the first part of the key, but there is a condition that provides a constant for that part:

```
SELECT c1, c2, c3 FROM t1 WHERE c1 = 'a' GROUP BY c2, c3;

```