---
id: 808
title: MySQL8.0迁移到5.7中的正则表达REGEXP坑
date: '2019-11-19T20:06:36+08:00'
author: Shuo
excerpt: 'MySQL从8.0.4开始支持International Components for Unicode(ICU)包，在此版本之前，用的是Henry Spencer''s引擎，所以在不通版本数据库中使用正则表达，可能会出现不通结果。'
layout: post
guid: 'http://codercoder.cn/?p=808'
permalink: /index.php/2019/11/the-regexp-difference-between-mysql57-and-80/
views:
    - '7175'
    - '7175'
bigfa_ding:
    - '1'
categories:
    - Fell-in-pit
tags:
    - MySQL
    - 正则表达式
    - Fell-in-pit
---

# 查不到数据？

最近在迁移一个MySQL的环境，出了基本的默认字符集相关的一些差异可以人为预见，迁移过程未出现报错，迁移之后，业务同事反馈说部分应用查不到数据。

反查了一下数据库，数据都是一致的，所以让业务打印出相应的日志。查看里面有这么一条sql：

```
select   c1, c2 from t1 where REGEXP '\\btest@111.com\\b'

```

在日志中拎出sql，手动分别在5.7和8.0上执行：

```
5.7：
Empty set (0.00 sec)

8.0:
…
1 row in set (0.00 sec)

```

所以猜想就是数据库版本不一致，导致的查询数据失败，查看8.0版本的官方文档：  
https://dev.mysql.com/doc/refman/8.0/en/regexp.html

> MySQL implements regular expression support using International Components for Unicode (ICU), which provides full Unicode support and is multibyte safe. (Prior to MySQL 8.0.4, MySQL used Henry Spencer’s implementation of regular expressions, which operates in byte-wise fashion and is not multibyte safe. For information about ways in which applications that use regular expressions may be affected by the implementation change, see Regular Expression Compatibility Considerations.)
> 
>  MySQL从8.0.4开始支持ICU包，在此版本之前，用的是Henry Spencer’s引擎，所以本次遇到的情况，恰还是由于两种正则表达式的规则不一致，所以在5.7版本查询不到数据。

ICU相关正则说明：http://userguide.icu-project.org/strings/regexp  
Spencer’s引擎正则介绍：https://www.codeproject.com/Articles/4421/Henry-Spencer-s-Regexp-Engine-Revisited

# 问题fix:

**对于8.0：\\b为单词边界的标识**  
**对于5.7：\\b为空格的标识**  
![](http://codercoder.cn/wp-content/uploads/2019/11/8a29327601ee2869805d9baf5df90231.png)  
所以在5.7中，原sql返回为空。  
![](http://codercoder.cn/wp-content/uploads/2019/11/8a29327601ee2869805d9baf5df90231.png)

在原8.0环境，由于sql语法为 **\\b** ，即想要匹配 test@111.com 两边的单词边界，查阅表中的数据，数据之间是使用分号;进行分隔的，所以在5.7版本中，换成：

```
select   c1, c2 from t1 where REGEXP ';test@111.com;'

```

也可以直接使用like语法：

```
select  c1, c2 from t1 where like '%test@111.com%'

```

# 小感想

MySQL8.0支持了更为丰富，也更加规范的正则规则，更加便于开发人员。同样DBA在处理正则匹配的sql时候，应当考虑版本的问题，还有查询性能。