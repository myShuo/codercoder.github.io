---
id: 925
title: 'MySQL手记0 — MySQL安装方式'
date: '2020-04-12T21:57:20+08:00'
author: Shuo
excerpt: MySQL的安装，是了解数据库的第一步，安装时的一些内容，可以让我们理解MySQL的文件基本结构。
layout: post
guid: 'http://codercoder.cn/?p=925'
permalink: /index.php/2020/04/mysql-note-0-mysql-installation/
views:
    - '1880'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

```
    **MySQL的安装，是了解数据库的第一步，安装时的一些内容，可以让我们理解MySQL的文件基本结构。**https://dev.mysql.com/doc/refman/5.7/en/installation-layouts.html

```

![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-1248-300x111.png)

# 一、源码安装

```
    使用源码安装需要首先连接使用cmake安装MySQL所需要的依赖。例如：

```

```
yum install ncurses-devel -y
yum install libaio -y
yum install glibc-devel.i686 glibc-devel -y
yum install gcc gcc-c++ -y

```

当然，对于其中的版本也有依赖，例如MySQL8.0版本需要gcc版本为4.8以上，但是CentOS6版本通过yum安装gcc，最高只能到4.7，所以还需手动安装gcc4.8。然后再进行cmake的安装：

```
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql  \
-DMYSQL_DATADIR=/usr/local/mysql/data/ -DSYSCONFDIR=/etc/mysql \
-DWITH_INNOBASE_STORAGE_ENGINE=1   \
-DMYSQL_TCP_PORT=3306   \
-DENABLED_LOCAL_INFILE=1   \
-DEXTRA_CHARSETS=all   \
-DDEFAULT_CHARSET=utf8   \
-DDEFAULT_COLLATION=utf8_general_ci  \
-DWITH_BOOST=/tmp/boost_1_60_0/

```

```
make && make install

```

对于其中的选项，还需查阅MySQL的官方文档（https://dev.mysql.com/doc/refman/8.0/en/source-configuration-options.html）。

# 二、yum安装

```
    默认Linux系统yum安装大多为MySQL5.5的稳定版本，版本较老。通常在自己测试，为了方便部署时候，临时可以使用yum的方式进行安装。

```

# 三、二进制包

```
    比较建议使用二进制包进行安装，二进制包里面的目录结构较为清晰。除了方便运维管理之外，还方便DBA做自动化部署的流程。二进制包直接展示了MySQL文件的基本结构。例如Percona公司，就提供了二进制包，方便用户的下载使用，用户只需要使用自己的数据库配置文件，就能启动MySQL实例。

```

https://www.percona.com/doc/percona-server/LATEST/index.html  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-125-300x139.png)

# 四、Docker安装

```
    Docker安装通常也是在自己测试的时候使用，并且与yum不同的是，Docker能够很方便的指定版本，譬如想要尝试下MySQL的最新版本，使用Docker可以快速的部署进行使用。通常不建议在生产环境使用Docker进行MySQL的部署，虽然其部署方便快捷，并且重启迁移也很便捷，但是考虑到数据库往往是业务敏感性的，稍微一点的波动或者延迟，可能就会影响到上层应用，所以数据库通常单独拎出来进行部署。
    对于Docker部署MySQL，后续会进行介绍，方便各位快速上手使用MySQL。

```

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)