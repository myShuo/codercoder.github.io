---
id: 440
title: MySQL版本区分
date: '2019-09-08T17:30:16+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=440'
permalink: /index.php/2019/09/mysql-sersion-introduction/
views:
    - '3634'
    - '3634'
categories:
    - Tech
tags:
    - MySQL
    - Tech
---

![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0899-1-300x208.jpg)  
**alpha**  
暗示这是一个以展示新特性为目的的版本，存在比较多的不稳定因素，还会向代码中添加新新特性

**beta**  
以后的beta版、发布版或产品发布中，所有API、外部可视结构和SQL命令列均不再更改,不再向代码中添加影响代码稳定性的新特性。

**RC**  
指 Release Candidate. Release candidates被认为是稳定的, 通过了mysql所有的内部测试, 修正了所有已知的致命bug. 但是rc版本还没有经历足够长的时间来确认所有bug都已经发现，但是对rc版本只会做些小的bug修正

**GA**  
如果没有后缀,则暗示这是一个大多数情况下可用版本或者是产品版本。. GA releases是稳定的, 并通过了早期版本的测试，并显示其可用性， 解决了所有严重的bug, 并且适合在生产环境中使用. 只有少数较为严重的bug修改才会添加到该版本中。

通常来讲我们在**生产环境中还是建议使用GA版本**🙂