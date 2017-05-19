---
layout: article
title:  "闰秒导致部分Linux服务器高CPU使用率"
categories: linux
toc: true
image:
    teaser: /teaser/leapsecond.jpg
---

> 本文主要介绍闰秒为什么会导致linux出现问题

### 前言
&emsp;&emsp;以下内容主要是通过在学习和工作中的一些总结和感悟，很多概念上的东西都主要来自于书籍和网络，毕竟这些基础知识还是以官网为准。
## 概述
&emsp;&emsp;从公司这次出现的事故说起吧，在2017年1月1日刚好是个周末。大清早的就接到运维电话。


    | 时间    | 事件| 
    | 09:34   | 生产环境有多台机器报警，2台Flume均有日志积压，同时测试服务器CPU持续100%。
                先将Flume重启，确认已经重启成功，正在读取缓存文件后，反馈运维。暂没有深入分析原因
    | 11:00   | 运维再次发现还是一直报警，并且flume还有大量的积压数据。
    | 13:00   | 两台Flume重新进行配置。由文件缓存改为内存缓存，并增加写入HDFS线程数到8个。 
                发现Flume正常无积压运行一段一个小时后Flume无响应，不接受数据，对配置参数进行调整，写入HDFS线程数减少到3个，增大了每次写入HDFS的batch size。 
                同时将生产环境active namenode的yarn关闭，提高namenode响应。
    |19:00    | 调整完毕。之后一直正常运行
    |21:00    | kill无用的高CPU java进程，解决了测试环境服务器持续数天高负载的问题
直到完全解决问题，一直认为是突然公司流量增大，导致Flume端消费不及时数据堆积增多导致的。但是又发现测试环境中基本没有什么任务。为什么一直是满负载。而且测试集群中有8台机器，为什么只有两台机器cpu持续是100%？
&emsp;&emsp;__最后找到罪魁祸首就是leap second（闰秒）：__
## 问题分析：
#### 1. 此次Flume故障开始于1月1日早上8点到9点，此时，在08:00测试集群的两台机器CPU 突然100%
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffqthgrs7sj20ak0a73z5.jpg)

后来了解到2017年1月1日早上08:00，UTC时间增加一秒（leap second，参考：http://www.cas.cn/tz/201612/t20161227_4586274.shtml  ）

#### 2. 检查系统日志，发现增加闰秒日志：
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffqtkfqzeyj20my02ot8y.jpg)
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffqtpb75ohj208x01ldfo.jpg)

&emsp;&emsp;__综上认为：闰秒触发了java的bug，导致java进程CPU占用过高，响应缓__

## 1. 什么是闰秒？
