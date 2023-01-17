---
id: 667
title: '项目重启后, Mybatis报错org.apache.ibatis.ognl.NoSuchPropertyException分析'
date: '2019-09-20T14:13:03+08:00'
author: Smile
excerpt: Mybatis报错org.apache.ibatis.ognl.NoSuchPropertyException分析
layout: post
guid: 'http://codercoder.cn/?p=667'
permalink: /index.php/2019/09/mybatis-nosuchpropertyexception/
views:
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
    - '133210'
bigfa_ding:
    - '1'
image: /wp-content/uploads/2019/09/2019-09-2054-320x110.png
categories:
    - Tech
    - Fell-in-pit
tags:
    - MyBatis
    - Tech
    - Fell-in-pit
---

##### Mybatis报错org.apache.ibatis.ognl.NoSuchPropertyException分析

###### 日志报错信息

org.apache.ibatis.ognl.NoSuchPropertyException: XxxExample&Criterion.condition  
或者  
org.apache.ibatis.ognl.NoSuchPropertyException: XxxExample&Criterion.value  
或者  
org.apache.ibatis.ognl.NoSuchPropertyException: XxxExample&Criterion.noValue等  
这些错误都是项目发布重启之后可能会出现。

###### 问题分析

查看报错信息+debug调试  
一. 获取某个对象的某个字段值流程如下  
![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-2084.png)  
![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-2058.png)

从图中可以看出，getProperty方法只有在获取不到值的情况下才会抛出上面的异常，也就是说只有存在OgnlRuntime#getMethodValue为空的时候发生（OgnlRuntime#getFieldValue方法在上面的举例中是肯定获取不到的，因为condition是类的私有变量，这个方法只能获取到非私有变量的值）

二.现在就来看什么情况下会导致OgnlRuntime#getMethodValue返回null  
![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-2072.png)

在这里同一同样有两种情况，getGetMethod返回为null，并且getReadMethod返回也为null的时候才会返回null。由于在这里传入给getReadMethod的numParms参数是0，导致getReadMethod里面根本也获取不到任何值（方法实现里只判断了大于小于0，没有等于0场景），所以问题只能出现在getGetMethod返回null时候

三.继续查看OgnlRuntime#getGetMethod什么情况下会返回null  
![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-202.png)

跟到这个方法可以看出，这个方法是存在并发问题的，线程非安全。项目重启后第一次查询当两个线程同时值了行了A处从缓存里面获取字段对应的method都为null，然后另一个线程去执行了C处将method放到了缓存，另一个线程此时去执行B处代码时候，判断了如果包含了这个key就返回null了。

###### 总结

mybatis在版本>=3.4.1之后修复了这个问题，主要修改是改动了上面提到的OgnlRuntime#getReadMethod这个方法的实现，即使第三段中提到的并发下依旧会返回null，但是之后可以通过这个方法重新能得到。