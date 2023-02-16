---
id: 1204
title: 'Linux磁盘监控-diskio_io_time'
date: '2023-02-03T18:34:40+08:00'
author: Shuo
excerpt: '查看Linux系统的磁盘监控，常常需要关注磁盘的情况，例如：diskio_io_time，能够间接反应磁盘的繁忙情况'
layout: post
guid: 'http://codercoder.cn/?p=1204'
permalink: /index.php/2023/02/2023-02-03-2023-02-03-linux-disk-monitor-diskio_io_time
categories:
    - Tech
tags:
    - Linux
    - Tech
---
# 一、含义
表示磁盘的I/O requests queue，即IO占用的时间，单位为毫秒。

参考：[SystemDiskRequestQueuedWarning](https://docs.mirantis.com/mcp/q4-18/mcp-operations-guide/lma/alerts/core-services-alerts/system-alerts.html)

# 二、表达式
（1）IO占比
```
rate(diskio_io_time{name=~"node-.*"}[1m]) / 1000
```
rate计算1分钟内的IO耗时占比，除以1000后单位为秒。意思为一秒内的IO占比。


（2）IO占比阈值
```
rate(diskio_io_time{name=~"node-.*"}[1m]) / 1000 > 0.9
```
I/O requests for the {{ $labels.name }} disk on the {{ $labels.host }} node spent 90% of the device time in queue during the last 10 minutes.

IO在1秒内的耗时占比大于0.9（即90%）的数据。

I/O requests queue 占用总的device_time的时间比例

# 三、作用
## （1）间接反应磁盘的繁忙情况
若占比越高，则当前的IO request所占用的时间越长，响应速度则越慢。

参考：[io-request-queueing](https://stackoverflow.com/questions/25479293/io-request-queueing)

## （2）blocking I/O call 
Broadly, a blocking I/O call does this: 
* (1) put a process/thread on a list of processes/threads which are waiting for I/O to complete
* (2) mark it as non-runnable 
* (3) invoke a context switch. 

Blocking：
Since the thread which made the blocking call is now marked as non-runnable, the OS' scheduler will never schedule it to run on the CPU, until the I/O operation completes, and the thread is marked as runnable again.

在IO操作完成后，线程才能继续执行。

## （3）non-blocking I/O calls
* (1) Your program first makes a system call to create an "asynchronous I/O context". 

**This is an object which includes a queue for notification events generated when an I/O operation completes.**

创建一个系统调用，包含IO操作结束时生成的通知事件队列。

* (2) The kernel passes back an AIO context identifier which your program uses when making subsequent calls. 

后续调用时，内核回传上步骤创建的AIO标识符

* (3) Then, when you want to do some I/O, you make another system call to "submit" an async I/O operation for execution. You can submit as many of these as you like. 

当需要IO操作时，创建一个新的系统调用来“提交”一个异步操作，可以提交任意多个数量。

* (4) As they complete, notifications will accumulate in your AIO context's event queue. 

当它们完成时，通知将累积在您的 AIO 上下文的事件队列中。

* (5) Later, whenever you want, you make another syscall to retrieve the notifications from that queue.

之后，你可以在任何时候，通过另外的系统调用从队列中获取通知。