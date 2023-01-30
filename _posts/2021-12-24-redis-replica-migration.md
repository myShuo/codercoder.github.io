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

With replicas migration what happens is that if a master is left without replicas, a replica from a master that has multiple replicas will migrate to the orphaned master. So after your replica goes down at 4am as in the example we made above, another replica will take its place, and when the master will fail as well at 5am, there is still a replica that can be elected so that the cluster can continue to operate.

So what you should know about replicas migration in short?

* The cluster will try to migrate a replica from the master that has the greatest number of replicas in a given moment.
* To benefit from replica migration you have just to add a few more replicas to a single master in your cluster, it does not matter what master.
* There is a configuration parameter that controls the replica migration feature that is called cluster-migration-barrier: you can read more about it in the example redis.conf file provided with Redis Cluster.


![](http://codercoder.cn/wp-content/uploads/2021/12/2021-12-24-redis-replica-migration.png)