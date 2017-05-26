---
layout: article
title:  "hadoop OutOfMemoryError"
categories: hadoop
toc: true
image:
    teaser: /teaser/OutOfMemory.png
---

> 本文主要介绍mapreduce计算过程中出现内存溢出的问题，然后是对Java内存溢出(OOM)异常的总结与思考

### 前言
&emsp;&emsp;以下内容主要是通过在学习和工作中的一些总结和感悟，很多概念上的东西都主要来自于书籍和网络，毕竟这些基础知识还是以官网为准。
## 概述
&emsp;&emsp;从这次出现的事故说起吧，在凌晨一点多接到运维电话。报ETL流程失败


    | 时间    | 事件| 
    | 01:34   | 日志检查失败，ETL流程异常退出，其中有一个任务：Caused by: java.lang.OutOfMemoryError: Java heap space



## 问题分析：

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

## 3.闰秒导致部分Linux服务器高CPU使用率
* 国际地球自转和参考坐标系统服务(IERS)在2012年6月30日午夜(北京时间7月1号7点59分59秒)增加一闰秒(即出现 7：59：60)。由于Linux kernel 2.6.29之前版本存在bug，在进行闰秒调整时可能会引起系统时钟服务ntpd进程死锁。Debian Lenny、RHEL/CentOS 5等旧发行版今天仍被广泛使用，部分供应商早已经发布了补丁。
* 但除了Linux服务器外，一些服务器程序也因为闰秒出现了问题，如Reddit、Mozilla、FourSquare、Yelp、 LinkedIn和Gawker等网站都短暂遭遇了技术问题，国内的一家云储存供应商发现运行在CentOS 6.2上的Java和MySQL因闰秒出现了不同程度的CPU利用率增长，猜测是JVM和MySQL试图通过CPU硬件晶振的数据获得当前精确的时间，由 于闰秒的关系，这个时间和操作系统维持的墙上时间(Wall Time，也就是显示给用户看的时间)不一致，导致了这个问题。简单的修正方法是强制重置系统时间，让系统中所有时间回到同步的状态
* 近日，国际地球自转和参考系统服务地球定向中心(IERS)通过推特重申，国际标准时间UTC将在格林尼治时间2016年12月31日23时59分59秒(北京时间2017年1月1日7时59分59秒)之后，在原子时钟实施一个正闰秒，即增加1秒，然后才会跨入新的一年。
* 每隔一段时间，目前世界范围内通用的协调世界时(UTC)会与依据地球围绕太阳运动计算的平太阳日和世界时(UT1)出现很小的偏差，需要对UTC增加或者减少一秒来消除。

## 4.解决方法
1. 简要解决方法：在发生闰秒前停掉ntpd服务，闰秒发生后再开启ntpd

2. 根解：放弃使用ntpd，使用简化的sntp协议，同时在实现直接调用settimeofday来完成，不会触发内核的事件调整异常
 Java Fortunately the fix is straightforward:

```
/etc/init.d/ntp stop
date -s "$(date)"
```

 Mysql

{% highlight bash %}
{% raw %}

The fix is quite simple – simply set the date. Alternatively, you can restart the machine, which also works. Restarting MySQL (or Java, or whatever) does NOT fix the problem. We put the following into puppet to run on all our machines:

$ cat files/bin/leap-second.sh

`#!/bin/bash`
`# this is a quick-fix to the 6/30/12 leap second bug`

if [ ! -f /tmp/leapsecond_2012_06_30 ]
then
/etc/init.d/ntpd stop; date -s "`date`" && /bin/touch /tmp/leapsecond_2012_06_30
fi

{% endraw %}
{% endhighlight %}

## 经验总结
1. 发现问题应及时处理，并与团队以及业务进行沟通
2.  闰秒问题每隔18个月出现，应提前准备