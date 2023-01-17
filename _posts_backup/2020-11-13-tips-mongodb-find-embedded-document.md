---
id: 1193
title: 'Tips：MongoDB中的embedded document（嵌套文档）查询注意事项'
date: '2020-11-13T18:34:40+08:00'
author: Shuo
excerpt: MongoDB查询嵌套的文档时候，字段顺序需要与存储时候的保持一致，否则会查询不到数据.
layout: post
guid: 'http://codercoder.cn/?p=1193'
permalink: /index.php/2020/11/tips-mongodb-find-embedded-document/
views:
    - '552'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MongoDB
    - Tech
---

# 背景

最近在查询MongoDB中的数据时，发现查询不到数据，查询语句为：db.person.find({“name”:{  
 last: “Matsumoto”,  
 first: “Yukihiro”  
}})  
数据库中数据为：

```
{
    "_id" : <value>,
    "name" : { "first" : <string>, "last" : <string> },       // embedded document
    "birth" : <ISODate>,
    "death" : <ISODate>,
    "contribs" : [ <string>, ... ],                           // Array of Strings
    "awards" : [
        { "award" : <string>, year: <number>, by: <string> }  // Array of embedded documents
        ...
    ]
}

```

# 问题定位

交换“name”中的属性顺序，发现必须与数据库中的一致，才能查询到数据～

## 官方文档

![](http://codercoder.cn/wp-content/uploads/2020/11/2020-11-1315.png)  
**The name field must match the embedded document exactly.**

果然，在查询嵌套的文档时候，字段顺序需要与存储时候的保持一致，否则会查询不到数据。  
https://docs.mongodb.com/manual/reference/method/db.collection.find/