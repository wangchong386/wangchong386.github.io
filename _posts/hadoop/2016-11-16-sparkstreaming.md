---
layout: article
title:  "sparkstreaming与kafka集成分析"
categories: hadoop
toc: true
image:
    teaser: /teaser/sparkstreaming.png
---

> 这篇内容主要介绍Spark Streaming 数据接收流程模块中与Kafka集成相关的功能。




### 前言
&emsp;&emsp;Spark Streaming 与 Kafka 集成接受数据的方式有两种：
* Receiver-based Approach
* Direct Approach (No Receivers)
主要对这两种方案详细的介绍以及原理的解析，同时对比两种方案的优劣
## Receiver-based Approach
## Direct Approach (No Receivers)
> This approach has the following advantages over the receiver-based approach
* 简化并行：不需要创建多个input kafka streams并联合它们。 使用`directStream`，Spark Streaming将创建尽可能多的RDD分区，因为有Kafka partition来进行consumer，这将分别从Kafka读取数据并行。 因此，在Kafka和RDD分区之间存在一对一映射，这更容易理解和调整