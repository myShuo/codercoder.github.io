---
id: 588
title: '安装及使用glance &#8211;查看Linux性能命令'
date: '2019-09-11T16:33:03+08:00'
author: Shuo
layout: post
guid: 'http://codercoder.cn/?p=588'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1182.png
permalink: /index.php/2019/09/glance-show-linux-status/
views:
    - '28992'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - Tool
    - Tech
---

# 环境准备

## python3

### 更新yum源

```
备份：
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
mv epel.repo epel.repo.backup

yum clean all    ##清理缓存

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo
yum makecache

```

### 安装python3

```
yum list python3*

```

找到相关的包后，进行安装：

```
yum install python3*

```

**在/usr/bin目录下，更换默认的python文件**

```
mv /usr/bin/python  /usr/bin/python2.6.bak
cp -rf /usr/bin/python3  /use/bin/python

```

# 安装glance

1. 使用curl安装  
  > curl -L https://bit.ly/glances | /bin/bash
2. 使用wget安装  
  > wget -O- http://bit.ly/glances | /bin/bash

## 报错及修复

### yum配置报错

```
[root@localhost ~]# sh install.sh
Detected system: CentOS
Loaded plugins: fastestmirror, security
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
 * webtatic: uk.repo.webtatic.com
Package python-pip-7.1.0-1.el6.noarch already installed and latest version
Package python-devel-2.6.6-66.el6_8.x86_64 already installed and latest version
Package gcc-4.4.7-23.el6.x86_64 already installed and latest version
Package lm_sensors-3.1.1-17.el6.x86_64 already installed and latest version
Package 1:wireless-tools-29-6.el6.x86_64 already installed and latest version
Nothing to do
Install dependancies
Traceback (most recent call last):
  File "/usr/bin/pip", line 7, in 
    from pip._internal import main
  File "/usr/lib/python2.6/site-packages/pip/_internal/__init__.py", line 19, in 
    from pip._vendor.urllib3.exceptions import DependencyWarning
  File "/usr/lib/python2.6/site-packages/pip/_vendor/urllib3/__init__.py", line 8, in 
    from .connectionpool import (
  File "/usr/lib/python2.6/site-packages/pip/_vendor/urllib3/connectionpool.py", line 92
    _blocking_errnos = {errno.EAGAIN, errno.EWOULDBLOCK}
                                    ^
SyntaxError: invalid syntax
Traceback (most recent call last):
  File "/usr/bin/pip", line 7, in 
    from pip._internal import main
  File "/usr/lib/python2.6/site-packages/pip/_internal/__init__.py", line 19, in 
    from pip._vendor.urllib3.exceptions import DependencyWarning
  File "/usr/lib/python2.6/site-packages/pip/_vendor/urllib3/__init__.py", line 8, in 
    from .connectionpool import (
  File "/usr/lib/python2.6/site-packages/pip/_vendor/urllib3/connectionpool.py", line 92
    _blocking_errnos = {errno.EAGAIN, errno.EWOULDBLOCK}
                                    ^
SyntaxError: invalid syntax
Install Glances
Traceback (most recent call last):
  File "/usr/bin/pip", line 7, in 
    from pip._internal import main
  File "/usr/lib/python2.6/site-packages/pip/_internal/__init__.py", line 19, in 
    from pip._vendor.urllib3.exceptions import DependencyWarning
  File "/usr/lib/python2.6/site-packages/pip/_vendor/urllib3/__init__.py", line 8, in 
    from .connectionpool import (
  File "/usr/lib/python2.6/site-packages/pip/_vendor/urllib3/connectionpool.py", line 92
    _blocking_errnos = {errno.EAGAIN, errno.EWOULDBLOCK}
                                    ^
SyntaxError: invalid syntax

```

**修复**  
可以看出报错是在yum时候，**由于yum配置为python2版本，现在的默认python为python3，所以解析报错**。修复：

```
vim  /usr/bin/yum
#把   #!/usr/bin/python 改为 #!/usr/bin/python2.6

```

### pip版本问题

```
[root@localhost  ~]# ./install.sh       
Detected system: CentOS
Loaded plugins: fastestmirror, security
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
 * webtatic: uk.repo.webtatic.com
Package python-pip-7.1.0-1.el6.noarch already installed and latest version
Package python-devel-2.6.6-66.el6_8.x86_64 already installed and latest version
Package gcc-4.4.7-23.el6.x86_64 already installed and latest version
Package lm_sensors-3.1.1-17.el6.x86_64 already installed and latest version
Package 1:wireless-tools-29-6.el6.x86_64 already installed and latest version
Nothing to do
Install dependancies
Traceback (most recent call last):
  File "/usr/bin/pip", line 7, in 
    from pip._internal import main
ImportError: No module named 'pip'
Traceback (most recent call last):
  File "/usr/bin/pip", line 7, in 
    from pip._internal import main
ImportError: No module named 'pip'
Install Glances
Traceback (most recent call last):
  File "/usr/bin/pip", line 7, in 
    from pip._internal import main
ImportError: No module named 'pip'

```

**修复**  
可以看出是因为pip版本较低的原因，所以升级pip：

```
wget https://bootstrap.pypa.io/get-pip.py -o get-pip.py 
python get-pip.py --force-reinstall

[root@localhost  ~]# python get-pip.py --force-reinstall
DEPRECATION: Python 3.4 support has been deprecated. pip 19.1 will be the last one supporting it. Please upgrade your Python as Python 3.4 won't be maintained after March 2019 (cf PEP 429).
Collecting pip
  Using cached https://files.pythonhosted.org/packages/5c/e0/be401c003291b56efc55aeba6a80ab790d3d4cece2778288d65323009420/pip-19.1.1-py2.py3-none-any.whl
Collecting wheel
  Downloading https://files.pythonhosted.org/packages/bb/10/44230dd6bf3563b8f227dbf344c908d412ad2ff48066476672f3a72e174e/wheel-0.33.4-py2.py3-none-any.whl
Installing collected packages: pip, wheel
Successfully installed pip-19.1.1 wheel-0.33.4

```

## 安装glance时的日志

安装文档<sup id="fnref-588-1">[1](#fn-588-1)</sup>

```
[root@localhost  ~]# ./install.sh 
Detected system: CentOS
Loaded plugins: fastestmirror, security
Setting up Install Process
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
 * webtatic: us-east.repo.webtatic.com
Package python-pip-7.1.0-1.el6.noarch already installed and latest version
Package python-devel-2.6.6-66.el6_8.x86_64 already installed and latest version
Package gcc-4.4.7-23.el6.x86_64 already installed and latest version
Package lm_sensors-3.1.1-17.el6.x86_64 already installed and latest version
Package 1:wireless-tools-29-6.el6.x86_64 already installed and latest version
Nothing to do
Install dependancies
DEPRECATION: Python 3.4 support has been deprecated. pip 19.1 will be the last one supporting it. Please upgrade your Python as Python 3.4 won't be maintained after March 2019 (cf PEP 429).
Requirement already up-to-date: pip in /usr/lib/python3.4/site-packages (19.1.1)
DEPRECATION: Python 3.4 support has been deprecated. pip 19.1 will be the last one supporting it. Please upgrade your Python as Python 3.4 won't be maintained after March 2019 (cf PEP 429).
Requirement already satisfied: setuptools in /usr/lib/python3.4/site-packages (19.6.2)
Collecting glances[action,batinfo,browser,cpuinfo,docker,export,folders,gpu,graph,ip,raid,snmp,web,wifi]
  Downloading https://files.pythonhosted.org/packages/32/34/72f9202ad5b7ada314507a50b9ab1fb604d2f468b138679e0a4fedeb91fa/Glances-3.1.0.tar.gz (6.7MB)
     |██████████████████████████████▋ | 6.4MB 22kB/s eta 0:00:13
     .....
Successfully built glances psutil
Installing collected packages: psutil, glances
Successfully installed glances-3.1.0 psutil-5.6.2

```

观察到***Successfully installed glances-3.1.0 psutil-5.6.2***，即为安装成功。

# 使用glance

在命令行直接输入：

```
    glances

```

显示如下：  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1182.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1182.png)

可以看出，得到了系统的多方面指标。  
对于各个指标的情况，还可以加上对应参数，例如CPU各个核的情况：

```
glances --percpu 

```

则会显示所有CPU分别的状态：  
[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1151.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1151.png)

查看帮助：

```
 glances --help

```

[![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1124.png)](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-1124.png)

## glance其它功能

### web

还可以使用web查看状态：

```
[root@localhost  ~]#  glances -w
Glances Web User Interface started on http://0.0.0.0:61208/

```

然后用浏览器即可看到监控的状态。  
.

### influxdb

将监控数据输出到influxdb<sup id="fnref-588-2">[2](#fn-588-2)</sup>：  
首先安装influxdb module：

```
pip install influxdb

```

```
glances -t 5 --export influxdb

```

安装grafana：  
https://grafana.com/grafana/download?platform=linux

<div class="footnotes" role="doc-endnotes">- - - - - -

1. [Glances](https://pypi.org/project/Glances/) [↩︎](#fnref-588-1)
2. [InfluxDB & Glances](https://glances.readthedocs.io/en/stable/gw/influxdb.html) [↩︎](#fnref-588-2)

</div>