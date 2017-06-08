---
layout: article
title:  "配置Bitvise SSH Client以及proxifier做全局代理"
categories: linux
image:
    teaser: /teaser/ssh.jpg
---

> 本文主要介绍配置Bitvise SSH Client以及proxifier做全局代理

### 前言
&emsp;&emsp;由于公司一直对于数据库管理很严格，除了运维部门，都需要通过跳板机等来进行连接，基本是不允许直连的。有时候公司给一远程电脑连接权限让大家连接。对于像很多数据开发人员是很头疼的事情。但是为了安全考虑也就只能接受了。但是今天我想说下我的方法



# 什么是SSH
---

简单说，SSH是一种网络协议，用于计算机之间的加密登录。
如果一个用户从本地计算机，使用SSH协议登录另一台远程计算机，我们就可以认为，这种登录是安全的，即使被中途截获，密码也不会泄露。
最早的时候，互联网通信都是明文通信，一旦被截获，内容就暴露无疑。1995年，芬兰学者Tatu Ylonen设计了SSH协议，将登录信息全部加密，成为互联网安全的一个基本解决方案，迅速在全世界获得推广，目前已经成为Linux系统的标准配置。
需要指出的是，SSH只是一种协议，存在多种实现，既有商业实现，也有开源实现。本文针对的实现是OpenSSH，它是自由软件，应用非常广泛。

---

## Bitvise SSH Client
首先配置Bitvise SSH Client(由于涉及到公司内部的地址和端口，所以将一些敏感词给屏蔽了)
### 将地址，账号，密码加进去。端口基本都是22
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffomhc5bzbj20hk0h1q51.jpg)

### 配置端口
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffomlbulenj20hx0h5q5c.jpg)

1. 监听端口这里设为7070，也可以设为其他的，也可以默认设置的。
但是如果设置一定要与后面的保持一致！
但是不要端口冲突就好
下来点击登录

2. 这样就代表我们成功登入服务器，下面来配置本地服务器
我们需要记住一个IP地址，就是我们刚刚在Services选项配置的
127.0.0.1 还有刚刚填写的端口号，我填写的是7070，好。。我们记下来了，开始配置proxifier

## Proxifier
Proxifier for Mac 是一款功能非常强大的socks5客户端，可以让不支持通过代理服务器工作的网络程序能通过HTTPS或SOCKS代理或代理链。

这款应用同时有 Mac 也有 Windows 版本，其中 Windows 版已经更新到 3.x 了，功能是一样的。

### 配置代理服务器
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffompy1ncsj20rl0iqtby.jpg)
### 配置完成检查测试
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffomqz450xj20g009v760.jpg)

> __表示通过代理服务器连接成功SUCCESS__

### 配置代理规则
由于是我想打算跳过跳板机直连oracle数据库，所以就把目标主机地址填进去就可以
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffomu7sia5j20de0dldgf.jpg)
![](http://ww1.sinaimg.cn/large/a8a646f9ly1ffomvmm9hvj20rq0izdim.jpg)
当显示以上信息的话，就表明我可以直连数据库了。
