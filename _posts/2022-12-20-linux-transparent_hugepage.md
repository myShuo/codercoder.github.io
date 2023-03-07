---
id: 1207
title: 'linux-transparent_hugepage'
date: '2022-12-20T18:34:40+08:00'
author: Shuo
excerpt: 'What is transparent_hugepage in Linux and why we should disable it in database instance?'
layout: post
guid: 'http://codercoder.cn/?p=1207'
permalink: /index.php/2022/12/2022-12-20-linux-transparent_hugepage
categories:
    - Tech
tags:
    - Linux
    - Tech
    - Fell-in-pit
---


# 透明大页说明
Transparent Huge Pages (THP) is a Linux memory management system that reduces the overhead of Translation Lookaside Buffer (TLB) lookups on machines with large amounts of memory by using larger memory pages.
**THP在Linux系统中，通过内存大页的方式，降低TLB(Translation Lookaside Buffer)查询大量的内存**

# 关闭
## 1. why we should disable it in database instance?
However, database workloads often perform poorly with THP enabled, because they tend to have sparse rather than contiguous memory access patterns. When running MongoDB on Linux, THP should be disabled for best performance.
由于数据库环境，包括MySQL、MongoDB、Redis、TiDB等，由于往往需要获取的页比较分散，而不是连续的页，所以THP可能会带来性能上的浪费，建议将其关闭。

[Redis-Disable-THP](https://access.redhat.com/solutions/46111)

[MongoDB-Disable-THP](https://www.mongodb.com/docs/v5.0/tutorial/transparent-huge-pages/)

## 2. 检查THP的启用状态
```
[root@localhost ~]# cat /sys/kernel/mm/transparent_hugepage/defrag
[always] madvise never
[root@localhost ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```
**注意：**

对于不同版本的操作系统，使用不同的THP文件路径！！

**查看系统的内存使用情况:**
```
cat /proc/meminfo && grep AnonHugePages /proc/meminfo 
```


## 3. 关闭THP
### （1）临时关闭
```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### （2）开机自动关闭
**1）添加systemd服务**
新建文件：/etc/systemd/system/disable-transparent-huge-pages.service
```
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never | tee /sys/kernel/mm/transparent_hugepage/enabled > /dev/null'

[Install]
WantedBy=basic.target

```
**注意：**
对于不同版本的操作系统，使用不同的THP文件路径！！

**2）Reload systemd unit files**
```
sudo systemctl daemon-reload
```

**3）Start the service**
```
sudo systemctl start disable-transparent-huge-pages
```
重启后，再次查看状态：
```
[root@localhost ~]# cat /sys/kernel/mm/transparent_hugepage/defrag
[always] madvise never
[root@localhost ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```

**4）Configure your operating system to run it on boot**
打开开机自启动：
```
sudo systemctl enable disable-transparent-huge-pages
```

**5）Customize tuned / ktune profile, if applicable**

### Using tuned and ktune
#### (1) Red Hat/CentOS 6
**1. Create a new profile**
Create a new profile from an existing profile by copying the relevant directory. This example uses the virtual-guest profile as the base, and uses virtual-guest-no-thp as the new profile:
```
sudo cp -r /etc/tune-profiles/virtual-guest /etc/tune-profiles/virtual-guest-no-thp 
```

**2. Edit ktune.sh**
Edit /etc/tune-profiles/virtual-guest-no-thp/ktune.sh and change the set_transparent_hugepages setting to the following:
```
set_transparent_hugepages never 
```

**3. Enable the new profile**
Enable the new profile:
```
sudo tuned-adm profile virtual-guest-no-thp 
```

#### (2) Red Hat/CentOS 7 and 8

**1. Create a new profile**

Create a new directory to hold the custom tuned profile. This example inherits from the existing virtual-guest profile, and uses virtual-guest-no-thp as the new profile:
```
sudo mkdir /etc/tuned/virtual-guest-no-thp 
``` 

**2. Edit tuned.conf**

Create and edit /etc/tuned/virtual-guest-no-thp/tuned.conf so that it contains the following:
```
[main]
include=virtual-guest 

[vm]
transparent_hugepages=never 
```
This example inherits from the existing virtual-guest profile. Select the profile most appropriate for your system.

**3. Enable the new profile**

Enable the new profile:
```
sudo tuned-adm profile virtual-guest-no-thp 

```
