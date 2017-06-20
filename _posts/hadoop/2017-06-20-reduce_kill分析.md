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
* 在Yarn JobHistory UI上Note项显示：	
{% highlight bash %}
{% raw %}
Reducer preempted to make room for pending map attempts
{% endraw %}
{% endhighlight %}
* 看日志
1. 开始 
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
2. 分配
{% highlight bash %}
{% raw %}
2017-06-20 03:54:56,752 INFO [RMCommunicator Allocator] org.apache.hadoop.mapreduce.v2.app.rm.RMContainerAllocator: Assigned container container_e04_1496750989788_76705_01_000003 to attempt_1496750989788_76705_m_000003_0
{% endraw %}
{% endhighlight %}
3. 启动
{% highlight bash %}
{% raw %}
2017-06-20 03:54:57,042 INFO [ContainerLauncher #0] org.apache.hadoop.mapreduce.v2.app.launcher.ContainerLauncherImpl: Processing the event EventType: CONTAINER_REMOTE_LAUNCH for container container_e04_1496750989788_76705_01_000003 taskAttempt attempt_1496750989788_76705_m_000003_0
2017-06-20 03:54:57,044 INFO [ContainerLauncher #0] org.apache.hadoop.mapreduce.v2.app.launcher.ContainerLauncherImpl: Launching attempt_1496750989788_76705_m_000003_0
2017-06-20 03:54:57,047 INFO [ContainerLauncher #0] org.apache.hadoop.yarn.client.api.impl.ContainerManagementProtocolProxy: Opening proxy : d1-datanode32:8041
2017-06-20 03:54:57,115 INFO [ContainerLauncher #0] org.apache.hadoop.mapreduce.v2.app.launcher.ContainerLauncherImpl: Shuffle port returned by ContainerManager for attempt_1496750989788_76705_m_000003_0 : 13562
{% endraw %}
{% endhighlight %}
4.开始跑
{% highlight bash %}
{% raw %}
2017-06-20 03:54:47,632 INFO [AsyncDispatcher event handler] org.apache.hadoop.mapreduce.v2.app.job.impl.JobImpl: job_1496750989788_76705Job Transitioned from INITED to SETUP
2017-06-20 03:54:47,635 INFO [CommitterEvent Processor #0] org.apache.hadoop.mapreduce.v2.app.commit.CommitterEventHandler: Processing the event EventType: JOB_SETUP
2017-06-20 03:54:47,642 INFO [AsyncDispatcher event handler] org.apache.hadoop.mapreduce.v2.app.job.impl.JobImpl: job_1496750989788_76705Job Transitioned from SETUP to RUNNING
2017-06-20 03:54:47,842 INFO [eventHandlingThread] org.apache.hadoop.mapreduce.jobhistory.JobHistoryEventHandler: Event Writer setup for JobId: job_1496750989788_76705, File: hdfs://nameservice1:8020/user/hdfs/.staging/job_1496750989788_76705/job_1496750989788_76705_1.jhist
2017-06-20 03:54:47,972 INFO [AsyncDispatcher event handler] org.apache.hadoop.mapreduce.v2.app.job.impl.TaskImpl: task_1496750989788_76705_m_000000 Task Transitioned from NEW to SCHEDULED
{% endraw %}
{% endhighlight %}
5.完成清理相关的信息，此时container已经退出
{% highlight bash %}
{% raw %}
2017-06-20 03:55:21,861 INFO [IPC Server handler 16 on 43744] org.apache.hadoop.mapred.TaskAttemptListenerImpl: Done acknowledgement from attempt_1496750989788_76705_m_000018_0
2017-06-20 03:55:21,863 INFO [AsyncDispatcher event handler] org.apache.hadoop.mapreduce.v2.app.job.impl.TaskAttemptImpl: attempt_1496750989788_76705_m_000018_0 TaskAttempt Transitioned from RUNNING to SUCCESS_FINISHING_CONTAINER
{% endraw %}
{% endhighlight %}
* 从实际监控来看，出现问题时，没跑完的map全在等待资源，而reduce在copy阶段已占用大量资源，由于map一直在等空闲资源，而reduce一直等未完成的map执行完，形成了一个死锁。AppMaster将reduce kill并释放资源。出现这种情况时，Job运行时间会增加几小时。甚至一直持续无法释放资源。这块需要了解下RMContainerAllocator
## RMContainerAllocator原理分析

1. ContainerAllocator通过与RM通信，为Job申请资源，同时其维护一个心跳信息，获取新分配的资源和各Container运行情况。

2. 在注释中可以看到，map生命周期为`scheduled->assigned->completed，reduce`生命周期为`pending->scheduled->assigned->completed`。只要收到map的请求后，map的状态即变为scheduled状态，reduce根据map完成数和集群资源情况在pending和scheduled状态中变动。
> Vocabulary Used:
> 
> pending -> requests which are NOT yet sent to RM
> 
> scheduled -> requests which are sent to RM but not yet assigned
> 
> assigned -> requests which are assigned to a container
> 
> completed -> request corresponding to which container has completed
3. ContainerAllocator将所有任务分成三类：
* Failed Map。Priority为5。
* Reduce。Priority为10。
* Map。Priority为20。
&emsp;&emsp;Priority越低，该任务优先级越高。即这三种任务同时请求资源时，资源优先分配给Failed Map，其次是Reduce，最后才是Map。
3. 源码分析：
## 解决办法
少部分map由于reduce占用过多资源，无法执行，Container中kill相关reduce，腾出资源让map继续执行。这里有个疑问，从源码和配置文件中，如果map出现资源不足的情况，reduce应该会立即释放资源，但为何map等待时间这么久？从log可以看到，container申请资源时间相当长，考虑到使用的是FairScheduler对于此问题，比较好的方案就是尽量错
开大Job的执行时间，另外可适当调大COMPLETED_MAPS_FOR_REDUCE_SLOWSTART值，尽量让map多完成，但这样可能造成job运行时间变长。我找到集群中该参数发现已经设置过；哎！所以看来是需要设置为1了。(主要还是同时段启动的任务太多导致的)
![reduce_kill图](/images/hadoop/reduce/reduce_kill3.png)
&emsp;&emsp;所以最好就是调整参数：`mapreduce.job.reduce.slowstart.completedmaps`