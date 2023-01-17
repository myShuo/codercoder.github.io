---
id: 930
title: 'MySQL手记8 — adminMongo与Mongo-express对比（结果格式差异）'
date: '2020-04-15T21:12:27+08:00'
author: Shuo
excerpt: adminMongo和Mongo-express两者都可以显示数据，就是展示的格式不同。
layout: post
guid: 'http://codercoder.cn/?p=930'
image: http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1525.png
permalink: /index.php/2020/04/mysql-note8-comparing-adminmongo-and-mongo-express-data-format/
views:
    - '4776'
bigfa_ding:
    - '1'
categories:
    - Tech
    - Fell-in-pit
tags:
    - MySQL手记
    - Tech
    - Fell-in-pit
---

# 一、adminMongo & Mongo-express

## 介绍：

```
    adminMongo（https://github.com/mrvautin/adminMongo）和Mongo-express（https://github.com/mongo-express/mongo-express）都是非常简单好用的MongoDB可视化展示的工具，在平常的调试或者排查问题中会使用到。

```

## 1.1 adminMongo安装

### （1）使用git拷贝仓库

```
git clone https://github.com/mrvautin/adminMongo

```

### （2）安装

```
cd adminMongo/
npm install

```

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1525.png)

### （3）启动

```
 npm start

```

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1571.png)  
可以看出，adminMongo的安装及启动都是非常的方便，启动后，直接访问http://localhost:1234 即可。

## 1.2 Mongo-express安装

```
    可使用docker进行快速便捷的安装：

```

### （1）编辑docker启动mongo-express所需的配置文件

```
cat start.yml
version: '3.1'

services:
  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8081:8081
    environment:
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: admin
      ME_CONFIG_MONGODB_ADMINUSERNAME: mongoadmin
      ME_CONFIG_MONGODB_ADMINPASSWORD: mongoroot
      ME_CONFIG_MONGODB_ENABLE_ADMIN: 'true'
      ME_CONFIG_MONGODB_PORT: 27017
      ME_CONFIG_MONGODB_SERVER: 127.0.0.1

```

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1549.png)

### （2）使用docker安装及启动

```
docker-compose -f start.yml up

```

# 二、两者的差异

```
    近日在查看MongoDB的数据时，发现得到的结果格式不一致，猜想是MongoDB客户端展示的问题，显示如下：

```

## adminMongo：

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1586.png)

### Mongo-express:

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1585.png)

```
插入的时候，插入的是一个kv键值对，其中value为json格式。

```

 可以看出，对于MongoDB中的document，adminMongo显示为一个包含“\_id”的JSON串（并且把原来的JSON串的双引号进行了转义），mongo-express则是展示这个document中的键值对，并保留插入时候的JSON。

 两者都可以显示数据，就是展示的格式不同，可以拿到json.cn进行查看：  
 adminMongo：需要先转换一次，再把去除了双引号的value提出来（json串）  
 mongo-express：删除key，直接使用value（json串）进行查看。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)