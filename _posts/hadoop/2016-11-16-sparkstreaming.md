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