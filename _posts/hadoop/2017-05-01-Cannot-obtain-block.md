---
layout: article
title: "Cannot obtain block length for LocatedBlock"
modified:
categories: hadoop
toc: true
#excerpt:

image:
#  feature:
    teaser: /teaser/hdfsblock.jpg
#  thumb:
date: 2017-05-01T17:12:48+08:00
---


> cat一下某天的HDFS文件内容的时候突然报Cannot obtain block length for LocatedBlock异常，get也一样，这样无法访问hdfs文件的问题必须解决，Mark一下问题背景和解决过程
{% highlight %}
{% raw %}
Error: java.io.IOException: Cannot obtain block length for LocatedBlock{BP-1387429737-172.21.150.11-1452521720772:blk_1110555732_36821087; getBlockSize()=1077321; corrupt=false; offset=0; locs=[DatanodeInfoWithStorage[172.21.150.36:50010,DS-80087dd8-e0de-4b85-81a4-f2f6d3519eaa,DISK]]} at 
{% endraw %}
{% endhighlight %}


## 一.问题背景

问题产生的原因可能是由于前几日Hadoop集群维护的时候，基础运维组操作不当，先关闭的Hadoop集群，然后才关闭的Flume agent导致的hdfs文件写入后状态不一致。排查和解决过程如下.

## 二.解决过程

* 1.既然是hdfs文件出问题,用fsck检查一下吧

hdfs fsck /

当然你可以具体到指定的hdfs路径,检查完打印结果没有发现任何异常，没有发现损坏或者Corrupt的block，继续排查

* 2.那么加上其他参数细查

`hdfs fsck / –openforwrite`

ok,这次检查出来不少文件打印显示都是 openforwrite状态,而且我测试相应文件确实不能读取,这很不正常不是吗？Flume已经写过的hdfs文件居然还处于openforwrite状态，而且无法cat和get

所以这里的”Cannot obtain block length for LocatedBlock”结合字面意思讲应该是当前有文件处于写入状态尚未关闭，无法与对应的datanode通信来成功标识其block长度.

那么分析其产生的可能性，举栗子如下

* 1 Flume客户端写入hdfs文件时的网络连接被不正常的关闭了

或者

* 2 Flume客户端写入hdfs失败了，而且其replication副本也丢失了

我这里应该属于第一种，总结一下就是Flume写入的hdfs文件由于什么原因没有被正常close，状态不一致随后无法正常访问.继续排查

* 3.推断:HDFS文件租约未释放

可以参考这篇文章来了解[HDFS租约机制](http://www.cnblogs.com/cssdongl/p/6699919.html)
了解过HDFS租约后我们知道,客户端在每次读写HDFS文件的时候获取租约对文件进行读写，文件读取完毕了，然后再释放此租约.文件状态就是关闭的了。

但是结合当前场景由于先关闭的hadoop集群，后关闭的Flume sink hdfs,那么hadoop集群都关了，Flume还在对hdfs文件写入，那么租约最后释放了吗？答案是肯定没释放.

* 4.恢复租约

对于这些状态损坏的文件来讲，rm掉的话是很暴力的做法，万一上游对应日期的数据已经没有rention呢？所以，既然没有释放租约，那么恢复租约close掉文件就是了,如下命令

`hdfs debug recoverLease -path <path-of-the-file> -retries <retry times>`

请将<path-of-the-file>修改成你需要恢复的租约状态不一致的hdfs文件的具体路径,如果要恢复的很多，可以写个自动化脚本来找出需要恢复的所有文件然后统一恢复租约.

ok，执行完命令后再次cat对应hdfs文件已无异常，顺利显示内容，问题解决.

