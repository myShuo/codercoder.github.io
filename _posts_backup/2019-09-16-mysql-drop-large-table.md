---
id: 621
title: 'MySQL安全删除大表drop table'
date: '2019-09-16T17:55:22+08:00'
author: Shuo
excerpt: 在删除过大的表时，除了会导致DML和DDL阻塞之外，还会产生大量的IO，造成资源占用严重，影响正常的业务。所以在生产环境中，我们需要将删除的过程影响降到最低。
layout: post
guid: 'http://codercoder.cn/?p=621'
permalink: /index.php/2019/09/mysql-drop-large-table/
views:
    - '7503'
bigfa_ding:
    - '2'
categories:
    - Tech
tags:
    - MySQL
    - Fell-in-pit
---

# 情况描述

## 背景

 在删除过大的表时，除了会导致DML和DDL阻塞之外，还会产生大量的IO，造成资源占用严重，影响正常的业务。

## 前提

 linux操作系统，MySQL数据库为独立表空间

## 思路

 在独立表空间环境中，删除一个表体现在OS上则为删除一个表空间文件。所以在OS上使用硬链接，在删除表时，实际是删除硬链接，该操作能迅速完成，然后再在系统上进行文件的逐步删除

# 删除大表步骤

## 建立硬链接

在OS上建立硬链接，*例如表**testbigfile***，则在datadir下对应位置：

```
ln testbigfile.ibd testbigfile.ibd.rm

```

 通过 **ll** 显示的第二列可以看到有两个文件使用了这个链接，所以删除其中一个时，删除的只是链接（会使得MySQL删表时速度更快），**只有在只有一个文件使用链接时，才是真正的删除。**

linux命令行 **ll -a** 显示的内容：  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1673.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1673.png)

# 在MySQL中进行表的删除

```
mysql> drop table bigtbdrop;

```

该步操作非常快，是因为删除的是硬链接，而不是真正的文件。

# 在操作系统删除硬链接文件

 由于突然删除上百GB的文件，同样会导致IO的占用，所以在Linux环境可以使用**truncate**工具进行递进删除，一次删除1GB；更好的做法是统计当前的IO使用资源，在空闲IO资源较多时，触发truncate的操作：

```
truncate -s -1GB regtime.ibd

```

> -s -1GB: 一次删除1GB，直到删完为止

同样可以通过**truncate –help**查看命令的帮助。

至此，则可安全地删除大表，妈妈再也不用担心我的***drop table***啦~