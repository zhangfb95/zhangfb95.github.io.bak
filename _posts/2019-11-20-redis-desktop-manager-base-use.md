---
layout: post
title:  深入使用RedisDesktopManager
date:   2019-11-20 20:25:32 +0800
categories: RedisDesktopManager
tag: RedisDesktopManager,工具
---

* content
{:toc}

工欲善其事，必先利其器。器者，工具也。

## 前言

Redis是一个强大的非关系型数据库，也是一个内存数据库，它基于键值对的方式对数据进行读写。因为所有数据操作都是基于内存，所以读写的效率都是非常高的。同时，它提供了很多命令来支持其读写，这些命令单个看上去都很简单，但是要想全部记住也非易事，尤其是这些命令基本都带有一些附加项。因此，有的时候，借助于一些可视化工具，可有效提升我们操作Redis的效率，也能降低我们使用Redis的门槛。

`RedisDesktopManager`是其中做得相对较好的一个工具，功能强大，简单易用，让我们来深入了解一下吧！

## 整体界面

因为安装过程非常简单，在此楼主省略掉了此步骤。安装之后，我们看到的是下面这样一个界面。

![整体界面](https://upload-images.jianshu.io/upload_images/845143-47452f72a3c13ea7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个图里面有以下几块内容：

1. Redis服务器列表
2. 导入/导出入口
3. 创建新的Redis服务器
4. 工作区
5. 操作日志及控制台命令
6. Tab标签

## 1. 创建一个Redis服务器

![创建Redis服务器](https://upload-images.jianshu.io/upload_images/845143-c3f5c7da799449e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个界面里，我们需要输入Redis的名称（自定义）、主机（主要是ip或域名）、端口（默认为6379）、授权信息（如登录密码）。注意：Redis服务器是没有用户名这个概念的。

有的场景，我们需要借助于跳板机来连接Redis服务器，这时我们需要配置SSH或者SSL。

![SSH配置界面](https://upload-images.jianshu.io/upload_images/845143-b1a8ed9174fef3d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个界面里，我们需要输入跳板机的主机、端口、用户名及私钥或密码。

## 2. 导入导出

导入和导出是基于Redis服务器的配置信息，交互的格式为json。

## 3. 基本操作

当我们对添加的redis信息或者具体的库进行单击操作，即打开了这个库。

![image.png](https://upload-images.jianshu.io/upload_images/845143-bd7cce09fd221d16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

+ `1` 为Redis服务器
+ `2` 为具体的Redis库（在Redis中，默认有16个库，库名为 0 - 15，我们可以通过redis.conf下的databases指令，来修改库的数量）
+ `3` 为redis的键名。为了简化我们的使用，我们可以通过将键名设置为英文冒号分割的形式，使其呈现为类似数据库的样式。如：`user:id:1`。

当点击具体的Redis键时，我们看到以下界面。包括：键、值、过期时间。同时我们可以对其进行简单的操作。包括：重命名、删除、重新加载和设置过期时间等。

![key/value操作界面](https://upload-images.jianshu.io/upload_images/845143-6db5a64a3e4df6c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4. 操作日志及控制台

在Redis服务器上，当我们右键点击`Console`，就可以进入控制台界面了。

![入口操作](https://upload-images.jianshu.io/upload_images/845143-6cf10c8a94ef0330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![控制台界面](https://upload-images.jianshu.io/upload_images/845143-2764999ef4c35f66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以在控制台上进行库的切换，切换的命令为：select {库名}。例如：select 0、select 1等。然后，我们就可以基于具体的库自由地输入我们的命令了。

![命令输入](https://upload-images.jianshu.io/upload_images/845143-8b111a71317d44b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

至此，我们已经对这个工具的简单使用做了最基本的入门式介绍。楼主使用过1.x版本，也是用过0.8.x版本，但最终发现0.8.x版本加载的效率会更高，尽管界面简单了一些。

另外，需要注意的是，这个工具其实是收费的，而且还不便宜。对于屌丝程序员来说，还是使用破解版比较好，网上有非常多的破解版本。
