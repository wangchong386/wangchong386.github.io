---
layout: article
title:  "Flume 拦截器interceptor（一）"
categories: flume
toc: true
image:
    teaser: /teaser/flume-intercepter1.png
---

> 这篇主要介绍Flume Interceptor 拦截器

## 拦截器
> 拦截器是一段代码，可以基于某些标准或者自定义的规则对需要的事件进行读取，修改或者删除。每个source可以配置使用多个拦截器，按照配置中定义的顺序被调用，将拦截器的结果传递给下一个单元。这就是所谓的责任链(chain-of-responsibility)的设计模式，一旦拦截器处理完事件。拦截器链返回的的事件列表传递到channel列表中，既通过channel选择器为每个事件进行，一般拦截器与channel选择器共同使用效果比较好。将事件传到多个channel.

![](/images/hadoop/flume/flume-intercepter1.png)

* (4) 将每个事件传递给Channel选择器
* (5) 返回写入事件的Channel列表
* (6) 将多有事件写入每个必需的channel
* (7) 利用可选channel重复相同的操作

所有拦截器通用的唯一参数是type参数，该参数必需是拦截器的别名或者Builder类的完全限定名(DQCN)，该Builder类用于创建拦截器

## 介绍几个常见的拦截器

* Timestamp Interceptor
这是Flume最常用的一个拦截器：时间戳拦截器，该拦截器将时间戳插入到Flume的事件报头中，
![](/images/hadoop/flume/flume-intercepter2.png)
例如：
{% highlight bash %}
{% raw %}
agent.sources.avro.interceptors = timestampInterceptor
agent.sources.avro.interceptors.timestampInterceptor.type = timestamp
agent.sources.avro.interceptors.preserveExisting = false

{% endraw %}
{% endhighlight %}

* Host Interceptor
顾名思义就是将ip地址或者主机名插入到Flume事件的报头中，
![](/images/hadoop/flume/flume-intercepter3.png)
样例：配置主机拦截器将主机名写入到事件报头中，如果事件报头已经存在，则不会替换
{% highlight bash %}
{% raw %}
agent.sources.avro.interceptors=hostinterceptor
agent.sources.avro.interceptors.hostinterceptor.type=host
agent.sources.avro.interceptors.hostinterceptor.useIp = false
agent.sources.avro.interceptors.hostinterceptor.preserveExisting = true
{% endraw %}
{% endhighlight %}

* 其他的还有：
Static Interceptor；
UUID Interceptor；
Morphline Interceptor；
Search and Replace Interceptor；
Regex Filtering Interceptor；
Regex Extractor Interceptor；



* Source读取events发送到Sink的时候，在events header中加入一些有用的信息，或者对events的内容进行过滤，完成初步的数据清洗。这在实际业务场景中非常有用