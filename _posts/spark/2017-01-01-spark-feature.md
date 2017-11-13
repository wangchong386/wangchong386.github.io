---
layout: article
title:  "谈谈spark"
categories: spark
toc: true

---

> 这篇内容主要介绍Spark的特点以及相比与hadoop一些优势的地方

## spark的特点
* 轻：spark 0.6的核心代码只有2万行，Hadoop 1.0为9万行，2.0为22万行。虽然spark很轻，但是容错能力却很强大。主创人Matei说:"不把容错当做特例处理"
* 快：spark对小数据集能够达到呀妙计的延迟。对于hadoop的mapreduce来说是无法实现的，仅仅任务启动就有数秒的延迟、相较于传统的mapreduce/hive快十倍到百倍。其中内存计算、数据本地性和传输优化、调度优化都起到了很重要的作用。
* 灵：Spark提供了不同层面的灵活性。在实现层，它完美演绎了Scala trait动态混入（mixin）策略（如可更换的集群调度器、序列化库）；在原语（Primitive）层，它允许扩展新的数据算子 （operator）、新的数据源（如HDFS之外支持DynamoDB）、新的language bindings（Java和Python）；在范式（Paradigm）层，Spark支持内存计算、多迭代批量处理、即席查询、流处理和图计算等多种 范式。
* 巧：巧在借势和借力。Spark借Hadoop之势，与Hadoop无缝结合；接着Shark（Spark上的数据仓库实现）借了Hive的势；图计算借 用Pregel和PowerGraph的API以及PowerGraph的点分割思想。一切的一切，都借助了Scala（被广泛誉为Java的未来取代 者）之势：Spark编程的Look'n'Feel就是原汁原味的Scala，无论是语法还是API。在实现上，又能灵巧借力。为支持交互式编 程，Spark只需对Scala的Shell小做修改（相比之下，微软为支持JavaScript Console对MapReduce交互式编程，不仅要跨越Java和JavaScript的思维屏障，在实现上还要大动干戈）
