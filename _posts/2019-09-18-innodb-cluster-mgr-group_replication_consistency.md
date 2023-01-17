---
id: 633
title: 'InnoDB Cluster(MGR)中的事务一致性等级配置'
date: '2019-09-18T09:25:45+08:00'
author: Shuo
excerpt: 'InnoDB cluster是基于MySQL Group Replication（MGR）搭建的，本文主要介绍在MySQL-8.0.14中新增的一致性配置参数：group_replication_consistency，用以控制在InnoDB cluster中的事务一致性等级。'
layout: post
guid: 'http://codercoder.cn/?p=633'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1861.png
permalink: /index.php/2019/09/innodb-cluster-mgr-group_replication_consistency/
views:
    - '4449'
    - '4449'
bigfa_ding:
    - '2'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

# 简介

 InnoDB cluster是基于MySQL Group Replication（MGR）搭建的，本文主要介绍在MySQL-8.0.14中新增的一致性配置参数：**group\_replication\_consistency**<sup id="fnref-633-1">[1](#fn-633-1)</sup>，用以控制在InnoDB cluster中的事务一致性等级，其有5种可选配置项：**EVENTUAL**(默认)、**BEFORE\_ON\_PRIMARY\_FAILOVER**、**BEFORE**、**AFTER**、**BEFORE\_AND\_AFTER**。此外，对于不同的事务（只读，或者读写），进行不同的一致性控制。本文的例子均基于“单写”（Single-primary mode）cluster环境。  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1861.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1861.png)

# group\_replication\_consistency配置

有如下几种可配选项，根据一致性等级程度增大顺序进行介绍：

## EVENTUAL

在执行”读”或者”写”事务之前，不用等待之前的事务完成。在group\_replication\_consistency参数出现之前，默认是使用此种方法。  
例如：

```
Secondary master上：    lock table t1 read;
Primary 上：            insert into t1 values (***);    ----执行成功，不会产生等待
Secondary master上：    select * from t1;    ----执行成功，返回结果为插入前的状态

Secondary master上：    unlock tables;  
Secondary master上：    select * from t1;    ----执行成功，返回结果为插入后的状态

```

## BEFORE\_ON\_PRIMARY\_FAILOVER

新primary上的RO或RW事务，在恢复老的primary的待办事务之前，将会被阻塞。 这可确保在primary发生故障转移发生时，客户端始终会在primary服务器上看到最新值。 这保证了一致性，但意味着客户端必须能够在应用积压的情况下处理延迟。 通常这种延迟应该是最小的，但取决于积压的任务大小。

## BEFORE

RW事务在执行之前等待所有先前的事务完成；RO事务在返回结果前需要等待所有先前的事务完成(Apply Queue为空才能返回结果)。这样可以确保RO返回的结果，为最新的状态。  
例如：

```
Secondary master上：    lock table t1 read;
Primary 上：            insert into t1 values (***);    ----执行成功，不会产生等待
Secondary master上：   
    set @@session.group_replication_consistency='BEFORE' ; 
    select * from t1;    ----等待，若超过“wait_timeout”的阈值，则报错：ERROR: 3797: Error while waiting for group transactions commit on group_replication_consistency= 'BEFORE'

Secondary master上：    unlock tables;   ----执行成功。此时Secondary master上的select线程返回插入后的结果
Secondary master上：    select * from t1;    ----执行成功，返回结果为插入后的状态

```

## AFTER

RW事务需要等待其在所有节点上的变更结束，才能执行成功。“AFTER”不影响只读事务。  
例如：

```
Secondary master上：    lock table t1 read;
Primary 上：     
    set @@session.group_replication_consistency='AFTER' ;        
    insert into t1 values (***);    ----等待，测试发现即使等待时间超过“wait_timeout”的阈值，也不会报错
Secondary master上：   
    select * from t1;    ----等待，若超过“wait_timeout”的阈值，则报错：ERROR: 3797: Error while waiting for group transactions commit on group_replication_consistency= 'BEFORE'

Secondary master上：    unlock tables;   ----执行成功。此时Primary 上的insert 执行成功
Secondary master上：    select * from t1;    ----执行成功，返回结果为插入后的状态

```

## BEFORE\_AND\_AFTER

RW事务等待之前所有的事务完成，并且等待其在所有节点上的变更结束；RO事务需要等待之前所有的事务完成。如上文“AFTER”部分的测试结果。  
（本文测试参考MySQL InnoDB Cluster – consistency levels<sup id="fnref-633-2">[2](#fn-633-2)</sup>）

# 总结

由此可见，在8.0.14后，提供了更为准确的数据一致性控制：  
1\. 相比之前默认的EVENTUAL，即保持最终的一致性，配置不同的选项，可以更好的控制在primary宕机时，能保证RO和RW事务的准确性。

```
**EVENTUAL**在Primary宕机后，在新的Primary重放完老primary的事务之前，若此时有RO事务，则会导致RO事务读取到老的数据；而RW事务则可能由于冲突被rollback，导致数据不一致。

```

2\. 当然，很多小伙伴会想，那么把所有节点都配置为“AFTER”不就好了吗？

```
事实并非如此，因为“AFTER”很有可能导致：当Primary上有大量的变更、DDL、备份等操作时，commit被延迟，并且延迟可能会导致独有的业务全部需要连接到Primary。

```

<div class="footnotes" role="doc-endnotes">- - - - - -

1. [官方文档：group\_replication\_consistency](https://dev.mysql.com/doc/refman/8.0/en/group-replication-options.html#sysvar_group_replication_consistency) [↩︎](#fnref-633-1)
2. [MySQL InnoDB Cluster – consistency levels](https://lefred.be/content/mysql-innodb-cluster-consistency-levels/) [↩︎](#fnref-633-2)

</div>