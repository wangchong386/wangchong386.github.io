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

