---
layout: article
title:  "流式计算flume-kafka-sparkstreaming案例(一)"
categories: hadoop
toc: true
image:
    teaser: /teaser/spark.png
---

> 这篇内容主要介绍流式计算之Flume阶段




### 前言
&emsp;&emsp;
{% highlight bash %}
{% raw %}
# memory channel called ch1 on web1
web1.channels.ch1.type = memory
web1.channels.ch1.capacity=500000
web1.channels.ch1.transactionCapacity=5000

# Define a tail source
web1.sources.tail1.channels=ch1
web1.sources.tail1.type=exec
#web1.sources.tail1.command=tail -n +0 -F /usr/local/serverlog/serverlog
web1.sources.tail1.command=tail -F /usr/local/serverlog/serverlog
web1.sources.tail1.interceptors = ts
web1.sources.tail1.restart=true
web1.sources.tail1.interceptors.ts.type = timestamp



# Define an Avro sink main
web1.sinks.avro_main.type = avro
web1.sinks.avro_main.hostname = 172.21.150.50
web1.sinks.avro_main.port = 41413
web1.sinks.avro_main.channel= ch1
web1.sinks.avro_main.batchsize =5000

# Define an Avro sink backup
web1.sinks.avro_backup.type = avro
web1.sinks.avro_backup.hostname = 172.21.150.50
web1.sinks.avro_backup.port = 41413
web1.sinks.avro_backup.channel = ch1
web1.sinks.avro_backup.batchsize = 5000

# Define web1 sink group
web1.sinkgroups.app1_group.sinks=avro_main avro_backup
web1.sinkgroups.app1_group.processor.type= failover
web1.sinkgroups.app1_group.processor.priority.avro_main = 5
web1.sinkgroups.app1_group.processor.priority.avro_backup = 10

# Define web1 sink group load balance
web1.sinkgroups.app2_group.sinks=avro_main avro_backup
web1.sinkgroups.app2_group.processor.type = load_balance
web1.sinkgroups.app2_group.processor.selector = ROUND_ROBIN

#web1.sinkgroups.app2_group.processor.backoff=true
#web1.sinkgroups.app2_group.processor.selector.maxTimeOut=15000


# Finally, now that we've defined all of our components, tell
# web1 which ones we want to activate.
web1.channels = ch1
web1.sources = tail1
web1.sinkgroups = app2_group
web1.sinks = avro_main avro_backup
                                   
{% endraw %}
{% endhighlight %}