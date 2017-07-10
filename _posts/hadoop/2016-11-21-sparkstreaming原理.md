---
layout: article
title:  "Spark Streaming整体架构以及原理"
categories: hadoop
toc: true
image:
    teaser: /teaser/spark.png
---

> 这篇内容主要介绍Spark Streaming整体架构以及原理

## 架构介绍
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

|Source |Artifact|
Kafka |spark-streaming-kafka_2.10
Flume |spark-streaming-flume_2.10
Kinesis |spark-streaming-kinesis-asl_2.10
Twitter |spark-streaming-twitter_2.10
ZeroMQ |spark-streaming-zeromq_2.10
MQTT |spark-streaming-mqtt_2.10

## 各部分功能介绍
* 初始化StreamingContext(Java版本)
&emsp;&emsp;为了初始化Spark Streaming程序，一个StreamingContext对象必需被创建，它是Spark Streaming所有流操作的主要入口。一个StreamingContext 对象可以用SparkConf对象创建。
{% highlight bash %}
{% raw %}
import org.apache.spark.*;
import org.apache.spark.streaming.api.java.*;
/*
 *第1步：创建spark的配置对象SparkConf，设置spark程序运行的配置信息
*/
SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
//appName 表示你的应用程序显示在集群UI上的名字， master 是一个Spark、Mesos、YARN集群URL 或者一个特殊字符串“local[*]”，它表示程序用本地模式运行。
JavaStreamingContext ssc = new JavaStreamingContext(conf, new Duration(1000));
//批时间片需要根据你的程序的潜在需求以及集群的可用资源来设置


//JavaStreamingContext也可以从一个已经存在的JavaSparkContext来进行创建
import org.apache.spark.streaming.api.java.*;

JavaSparkContext sc = ...   //existing JavaSparkContext
JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(1));
{% endraw %}
{% endhighlight %}