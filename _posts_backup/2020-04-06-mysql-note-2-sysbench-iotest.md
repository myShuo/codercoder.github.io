---
id: 868
title: 'MySQL手记2 &#8212; sysbench测试磁盘IO'
date: '2020-04-06T15:26:17+08:00'
author: Shuo
excerpt: 对于基础硬件资源的性能测试，刚工作时，我也是人云亦云，不知道为什么要这么去做。在后面的工作中，逐渐意识到了性能压测，又或称其为基准测试的重要性。
layout: post
guid: 'http://codercoder.cn/?p=868'
permalink: /index.php/2020/04/mysql-note-2-sysbench-iotest/
views:
    - '7371'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、磁盘性能测试

 对于基础硬件资源的性能测试，刚工作时，我也是人云亦云，不知道为什么要这么去做。在后面的工作中，逐渐意识到了性能压测，又或称其为基准测试的重要性。在MySQL数据库界的一个宝藏书籍：**《高性能MySQL》**中，就有详细的介绍。

# 二、Sysbench测试磁盘IO

 很多人性能压测，就是按照文档跑一遍，这种做法比较差强人意，在生产环境中，上线前对于机器的压测显得尤为重要。

## 1. 文件大小的选择

 光是这一点列出来，想必大部分人都有恍然大悟的感觉，对于测试IO时候，应当选取多大的数据文件，也是有讲究的：例如线上实际是500GB的数据文件，但是在测试时候只生成了10GB的文件进行IO测试，这样得到的结果，就与实际结果具有较大的误差。

## 2.测试过程

 Sysbench测试的过程为：  
（1）生成指定大小和数目的数据文件；  
（2）按照指定算法运行IO测试；  
（3）清除数据文件

### 2.1 生成文件

```
sysbench --test=fileio --file-total-size=1GB  --file-num=128 prepare

```

```
​显而易见，生成128个总大小为1GB的文件进行IO测试。
​sysbench还有很多的指标，可供测试的时候选用，用以得到最为符合线上情况的效果。

```

[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0678.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0678.png)  
 ​在生成文件时，是顺序写入，可以看出该磁盘的顺序写入性能为：179 MiB/s。

### 2.2 run

```
sysbench fileio --num-threads=32 --file-total-size=1GB  --file-num=10 --file-test-mode=rndrw  run

```

主要关注：  
 –file-test-mode，可选的算法有：seqwr, seqrewr, seqrd, rndrd, rndwr, rndrw，即顺序写、顺序读写、顺序读、随机读、随机写、随机读写  
 虽然MySQL的数据在逻辑上，是按照主键（聚集索引）进行排布的，但是实际在物理，为随机读写rndrw。这点也是在基准测试时，需要着重注意的，而不是当被问到：为什么使用随机读写的时候，自己也一脸懵  
 由于MySQL的这个属性，所以我们在测试IO的时候同样选择**随机读写rndrw**进行测试。

```
​sysbench在运行时，会打印相关的信息

```

```
[admin@ws_001 sysbench]$ sysbench fileio --num-threads=32 --file-total-size=1GB  --file-num=128  --file-test-mode=rndrw  run
WARNING: --num-threads is deprecated, use --threads instead
sysbench 1.0.17 (using system LuaJIT 2.0.4)
​
Running the test with following options:
Number of threads: 32
Initializing random number generator from current time
​
​
Extra file open flags: (none)
128 files, 8MiB each
1GiB total file size
Block size 16KiB
Number of IO requests: 0
Read/Write ratio for combined random IO test: 1.50
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing random r/w test
Initializing worker threads...
​
​
Threads started!
​
​
File operations:
    reads/s:                      2070.70
    writes/s:                     1379.97
    fsyncs/s:                     4819.37
​
​
Throughput:
    read, MiB/s:                  32.35
    written, MiB/s:               21.56
​
​
General statistics:
    total time:                          10.1124s
    total number of events:              79547
​
​
Latency (ms):
         min:                                    0.00
         avg:                                    4.03
         max:                                  263.72
         95th percentile:                       23.52
         sum:                               320278.15
​
​
Threads fairness:
    events (avg/stddev):           2485.8438/187.43
    execution time (avg/stddev):   10.0087/0.02
​

```

```
​Mbit/s：每秒传输10^6 bit的数据，也写成Mbps
​MB/s：每秒传输10^6 byte的数据
​MiB/s：每秒传输2^20 byte的数据

​可以看到，读写速率为2070reads/s、1379writes/s，读写吞吐量为：32MiB/s、21MiB/s，并且读写的延迟为23.52毫秒。对比在上个步骤中顺序写入的性能，可以大致猜测该磁盘为机械盘，而非SSD，因为SSD的随机读写的性能很高，并且SSD的随机读写和顺序读写不会相差那么高（这个结论也是多次在机房做基准测试得到的结论）。

```

此次测试的误差其实较大，因为使用较小的文件进行测试，往往很容易受硬件资源抖动的影响。本案例只是用来说明测试需要关注的要点。  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0633.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0633.png)  
参考：https://en.wikipedia.org/wiki/Data-rate\_units

根据**读写量、吞吐量、延迟**的情况，可以初步得出结论：**该磁盘是否能满足生产环境的要求**？  
​

此外，通过查看sysbench –help信息，sysbench还可以用来测试CPU、内存等的信息：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0625.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0625.png)

至此，sysbench测试磁盘IO的介绍到一段落。测试不是目的，而为什么测试、想要获得什么样的结果，才是我们应该着重关注的，只有测试多种场景，才能使我们的系统在生产环境中更加稳健。  
欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)