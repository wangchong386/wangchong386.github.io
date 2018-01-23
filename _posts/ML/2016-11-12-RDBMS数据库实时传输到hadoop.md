---
layout: article
title:  "kafka将关系数据库实时传输到hdfs中"
categories: kafka
toc: true
image:
    teaser: /teaser/Kafka_Connect_graphic.jpg
---

> 这篇内容主要介绍如何通过kafka将在RDMBS上的数据实时传输到hive数据仓库中，进而实时分析处理。




### 前言
&emsp;&emsp;由于w公司的交易数据和商品数据是保存在RDBMS数据库中，目前情况还是通过sqoop将oralce数据库中的数据每天批量导过来，在hive中做离线分析。所以一直想尝试将这关系数据库中这部分数据实时接过来。进而可以做实时分析处理

## 总体架构
![](/images/kafka/kafka1.png)