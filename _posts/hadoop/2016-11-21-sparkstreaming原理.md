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

* 当一个上下文（context）定义之后，你必须按照以下几步进行操作：
(1) 定义输入源；
(2) 准备好流计算指令；
(3) 利用 JavaStreamingContext.start() 方法接收和处理数据；
(4) 处理过程将一直持续，直到 JavaStreamingContext.stop() 方法被调用。

* 几点需要注意的地方：
(1) 一旦一个context已经启动，就不能有新的流算子建立或者是添加到context中。
(2) 一旦一个context已经停止，它就不能再重新启动
(3) 在JVM中，同一时间只能有一个JavaStreamingContext处于活跃状态
(4) 在JavaStreamingContext上调用 stop() 方法，也会关闭JavaSparkContext对象。如果只想仅关闭JavaStreamingContext对象，设置 stop() 的可选参数为false
(5) 一个JavaSparkContext对象可以重复利用去创建多个JavaStreamingContext对象，前提条件是前面的JavaStreamingContext在后面JavaStreamingContext创建之前关闭（不关闭JavaSparkContext）。