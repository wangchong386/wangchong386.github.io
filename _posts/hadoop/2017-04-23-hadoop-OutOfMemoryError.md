---
layout: article
title:  "Yarn在Shuffle阶段内存不足问题(error in shuffle in fetcher)"
categories: hadoop
toc: true
image:
    teaser: /teaser/OutOfMemory.png
---

> 本文主要介绍mapreduce计算过程中出现内存溢出的问题，然后是对Java内存溢出(OOM)异常的总结与思考

### 前言
&emsp;&emsp;以下内容主要是通过在学习和工作中的一些总结和感悟，很多概念上的东西都主要来自于书籍和网络，毕竟这些基础知识还是以官网为准。
## 故障现象
&emsp;&emsp;从这次出现的事故说起吧，在凌晨一点多接到运维电话。报ETL流程失败


    | 时间    | 事件| 
    | 01:34   | 日志检查失败，ETL流程异常退出，其中有一个任务：Caused by: java.lang.OutOfMemoryError: Java heap space


* 查看日志发现有任务的确是java.lang.OutOfMemoryError错误，主要错误日志如下：
{% highlight bash %}
{% raw %}

 Examining task ID: task_1495604758492_3152$m_000477 (and more) from job job_1495604758492_3152

 Task with the most failures(4):
 -----
 Task ID:
   task_1495604758492_3152_r_000001

 URL:
   http://d1-datanode32:8088/taskdetails.js$?jobid=job_1495604758492_3152&tipid=task_1495604758492_3152_r_000001
 -----
 Diagnostic Messages for this Task:
 Error: org.apache.hadoop.mapreduce.task.re$uce.Shuffle$ShuffleError: error in shuffle in fetcher#16
   at org.apache.hadoop.mapreduce.task.redu$e.Shuffle.run(Shuffle.java:134)
   at org.apache.hadoop.mapred.ReduceTask.r$n(ReduceTask.java:376)
   at org.apache.hadoop.mapred.YarnChild$2.$un(YarnChild.java:163)
   at java.security.AccessController.doPriv$leged(Native Method)
   at javax.security.auth.Subject.doAs(Subj$ct.java:415)
   at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1671)
   at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
 Caused by: java.lang.OutOfMemoryError: Java heap space
   at org.apache.hadoop.io.BoundedByteArrayOutputStream.<init>(BoundedByteArrayOutputStream.java:56)
   at org.apache.hadoop.io.BoundedByteArrayOutputStream.<init>(BoundedByteArrayOutputStream.java:46)
   at org.apache.hadoop.mapreduce.task.reduce.InMemoryMapOutput.<init>(InMemoryMapOutput.java:63)
   at org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl.unconditionalReserve(MergeManagerImpl.java:305)
   at org.apache.hadoop.mapreduce.task.reduce.MergeManagerImpl.reserve(MergeManagerImpl.java:295)
   at org.apache.hadoop.mapreduce.task.reduce.Fetcher.copyMapOutput(Fetcher.java:514)
   at org.apache.hadoop.mapreduce.task.reduce.Fetcher.copyFromHost(Fetcher.java:336)
   at org.apache.hadoop.mapreduce.task.reduce.Fetcher.run(Fetcher.java:193)
 FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask
 MapReduce Jobs Launched:
 Stage-Stage-1: Map: 485  Reduce: 7   Cumulative CPU: 11740.0 sec   HDFS Read: 13431704785 HDFS Write: 0 FAIL
{% endraw %}
{% endhighlight %}

* 相信对于java人员来说出现java.lang.OutOfMemoryError: Java heap space时很常见的。一般是因为：当应用程序试图向堆空间添加更多的数据，但堆却没有足够的空间来容纳这些数据时，将会触发java.lang.OutOfMemoryError: Java heap space异常。需要注意的是：即使有足够的物理内存可用，只要达到堆空间设置的大小限制，此异常仍然会被触发。
> __将该任务的hiveql找出来，发现该表分区一天有12G数据量，由于是全量存取的用户登录数据。虽然数据量很大，但是以前为什么没有出现过这种内存溢出的错误？难道是小概率问题？最后重跑了一次脚本结果可以正确执行，也没有报错__

## 解决方法：限制reduce的shuffle内存使用 
1. 具体的来说，shuffle过程的输入是：map任务的输出文件，它的输出接收者是：运行reduce任务的机子上的内存buffer，并且shuffle过程以并行方式运行
参数mapreduce.reduce.shuffle.input.buffer.percent控制运行reduce任务的机子上多少比例的内存用作上述buffer(默认值为0.70)，参数mapreduce.reduce.shuffle.parallelcopies控制shuffle过程的并行度(默认值为5)
那么"mapreduce.reduce.shuffle.input.buffer.percent" * "mapreduce.reduce.shuffle.parallelcopies" 必须小于等于1，否则就会出现如上错误
因此，我将mapreduce.reduce.shuffle.input.buffer.percent设置成值为0.1，就可以正常运行了（设置成0.2，还是会抛同样的错）


2. 另外，可以发现如果使用两个参数的默认值，那么两者乘积为3.5，大大大于1了，为什么没有经常抛出以上的错误呢？

&emsp;&emsp; 首先，把默认值设为比较大，主要是基于性能考虑，将它们设为比较大，可以大大加快从map复制数据的速度

&emsp;&emsp;其次，要抛出如上异常，还需满足另外一个条件，就是map任务的数据一下子准备好了等待shuffle去复制，在这种情况下，就会导致shuffle过程的“线程数量”和“内存buffer使用量”都是满负荷的值，自然就造成了内存不足的错误；而如果map任务的数据是断断续续完成的，那么没有一个时刻shuffle过程的“线程数量”和“内存buffer使用量”是满负荷值的，自然也就不会抛出如上错误

3. 另外，如果在设置以上参数后，还是出现错误，那么有可能是运行Reduce任务的进程的内存总量不足，可以通过mapred.child.Java.opts参数来调节，比如设置mapred.child.java.opts=-Xmx2024m。
