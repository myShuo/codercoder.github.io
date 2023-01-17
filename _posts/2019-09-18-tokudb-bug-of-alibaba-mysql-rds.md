---
id: 659
title: 阿里云RDS中MySQL实例TokuDB的BUG
date: '2019-09-18T10:42:13+08:00'
author: Shuo
excerpt: 在进行表结构变更时，发现一个存储引擎为TokuDB的表变更字段花费了10个小时。但是对于TokuDB，字段变更应该是秒级完成的。至此向阿里云提交了bug。
layout: post
guid: 'http://codercoder.cn/?p=659'
permalink: /index.php/2019/09/tokudb-bug-of-alibaba-mysql-rds/
views:
    - '4956'
    - '4956'
bigfa_ding:
    - '1'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - Fell-in-pit
---

[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1857-150x150.jpg)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1857.jpg)

# **背景：**

近日在进行表结构变更时，发现一个存储引擎为TokuDB的表变更字段花费了10个小时。但是对于TokuDB，字段变更应该是秒级完成的。至此向阿里云提交了bug。

# **环境：**

阿里云RDS for MySQL 5.6版本  
表的存储引擎：TokuDB

# **回复：**

阿里云目前不再进行TokuDB的更新和维护，所以建议不再使用该存储引擎。

# **相似bug：**

[tokudb-hot-column-expansion](https://dba.stackexchange.com/questions/144919/tokudb-hot-column-expansion)  
[\[bugfix\] Issue: #62 alter TokuDB table comment rebuild whole engine data](https://jira.mariadb.org/browse/MDEV-19107)

# 规避措施：

（1）不使用TokuDB存储引擎  
（2）监控结构变更的状态，控制执行时间，若某个时间内没有执行成功，则kill变更线程  
（3）使用pt-online-schema-change进行结构变更