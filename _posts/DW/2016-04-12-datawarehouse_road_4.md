---
layout: article
title:  "数据仓库之路(四)"
categories: DW
toc: true
image:
    teaser: /teaser/hive_sql1.png
---

> 本文主要介绍在w公司的数据仓库之数据存储以及HA高可用


### 前言
&emsp;&emsp;主要想记录自己在w公司以来的经历和成长，挤出些时间对自己两年以来坐下总结，以便对下一步的道路进行更好的规划。
## 概述
w公司大数据在存储这部分主要是以HDFS为主。方便下游mapreduce,或者spark等引擎计算处理（由于为了节约管理成本以及维护成本，使用Cloudera manager进行管理）
