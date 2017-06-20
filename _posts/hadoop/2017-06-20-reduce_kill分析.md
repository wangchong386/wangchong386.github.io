---
layout: article
title:  "mapreduce中reduce一直被kill分析"
categories: hadoop
toc: true
image:
    teaser: /teaser/hive_sql1.png
---

> 本文主要介绍mapreduce出现类似死锁应该如何处理


### 前言
&emsp;&emsp;最近离线任务做了调整，为了缩短整个离线运行的时间。所以将大量的任务并行在同一时间点进行启动。结果就出现大量的reduce被kill掉。
## 现象
![reduce_kill图](/images/hadoop/reduce/reduce_kill1.png)

* __花费时间__：该任务从凌晨03:53:17一直跑到早上7点多，然后我手动将任务kill掉。Elapsed:	3hrs, 50mins, 56sec。（如果不手动kill掉该任务，在得不到资源的情况下，将会一直kill下去。以至于影响下游的数据运行）
* __map__:map数总共有185个，已经完成了183个，还有2个map一直抢不到资源，一直不能完成
* __reduce__:reduce数总共只有20，reduce已经kill掉46151个reduce

就以上这种现象一直持续了4个小时，一直不能完成。类似于死锁状态

![reduce_kill图](/images/hadoop/reduce/reduce_kill2.png)
## 原因分析
