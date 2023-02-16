---
id: 1205
title: 'Linux swap 内存溢出'
date: '2022-12-19T18:34:40+08:00'
author: Shuo
excerpt: '由于磁盘IO较弱的原因，不再在数据库的实例上进行swap的内存溢出，需要进行关闭。'
layout: post
guid: 'http://codercoder.cn/?p=1205'
image: http://codercoder.cn/wp-content/uploads/2022/12/2022-12-19-linux-swap-2.png
permalink: /index.php/2022/12/2022-12-19-Linux-swap
categories:
    - Tech
tags:
    - Linux
    - Tech
---

# 一、目的
由于磁盘IO较弱的原因，不再在数据库的实例上进行swap的内存溢出，需要进行关闭。

# 二、操作
## （1）查看内存使用情况

```
free -m
```
![](http://codercoder.cn/wp-content/uploads/2022/12/2022-12-19-linux-swap-1.png)

## （2）swapoff
执行：
```
swapoff -a
```

对于磁盘较弱的环境：8GB的swap，使用5GB的情况下，大约花费1小时。

查看swapoff进程：

![](http://codercoder.cn/wp-content/uploads/2022/12/2022-12-19-linux-swap-2.png)


**swapoff速度较慢原因：**
* 磁盘性能
* The way it works now, swapoff looks at each swapped out memory page in the swap partition, and tries to find all the programs that use it. If it can’t find them right away, it will look at the page tables of every program that’s running to find them. In the worst case, it will check all the page tables for every swapped out page in the partition. That’s right–the same page tables get checked over and over again.

它现在的工作方式是，swapoff 查看交换分区中每个换出的内存页，并尝试找到所有使用它的程序。 如果不能立即找到它们，它将查看正在运行的每个程序的页表以找到它们。 在最坏的情况下，它将检查分区中每个换出页的所有页表。 没错——相同的页表被一遍又一遍地检查。

参考：[https://unix.stackexchange.com/questions/45673/how-can-swapoff-be-that-slow](https://unix.stackexchange.com/questions/45673/how-can-swapoff-be-that-slow)

## （3）调整/etc/fstab
为避免重启后swap继续打开，需要关闭swap挂载：
```
vim /etc/fstab
注释swap所在行
```
