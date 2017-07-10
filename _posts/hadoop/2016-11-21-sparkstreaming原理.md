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