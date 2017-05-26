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

## 故障产生原因
