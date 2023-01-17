---
id: 415
title: MongoDB升级注意事项
date: '2019-09-08T15:38:14+08:00'
author: Shuo
excerpt: 本篇文章介绍的“升级检查”，同样可以适用于其它数据库关于升级前的检查项。
layout: post
guid: 'http://codercoder.cn/?p=415'
permalink: /index.php/2019/09/checks-to-successfully-upgrade-mongodb/
views:
    - '422'
categories:
    - Tech
tags:
    - MongoDB
    - Tech
---

[7 Checks to Successfully Upgrade MongoDB Replica Set in Production](https://www.percona.com/community-blog/2018/08/29/7-checks-successfully-upgrade-mongodb-replica-set-production/)

**注：本篇文章介绍的“升级检查”，同样可以适用于其它数据库关于升级前的检查项。**  
![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0885-300x238.jpg)

# 1. 数据兼容性

 在release changes里面进行查看，是否有configurations, metadata, protocol version, validations, indexes or options等的一些变化

# 2. 驱动兼容性

 可以在网站上进行对比: The driver compatibility matrix lists the versions of MongoDB and language-specific versions that are compatible with those versions.

# 3. 更新的顺序

 不要跨多个版本进行升级，例如想要从3.4升级到4.0，请先升级到3.6

- Upgrade your current MongoDB version to the latest revision of current release series
- Go to check 1 and plan your upgrade

# 4. 设置兼容配置

 Feature Compatibility Flags， 可以[setFeatureCompatibilityVersion](https://docs.mongodb.com/manual/reference/command/setFeatureCompatibilityVersion/#dbcmd.setFeatureCompatibilityVersion) allows you to set the features those are incompatible with the previous versions ON or OFF.

# 5. 在测试环境先进行演练

 Now that you’re prepared, you can upgrade the MongoDB replica set in rolling fashion. This upgrade will involve the DB upgrade, driver upgrades and application code that is compatible with this driver version and DB version. To minimize the impact, upgrade secondaries in a replica set first, followed by stepping down a primary and its upgrade.

 Test your downgrade path:Prepare for downgrading in the test environment. A MongoDB replica set must follow the downgrade path from the path to be upgraded to, to the latest revision of the currently used release series.

**为使影响最小，先升级从节点secondaries ，再升级主节点primary。**

**制定降级方案。**

# 6. 预发布 Allow a Burn-in Period

 出现问题可以按照4进行配置（甚至是回滚）

# 7. Upgrade MongoDB Tools

- Upgrade the mongo shell to the same version as the MongoDB deployment
- Upgrade mongodump and mongorestore versions used in your backup and restore scripts.
- Use the same version of mongodump/mongorestore to backup/restore deployment of the same version on MongoDB.

 **使用相同版本的mongo shell，升级mongodump和mongorestore 工具。**