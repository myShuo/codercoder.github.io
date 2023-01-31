---
id: 1197
title: 'Redis ERR Protocol error: too big inline request'
date: '2021-09-17T18:34:40+08:00'
author: Shuo
excerpt: 在运维过程中，需要传输文件的时候，可以使用python命令，直接开启一个ftp服务的进程，网络通即可直接传输文件。
layout: post
guid: 'http://codercoder.cn/?p=1197'
permalink: /index.php/2021/09/2021-09-17-create-a-ftp-server-using-python
categories:
    - Tech
tags:
    - python
    - Tech
---

在运维过程中，需要传输文件的时候，可以使用python命令，直接开启一个ftp服务的进程，网络通即可直接传输文件。

## 在源端机器上

进入到需要传输的文件的目录下，执行：
```
[python2]
python -m SimpleHTTPServer 8081

[python3]
python -m http.server 8081
```

其中8081为自定义的端口，可进行修改。

## 在目标端
（1）浏览器访问 localhost:8081

（2）可以直接使用wget命令：
```
wget "http://源端ip:8081/文件名"
```
