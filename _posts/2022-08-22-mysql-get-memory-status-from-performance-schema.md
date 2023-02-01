---
id: 1199
title: 'MySQL Get Memory Status From Performance_Schema'
date: '2022-08-22T18:34:40+08:00'
author: Shuo
excerpt: '当MySQL进程mysqld占用了太多内存时，需要定位对应的用户、线程、event等信息，可以使用performance_schema表信息进行查看'
layout: post
guid: 'http://codercoder.cn/?p=1199'
image: http://codercoder.cn/wp-content/uploads/2022/08/2022-08-22-mysql-get-memory-status-from-performance-schema.png
permalink: /index.php/2022/08/2022-08-22-mysql-get-memory-status-from-performance-schema
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 确认是否打开
```
mysql>  show global variables like "performance_schema";
```
若为ON，则为打开。

# 分类查询
## 统计事件消耗内存
event_name排最前,就代表这个事件模块占内存最多。
例如:JOIN_CACHE就是join的操作,mem0mem就是客户端连接,没开启统计的话,数据可能并不会很多.
```
mysql> select event_name, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_global_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;
```

## 统计线程消耗内存
thread_id就是show processlist的thread_id,如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计
```
mysql> select thread_id, event_name, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_thread_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;
```

## 统计账户消耗内存
如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计
```
mysql> select USER, HOST, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_account_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;
```

## 统计主机消耗内存
如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计
```
mysql > select HOST, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_host_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;
```

## 统计用户消耗内存
如果SUM_NUMBER_OF_BYTES_ALLOC为零就代表没开启统计
```
mysql> select USER, EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED,SUM_NUMBER_OF_BYTES_ALLOC from performance_schema.memory_summary_by_user_by_event_name order by CURRENT_NUMBER_OF_BYTES_USED desc limit 10;
```

## 附：MySQL内存使用拆分
![](http://codercoder.cn/wp-content/uploads/2022/08/2022-08-22-mysql-get-memory-status-from-performance-schema.png)