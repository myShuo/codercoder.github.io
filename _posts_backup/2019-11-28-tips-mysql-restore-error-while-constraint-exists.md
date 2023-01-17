---
id: 820
title: 'Tips: MySQL导入备份sql文件时，外键约束报错'
date: '2019-11-28T17:27:03+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=820'
permalink: /index.php/2019/11/tips-mysql-restore-error-while-constraint-exists/
views:
    - '1951'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - 备份恢复
    - 外键
    - Fell-in-pit
---

# 现象

最近在导入一个环境的数据的时候，出现了这个报错，备份为使用mysqldump进行的。  
在导入新环境时，提示：**Fail to open the referenced table ….**  
![](http://codercoder.cn/wp-content/uploads/2019/11/b57a8a3d542efea23789889665436dcb.png)

# 原因

根据报错是外键的问题，此时，由于sql文件中建表的先后顺序，导致打不开需要关联的其它表，会报错。但是由于此阶段还在初始化，所以可以暂时忽略。在导入结束后再去检查外键情况。

# 解决

操作如下，首先创建一个文件，第一行为：

```
echo "set FOREIGN_KEY_CHECKS=0; " > testdb_dump.sql 
mysqldump  -B testdb  --compact  >> testdb_dump.sql 
echo "set FOREIGN_KEY_CHECKS=1; " > testdb_dump.sql 

```

# 注意

不要直接vi编辑sql文件，因为如果备份文件过大，则可能会打不开文件，且消耗系统的IO资源。