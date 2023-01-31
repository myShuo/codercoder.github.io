---
id: 1196
title: 'Redis ERR Protocol error: too big inline request'
date: '2021-09-24T18:34:40+08:00'
author: Shuo
excerpt: 'Redis ERR Protocol error: too big inline request'
layout: post
guid: 'http://codercoder.cn/?p=1196'
permalink: /index.php/2021/09/2021-09-24-redis-too-big-inline-request
categories:
    - Tech
    - Fell-in-pit
tags:
    - Redis
    - Tech
    - Fell-in-pit
---
# How to repeat?
在使用redis-cli set大key的时候，由于一行太长，报错：
```
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
All data transferred. Waiting for the last reply...
ERR Protocol error: too big inline request
ERR server closed connection
```

# How to fix?
使用其它client, 例如python
    
```
# from redis import StrictRedis, ConnectionPool
import redis

def getConnectFromPool(host,port,password=''):
    try:
        pool = redis.ConnectionPool(host=host, port=port, password = password)
        conn = redis.Redis(connection_pool=pool)
    except Exception as e:
        print ("Error:connect to redis %s:%s failed." %(host,port))
        return 0
    return conn

if __name__ == '__main__':
    redis_conn = getConnectFromPool('172.12.1.100', 6379, password="123aaa")
    file = "/home/admin/rediskey"
    keyname = "galaxy_dcaeg325g24t"
    with open(file) as fr:
        rows = fr.readlines()[0]
    
    print(redis_conn.set(keyname, rows))

```
