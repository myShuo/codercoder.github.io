---
id: 1200
title: 'MySQL Partition Tables: Split New Partition From Maxvalue Partition'
date: '2022-07-04T18:34:40+08:00'
author: Shuo
excerpt: 'MysQL的分区表中，已经有maxvalue分区时，需要再添加分区，则需要重新进行分配'
layout: post
guid: 'http://codercoder.cn/?p=1200'
image: http://codercoder.cn/wp-content/uploads/2022/07/2022-07-04-mysql-split-new-partition-from-maxvalue-partition-table.png
permalink: /index.php/2022/07/2022-07-04-mysql-split-new-partition-from-maxvalue-partition-table
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、环境准备
建表：
```
create table test_part(id int auto_increment ,
days int unsigned not null default 19700101,
primary key (id,days)) 
partition by range(days) (
partition p202206 values less than (20220601),
partition p202207 values less than (20220701),
partition p2022 values less than maxvalue
);
```

## （1）在已有maxvalue的分区表中添加新的分区
```
ALTER TABLE test_part ADD PARTITION (PARTITION p202208 VALUES LESS THAN (20220801) ENGINE = InnoDB);
报错：
ERROR 1481 (HY000): MAXVALUE can only be used in last partition definition
```
![](http://codercoder.cn/wp-content/uploads/2022/07/2022-07-04-mysql-split-new-partition-from-maxvalue-partition-table.png)

## （2）重新定义分区分布
```
alter table test_part
PARTITION BY RANGE (`days`)
(PARTITION p202206 VALUES LESS THAN (20220601) ENGINE = InnoDB,
 PARTITION p202207 VALUES LESS THAN (20220701) ENGINE = InnoDB,
PARTITION p202208 VALUES LESS THAN (20220801) ENGINE = InnoDB,
 PARTITION p2022 VALUES LESS THAN MAXVALUE ENGINE = InnoDB);

```

可以执行，会迁移在原来p2022分区中的数据，迁移过程加share lock：
* Only permits ALGORITHM=DEFAULT, LOCK=DEFAULT. Does not copy existing data for tables partitioned by RANGE or LIST. Concurrent queries are permitted for tables partitioned by HASH or LIST. MySQL copies the data while holding a shared lock.

## （3）直接拆分maxvalue的分区
MySQL支持重新分配分区：
```
alter table test_part REORGANIZE PARTITION p2022 into ( partition p202209 VALUES LESS THAN (20220901) ENGINE = InnoDB, PARTITION p2022 VALUES LESS THAN MAXVALUE ENGINE = InnoDB)
```
![](http://codercoder.cn/wp-content/uploads/2022/07/2022-07-04-mysql-split-new-partition-from-maxvalue-partition-table-2.png)

参考：[Management of RANGE and LIST Partitions](https://dev.mysql.com/doc/mysql-partitioning-excerpt/5.7/en/partitioning-management-range-list.html)