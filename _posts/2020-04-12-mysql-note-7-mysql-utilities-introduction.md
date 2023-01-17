---
id: 906
title: 'MySQL手记7 &#8212; MySQL Utilities工具包'
date: '2020-04-12T20:51:38+08:00'
author: Shuo
excerpt: '从安装过程打印的信息来看，MySQL Utilities工具包是使用Python进行开发的，并在安装过程把python脚本复制在/use/bin目录下，新增了多个mysql*开头的可执行文件，这些文件，就是Utilities中的工具。'
layout: post
guid: 'http://codercoder.cn/?p=906'
image: http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1245.png
permalink: /index.php/2020/04/mysql-note-7-mysql-utilities-introduction/
views:
    - '2531'
    - '2531'
    - '2531'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、MySQL Utilities简介

 MySQL Utilities是MySQL官方的数据库工具包，下载地址为：https://downloads.mysql.com/archives/utilities/。 该工具包主要是给DBA提供了在运维上可能需要的数据库工具。  
（官方文档：https://downloads.mysql.com/docs/mysql-utilities-1.6-en.pdf）  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1245.png)  
 下载后，解压得到如下目录：

```
build  
CHANGES.txt  
docs  
info.py  
info.pyc  
LICENSE.txt  
mysql  
PKG-INFO  
README.txt  
scripts  
setup.py  
unit_tests

```

 执行python setup.py install：

```
......
running install_scripts
copying build/scripts-2.7/mysqlfrm -> /usr/bin
copying build/scripts-2.7/mysqlgrants -> /usr/bin
copying build/scripts-2.7/mysqlbinlogmove -> /usr/bin
copying build/scripts-2.7/mysqldbexport -> /usr/bin
copying build/scripts-2.7/mysqluc -> /usr/bin
copying build/scripts-2.7/mysqlrplshow -> /usr/bin
copying build/scripts-2.7/mysqlrplsync -> /usr/bin
copying build/scripts-2.7/mysqldbcopy -> /usr/bin
copying build/scripts-2.7/mysqlindexcheck -> /usr/bin
copying build/scripts-2.7/mysqlauditgrep -> /usr/bin
copying build/scripts-2.7/mysqlbinlogpurge -> /usr/bin
copying build/scripts-2.7/mysqldbimport -> /usr/bin
copying build/scripts-2.7/mysqldiskusage -> /usr/bin
copying build/scripts-2.7/mysqlfailover -> /usr/bin
copying build/scripts-2.7/mysqlrplcheck -> /usr/bin
copying build/scripts-2.7/mysqlreplicate -> /usr/bin
copying build/scripts-2.7/mysqlmetagrep -> /usr/bin
copying build/scripts-2.7/mysqlauditadmin -> /usr/bin
copying build/scripts-2.7/mysqlslavetrx -> /usr/bin
copying build/scripts-2.7/mysqlrplms -> /usr/bin
copying build/scripts-2.7/mysqldbcompare -> /usr/bin
copying build/scripts-2.7/mysqluserclone -> /usr/bin
copying build/scripts-2.7/mysqlserverinfo -> /usr/bin
copying build/scripts-2.7/mysqlserverclone -> /usr/bin
copying build/scripts-2.7/mysqldiff -> /usr/bin
copying build/scripts-2.7/mysqlbinlogrotate -> /usr/bin
copying build/scripts-2.7/mysqlprocgrep -> /usr/bin
copying build/scripts-2.7/mysqlrpladmin -> /usr/bin
changing mode of /usr/bin/mysqlfrm to 755
changing mode of /usr/bin/mysqlgrants to 755
changing mode of /usr/bin/mysqlbinlogmove to 755
changing mode of /usr/bin/mysqldbexport to 755
changing mode of /usr/bin/mysqluc to 755
changing mode of /usr/bin/mysqlrplshow to 755
changing mode of /usr/bin/mysqlrplsync to 755
changing mode of /usr/bin/mysqldbcopy to 755
changing mode of /usr/bin/mysqlindexcheck to 755
changing mode of /usr/bin/mysqlauditgrep to 755
changing mode of /usr/bin/mysqlbinlogpurge to 755
changing mode of /usr/bin/mysqldbimport to 755
changing mode of /usr/bin/mysqldiskusage to 755
changing mode of /usr/bin/mysqlfailover to 755
changing mode of /usr/bin/mysqlrplcheck to 755
changing mode of /usr/bin/mysqlreplicate to 755
changing mode of /usr/bin/mysqlmetagrep to 755
changing mode of /usr/bin/mysqlauditadmin to 755
changing mode of /usr/bin/mysqlslavetrx to 755
changing mode of /usr/bin/mysqlrplms to 755
changing mode of /usr/bin/mysqldbcompare to 755
changing mode of /usr/bin/mysqluserclone to 755
changing mode of /usr/bin/mysqlserverinfo to 755
changing mode of /usr/bin/mysqlserverclone to 755
changing mode of /usr/bin/mysqldiff to 755
changing mode of /usr/bin/mysqlbinlogrotate to 755
changing mode of /usr/bin/mysqlprocgrep to 755
changing mode of /usr/bin/mysqlrpladmin to 755
......


```

 从安装过程打印的信息来看，该工具包是使用Python进行开发的，并在安装过程把python脚本复制在/use/bin目录下，新增了多个mysql\*开头的可执行文件，这些文件，就是Utilities中的工具。后续将逐步介绍常用的集中工具：

 和pt工具相比，两个工具包相辅相成，虽说在实际工作中，pt工具用得更多，但是，在部分场景，MySQL Utilities也可以发挥其的作用，灵活运用好这两个工具包，可以让工作更加得心应手。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)