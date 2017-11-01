> 本文主要介绍使用intellijidea基础功能

### 前言
&emsp;&emsp;以下内容主要是通过在学习和工作中的一些总结和感悟，很多概念上的东西都主要来自于书籍和网络，毕竟这些基础知识还是以官网为准。
## 概述
Mac版IntelliJ Idea升级到2017.01后，运行程序时，Consoles提示框中会出现如下红色警告：
{% highlight bash %}
{% raw %}
objc[2163]: Class JavaLaunchHelper is implemented in both /Library/Java/JavaVirtualMachines/jdk1.8.0_65.jdk/Contents/Home/bin/java (0x10ba504c0) and /Library/Java/JavaVirtualMachines/jdk1.8.0_65.jdk/Contents/Home/jre/lib/libinstrument.dylib (0x10cab54e0). One of the two will be used. Which one is undefined.
{% endraw %}
{% endhighlight %}
### 目前stackoverflow上主流的意见如下：
{% highlight bash %}
{% raw %}
You can find all the details here:

IDEA-170117 "objc: Class JavaLaunchHelper is implemented in both ..." warning in Run consoles
It's the old bug in Java on Mac that got triggered by the Java Agent being used by the IDE when starting the app. This message is harmless and is safe to ignore. Oracle developer's comment:

The message is benign, there is no negative impact from this problem since both copies of that class are identical (compiled from the exact same source). It is purely a cosmetic issue.
The problem is fixed in Java 9 and in Java 8 update 152.

If it annoys you or affects your apps in any way, the workaround for IntelliJ IDEA is to disable idea_rt launcher agent by adding idea.no.launcher=true into idea.properties (Help | Edit Custom Properties...).
{% endraw %}
{% endhighlight %}
* 总的来说，上述警告信息是Java Mac版本的一个Bug，即IDE启动app时被java代理器触发了这个警告。不过这个警告对程序没有任务伤害，你可以选择忽视它。
* a simple and safe workaround is that you can fold this buzzing message in console by IntelliJ IDEA settings:
    * 【Preferences】- 【Editor】-【General】-【Console】- 【Fold console lines that contain】
Of course, you can use 【Find Action...】(cmd+shift+A on mac) and type Fold console lines that contain so as to navigate more effectively.
    * add `Class JavaLaunchHelper is implemented in both`
![idea配置图](/images/linux/huge_page/Snip20171101_22.png)
* 这样就可以解决

