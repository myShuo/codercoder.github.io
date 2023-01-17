---
id: 1111
title: 'MySQL手记19 &#8212; MySQL代理工具ProxySQL'
date: '2020-05-24T18:04:29+08:00'
author: Shuo
excerpt: 本文介绍了ProxySQL这一强大且灵活的MySQL代理，需要按需进行测试，包括对于连接的转发是否均衡、节点宕机是否能够将连接发到其它节点、是否能够承受住非常大量的连接、自身的高可用等。数据库加了一层代理，还需要考虑到代理与实例间的延迟，很灵敏的业务是否使用。对于MySQL的集群，需要稳定、灵活且方便的进行管理，包括之前介绍的MySQL高可用集群拓扑结构管理工具Orchestrator，本篇的ProxySQL等，都是集群运维中的一个部分，需要我们谨慎的完成管理。
layout: post
guid: 'http://codercoder.cn/?p=1111'
permalink: /index.php/2020/05/mysql-note19-mysql-proxy-tool-proxysql/
views:
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
    - '55755'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 介绍

ProxySQL是一个MySQL集群架构的代理，由于其本身支持高可用，常被用作包括Master-Slave，MGR在内的集群结构的代理。  
https://proxysql.com/documentation/  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-2481.png)  
图片来源：https://www.percona.com/blog/2017/07/20/where-do-i-put-proxysql/

# 一、安装部署

proxySQL支持直接使用rpm包安装，所以过程较为简单。  
在官网上下载rom包：  
https://proxysql.com/documentation/installing-proxysql/

```
[root@node1 proxysql]# rpm -ivh proxysql-2.0.12-1-centos7.x86_64.rpm
warning: proxysql-2.0.12-1-centos7.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 79953b49: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:proxysql-2.0.12-1                warning: group proxysql does not exist - using root
warning: group proxysql does not exist - using root
################################# [100%]
Created symlink from /etc/systemd/system/multi-user.target.wants/proxysql.service to /etc/systemd/system/proxysql.service.

```

其中：  
 默认配置文件：/etc/proxysql.cnf  
 默认数据路径：/var/lib/proxy  
 ProxySQL默认管理端口：6032  
 ProxySQL默认client端口：6033

启动：  
proxySQL可以先启动，再进行配置，所以可以在启动后添加相应的配置项：

```
systemctl start proxysql

```

查看启动状态：

```
systemctl status proxysql -l
● proxysql.service - LSB: High Performance Advanced Proxy for MySQL
   Loaded: loaded (/etc/rc.d/init.d/proxysql; bad; vendor preset: disabled)
   Active: active (running) since Fri 2020-05-22 12:08:10 UTC; 59s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 62807 ExecStart=/etc/rc.d/init.d/proxysql start (code=exited, status=0/SUCCESS)
   CGroup: /docker/b0d7120efb2f7b0ae3a5344d6e1ce509d1e841ad238f2fc1ae4ca22ca5358aff/system.slice/proxysql.service
           ├─62811 proxysql -c /etc/proxysql.cnf -D /var/lib/proxysql
           └─62812 proxysql -c /etc/proxysql.cnf -D /var/lib/proxysql

May 22 12:08:10 dev-mysql-248110 systemd[1]: Starting LSB: High Performance Advanced Proxy for MySQL...
May 22 12:08:10 dev-mysql-248110 proxysql[62807]: Starting ProxySQL: 2020-05-22 12:08:10 [INFO] Using config file /etc/proxysql.cnf
May 22 12:08:10 dev-mysql-248110 proxysql[62807]: DONE!
May 22 12:08:10 dev-mysql-248110 systemd[1]: Started LSB: High Performance Advanced Proxy for MySQL.

```

登录proxySQL进行管理（默认只能本地127.0.0.1登录，由于proxySQL所需的用户权限要求较高，防止出现安全问题）：  
 用户名/密码：admin:admin  
 web页面开关（默认用户名密码：stats/stats）：

```
mysql> show variables like '%web%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| admin-web_enabled | false |
| admin-web_port    | 6080  |
+-------------------+-------+
2 rows in set (0.00 sec)

mysql> set admin-web_enabled=1;
Query OK, 1 row affected (0.00 sec)

mysql> load admin variables to run;
Query OK, 0 rows affected (0.00 sec)

```

## ProxySQL的结构

**第一层：RUNTIME**  
 目前在运行中的ProxySQL的配置，RUNTIME的配置是不能修改的，可以将其当作一个状态。修改需要从MEMORY层加载。

**第二层：MEMORY**  
 已在内存中的配置，为数据加载的中间层。ProxySQL启动时，从磁盘中读取配置文件，放到内存中，再将内存中的数据加载到RUNTIME层。

**第三层：DISK/CONFIG FILE**  
 ProxySQL在磁盘上为SQLite数据库，所以需要将在内存中的数据，加载到磁盘中，否则重启proxySQL后会丢失内存中的信息。  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-2479.png)

   
例如需要修改MySQL的变量，对应的指令为：

```
LOAD MYSQL VARIABLES TO RUNTIME;
 SAVE MYSQL VARIABLES TO DISK;

```

# 三、基本信息

登录管理后台，执行：

```
mysql> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)


mysql> show tables;
+--------------------------------------------+
| tables                                     |
+--------------------------------------------+
| global_variables                           |
| mysql_collations                           |
| mysql_group_replication_hostgroups         |
| mysql_query_rules                          |
| mysql_query_rules_fast_routing             |
| mysql_replication_hostgroups               |
| mysql_servers                              |
| mysql_users                                |
| proxysql_servers                           |
| runtime_checksums_values                   |
| runtime_global_variables                   |
| runtime_mysql_group_replication_hostgroups |
| runtime_mysql_query_rules                  |
| runtime_mysql_query_rules_fast_routing     |
| runtime_mysql_replication_hostgroups       |
| runtime_mysql_servers                      |
| runtime_mysql_users                        |
| runtime_proxysql_servers                   |
| runtime_scheduler                          |
| scheduler                                  |
+--------------------------------------------+

```

## 常用的三类表

### （1）变量：

ProxySQL作为一个代理，其必然会包含许多MySQL的参数：  
 **数据库相关：**max\_connection、max\_allowed\_packed、wait\_timeout、字符集等等。  
 **限制相关：**连接请求次数、数据包大小…  
 **检测相关：**连接的心跳状态、失败情况…  
在配置上极为灵活，可以通过global\_variables表进行查看，修改后，别忘记：

```
LOAD MYSQL VARIABLES TO RUNTIME;
 SAVE MYSQL VARIABLES TO DISK;

```

修改的值加载到runtime，并保存在磁盘中。

### （2）MySQL实例、用户：

**添加MySQL的实例：**

```
mysql> insert into mysql_servers (hostgroup_id, hostname, port) values(1, '172.16.120.209', 5501);

mysql> select * from mysql_servers \G
*************************** 1. row ***************************
      hostgroup_id: 1
          hostname: 172.16.120.209
              port: 5501
            status: OFFLINE_HARD
            weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
    max_latency_ms: 0
            comment:

```

可以从该表中，看到对应实例的运行情况：**报错host、status、weight**等。

修改MySQL的实例信息:

```
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

```

**添加MySQL用户：**

```
INSERT INTO mysql_users(username,password,default_hostgroup) VALUES ('proxyuser','proxypasswd',0);

```

   
直接查看该表，密码是明文的，不安全，所以需要进行密码的加密：

```
mysql> save mysql users from runtime;

 mysql> select * from mysql_users \G
*************************** 1. row ***************************
              username: root
              password: *FE1E37A7390CE06FF73D46CE034FE0C9A59A9681
                active: 1
              use_ssl: 0
    default_hostgroup: 1
        default_schema:
        schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
              backend: 1
              frontend: 1
      max_connections: 10000

```

   
**transaction\_persistent**，同一个事务的查询，必须分发在一个节点，1.4版本后默认为打开：  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-2467.png)

https://github.com/sysown/proxysql/commit/b89de59f06261eac038c89ed3be5c083ceadfaa8

修改MySQL的用户信息，修改后，同样别忘记：

```
LOAD MYSQL USERS TO RUNTIME;
 SAVE MYSQL USERS TO DISK;

```

### （3）scheduler定时任务

 对于集群的管理，我们还可以写脚本，来完成自动的节点上下线，切换为从节点等的一些操作。这个时候，就可以使用scheduler表进行配置：

```
insert into scheduler(id, active, interval_ms, filename, arg1, arg2, arg3, arg4) values(1, 1, 3000, '/var/lib/proxysql/gr_mw_mode_sw_cheker.sh', 1, 2, 1, '/var/lib/proxysql/checker.log');

mysql> select * from scheduler \G
*************************** 1. row ***************************
        id: 1
    active: 1
interval_ms: 3000
  filename: /var/lib/proxysql/gr_mw_mode_sw_cheker.sh
      arg1: 1
      arg2: 1
      arg3: 1
      arg4: /var/lib/proxysql/checker.log
      arg5: NULL
    comment:

```

按照文档的介绍：  
![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-2466.png)

 **id：**scheduler的全剧唯一job id  
 **active**：1为在运行中，否则即没有激活  
 **interval\_ms**：执行周期，单位毫秒，最小值为100ms  
 **filename**：可执行脚本的文件绝对路径  
 **arg1~arg5**：可以传递给脚本的参数  
 **comment**：备注

# 总结：

 本文简单介绍了ProxySQL这一强大且灵活的MySQL代理，需要按需进行测试，包括对于连接的转发是否均衡、节点宕机是否能够将连接发到其它节点、是否能够承受住非常大量的连接、自身的高可用等。  
 数据库加了一层代理，还需要考虑到代理与实例间的延迟，对延迟敏感的业务是否适用。对于MySQL的集群，需要稳定、灵活且方便的进行管理，包括之前介绍的MySQL高可用集群拓扑结构管理工具Orchestrator，本篇的ProxySQL等，都是集群运维中的一个部分，需要我们谨慎的完成管理。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)