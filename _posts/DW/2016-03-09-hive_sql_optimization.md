---
layout: article
title:  "hive sql优化基础篇"
categories: DW
toc: true
image:
    teaser: /teaser/hive_sql1.jpg
---

> 本文主要介绍hive sql是如何进行编译执行的，只有知道是如何编译才可以对hive sql进行优化

### 前言
&emsp;&emsp;以下内容主要是通过在学习和工作中的一些总结和感悟，很多概念上的东西都主要来自于书籍和网络，毕竟这些基础知识还是以官网为准。对于我的公司我会成为W公司。
## 概述
&emsp;&emsp;Hive是基于Hadoop的一个数据仓库系统，基本在很多公司都有使用，特别是在w公司。由于基本都是离线处理，并且流量数据特别大，使用hive处理简直再适合不过了。
## 问题分析：

* 1. 此次Flume故障开始于1月1日早上8点到9点，此时，在08:00测试集群的两台机器CPU 突然100%
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffqthgrs7sj20ak0a73z5.jpg)

后来了解到2017年1月1日早上08:00，UTC时间增加一秒（leap second，参考：http://www.cas.cn/tz/201612/t20161227_4586274.shtml  ）

* 2. 检查系统日志，发现增加闰秒日志：
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffqtkfqzeyj20my02ot8y.jpg)
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffqtpb75ohj208x01ldfo.jpg)

&emsp;&emsp;__综上认为：闰秒触发了java的bug，导致java进程CPU占用过高，响应缓__

## 1. 什么是闰秒？
{% highlight bash %}
{% raw %}

* 闰秒，是指为保持协调世界时接近于世界时时刻，由国际计量局统一规定在年底或年中（也可能在季末）对协调世界时增加或减少1秒的调整。由于地球自转的不均匀性和长期变慢性（主要由潮汐摩擦引起的），会使世界时（民用时）和原子时之间相差超过到±0.9秒时，就把协调世界时向前拨1秒（负闰秒，最后一分钟为59秒）或向后拨1秒（正闰秒，最后一分钟为61秒）； 
* 闰秒一般加在公历年末或公历六月末。
* 目前，全球已经进行了27次闰秒，均为正闰秒。
* __最近一次闰秒在北京时间2017年1月1日7时59分59秒（时钟显示07:59:60）出现。这也是本世纪的第五次闰秒。__

{% endraw %}
{% endhighlight %}

## 2. 闰秒为什么会导致Linux出现问题？
* 由于Linux kernel 2.6.29之前版本存在bug，在进行闰秒调整时可能会引起系统时钟服务ntpd进程死锁。Debian Lenny、RHEL/CentOS 5等旧发行版今天仍被广泛使用，部分供应商早已经发布了补丁。

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