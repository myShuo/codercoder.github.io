---
id: 885
title: 'MySQL手记4 &#8212; Sysbench进行QPS性能测试'
date: '2020-04-08T21:47:56+08:00'
author: Shuo
excerpt: '加上上篇的MySQL手记2 -- sysbench测试磁盘IO，已经可以使用sysbench测试得到IOPS和QPS/TPS的结果了，这对于业务上线，提供了参考​。压测这一步，也不能马虎​。'
layout: post
guid: 'http://codercoder.cn/?p=885'
image: http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0847.png
permalink: /index.php/2020/04/mysql-note-4-sysbench-qps-test/
views:
    - '3390'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、简介

```
Sysbench是业内较为常用的测试工具，常用以测试数据库QPS（Queries Per Second），还有系统IO等等。https://launchpad.net/sysbench/

```

在《高性能MySQL》书中也有提到使用Sysbench来进行数据库的压测。  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-0847.png)

# 二、基准测试

```
在做压测之前，需要评估好数据库环境的情况，例如：需要多少数据进行测试，测试选用什么测试算法等等。

```

## 2.1 数据量评估

```
根据业务评估最大需要存放多少的数据量，并且做一定的冗余。例如：实际生产需存放50G的数据，最大有200个并发，那么，我们在测试的时候就可以选择30GB～100GB的范围区间进行“阶梯形”压测。
为了防止出现内存很大会放下所有数据的情况，所以在压测时候，可使压测的数据量大于数据库（innodb_buffer_pool_size）的大小，这样能测试出更为确切的磁盘IO。

```

## 2.2 IO测试模式

```
本系列主要是以数据库为中心进行介绍，所以本篇也是站在数据库的角度选择测试的模式。

对于MySQL数据库，由于数据的存储在逻辑上是按照主键（聚集索引）顺序进行排列，但物理上，由于行信息，头信息（page-head），碎片等因素，真实在物理上则不是顺序的。所以在测试IO的时候，我们选择的test-mode为——“rndrw”（Random Read Write），这也就是为什么我们通常把数据放在SSD上，是因为SSD具有更高的随机读写性能。

```

（PS:对于顺序读写，机械硬盘的性能和SSD相差不会很大，这点也是可以使用sysbench进行测试得到）

### 2.2.1 IO测试

```
由于数据库的性能瓶颈往往在于磁盘的IO，所以测试IO成为了第一个步骤。可参照：MySQL手记2 -- sysbench测试磁盘IO

```

### 2.2.2 IO测试结果解读

```
执行sysbench进行测试后，我们主要关注几个方面：IOPS、吞吐量、延迟情况。

```

**（1）IOPS：**

```
在我们实际环境中，IOPS间接影响了应用的并发，IOPS越高，应用就能够在相同时间内执行更多的查询。而若IOPS较低，那么除了并发的QPS得不到提升之外，SQL可能会因此阻塞，导致大量的慢查，消耗数据库资源。

```

**（2）吞吐量：**

```
部分业务可能会较大的字符串，所以磁盘的吞吐量同样是一个重要的指标，吞吐量越大，能够瞬时IO（INPUT, OUTPUT）的数据也就越多。

```

**（3）延迟情况：**

```
很多对于延迟很敏感的应用，需要及时返回查询结果，读写都很频繁的场景，就会需要我们的磁盘延迟非常低。通常情况下，随着负载的升高，延迟可能会有一定的波动。在测试时，建议：

```

a. 测试的时间需要至少测试1个小时  
b. 测试的数据量最好是在预估数据量附近的一个范围段，例如需要上线的最大数据量是100GB，则测试80～120GB区间（例如：80GB/90GB/100GB…）  
c. 相同条件测试3次，取平均值，防止出现偶然情况

## 2.3 QPS测试

```
QPS(Queries Per Second)为衡量数据库性能的重要指标。数据库实例QPS越高，那么在生产环境中，就能满足更多的业务负载。对于QPS的测试，尤为重要。

```

例如选择如下的条件：

```
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=172.16.3.3 --mysql-port=3306  --mysql-user=root --mysql-password=abcabc  --mysql-db=sbtestdb --tables=3 --table-size=50000000 --threads=30  --max-time=90 --report-interval=10  run &

```

**（1）sysbench –help**

```
sysbench --help
​
Usage:
​
  sysbench [options]... [testname] [command]
​
Commands implemented by most tests: prepare run cleanup help
​
General options:
  --threads=N                     number of threads to use [1]
  --events=N                      limit for total number of events [0]
  --time=N                        limit for total execution time in seconds [10]
  --forced-shutdown=STRING        number of seconds to wait after the --time limit before forcing shutdown, or 'off' to disable [off]
  --thread-stack-size=SIZE        size of stack per thread [64K]
  --rate=N                        average transactions rate. 0 for unlimited rate [0]
  --report-interval=N             periodically report intermediate statistics with a specified interval in seconds. 0 disables intermediate reports [0]
  --report-checkpoints=[LIST,...] dump full statistics and reset all counters at specified points in time. The argument is a list of comma-separated values representing the amount of time in seconds elapsed from start of test when report checkpoint(s) must be performed. Report checkpoints are off by default. []
  --debug[=on|off]                print more debugging info [off]
  --validate[=on|off]             perform validation checks where possible [off]
  --help[=on|off]                 print help and exit [off]
  --version[=on|off]              print version and exit [off]
  --config-file=FILENAME          File containing command line options
  --tx-rate=N                     deprecated alias for --rate [0]
  --max-requests=N                deprecated alias for --events [0]
  --max-time=N                    deprecated alias for --time [0]
  --num-threads=N                 deprecated alias for --threads [1]
​
Pseudo-Random Numbers Generator options:
  --rand-type=STRING random numbers distribution {uniform,gaussian,special,pareto} [special]
  --rand-spec-iter=N number of iterations used for numbers generation [12]
  --rand-spec-pct=N  percentage of values to be treated as 'special' (for special distribution) [1]
  --rand-spec-res=N  percentage of 'special' values to use (for special distribution) [75]
  --rand-seed=N      seed for random number generator. When 0, the current time is used as a RNG seed. [0]
  --rand-pareto-h=N  parameter h for pareto distribution [0.2]
​
....
See 'sysbench <testname> help' for a list of options for each test.

```

**（2）lua脚本：**

bulk\_insert.lua oltp\_insert.lua oltp\_read\_write.lua oltp\_write\_only.lua tests  
​  
oltp\_common.lua oltp\_point\_select.lua oltp\_update\_index.lua select\_random\_points.lua  
​  
oltp\_delete.lua oltp\_read\_only.lua oltp\_update\_non\_index.lua select\_random\_ranges.lua

lua脚本的选择：  
 sysbench提供了多种不同的类型，其中较为常用的为：oltp\_read\_write.lua（即oltp场景的read&write），当然，可以根据实际的需要，选择只读（oltp\_read\_only.lua ）、只写（oltp\_write\_only.lua ）、批量插入（bulk\_insert.lua）等等。

**（3）其它选项**  
 –threads：并发执行的线程数 通常使用线上可能出现的并发数进行测试  
 –max-time：运行测试的最长时间，至少测试15分钟  
 –tables：表的数目，可根据应用情况而定，主要根据大表的数目进行判断  
 –table-size：表的大小，若为多个表，则每个表的大小均为table-size

## 2.4 测试过程

**（1）数据准备**

```
# sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=172.16.3.3 --mysql-port=3306  --mysql-user=root --mysql-password=abcabc  --mysql-db=sbtestdb --tables=3 --table-size=50000 --threads=30  --max-time=90 --report-interval=10  prepare
​
sysbench 1.0.17 (using system LuaJIT 2.0.4)
Initializing worker threads...
​
Creating table 'sbtest2'...
Creating table 'sbtest3'...
Creating table 'sbtest1'...
Inserting 50000 records into 'sbtest1'
Inserting 50000 records into 'sbtest2'
Inserting 50000 records into 'sbtest3'
Creating a secondary index on 'sbtest1'...
Creating a secondary index on 'sbtest3'...
Creating a secondary index on 'sbtest2'...

```

```
​生成3个表，每个表5​0000行数据。

```

**（2）oltp测试：**

```
# sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=172.16.3.3 --mysql-port=3306  --mysql-user=root --mysql-password=abcabc  --mysql-db=sbtestdb --tables=3 --table-size=50000 --threads=30  --max-time=90 --report-interval=10  run
​
sysbench 1.0.17 (using system LuaJIT 2.0.4)
Running the test with following options:
Number of threads: 30
​
Report intermediate results every 10 second(s)
Initializing random number generator from current time
Initializing worker threads...
​
Threads started!
​
[ 10s ] thds: 30 tps: 168.27 qps: 3402.24 (r/w/o: 2388.13/674.57/339.53) lat (ms,95%): 634.66 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 30 tps: 196.39 qps: 3923.33 (r/w/o: 2744.41/786.15/392.77) lat (ms,95%): 196.89 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 30 tps: 228.62 qps: 4578.77 (r/w/o: 3207.96/913.57/457.24) lat (ms,95%): 207.82 err/s: 0.00 reconn/s: 0.00
[ 40s ] thds: 30 tps: 254.63 qps: 5090.31 (r/w/o: 3561.16/1019.90/509.25) lat (ms,95%): 183.21 err/s: 0.00 reconn/s: 0.00
[ 50s ] thds: 30 tps: 217.37 qps: 4348.62 (r/w/o: 3045.12/868.86/434.63) lat (ms,95%): 189.93 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 30 tps: 246.20 qps: 4922.70 (r/w/o: 3445.80/984.40/492.50) lat (ms,95%): 186.54 err/s: 0.00 reconn/s: 0.00
[ 70s ] thds: 30 tps: 210.20 qps: 4207.09 (r/w/o: 2943.39/843.30/420.40) lat (ms,95%): 193.38 err/s: 0.00 reconn/s: 0.00
[ 80s ] thds: 30 tps: 216.30 qps: 4324.93 (r/w/o: 3030.35/861.99/432.59) lat (ms,95%): 196.89 err/s: 0.00 reconn/s: 0.00
[ 90s ] thds: 30 tps: 198.00 qps: 3963.07 (r/w/o: 2774.65/792.51/395.91) lat (ms,95%): 200.47 err/s: 0.00 reconn/s: 0.00
​
SQL statistics:
    queries performed:
        read:                            271460
        write:                           77560
        other:                           38780
        total:                           387800
    transactions:                        19390  (215.21 per sec.)
    queries:                             387800 (4304.16 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)
​
General statistics:
    total time:                          90.0974s
    total number of events:              19390
Latency (ms):
         min:                                    8.11
         avg:                                  139.32
         max:                                 1757.42
         95th percentile:                      196.89
         sum:                              2701326.42
​
Threads fairness:
    events (avg/stddev):           646.3333/6.16
    execution time (avg/stddev):   90.0442/0.02
​

```

 命令指定为每10秒打印一次状态，一共执行90秒（由于此次为展示，所以执行的表数据、时间等都不符合规范）。  
可以看到平均的结果汇总为：

```
**TPS：215.21             ## transactions:   19390  (215.21 per sec.)
QPS：4304.16                ##queries:     387800 (4304.16 per sec.) 
95%的延时为：196.89ms    ##95th percentile:                      196.89**

```

```
​通常，结合业务的情况，判读是否能够满足业务需求，例如：
​    ​业务需要单实例的QPS在6000左右，那么这个测试的结果就不能够满足需求
​    ​有的业务对于数据库响应的延时要求很高，若需要的延时不能超过50毫秒，那么该实例同样也不能满足需求

```

此外，还有许多有趣的测试案例，之前看到的一个有趣的测试案例，测试MySQL的创建连接的性能：https://github.com/jeremycole/yesmark

**（3）清理**

```
​测试结束后，记得及时清理数据，防止占用磁盘空间，造成不必要的磁盘浪费：

```

```
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=172.16.3.3 --mysql-port=3306  --mysql-user=root --mysql-password=abcabc  --mysql-db=sbtestdb --tables=3 --table-size=50000 --threads=30  --max-time=90 --report-interval=10  cleanup

```

## 2.4 其它注意事项

```
​每测试一次，停顿900秒：（1）用于预热数据，避免预热时的I/O影响测试结果；（2）CPU也需要进行“冷处理”，同样《高性能MySQL》一书中推荐两次测试中间间隔15分钟

​sysbench所在的实例也需要预留足够的资源：因为sysbench运行的时候，会消耗CPU和内存，所以为了得到更准确的结果，sysbench所在的机器也需要预留足够的配置

```

​

```
​至此，加上上篇的[MySQL手记2 -- sysbench测试磁盘IO](http://codercoder.cn/index.php/2020/04/mysql-note-2-sysbench-iotest/ "MySQL手记2 -- sysbench测试磁盘IO")，已经可以使用sysbench测试得到IOPS和QPS/TPS的结果了，这对于业务上线，提供了参考​。压测这一步，也不能马虎​。

```

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)