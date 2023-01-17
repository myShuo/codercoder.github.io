---
id: 1178
title: 你真的了解线程池ThreadPoolExecutor吗？
date: '2020-07-29T15:19:34+08:00'
author: Smile
layout: post
guid: 'http://codercoder.cn/?p=1178'
permalink: /index.php/2020/07/know-threadpoolexecutor/
views:
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
    - '103655'
bigfa_ding:
    - '2'
categories:
    - Tech
tags:
    - Java
    - 线程池
---

### 背景

最近被别人问到有关线程池的问题，自己没有答上来，自己觉得之前还是比较了解线程池的，所以又重新学习了一下这块内容，然后记录一下与大家分享。

### 从两个问题说起

- 线程池线程数增加过程是怎样的？
- 如果线程池线程运行过程中抛异常了，线程池怎么处理该异常线程（是否抛异常、是否回收线程再次利用）

### Part 1:

#### 线程池线程增加逻辑

参考图：  
![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2972.png)

#### 如果线程池队列设置为无限大，最大线程数还有用吗？

```
从图中可以得知，当线程池队列设置为无限大的时候，最大线程数是没有用的，线程池的活跃线程最大就为核心线程数大小。

```

#### JDK线程池为什么要实现成先放队列然后在增加到最大线程数，为什么不是像Tomcat线程池实现一样先增大到最大线程数在放队列?

1. 自我感觉没有好坏之分，可能适用场景不一样。如果不能容忍延迟，期望应用能尽快的为用户提供服务，就选tomcat实现的，如果能容忍一定的延迟来换取性能上的提升就采用JDK方式。
2. 而且也区分应用是CPU密集型还是IO密集型，如果是CPU密集型，是需要线程长时间进行复杂运算，增加线程会造成线程上下文切换频繁，处理速度反而会降低。如果是IO密集型，线程大部分时间是在等待IO读取和写入，增加线程可以提高并发度，处理更多任务。
3. 还有一个原因可能就是创建线程addWorker的是时候是需要获取mainLock这个全局锁，影响并发效率，先放队列时候使用到同步队列可以做为一个缓冲。

上面的是自我的几点理解，如果大家有觉得有更好的理解的，希望补充。

#### 如果想要先增加线程到最大之后才放队列怎么做？

##### 两种做法：

1. 调整核心线程数大小和最大线程数大小一样
2. 就是基于ArrayBlockingQueue实现自己的BlockingQueue，重写其中的offer方法，在里面增加判断如果当前线程数小于最大线程数，则返回false，否则就执行队列自身的offer方法。

### Part 2

#### 线程池线程抛异常了会打印错误日志吗?

线程池执行任务分为两种方式，一种execute，一种submit方式：

- 如果是execute方式，则会在  
  java.util.concurrent.ThreadPoolExecutor#runWorker中会抛出异常  
  ![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2923.png)  
  在java.lang.ThreadGroup#uncaughtException中会将异常打印到控制台  
  ![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2985.png)
- 如果是通过submit方式提交，则不会有异常打印，看下submit代码：java.util.concurrent.AbstractExecutorService#submit ![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2968.png)  
  task被包装成一个FutureTask执行，在java.util.concurrent.FutureTask#run方法里看到异常被捕获并没有抛出，而是设置到了对象中一个字段中![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2916.png)

#### 线程池线程抛异常了线程会被回收吗?

答案是线程会被remove掉，然后重新创建一个新的线程加入到线程池  
查看java.util.concurrent.ThreadPoolExecutor#runWorker代码，发现如果抛出异常时候，最后走到java.util.concurrent.ThreadPoolExecutor#processWorkerExit方法，核心代码如下图  
![](http://codercoder.cn/wp-content/uploads/2020/07/2020-07-2965.png)

#### 如果想要获取到异常怎么办?

##### 两种做法：

1. 通过submit方式提交时候，future.get时候会抛出异常
2. 通过实现线程池java.util.concurrent.ThreadPoolExecutor#afterExecute方式，可以获得到异常信息

#### 为什么线程抛异常被移除之后又会创建一个新的线程加入？

```
个人理解：可能是为了避免线程池还有任务但是线程异常被移除了之后没有线程在工作了，所以又新创建了一个新的线程到线程池。

```