---
layout: article
title:  "sparkstreaming与kafka集成分析"
categories: hadoop
toc: true
image:
    teaser: /teaser/spark.png
---

> 这篇内容主要介绍Spark Streaming整体架构以及原理

> sparkstreaming是核心Spark API的扩展，可以实现实时数据流的可扩展，高吞吐，容错的流处理。数据源可以从Flume、kafka、kinesis 以及TCP sockets来获取。并且可以使用map,reduce,joinhe windows等高级功能开进行处理。最后处理好的数据可以存储到文件系统，数据库以及实时仪表盘上展现

![](/images/hadoop/sparkstreaming/sparkstreaming1.png)