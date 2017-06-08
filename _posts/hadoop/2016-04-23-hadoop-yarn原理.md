---
layout: article
title:  "hadoop2.0 yarn原理"
categories: hadoop
toc: true
image:
    teaser: /teaser/Hadoop-YARN.png
---

> 本文简单地讲述hadoop mapreduceV2 框架 yarn的工作原理。由于mapreduceV1框架在大集群中面临着扩展性瓶颈，所以2010年雅虎团队开始设计了mapreduceV2框架Yarn (yet another resource negotiaor), 旨在解决mrV1中JobTrack职能过重的瓶颈问题。

### 前言
&emsp;&emsp;以下内容主要是通过在学习和工作中的一些总结和感悟，很多概念上的东西都主要来自于书籍和网络，毕竟这些基础知识还是以官网为准。
### yarn框架图

来源于官网的yarn架构图：
/hadoop/YARN/yarn_Architecture.png



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
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffz6awdedrj20p3042q2z.jpg)
因此，我将mapreduce.reduce.shuffle.input.buffer.percent设置成值为0.1，就可以正常运行了（设置成0.2，还是会抛同样的错）


2. 另外，可以发现如果使用两个参数的默认值，那么两者乘积为3.5，大大大于1了，为什么没有经常抛出以上的错误呢？

&emsp;&emsp; 首先，把默认值设为比较大，主要是基于性能考虑，将它们设为比较大，可以大大加快从map复制数据的速度

&emsp;&emsp;其次，要抛出如上异常，还需满足另外一个条件，就是map任务的数据一下子准备好了等待shuffle去复制，在这种情况下，就会导致shuffle过程的“线程数量”和“内存buffer使用量”都是满负荷的值，自然就造成了内存不足的错误；而如果map任务的数据是断断续续完成的，那么没有一个时刻shuffle过程的“线程数量”和“内存buffer使用量”是满负荷值的，自然也就不会抛出如上错误

3. 另外，如果在设置以上参数后，还是出现错误，那么有可能是运行Reduce任务的进程的内存总量不足，可以通过mapred.child.Java.opts参数来调节，比如设置mapred.child.java.opts=-Xmx2024m。
 ![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffz67n05oqj20ux0axjsb.jpg)
## MapReduce任务参数调优
### 1. 操作系统调优
增大打开文件数据和网络连接上限，调整内核参数`net.core.somaxconn`，提高读写速度和网络带宽使用率
适当调整`epoll`的文件描述符上限，提高Hadoop RPC并发
关闭`swap`。如果进程内存不足，系统会将内存中的部分数据暂时写入磁盘，当需要时再将磁盘上的数据动态换置到内存中，这样会降低进程执行效率
增加预读缓存区大小。预读可以减少磁盘寻道次数和I/O等待时间
设置`openfile`
### 2. Hdfs参数调优
* 2.1 core-default.xml：

`hadoop.tmp.dir`：

默认值： /tmp
说明： 尽量手动配置这个选项，否则的话都默认存在了里系统的默认临时文件/tmp里。并且手动配置的时候，如果服务器是多磁盘的，每个磁盘都设置一个临时文件目录，这样便于mapreduce或者hdfs等使用的时候提高磁盘IO效率。
`fs.trash.interval`：

默认值： 0
说明： 这个是开启hdfs文件删除自动转移到垃圾箱的选项，值为垃圾箱文件清除时间。一般开启这个会比较好，以防错误删除重要文件。单位是分钟。
`io.file.buffer.size`：

默认值：4096
说明：SequenceFiles在读写中可以使用的缓存大小，可减少 I/O 次数。在大型的 Hadoop cluster，建议可设定为 65536 到 131072。
* 2.2 hdfs-default.xml：

`dfs.blocksize`：

默认值：134217728
说明： 这个就是hdfs里一个文件块的大小了，CDH5中默认128M。太大的话会有较少map同时计算，太小的话也浪费可用map个数资源，而且文件太小namenode就浪费内存多。根据需要进行设置。
`dfs.namenode.handler.count`：

默认值：10
说明：设定 namenode server threads 的数量，这些 threads 會用 RPC 跟其他的 datanodes 沟通。当 datanodes 数量太多时会发現很容易出現 RPC timeout，解決方法是提升网络速度或提高这个值，但要注意的是 thread 数量多也表示 namenode 消耗的内存也随着增加
### 3. MapReduce参数调优
包括以下节点：

合理设置槽位数目
调整心跳配置
磁盘块配置
设置RPC和线程数目
启用批量任务调度
* 3.1 mapred-default.xml：

`mapred.reduce.tasks（mapreduce.job.reduces）`：

默认值：1
说明：默认启动的reduce数。通过该参数可以手动修改reduce的个数。
`mapreduce.task.io.sort.factor`：

默认值：10
说明：Reduce Task中合并小文件时，一次合并的文件数据，每次合并的时候选择最小的前10进行合并。

`mapreduce.task.io.sort.mb`：

默认值：100
说明： Map Task缓冲区所占内存大小。
`mapred.child.java.opts`：

默认值：-Xmx200m
说明：jvm启动的子线程可以使用的最大内存。建议值-XX:-UseGCOverheadLimit -Xms512m -Xmx2048m -verbose:gc -Xloggc:/tmp/@taskid@.gc

`mapreduce.jobtracker.handler.count`：

默认值：10
说明：JobTracker可以启动的线程数，一般为tasktracker节点的4%。

`mapreduce.reduce.shuffle.parallelcopies`：

默认值：5
说明：reuduce shuffle阶段并行传输数据的数量。这里改为10。集群大可以增大。

`mapreduce.tasktracker.http.threads`：

默认值：40
说明：map和reduce是通过http进行数据传输的，这个是设置传输的并行线程数。
`mapreduce.map.output.compress`：

默认值：false
说明： map输出是否进行压缩，如果压缩就会多耗cpu，但是减少传输时间，如果不压缩，就需要较多的传输带宽。配合`mapreduce.map.output.compress.codec`使用，默认是`org.apache.hadoop.io.compress.DefaultCodec`，可以根据需要设定数据压缩方式。

`mapreduce.reduce.shuffle.merge.percent`：

默认值： 0.66
说明：reduce归并接收map的输出数据可占用的内存配置百分比。类似`mapreduce.reduce.shuffle.input.buffer.percent`属性。

`mapreduce.reduce.shuffle.memory.limit.percent`：

默认值： 0.25
说明：一个单一的shuffle的最大内存使用限制。

`mapreduce.jobtracker.handler.count`：

默认值： 10
说明：可并发处理来自tasktracker的RPC请求数，默认值10。

`mapred.job.reuse.jvm.num.tasks（mapreduce.job.jvm.numtasks）`：

默认值： 1
说明：一个jvm可连续启动多个同类型任务，默认值1，若为-1表示不受限制。

`mapreduce.tasktracker.tasks.reduce.maximum`：

默认值： 2
说明：一个tasktracker并发执行的reduce数，建议为cpu核数
### 4. 系统优化
* 4.1 避免排序

对于一些不需要排序的应用，比如hash join或者limit n，可以将排序变为可选环节，这样可以带来一些好处：

在Map Collect阶段，不再需要同时比较partition和key，只需要比较partition，并可以使用更快的计数排序（O(n)）代替快速排序（O(NlgN)）
在Map Combine阶段，不再需要进行归并排序，只需要按照字节合并数据块即可。
去掉排序之后，Shuffle和Reduce可同时进行，这样就消除了Reduce Task的屏障（所有数据拷贝完成之后才能执行reduce()函数）。
* 4.2 Shuffle阶段内部优化

Map端–用Netty代替Jetty
Reduce端–批拷贝
将Shuffle阶段从Reduce Task中独立出来
5. 总结
在运行mapreduce任务中，经常调整的参数有：

`mapred.reduce.tasks`：手动设置reduce个数

`mapreduce.map.output.compress`：map输出结果是否压缩

`mapreduce.map.output.compress.codec`

`mapreduce.output.fileoutputformat.compress`：job输出结果是否压缩

`mapreduce.output.fileoutputformat.compress.type`

`mapreduce.output.fileoutputformat.compress.codec`