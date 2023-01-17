---
id: 653
title: '怎么查找MySQL中的重复索引和无用索引，并且安全地drop index删除索引？'
date: '2019-09-18T10:35:49+08:00'
author: Shuo
excerpt: '查找MySQL中的重复索引和无用索引，并且安全地drop index删除索引。'
layout: post
guid: 'http://codercoder.cn/?p=653'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1858-300x274.png
permalink: /index.php/2019/09/mysql-drop-index-safely/
views:
    - '5901'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 好文分享

[Dropping useless MySQL indexes](https://federico-razzoli.com/dropping-useless-mysql-indexes)  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1858-300x274.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1858.png)

# 总结

## 重复索引查找方法：

可以使用**pt-duplicate-key-checker**查找出MySQL数据库中的重复索引。  
**注意事项：**  
1\. 重复索引有部分是业务需要的，用以做冗余，或者是完成覆盖索引的sql，不能简单的找到重复索引后进行删除

## 无用索引查找方法：

1. 对于MariaDB或者Percona Server for MySQL，可以使用[user\_statistics](https://www.percona.com/doc/percona-server/LATEST/diagnostics/user_stats.html)，查询information\_schema.INDEX\_STATISTICS，可以找到未被使用的索引（ps.可以查看我之前的文章[MySQL索引统计信息information\_schema.INDEX\_STATISTICS](https://blog.csdn.net/woson_wang/article/details/89916572)）
2. 对于官方版MySQL，可以使用Performance\_schema中的table\_io\_waits\_summary\_by\_index\_usage表进行查找：

```
SELECT object_schema, object_name, index_name
    FROM performance_schema.table_io_waits_summary_by_index_usage
    WHERE index_name IS NOT NULL AND count_star = 0
    ORDER BY object_schema, object_name, index_name;

```

**注：需启动实例前设置performance\_schema = on。**

> 注意事项：  
>  1. 打开performance\_schema统计对系统性能有一定影响  
>  2. MariaDB10默认为关闭performance\_schema  
>  3. **此种统计方法是不准确的，因为其是把未产生IO等待的索引，看作为未被使用的索引。**

## 删除索引

由于对于索引的下线具有业务风险，所以，MySQL8开始，支持索引不可见，语法为：

```
ALTER TABLE t ALTER INDEX idx_a INVISIBLE[VISIBLE];

```

并且该语句可以瞬间完成，以此降低drop index的风险。