---
id: 1198
title: 'Redis ERR Protocol error: too big inline request'
date: '2022-12-01T18:34:40+08:00'
author: Shuo
excerpt: 慢查较多的时候，慢查日志需要进行定期切换，防止需要分析时，文件过大不便于排查。
layout: post
guid: 'http://codercoder.cn/?p=1198'
permalink: /index.php/2022/12/2022-12-01-flush-slow-log
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、目的
慢查较多的时候，慢查日志需要进行定期切换，防止需要分析时，文件过大不便于排查。

# 二、步骤
（1）备份当前的slow log
```
mv mysql-slow.log mysql-slow.log-20221202
```
**注意：**

此时，如果继续产生慢查，虽然已经更改了文件名，但是会继续往mysql-slow.log-20221202文件中追加慢查日志。

（2）切换慢查日志
```
flush slow logs;
```
此时，新增的日志将写入到一个新建的“mysql-slow.log”文件中。