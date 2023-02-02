---
id: 1203
title: 'MySQL写入重做日志（redo log files）的顺序'
date: '2022-12-01T18:34:40+08:00'
author: Shuo
excerpt: '在redo logs中，如果有多个files，那么MySQL会怎么使用这些文件，访问的时间，第一个使用结束再使用第二个依次循环？还是不同的事务访问不同的file？'
layout: post
guid: 'http://codercoder.cn/?p=1203'
permalink: /index.php/2022/12/2022-12-01-the-order-of-writing-redo-logs
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、提出问题
在redo logs中，如果有多个files，那么MySQL会怎么使用这些文件，访问的时间，第一个文件写完，再使用第二个依次循环？还是不同的事务访问不同的file？

# 二、查看源码
通过查看代码，其是通过计算：当前lsn-文件的lsn，差值对文件个数进行取模，分发到不同的文件。

**结论：**
* **不是写完一个文件，再去写另一个文件**

```
uint64_t log_files_real_offset_for_lsn(const log_t &log, lsn_t lsn) {
  uint64_t size_offset;
  uint64_t size_capacity;
  uint64_t delta;

  ut_ad(log_writer_mutex_own(log));

  size_capacity = log.n_files * (log.file_size - LOG_FILE_HDR_SIZE);

  if (lsn >= log.current_file_lsn) {
    delta = lsn - log.current_file_lsn;

    delta = delta % size_capacity;

  } else {
    /* Special case because lsn and offset are unsigned. */

    delta = log.current_file_lsn - lsn;

    delta = size_capacity - delta % size_capacity;
  }

  size_offset = log_files_size_offset(log, log.current_file_real_offset);

  size_offset = (size_offset + delta) % size_capacity;

  return (log_files_real_offset(log, size_offset));
}
```
![](http://codercoder.cn/wp-content/uploads/2022/12/2022-12-01-the-order-of-writing-redo-logs-1.png)

# 三、相关配置的说明
innodb_log_files_in_group
* The number of log files in the log group. InnoDB writes to the files in a circular fashion. The default (and recommended) value is 2. The location of the files is specified by innodb_log_group_home_dir. The combined size of log files (innodb_log_file_size * innodb_log_files_in_group) can be up to 512GB.
即：
配置日志组中的日志个数，InnoDB以循环的方式写入文件。
![](http://codercoder.cn/wp-content/uploads/2022/12/2022-12-01-the-order-of-writing-redo-logs-2.png)
