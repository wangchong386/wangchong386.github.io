---
layout: article
title:  "Spark Streaming整体架构以及原理"
categories: hadoop
toc: true
image:
    teaser: /teaser/spark.png
---

> 这篇内容主要介绍Spark Streaming整体架构以及原理

> sparkstreaming是核心Spark API的扩展，可以实现实时数据流的可扩展，高吞吐，容错的流处理。数据源可以从Flume、kafka、kinesis 以及TCP sockets来获取。并且可以使用map,reduce,joinhe windows等高级功能开进行处理。最后处理好的数据可以存储到文件系统，数据库以及实时仪表盘上展现

![](/images/hadoop/sparkstreaming/sparkstreaming1.png)

* Spark Streaming接收实时输入数据流并将数据分成批，然后由Spark引擎进行处理，以批量生成最终的结果流。
![](/images/hadoop/sparkstreaming/sparkstreaming2.png)

* 可以使用Maven仓库来编写sparkstreaming程序
{% highlight bash %}
{% raw %}

<dependency>
<groupId>org.apache.spark</groupId>
<artifactId>spark-streaming_2.10</artifactId>
<version>1.2</version>
</dependency>

{% endraw %}
{% endhighlight %}

为了从Kafka, Flume和Kinesis这些不在Spark核心API中提供的源获取数据，我们需要添加相关的模块 spark-streamingxyz_2.10 到依赖中。例如，一些通用的组件如下表所示：

|Source ||Artifact|
Kafka |spark-streaming-kafka_2.10
Flume |spark-streaming-flume_2.10
Kinesis |spark-streaming-kinesis-asl_2.10
Twitter |spark-streaming-twitter_2.10
ZeroMQ |spark-streaming-zeromq_2.10
MQTT |spark-streaming-mqtt_2.10