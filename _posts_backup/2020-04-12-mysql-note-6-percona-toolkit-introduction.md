---
id: 900
title: 'MySQL手记6 &#8212; percona-toolkit工具包'
date: '2020-04-12T20:40:57+08:00'
author: Shuo
excerpt: 对于经常用到的工具，例如使用pt-archiver进行数据的归档、使用pt-online-schema-change进行表结构的变更、使用pt-table-checksum对比两个表的checksum是否一致等等，都能灵活的进行。
layout: post
guid: 'http://codercoder.cn/?p=900'
permalink: /index.php/2020/04/mysql-note-6-percona-toolkit-introduction/
views:
    - '7206'
    - '7206'
    - '7206'
    - '7206'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、percona-toolkit工具包介绍

 **percona-toolkit（下称：pt工具）**，是Percona公司开发的用于数据库的工具包，品类很多，再次就不多做介绍，灵活的使用pt工具，可以极大的简化DBA日常的运维工作，提高工作效率。  
 *（在日常工作中，其实常常用到pt的工具，公司对工具进行了封装，做成为直观的工具）*  
 可在其网站上进行查看：https://www.percona.com/downloads/percona-toolkit/LATEST/  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1234.png)  
**pt工具包可大致分为以下几类：**  
**（1）开发类工具：**  
 pt-duplicate-key-checker  
 pt-online-schema-change  
 pt-query-advisor  
 pt-show-grants  
 pt-upgrade  
**（2）性能类工具：**  
 pt-index-usage  
 pt-pmp  
 pt-visual-explain  
（3）配置类工具：  
 pt-config-diff  
 pt-mysql-summary  
 pt-variable-advisor  
**（4）监控类工具：**  
 pt-deadlock-logger  
 pt-fk-error-logger  
 pt-mext  
 pt-query-digest  
 pt-trend  
**（5）复制类工具：**  
 pt-heartbeat  
 pt-slave-delay  
 pt-slave-find  
 pt-slave-restart  
 pt-table-checksum  
 pt-table-sync  
**（6）系统类工具：**  
 pt-diskstats  
 pt-fifo-split  
 pt-summary  
 pt-stalk  
**（7）实用类工具：**  
 pt-archiver  
 pt-find  
 pt-kill

# 二、安装

 在官网下载安装包后，解压，得到如下目录及文件：

```
bin        
CONTRIBUTE.md    
COPYING             
docs        
Gopkg.toml  
lib          
MANIFEST
Changelog  
CONTRIBUTING.md  
docker-compose.yml  
Gopkg.lock  
INSTALL     
Makefile.PL  
README.md

```

 其中bin目录下，即为我们所需要的可执行文件：

|  |  |
|:--|:--|
| pt-align | pt-fk-error-logger |
| pt-online-schema-change | pt-slave-restart |
| pt-archiver | pt-heartbeat |
| pt-pg-summary | pt-stalk |
| pt-config-diff | pt-index-usage |
| pt-pmp | pt-summary |
| pt-deadlock-logger | pt-ioprofile |
| pt-query-digest | pt-table-checksum |
| pt-diskstats | pt-kill |
| pt-secure-collect | pt-table-sync |
| pt-duplicate-key-checker | pt-mext |
| pt-show-grants | pt-table-usage |
| pt-fifo-split | pt-mongodb-query-digest |
| pt-sift | pt-upgrade |
| pt-find | pt-mongodb-summary |
| pt-slave-delay | pt-variable-advisor |
| pt-fingerprint | pt-mysql-summary |
| pt-slave-find | pt-visual-explain |

 对于经常用到的工具，例如使用pt-archiver进行数据的归档、使用pt-online-schema-change进行表结构的变更、使用pt-table-checksum对比两个表的checksum是否一致等等，都能灵活的进行，并且符合运维的思维，添加了很多选项，让我们能够精准的控制整个维护的过程。（后续将介绍常用的几种工具的使用场景）​  
欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)