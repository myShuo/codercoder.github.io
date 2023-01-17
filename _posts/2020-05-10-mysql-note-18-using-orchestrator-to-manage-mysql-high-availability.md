---
id: 1073
title: 'MySQL手记18 &#8212; MySQL高可用及复制管理工具Orchestrator'
date: '2020-05-10T15:54:50+08:00'
author: Shuo
excerpt: Orchestrator是一款提供页面和命令行和MySQL高可用和复制拓扑关系管理工具，Github也使用了Orchestrator进行了MySQL高可用拓扑结构的管理。还能进行主从等切换，只需在页面上进行节点的拖拽，就能完成切换。
layout: post
guid: 'http://codercoder.cn/?p=1073'
image: http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1024.png
permalink: /index.php/2020/05/mysql-note-18-using-orchestrator-to-manage-mysql-high-availability/
views:
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
    - '124327'
enclosure:
    - "http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1012.mp4\n4544782\nvideo/mp4\n"
    - "http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1082.mp4\n5927056\nvideo/mp4\n"
categories:
    - Tech
tags:
    - MySQL手记
    - Tech
---

 Orchestrator是一款提供页面和命令行和MySQL高可用和复制拓扑关系管理工具，可以用其方便的管理和调整拓扑结构。著名交友网站“Github”也使用了Orchestrator进行了MySQL高可用拓扑结构的管理。

 除了直观的展示拓扑关系外，还能进行主从、双主等花式切换，只需在页面上进行节点的拖拽，就能完成切换，可以说非常直观有效。  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1024.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1024.png)

# 一、安装部署

## （1）下载安装

下载Orchestrator的安装包，解压后编辑**orchestrator.conf.json**文件：

```
...
"MySQLOrchestratorHost": "127.0.0.1",
"MySQLOrchestratorPort": 3306,
"MySQLOrchestratorDatabase": "orchestrator",
"MySQLOrchestratorUser": "orchestrator",
"MySQLOrchestratorPassword": "orch_backend_password",
...

```

## （2）授权

 Orchestrator的元数据信息需要使用MySQL或者Sqlite进行管理，所以需要在数据库中授予相应的权限.

Orchestrator所需的元数据库建库：

```
CREATE DATABASE IF NOT EXISTS orchestrator;
CREATE USER 'orchestrator'@'127.0.0.1' IDENTIFIED BY 'orch_backend_password';
GRANT ALL PRIVILEGES ON `orchestrator`.* TO 'orchestrator'@'127.0.0.1';

```

添加的实例数据源授权：

```
CREATE USER 'orchestrator'@'orch_host' IDENTIFIED BY 'orch_topology_password';
GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orchestrator'@'orch_host';
GRANT SELECT ON mysql.slave_master_info TO 'orchestrator'@'orch_host';
GRANT SELECT ON ndbinfo.processes TO 'orchestrator'@'orch_host'; -- Only for NDB Cluster

```

权限说明：  
 *REPLICATION SLAVE is required for SHOW SLAVE HOSTS, and for scanning binary logs in favor of Pseudo GTID RELOAD required for RESET SLAVE operation PROCESS required to see replica processes in SHOW PROCESSLIST On MySQL 5.6 and above, and if using master\_info\_repository = ‘TABLE’, let orchestrator have access to the mysql.slave\_master\_info table. This will allow orchestrator to grab replication credentials if need be.*

## （3）启动

```
cd /usr/local/orchestrator &amp;&amp; ./orchestrator --debug --config=/path/to/config.file http

```

参考：https://github.com/openark/orchestrator/blob/master/docs/install.md

# 二、使用介绍

## 2.1 基础面板

 安装完成后，打开页面（默认端口为3000），Home —-> About中有Orchestrator的介绍：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1042.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1042.png)

选择Cluster —-> Dashboard，即可看到集群的拓扑情况。

### （1）展示

 部署好一个主从集群，添加到Orchestrator中：在Orchestrator中选择Cluster—->Discover：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1046.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1046.png)

   
填写数据库的信息：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1067.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1067.png)

   
 若填写的是从库的信息，则会自动把主库实例也添加进来（show slave status得到的主库的信息），可以看到新添加的集群的Instances = 2：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1050.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1050.png)  
   
   
 在Cluster —-> Dashboard中，选择需要管理的集群，进入后，即可看到拓扑情况：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1092.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1092.png)

   
 **点击各个节点上的设置按钮，**可以看到节点连接的详细信息，并且可以修改其中的部分配置：  
[![](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1090.png)](http://codercoder.cn/wp-content/uploads/2020/05/2020-05-1090.png)

## 2.2 基础测试

### （1）主从 —-> 双主

 **Orchestrator在切换时，并不是暴力的进行切换，而是会去避免出现数据不一致的情况。**例如若需要将主从 —-> 双主的时候，会提示需要先将原来的从库设置为read-only，才能切换为双主：

<div class="aiovg-player-container" style="max-width: 100%;"><div class="aiovg-player" style="padding-bottom: 56.25%;"><iframe allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen="" frameborder="0" height="315" mozallowfullscreen="" scrolling="no" src="http://codercoder.cn/index.php/player-embed/id/1073/?mp4=http%3A%2F%2Fcodercoder.cn%2Fwp-content%2Fuploads%2F2020%2F05%2F2020-05-1012.mp4" webkitallowfullscreen="" width="560"></iframe></div></div> **切换后，需要重新在Cluster —-> Dashboard中，选择集群，即可看到已经修改为了co-master集群**。  
 若需要交换主从，则需要进行再次切换，主库上stop slave后，将原主库拖拽到从库后，即可​。

## （2）同级从库 —-> 级联主从

 即B,C均为A的从库，现把C切换为B的从库，得到A —-> B —-> C的集群架构：

<div class="aiovg-player-container" style="max-width: 100%;"><div class="aiovg-player" style="padding-bottom: 56.25%;"><iframe allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen="" frameborder="0" height="315" mozallowfullscreen="" scrolling="no" src="http://codercoder.cn/index.php/player-embed/id/1073/?mp4=http%3A%2F%2Fcodercoder.cn%2Fwp-content%2Fuploads%2F2020%2F05%2F2020-05-1082.mp4" webkitallowfullscreen="" width="560"></iframe></div></div> 以上介绍了两种常见的情况，当然还有很多花样的切换方式，均可进行拖拽完成，Orchestrator支持命令行的方式进行切换，方便更加自动化的切换维护。

# 三、命令行

## 3.1 列出集群

```
bash-4.4# orchestrator-client -c clusters
2bd38a52527e:5502
3a95cdf097fd:4407
5daa942fb07c:4407
70b353672cdc:5501

```

## 3.2 查看集群状态

```
bash-4.4# orchestrator-client -c replication-analysis
5daa942fb07c:4407 (cluster 5daa942fb07c:4407): ErrantGTIDStructureWarning
2bd38a52527e:5502 (cluster 2bd38a52527e:5502): DeadMasterWithoutSlaves
3a95cdf097fd:4407 (cluster 3a95cdf097fd:4407): DeadMasterWithoutSlaves

```

## 3.3 查看拓扑关系

```
bash-4.4# orchestrator-client -c topology-tabulated -i 5daa942fb07c:4407
5daa942fb07c:4407  |0s|ok|8.0.18|rw|ROW|&gt;&gt;,GTID
+ b96697e765cc:4408|0s|ok|8.0.18|rw|ROW|&gt;&gt;,GTID

```

## 3.4 主-主结构

```
orchestrator-client -c make-co-master -i test1:3307

```

## 3.5 同级slave变为自己的slave

```
orchestrator-client -c take-siblings -i test3:3307

```

## 3.6 将实例转换为自己主人的主人，切换两个：take-master

```
orchestrator-client -c take-master -i test3:3307

```

## 3.7 将slave在拓扑上向上移动一级

对应web上的是在Classic Model下进行拖动：move-up

```
bash-4.4# orchestrator-client -c move-up -i b96697e765cc:4408 -d 5daa942fb07c:4407
b96697e765cc:4408&lt;5daa942fb07c:4407

```

参考：  
https://github.com/openark/orchestrator/blob/master/docs/executing-via-command-line.md

 相信看了那么多图，觉得想要蠢蠢欲动想要操作一把，原来数据库的维护也可以如此简单直观，可以说是很贴心的一个工具了。

欢迎关注公众号：**朔的话**：  
![](http://codercoder.cn/wp-content/uploads/2020/04/2020-04-2693.jpg)