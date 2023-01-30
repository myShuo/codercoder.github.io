---
id: 1194
title: 'Redis集群中slave漂移的问题'
date: '2021-12-24T18:34:40+08:00'
author: Shuo
excerpt: 在shutdown集群中某一个master的时候，集群中其它master的slave，会自动迁移到新的主上。
layout: post
guid: 'http://codercoder.cn/?p=1194'
image: http://codercoder.cn/wp-content/uploads/2021/12/2021-12-24-redis-replica-migration.png
permalink: /index.php/2021/12/2021-12-24-redis-replica-migration
categories:
    - Tech
tags:
    - Redis
    - Tech
---

参考：[cluster-tutorial](https://redis.io/topics/cluster-tutorial)

**现象：**

在shutdown集群中某一个master的时候，集群中其它master的slave，会自动迁移到新的主上

**原因：**

自动均衡

![](http://codercoder.cn/wp-content/uploads/2021/12/2021-12-24-redis-replica-migration.png)