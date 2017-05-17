---
layout: article
title:  "配置Bitvise SSH Client以及proxifier做全局代理"
categories: linux
image:
    teaser: /teaser/tic_tok.png
---

>背景：由于公司一直对于数据库管理很严格，除了运维部门，都需要通过跳板机等来进行连接，基本是不允许直连的。有时候公司给一远程电脑连接权限让大家连接。对于像很多数据开发人员是很头疼的事情。但是为了安全考虑也就只能接受了。但是今天我想说下我的方法



# 什么是SSH
---

简单说，SSH是一种网络协议，用于计算机之间的加密登录。
如果一个用户从本地计算机，使用SSH协议登录另一台远程计算机，我们就可以认为，这种登录是安全的，即使被中途截获，密码也不会泄露。
最早的时候，互联网通信都是明文通信，一旦被截获，内容就暴露无疑。1995年，芬兰学者Tatu Ylonen设计了SSH协议，将登录信息全部加密，成为互联网安全的一个基本解决方案，迅速在全世界获得推广，目前已经成为Linux系统的标准配置。
需要指出的是，SSH只是一种协议，存在多种实现，既有商业实现，也有开源实现。本文针对的实现是OpenSSH，它是自由软件，应用非常广泛。

![tic-toc roadmap]({{ site.url }}/images/linux/intel-tic-toc/tic-toc-roadmap.jpg)


# 近几年的tic-toc线路图
---

tic/toc|micro-architecture|fabrification process
----|----|-----
tic|P6/NetBurst|65nm
toc|Core|65nm
tic|Core|45nm
toc|Nehalem|45nm
tic|Nehalem|32nm
toc|Sandy Bridge|32nm
tic|Sandy Bridge|22nm
toc|Haswell|22nm
tic|Haswell|14nm
toc|Skylake|14nm
tic|Skylake|10nm



## Reference

1. [Tick-Tock on wikipidia](http://en.wikipedia.org/wiki/Intel_Tick-Tock)
2. [The Tick-Tock Model Through the Years](http://www.intel.com/content/www/us/en/silicon-innovations/intel-tick-tock-model-general.html)


