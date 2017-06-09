---
layout: article
title:  "Kafka Manager安装使用"
categories: hadoop
toc: true
image:
    teaser: /teaser/kafkamanager.jpg
---

> 本文主要介绍kafka manager的安装和使用


### 前言
&emsp;&emsp;Kafka manager是Yahoo开源的一个kafka GUI管理软件。
## 主要作用
为了简化开发者和服务工程师维护Kafka集群的工作，yahoo构建了一个叫做Kafka管理器的基于Web工具，叫做 Kafka Manager。这个管理工具可以很容易地发现分布在集群中的哪些topic分布不均匀，或者是分区在整个集群分布不均匀的的情况。它支持管理多个集群、选择副本、副本重新分配以及创建Topic。同时，这个管理工具也是一个非常好的可以快速浏览这个集群的工具，有如下功能：

* 管理多个kafka集群
* 便捷的检查kafka集群状态(topics,brokers,备份分布情况,分区分布情况)
* 选择你要运行的副本
* 基于当前分区状况进行
* 可以选择topic配置并创建topic(0.8.1.1和0.8.2的配置不同)
* 删除topic(只支持0.8.2以上的版本并且要在broker配置中设置delete.topic.enable=true)
* Topic list会指明哪些topic被删除（在0.8.2以上版本适用）
* 为已存在的topic增加分区
* 为已存在的topic更新配置
* 在多个topic上批量重分区
* 在多个topic上批量重分区(可选partition broker位置)

## 运行的环境要求
1. Kafka 0.8.1.1+
2. sbt 0.13.x
3. Java 7+
## 下载,编译,打包
获取kafka-manager源码，并编译打包，包会生成在(/usr/local/kafka-manager)
{% highlight bash %}
{% raw %}
git clone https://github.com/yahoo/kafka-manager
cd kafka-manager
./sbt clean dist
{% endraw %}
{% endhighlight %}
## 安装,配置,启动
在conf/application.conf中将kafka-manager.zkhosts的值设置为你的zk地址
```
kafka-manager.zkhosts="hostname1,hostname2:2181"
kafka-manager.zkhosts=${?ZK_HOSTS}

```
> 1. 解压

`unzip kafka-manager-1.3.4.zip`
> 2. 修改配置

`vim /usr/local/kafka-manager/kafka-manager/conf/application.conf`

> 3. 启动,指定配置文件位置和启动端口号，默认为9000(可以将启动命令和参数放到启动脚本里)

`vim start-kafka-manager.sh`

{% highlight bash %}
{% raw %}
KAFKA_MANAGER_HOME=/usr/local/kafka-manager/kafka-manager
$KAFKA_MANAGER_HOME/bin/kafka-manager -java-home /usr/java/jdk1.8.0_131/ -Dconfig.file=$KAFKA_MANAGER_HOME/conf/application.conf -Dhttp.port=9020 >> $KAFKA_MANAGER_HOME/logs/kafka-manager_start.log 2>&1 &
{% endraw %}
{% endhighlight %}

### tips:

使用sbt编译打包的时候时间可能会比较长，如果你hang在

`Loading project definition from /kafka-manager/project`
可以修改`project/plugins.sbt`中的`LogLevel`参数

将`logLevel := Level.Warn`修改为`logLevel := Level.Debug`

[kafka manager Git地址](https://github.com/yahoo/kafka-manager)