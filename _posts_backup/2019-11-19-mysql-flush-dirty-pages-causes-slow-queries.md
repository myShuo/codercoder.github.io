---
id: 810
title: MySQL刷脏让应用抖了一下？
date: '2019-11-19T20:58:54+08:00'
author: Shuo
excerpt: 对于MySQL的刷脏，应当调整数据库的相关配置，使该过程平滑进行，不要影响到业务。在本片文章中，介绍了排查的思路，若在日志中看到刷脏耗时较长，且导致了慢查，可参考本文的思路进行排查。
layout: post
guid: 'http://codercoder.cn/?p=810'
permalink: /index.php/2019/11/mysql-flush-dirty-pages-causes-slow-queries/
views:
    - '3183'
    - '3183'
    - '3183'
bigfa_ding:
    - '2'
categories:
    - Fell-in-pit
tags:
    - IO
    - MySQL
    - Fell-in-pit
---

[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0899-1-300x208.jpg)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0899-1.jpg)

# 数据库突然出现大量慢查

近期应用反馈出现连接被数据库断开的情况，查看应用日志，超过了jdbc的连接时间，没有返回结果，所以应该是慢查导致。但是为什么会突然出现耗时较长的慢查呢？

# 问题追溯

对照应用的报错时间点，看到在MySQL的错误日志中有如下：

> InnoDB: page\_cleaner: 1000ms intended loop took 27154ms. The settings might not be optimal. (flushed=27 and evicted=0, during the time.)

所以是在那个时间的时候，数据库在进行刷脏操作，并且持续了27154ms，page\_cleaner\_thread接受脏页的数量远远大于它每秒能够处理脏页的能力，所以应用在这个期间内出现了慢查的情况。

# How to fix?

既然找到了原因，那么就想怎么去规避？ 首先想到的是对于刷脏的控制，查看相关的数据库配置：

```
innodb_buffer_pool_instances  2
innodb_lru_scan_depth  1024
innodb_max_dirty_pages_pct  75
innodb_max_dirty_pages_pct_lwm  0
innodb_io_capacity  20000
innodb_io_capacity_max  40000
innodb_page_size  16384                             

```

## 优化刷新相关配置：

**1.innodb\_buffer\_pool\_instances**  
通常我们配置和innodb\_buffer\_pool\_instances配置一样。但是在本环境中， 配置为一样的，所以排除这个因素。

**2.innodb\_lru\_scan\_depth**  
由于此次刷脏，消耗了大量的io资源，导致大量慢查，所以需要进行优化，以缓解刷脏的消耗，用时间换资源：  
innodb\_lru\_scan\_depth=1024，默认值较大，若配置较低，或者磁盘的性能不是很高，则可以设置为较小的值  
官方文档描述：  
A parameter that influences the algorithms and heuristics for the flush operation for the InnoDB buffer pool. Primarily of interest to performance experts tuning I/O-intensive workloads. It specifies, per buffer pool instance, how far down the buffer pool LRU page list the page cleaner thread scans looking for dirty pages to flush. This is a background operation performed once per second.

控制LRU列表中可用页的数量，从LRU上清理block并将其放到free list上的扫描深度。  
**所以可以降低该值，设置innodb\_lru\_scan\_depth=256；**

**3.innodb\_max\_dirty\_pages\_pct**  
脏页比例，75%为默认值，由于此次刷脏的主要原因更多是在io上，所以暂时不变更这个参数

**4.innodb\_io\_capacity**  
官方：  
The innodb\_io\_capacity variable defines the number of I/O operations per second (IOPS) available to InnoDB background tasks, such as flushing pages from the buffer pool and merging data from the change buffer.  
该值为InnoDB后台进程（例如从缓冲池刷脏、merge change buffer）能够使用的IOPS的大小。

该实例的配置为20000，根据现状来看，业务在刷脏期间出现大量慢查，所以判断SSD的IOPS不足以分配20000给InnoDB的后台刷脏线程。  
（由于使用的是tx云数据库，厂商未提供IOPS的监控，所以不能得到确切的值）  
调整innodb\_io\_capacity=2000

# 后记

对于刷脏，应当调整数据库的相关配置，使该过程平滑进行，不要影响到业务。在本片文章中，介绍了排查的思路，若在日志中看到刷脏耗时较长，且导致了慢查，可参考本文的思路进行排查。