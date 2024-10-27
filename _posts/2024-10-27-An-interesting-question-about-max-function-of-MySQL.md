---
id: 20241027
title: 'An interesting question about max function of MySQL'
date: '2024-10-27T00:23:40+08:00'
author: Shuo
excerpt: "Why can't we use the 'max(point), id' to get the additional columns?"
layout: post
guid: 'http://codercoder.cn/?p=20241027'
image: http://codercoder.cn/wp-content/uploads/2024/20241027.jpg
permalink: /index.php/2024/10/2024-10-27-An-interesting-question-about-max-function-of-MySQL
categories:
    - LOL-7788
    - Shuo
    - Tech
tags:
    - LOL-7788
---

# Make the engineers busy?

Recently I talked with my colleague about why some people can make many creative products in their work areas. And he said it's because these people have enough time to think. I totally agree with this.

Some companies try to keep their staff busy enough, some want the enough overtime work hours, some want more code lines... 

It's hard to measure the efficiency of software engineers, even harder for DBAs. So when I'm working, I have to produce some phased results, but sometimes the system is so stable that we don't need to do troubleshooting, and sometimes it's not, and it's a usual thing that we spend lots of time on one-time CPU drops.

# From an on-call work day
Once, on my on-call work day, I met a MySQL problem where the column data type is Date type, and when inserting a row like "2024-08-01", it would show as  "2024:08:01". But I was so busy at that time, so I had to deal with all the other issues, and then I repeated the situation on my database instance, it was  the same result. Then I reported this bug to the MySQL official bug website.

[Binary log format bug](https://bugs.mysql.com/bug.php?id=115740)

If I'm always busy, I think maybe I'll have no time to find out the problem, and after many times giving up, I'll even lose the ability to test and think more.

A constant learning habit is necessary for an engineer's work, so we should free our minds sometimes,  make more creative things, think different and think big.

![](http://codercoder.cn/wp-content/uploads/2024/20240801.jpg)
