---
layout: article
title:  "sparkstreaming与kafka集成分析"
categories: hadoop
toc: true
image:
    teaser: /teaser/spark.png
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
* 效率高（zero copy）：Receiver-based Approach中要实现zero-data loss需要将数据存储在write Ahead log中，然后在进一步replicated复制数据，这种做法实际上是很低效的，数据被replicated复制了两次.一次被kafka,第二次被写入Ahead log中。然而第二种方法Direct Approach (No Receivers)消除了这个问题，因为没有receiver,所以也不需要写入Ahead log中，只要你有足够的kafka retention,消息就可以从kafka中进行恢复消费
* Exactly-once ：第一种方法使用Kafka的高级API在Zookeeper中存储consumeed offset。这通常是从kafka消费数据的方式。虽然这种方法（结合写入Ahead log）可以确保zero data loss（即at-least once），但是在某些故障下，一些记录可能会被消耗两次的可能性很小。这是因为Spark Streaming可靠接收的数据与Zookeeper跟踪的偏移量之间的不一致。因此，在第二种方法中，我们使用不使用Zookeeper的简单Kafka API。 Spark Streaming在其检查点内跟踪偏移量。这消除了Spark Streaming和Zookeeper / Kafka之间的不一致，因此，尽管出现故障，Spark Streaming仍然可以有效地收到每个记录。为了实现一致的语义输出结果，将数据保存到外部数据存储区的输出操作必须是幂等的，也可以是保存结果和偏移量的原子事务。
## Direct Approach VS Receiver-based Approach
经过上面对两种数据接收方案的介绍，我们发现， Receiver-based Approach 存在各种内存折腾，对应的Direct Approach (No Receivers)则显得比较纯粹简单些，这也给其带来了较多的优势，主要有如下几点：

1. 因为按需拉数据，所以不存在缓冲区，就不用担心缓冲区把内存撑爆了。这个在Receiver-based Approach 就比较麻烦，你需要通过spark.streaming.blockInterval等参数来调整。
数据默认就被分布到了多个Executor上。Receiver-based Approach 你需要做特定的处理，才能让 Receiver分不到多个Executor上。
2. Receiver-based Approach 的方式，一旦你的Batch Processing 被delay了，或者被delay了很多个batch,那估计你的Spark Streaming程序离奔溃也就不远了。 Direct Approach (No Receivers) 则完全不会存在类似问题。就算你delay了很多个batch time,你内存中的数据只有这次处理的。
3. Direct Approach (No Receivers) 直接维护了 Kafka offset,可以保证数据只有被执行成功了，才会被记录下来，透过 checkpoint机制。如果采用Receiver-based Approach，消费Kafka和数据处理是被分开的，这样就很不好做容错机制，比如系统当掉了。所以你需要开启WAL,但是开启WAL带来一个问题是，数据量很大，对HDFS是个很大的负担，而且也会对实时程序带来比较大延迟。

我原先以为Direct Approach 因为只有在计算的时候才拉取数据，可能会比Receiver-based Approach 的方式慢，但是经过我自己的实际测试，总体性能 Direct Approach会更快些，因为Receiver-based Approach可能会有较大的内存隐患，GC也会影响整体处理速度。