---
layout: article
title:  "mapreduce中reduce一直被kill分析"
categories: hadoop
toc: true
image:
    teaser: /teaser/hive_sql1.png
---

> 本文主要介绍mapreduce出现类似死锁应该如何处理


### 前言
&emsp;&emsp;最近离线任务做了调整，为了缩短整个离线运行的时间。所以将大量的任务并行在同一时间点进行启动。结果就出现大量的reduce被kill掉。
## 现象
![reduce_kill图](/images/hadoop/reduce/reduce_kill1.png)

* __花费时间__：该任务从凌晨03:53:17一直跑到早上7点多，然后我手动将任务kill掉。Elapsed:	3hrs, 50mins, 56sec。（如果不手动kill掉该任务，在得不到资源的情况下，将会一直kill下去。以至于影响下游的数据运行）
* __map__:map数总共有185个，已经完成了183个，还有2个map一直抢不到资源，一直不能完成
* __reduce__:reduce数总共只有20，reduce已经kill掉46151个reduce

就以上这种现象一直持续了4个小时，一直不能完成。类似于死锁状态

![reduce_kill图](/images/hadoop/reduce/reduce_kill2.png)
## 原因分析
* 在Yarn JobHistory UI上Note项显示：	`Reducer preempted to make room for pending map attempts`
* 看日志
{% highlight bash %}
{% raw %}
2017-06-20 03:54:31,683 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.jobhistory.EventType for class org.apache.hadoop.mapreduce.jobhistory.JobHistoryEventHandler
2017-06-20 03:54:31,685 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.job.event.JobEventType for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$JobEventDispatcher
2017-06-20 03:54:31,686 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.job.event.TaskEventType for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$TaskEventDispatcher
2017-06-20 03:54:31,687 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.job.event.TaskAttemptEventType for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$TaskAttemptEventDispatcher
2017-06-20 03:54:31,688 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.commit.CommitterEventType for class org.apache.hadoop.mapreduce.v2.app.commit.CommitterEventHandler
2017-06-20 03:54:31,706 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.speculate.Speculator$EventType for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$SpeculatorEventDispatcher
2017-06-20 03:54:31,707 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.rm.ContainerAllocator$EventType for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$ContainerAllocatorRouter
2017-06-20 03:54:31,708 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.launcher.ContainerLauncher$EventType for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$ContainerLauncherRouter
2017-06-20 03:54:31,837 INFO [main] org.apache.hadoop.mapreduce.v2.jobhistory.JobHistoryUtils: Default file system [hdfs://nameservice1:8020]
2017-06-20 03:54:31,894 INFO [main] org.apache.hadoop.mapreduce.v2.jobhistory.JobHistoryUtils: Default file system [hdfs://nameservice1:8020]
2017-06-20 03:54:32,855 INFO [main] org.apache.hadoop.mapreduce.v2.jobhistory.JobHistoryUtils: Default file system [hdfs://nameservice1:8020]
2017-06-20 03:54:33,100 INFO [main] org.apache.hadoop.mapreduce.jobhistory.JobHistoryEventHandler: Emitting job history data to the timeline server is not enabled
2017-06-20 03:54:33,243 INFO [main] org.apache.hadoop.yarn.event.AsyncDispatcher: Registering class org.apache.hadoop.mapreduce.v2.app.job.event.JobFinishEvent$Type for class org.apache.hadoop.mapreduce.v2.app.MRAppMaster$JobFinishEventHandler
2017-06-20 03:54:33,786 INFO [main] org.apache.hadoop.metrics2.impl.MetricsConfig: loaded properties from hadoop-metrics2.properties
2017-06-20 03:54:33,937 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: Scheduled snapshot period at 10 second(s).
2017-06-20 03:54:33,937 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: MRAppMaster metrics system started
{% endraw %}
{% endhighlight %}