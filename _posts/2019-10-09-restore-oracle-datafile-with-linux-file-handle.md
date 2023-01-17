---
id: 698
title: Linux下基于文件句柄的方式恢复oracle数据文件
date: '2019-10-09T16:17:45+08:00'
author: rondon
layout: post
guid: 'http://codercoder.cn/?p=698'
permalink: /index.php/2019/10/restore-oracle-datafile-with-linux-file-handle/
views:
    - '2849'
    - '2849'
bigfa_ding:
    - '2'
categories:
    - Tech
tags:
    - Oracle
    - Tech
---

# 前言

![](http://codercoder.cn/wp-content/uploads/2019/10/2019-10-0956-300x225.jpg)  
对于Linux下误删除Oracle数据文件，可以通过句柄方式来快速恢复数据文件，但是需要存在一定的前提，数据库与服务器均未发生重启，这样可以快速恢复。不然就需要通过RMAN进行restore/recover。**<u>*重启需谨慎，关库有风险！！！*</u>**

# 模拟删除数据文件

```
[oracle@dba1 ~]$ cd /home/oracle/app/oradata/orcl/ORCL/datafile/
[oracle@dba1 datafile]$ rm -rf o1_mf_tbs_face_fnrowob1_.dbf

```

# 查看数据库状态

```
SQL> select open_mode from v$database;
OPEN_MODE
------------------------------------------------------------
READ WRITE

```

# 获取DBW0进程号

通过进程号来获取到各Oracle文件的句柄号

```
[oracle@dba1 ~]$ ps -ef|grep -v grep |grep dbw0
oracle   16678     1  0 Jul16 ?        00:00:06 ora_dbw0_dssc
[oracle@dba1 ~]$ cd /proc/16678/fd
[oracle@dba1 fd]$ ls -l
total 0
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 0 -> /dev/null
l-wx------ 1 oracle oinstall 64 Jul 17 09:23 1 -> /dev/null
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 10 -> /home/oracle/app/oracle/product/11.2.0/dbhome_1/dbs/lkDSSC
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 11 -> /home/oracle/app/oracle/product/11.2.0/dbhome_1/rdbms/mesg/oraus.msb
l-wx------ 1 oracle oinstall 64 Jul 17 09:23 2 -> /dev/null
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 256 -> /home/oracle/app/oradata/orcl/control01.ctl
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 257 -> /home/oracle/app/oradata/orcl/control02.ctl
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 258 -> /home/oracle/app/oradata/orcl/control03.ctl
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 259 -> /home/oracle/app/oradata/orcl/system01.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 260 -> /home/oracle/app/oradata/orcl/sysaux01.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 261 -> /home/oracle/app/oradata/orcl/undotbs01.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 262 -> /home/oracle/app/oradata/orcl/users01.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 263 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_fmd5mfgl_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 264 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_idx_fmd5mhck_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 265 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_bigd_fmd5mjn8_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 266 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_bigd_fmd5mlhj_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 267 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_test_fmd5mmpc_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 268 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_test_fmd5mnr3_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 269 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_fmdsdobv_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 270 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_idx_fmdsj5dg_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 271 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_test_fmdsm22d_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 272 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_test_fmdsp0k8_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 273 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_bigd_fmdsryq1_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 274 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_bigd_fmdsxpv7_.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 275 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fmttspfy_.dbf
<u><b><i>**lrwx------ 1 oracle oinstall 64 Jul 17 09:23 276 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf (deleted)**
</i></b></u>lrwx------ 1 oracle oinstall 64 Jul 17 09:23 277 -> /home/oracle/app/oradata/orcl/undotbs03.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 278 -> /home/oracle/app/oradata/orcl/temp01.dbf
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 279 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_temp_fmc6m2q4_.tmp
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 280 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_temp_fmc6m2ts_.tmp
lrwx------ 1 oracle oinstall 64 Jul 17 09:23 281 -> /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_temp_fmc6m2vz_.tmp
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 3 -> /dev/null
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 4 -> /dev/null
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 5 -> /dev/null
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 6 -> /home/oracle/app/oracle/product/11.2.0/dbhome_1/rdbms/mesg/oraus.msb
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 7 -> /proc/16678/fd
lr-x------ 1 oracle oinstall 64 Jul 17 09:23 8 -> /dev/zero

```

# 数据文件已提示删除

Oracle数据文件/home/oracle/app/oradata/orcl/ORCL/datafile/o1\_mf\_tbs\_face\_fnrowob1\_.dbf已提示被删除，物理文件已不存在

```
[oracle@dba1 fd]$ ls -l /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf
ls: cannot access /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf: No such file or directory

```

# 还原数据文件

通过句柄号276，进行复制

```
[oracle@dba1 fd]$ cp /proc/16678/fd/276 /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf
[oracle@dba1 fd]$ ls -l /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf
-rw-r----- 1 oracle oinstall 10737426432 Jul 17 09:37 /home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf

```

# 数据文件的offline与online

确认复制完成后在去offline操作，修复好之后再去online操作

```
SQL> alter database datafile '/home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf' offline;
数据库已更改。
SQL> recover datafile '/home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf';
完成介质恢复。
SQL> alter database datafile '/home/oracle/app/oradata/orcl/ORCL/datafile/o1_mf_tbs_face_fnrowob1_.dbf' online;
数据库已更改。

```

# 检查数据库告警日志

```
[oracle@dba1 fd]$tail -f /home/oracle/alert_orcl.log

```

# 后续

其实通过句柄方式恢复过程中最重要的是数据文件的offline与online的顺序，一定先offline数据文件后执行recover在去online，如果顺序发生错误，那么这种方式基本就行不通了。需要考虑rman方式进行restore与recover。任何时候重启数据库与服务器都需要谨慎，避免带来更为严重的故障。