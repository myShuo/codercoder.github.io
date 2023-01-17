---
id: 536
title: 'MySQL使用原生OnlineDDL进行结构变更报错：The total number of locks exceeds the lock table size'
date: '2019-09-10T17:08:46+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=536'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1036.png
permalink: /index.php/2019/09/mysql-onlineddl-error-the-total-number-of-locks-exceeds-the-lock-table-size/
views:
    - '692'
    - '692'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

# 情况介绍

阿里云RDS MySQL5.6在使用原生OnlineDDL进行结构变更时报错：

```
ERROR 1206 (HY000): The total number of locks exceeds the lock table size，

```

在错误日志中：

```
=====================================

InnoDB: WARNING: over 67 percent of the buffer pool is occupied by

lock heaps or the adaptive hash index! Check that your

transactions do not set too many row locks.

Your buffer pool size is 64 MB. Maybe you should make the buffer pool bigger?

Starting the InnoDB Monitor to print diagnostics, including lock heap and hash index sizes.

......

```

# 问题排查

## 1. 确认环境参数：

查看当前的配置：**max\_write\_lock\_count =102400**，该参数在MySQL5.6 64位的环境默认值为18446744073709551615 ~

## 2.解决

发现这个参数配置得太小了，但是阿里云RDS默认权限不允许大于102400，所以采用pt-online-schema-change进行。

> 若为自建的MySQL Server，可以配置：  
>  set global max\_write\_lock\_count =18446744073709551615

在MySQL官方文档中，对于该参数的说明：  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1036.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1036.png)