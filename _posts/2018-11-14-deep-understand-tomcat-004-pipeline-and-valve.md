---
layout: post
title:  深入理解Tomcat（四）Pipeline和Valve
date:   2018-11-14 16:59:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

Tomcat的前身为Catalina，而Catalina又是一个轻量级的Servlet容器。在美国，catalina是一个很美的小岛。所以Tomcat作者的寓意可能是想把Tomcat设计成一个优雅美丽且轻量级的web服务器。Tomcat从4.x版本开始除了作为支持Servlet的容器外，额外加入了很多的功能，比如：jsp、el、naming等等，所以说Tomcat不仅仅是Catalina。

既然Tomcat首先是一个Servlet容器，我们应该更多的关心Servlet。

那么，什么是Servlet呢？

在互联网兴起之初，当时的Sun公司（后面被Oracle收购）已然看到了这次机遇，于是设计出了Applet来对Web应用的支持。不过事实却并不是预期那么得好，Sun悲催地发现Applet并没有给业界带来多大的影响。经过反思，Sun就想既然机遇出现了，市场前景也非常不错，总不能白白放弃了呀，怎么办呢？于是又投入精力去搞一套规范出来，这时Servlet诞生了！

> 所谓Servlet，其实就是Sun为了让Java能实现动态可交互的网页，从而进入Web编程领域而制定的一套标准！

一个Servlet主要做下面三件事情：

1. 创建并填充Request对象，包括：URI、参数、method、请求头信息、请求体信息等
2. 创建Response对象
3. 执行业务逻辑，将结果通过Response的输出流输出到客户端

Servlet没有main方法，所以，如果要执行，则需要在一个`容器`里面才能执行，这个容器就是为了支持Servlet的功能而存在，Tomcat其实就是一个Servlet容器的实现。

## 整体架构图

![整体架构图](https://upload-images.jianshu.io/upload_images/845143-523d7f34094a2911.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图我们看出，最核心的两个组件--连接器（Connector）和容器（Container）起到`心脏`的作用，他们至关重要！下面我们逐一来分析其功能：

1. `Server`表示服务器，提供了一种优雅的方式来启动和停止整个系统，不必单独启停连接器和容器
2. `Service`表示服务，`Server`可以运行多个服务。比如一个Tomcat里面可运行订单服务、支付服务、用户服务等等
3. 每个`Service`可包含`多个Connector`和`一个Container`。因为每个服务允许同时支持多种协议，但是每种协议最终执行的Servlet却是相同的
4. `Connector`表示连接器，比如一个服务可以同时支持AJP协议、Http协议和Https协议，每种协议可使用一种连接器来支持
5. `Container`表示容器，可以看做Servlet容器
    + `Engine` -- 引擎
    + `Host` -- 主机
    + `Context` -- 上下文
    + `Wrapper` -- 包装器
6. Service服务之下还有各种`支撑组件`，下面简单罗列一下这些组件
    + `Manager` -- 管理器，用于管理会话Session
    + `Logger` -- 日志器，用于管理日志
    + `Loader` -- 加载器，和类加载有关，只会开放给Context所使用
    + `Pipeline` -- 管道组件，配合Valve实现过滤器功能
    + `Valve` -- 阀门组件，配合Pipeline实现过滤器功能
    + `Realm` -- 认证授权组件

除了连接器和容器，管道组件和阀门组件也很关键，我们通过一张图来看看这两个组件

![pipeline+valve](https://upload-images.jianshu.io/upload_images/845143-286605040f90d472.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 换位思考Tomcat架构

我们从大的方向来看下Tomcat架构，大体涉及到下面3个方面。

1. 基于组件的架构
2. 基于JMX
3. 基于生命周期

首先，我们说一下什么是`基于组件的架构`

> 通过上一小节，我们知道了组成Tomcat的是各种各样的组件，每个组件各司其职，组件与组件之间有明确的职责划分，同时组件与组件之间又通过一定的联系相互通信。Tomcat整体就是一个个组件的堆砌！

其次，我们说一下为什么是`基于JMX`

> 我们在后续阅读Tomcat源码的时候，会发现代码里充斥着大量的类似于下面的代码。
`Registry.getRegistry(null, null).invoke(mbeans, "init", false); `
`Registry.getRegistry(null, null).invoke(mbeans, "start", false); `
而这实际上就是通过JMX来管理相应对象的代码。这儿我们不会详细讲述什么是JMX，我们只是简单地说明一下JMX的概念，参考[JMX百度百科](https://baike.baidu.com/item/JMX/2829357?fr=aladdin)。
JMX（Java Management Extensions，即Java管理扩展）是一个为应用程序、设备、系统等[植入](https://baike.baidu.com/item/%E6%A4%8D%E5%85%A5/7958584)管理功能的框架。JMX可以跨越一系列异构操作系统平台、[系统体系结构](https://baike.baidu.com/item/%E7%B3%BB%E7%BB%9F%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84/6842760)和[网络传输协议](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/332131)，灵活的开发无缝集成的系统、网络和服务管理应用。

最后，我们说一下为什么是基于生命周期。

> 我们在第二篇文章中介绍了`Lifecycle接口`。如果我们查阅各个组件的源代码，会发现绝大多数组件实现了该接口，这也就是我们所说的`基于生命周期`。生命周期的各个阶段的触发又是`基于事件`的方式。
参考link：[深入理解Tomcat（二）Lifecycle](https://www.jianshu.com/p/2a9ffbd00724)

## 总结

好了，我们已经从整体上看到了Tomcat的结构，但是对于每个组件我们并没有详细分析。后续章节我们会从几个方面来学习Tomcat：

1. 逐一分析各个组件
2. 通过断点的方式来跟踪Tomcat代码中的一次完整请求

希望通过这种方式加深自己对Tomcat的理解，同时给想要深入学习Tomcat的同学带来一些帮助。