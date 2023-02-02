---
id: 1201
title: '使用pt-online-schema-change添加唯一索引导致数据丢失'
date: '2022-05-31T18:34:40+08:00'
author: Shuo
excerpt: '由于pt-osc（pt-online-schema-change）在进行ddl表更时候，步骤中insert的语法为insert ignore，所以会导致在对于表中加唯一索引时候，重复的数据会被丢弃.'
layout: post
guid: 'http://codercoder.cn/?p=1201'
permalink: /index.php/2022/05/2022-05-31-Using-pt-osc-add-an-unique-index-causes-data-lose
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
    - Fell-in-pit
---

# 一、背景
由于pt-osc（pt-online-schema-change）在进行ddl表更时候，步骤中insert的语法为insert ignore，所以会导致在对于表中加唯一索引时候，**重复的数据** 会被丢弃。

pt从3.0版本开始，提供了参数：

``` 
--[no]check-unique-key-change    Avoid pt-online-schema-change to run if the
                                 specified statement for --alter is trying to
                                 add an unique index (default yes)
```
即是否允许添加唯一索引。

**注意：**

* 若需要此参数生效，需要打开check-alter参数：

``` 
--[no]check-alter                Parses the --alter specified and tries to
                                   warn of possible unintended behavior (
                                   default yes)
```
 
# 二、对于Inception
goInception兼容了osc_check_unique_key_change这个参数。
``` 
[ops@127.0.0.1][(none)]> inception show variables like "%check_al%";
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| osc_check_alter | true  |
+-----------------+-------+
1 row in set (0.00 sec)

[ops@127.0.0.1][(none)]> inception show variables like "%uniq%";
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| uniq_index_prefix               | unq_  |
| osc_check_unique_key_change     | true  |
| ghost_allow_nullable_unique_key | false |
+---------------------------------+-------+
3 rows in set (0.01 sec)

```

即：
* osc_check_alter = 1 and osc_check_unique_key_change = 1 --> 不允许添加唯一索引
* osc_check_alter = 1 and osc_check_unique_key_change = 0 --> 允许添加唯一索引
* osc_check_alter = 0 and osc_check_unique_key_change = 0/1 --> 允许添加唯一索引

# 附
## 为什么pt-online-schema-change会使用insert ignore语法进行全量数据的迁移
* That IGNORE is needed in case we're changing an active table and there's an INSERT or an UPDATE on the original, because a trigger will turn that into a REPLACE that is run on the new table, but the original data might've not been inserted yet.

即：
pt-online-schema-change创建的trigger，使用replace将增量数据写入新表。使用insert ignore进行全量数据的迁移，避免出现全量数据插入新表之前，被变更，导致在新表中被覆盖的情况。