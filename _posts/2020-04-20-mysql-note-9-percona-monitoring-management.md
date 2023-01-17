---
id: 952
title: 'MySQL手记9 &#8212; Percona Monitoring Management（PMM监控）'
date: '2020-04-20T21:02:52+08:00'
author: Shuo
excerpt: PMM分为两个部分：Client和Server。使用pmm-admin，可以快捷添加和删除MySQL、Mongodb、Redis实例。
layout: post
guid: 'http://codercoder.cn/?p=952'
image: http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2092.png
permalink: /index.php/2020/04/mysql-note-9-percona-monitoring-management/
views:
    - '7843'
    - '7843'
    - '7843'
    - '7843'
    - '7843'
    - '7843'
    - '7843'
    - '7843'
    - '7843'
bigfa_ding:
    - '1'
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

# 一、主要技术方案

## 1.1 PMM

 Percona Monitoring Management，下称PMM，Percona的监控管理系统，详情：  
https://www.percona.com/doc/percona-monitoring-and-management/index.html  
PMM分为两个部分：Client和Server  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2092.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2092.png)

### （1）Clinet端：

 由各种exporter（mysql\_exporter、mongodb\_exporter、node\_exporter、redis\_exporter…）外加queries工具、pmm-admin工具组成。  
 **pmm-admin**为管理工具，例如用以添加、删除所监控数据库实例  
 **pmm-mysql-queries-0**  
 **pmm-mongodb-queries-0**

 这两者用以收集MySQL和MongoDB的query performance发送到QAN API  
(https://www.percona.com/doc/percona-monitoring-and-management/conf-mysql.html  
)  
*（其中query的监控需要数据库按照文档进行配置，否则不能进行query监控）*  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2092-1.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2092-1.png)

### （2）Server端：

**Query Analytics：**  
 **QAN API：**接收QAN agent传来的数据  
 **QAN Web App：**用来展示Query Analytics data的wen应用  
**Metrics Monitor：**展示监控数据metrics：  
 **Percona Dashboard：**Percona一系列Grafana的dashboards  
 **Consul：**为提供Prometheus hosts的操作（添加、删除）等  
 **Prometheus：**连接exporters并聚合其传入的metrics数据  
 **Grafana：**用以展示Prometheus的数据  
**Orchestrator：**MySQL复制拓扑结构的可视化工具（后续将单独介绍此工具）

 虽然新版本的PMM默认为打开，最好在启动时候指定打开Orchestrator：

```
docker ... -e ORCHESTRATOR_ENABLED=true -e ORCHESTRATOR_USER=root -e ORCHESTRATOR_PASSWORD=root ...

```

Orchestrator地址：http://127.0.0.1/orchestrator

通过如下页面进行添加：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2042.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2042.png)

添加后，可以看到添加的实例的拓扑关系：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2071-2.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2071-2.png)

此外，还可以修改MySQL复制的相关配置：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2070.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2070.png)

## 1.2 监控实例

 监控实例的添加和删除使用pmm-admin工具进行管理。pmm-admin需要使用root用户运行：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2095.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2095.png)  
pmm-admin –help可以看到使用详情。

## 1.3 PMM的安装

 可使用docker进行安装：

```
 docker run -d -e METRICS_RETENTION=1440h --name pmm-server-1-2020-2 percona/pmm-server:1

```

```
可以看到，使用dock er安装时，可以定制化一些需要的配置，例如：

```

```
-e METRICS_RETENTION=1440h：metrics保留1440小时

```

 还有其它的配置可供挑选：https://www.percona.com/doc/percona-monitoring-and-management/glossary.option.html

# 二、添加/删除实例

## 2.1 添加MySQL实例

```
pmm-admin add mysql --user testuser --password passwd --host 192.168.0.1 --port 3306 testdb-master

```

添加完成后，使用pmm-admin工具显示监控的实例列表信息：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2070-1.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2070-1.png)

## 2.2 添加Mongodb实例

```
  pmm-admin add mongodb --uri mongodb://admin:password@172.27.2.1:27017/admin mongodb-001[linux:metrics] OK, already monitoring this system.[mongodb:metrics] OK, now monitoring MongoDB metrics using URI admin:***@172.27.2.1:27017/admin[mongodb:queries] OK, now monitoring MongoDB queries using URI admin:***@172.27.2.1:27017/admin[mongodb:queries] It is required for correct operation that profiling of monitored MongoDB databases be enabled.[mongodb:queries] Note that profiling is not enabled by default because it may reduce the performance of your MongoDB server.[mongodb:queries] For more information read PMM documentation (https://www.percona.com/doc/percona-monitoring-and-management/conf-mongodb.html).

```

```
​同理查看pmm-admin list：

```

[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2089.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2089.png)

## 2.3 添加Redis实例

### （1）redis\_exporter

 由于PMM本身是不自带Redis监控的，但是其中的Prometheus可以收集到redis的数据，所以可以使用pmm-admin中的external:service来添加Redis实例（例如：https://github.com/oliver006/redis\_exporter）:  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2043.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2043.png)

**使用redis\_exporter收集metrics**，并暴露端口给Prometheus：

```
 ./redis_exporter  -redis.addr 172.27.0.1:7000 --redis.password redisPasswd  -web.listen-address :9121 

```

默认的端口为9121。

### （2）PMM添加Redis实例

```
pmm-admin add external:service  S-Redis-master-001 --service-port=9121

```

其中–service-port=9121即为上步骤中redis\_exporter暴露的端口号。

### （3）查看redis实例添加情况

[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2071.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2071.png)

# 三、 删除监控实例

 使用pmm-admin remove，即可方便地删除监控的节点：

```
 pmm-admin remove mysql:metrics tuya_testdb-ind-master

 OK, removed MySQL metrics mysql-3306-master from monitoring.

```

# 四、页面上进行添加/删除实例

 PMM的dashboards上提供了页面上的操作，详情如下：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2094.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2094.png)

点击对应的添加项，可以添加不同类型的实例：  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2071-1.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2071-1.png)

 至此，PMM基本介绍完毕，实际环境中，需要对于很多细节进行修改优化，以便更加适用，并降低监控对于​数据库实例的影响。  
 对于监控数据来说，收集得越仔细，就更容易排查问题，但是也对数据库实例的负载造成影响，所以需要进行权衡，选择更为适宜的比例进行监控​。  
[![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2088.png)](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2088.png)

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)