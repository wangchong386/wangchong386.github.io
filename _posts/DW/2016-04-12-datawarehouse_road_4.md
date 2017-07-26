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
&emsp;&emsp;首先介绍下w公司大数据平台规模，由于w公司是B2B模式的互联网电商平台，跟很多大的公司不管是集群规模或者数据量都没办法相提并论。但是还是想说说：
目前负责约100台服务器：Dell PowerEdge R730xd。32Core 64G内存，2TB/3TB X 6 300GB X 1

|名称  |规模 |
集群管理系统|CDH5.9.0
已使用的组件|HDFS、YARN、Flume、Spark、Oozie、Sqoop、Kafka、Hive、Hue、Zookeeper、Impala
hadoop机器数|45个
总数据量|420TB
数据日增量|2T
日任务job数|10000个
核心任务job数|4000个
