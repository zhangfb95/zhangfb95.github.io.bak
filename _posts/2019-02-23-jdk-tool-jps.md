---
layout: post
title:  jdk工具-jps
date:   2019-02-23 00:20:00 +0800
categories: JVM工具
tag: java
---

* content
{:toc}

## 简介

jps，英文为"Java Virtual Machine Process Status Tool"，中文可翻译成"Java虚拟机进程状态工具"。jps的作用主要是查看有权访问Hotspot虚拟机的进程。

我们可以使用`jps -help`命令列出其主要语法，如下所示：

```
usage: jps [-help]
       jps [-q] [-mlvV] [<hostid>]

Definitions:
    <hostid>:      <hostname>[:<port>]
```

## 基本用法

1. 不带任何参数，默认列出进程pid和简单的class或jar名称

![不带任何参数](https://upload-images.jianshu.io/upload_images/845143-d465e00f84158a4c.png)


2. -q，仅显示进程编号，不显示class、jar及传入参数等其他信息.

![参数，-q](https://upload-images.jianshu.io/upload_images/845143-132818c709bd4127.png)

3. -m，输出main()函数传入的参数

![参数，-m](https://upload-images.jianshu.io/upload_images/845143-95d0470146aedf28.png)

4. -l，输出应用程序主类的完整package名称或完整jar名称.

![参数，-l](https://upload-images.jianshu.io/upload_images/845143-564e79de0329e9cc.png)

5. -v，列出jvm参数。如：-Xms20m -Xmx50m，是启动程序时所指定的jvm参数

![参数，-v](https://upload-images.jianshu.io/upload_images/845143-b2caafef4c0863dd.png)

6. -V:输出通过.hotsportrc或-XX:Flags=<filename>指定的jvm参数

暂无例子

## 获取远程物理机的jps信息

jps除了查看本机jvm进程的信息之外，还可以查看远程物理机的jvm进程信息。要想远程查看jps信息，需要在远程物理机上启动jstatd服务。

那么，如何启动呢？

第一步，启动jstatd需要足够的权限，因此我们需要基于java的安全策略分配相应的权限。在`{JAVA_HOME}/bin/`目录下创建`jstatd.all.policy`策略文件，输入如下信息：

```
grant codebase "file:${java.home}/../lib/tools.jar" {
    permission java.security.AllPermission;
}
```

第二步，通过下述命令启动jstatd

```
jstatd -J-Djava.security.policy=jstatd.all.policy
```

第三步，默认情况下，jstatd启动的端口为1099，若想指定端口，需要加上-p参数。另外，我们可以使用netstat命令查看jstatd启动之后的端口号

```
# 指定端口
jstatd -J-Djava.security.policy=jstatd.all.policy -p 9030
# 查看jstatd启动后的端口
netstat -apn | grep jstatd
```

## jps实现原理

Java程序在启动以后，会在`java.io.tmpdir`所在的目录下，生成一个类似于`hsperfdata_{User}`的文件夹，这个文件夹里面有几个文件，名称就是Java进程的pid。因此，要想列出当前运行的Java进程，只需要将这个目录下的文件名遍历一下即可。 至于系统的参数等信息，则需要解析这几个文件的内容才能获得。

【注意】：

1. `java.io.tmpdir`所在的目录即为临时文件夹
2. `hsperfdata_{User}`文件夹在Linux和Windows下分别如下
    + Linux：/tmp/hsperfdata_{userName}/
    + Windows：C:\Users\\{userName}\AppData\Local\Temp\hsperfdata_{userName}

![linux演示](https://upload-images.jianshu.io/upload_images/845143-3a095371d335e6d9.png)

![Windows演示](https://upload-images.jianshu.io/upload_images/845143-0509ca024eb66b91.png)

## 总结

通过这篇文章，我们对jps有了一个清晰直观的了解，基本掌握了其语法及使用。在最后，我们给出了jps的实现原理。

## 参考链接

+ https://www.cnblogs.com/tulianghui/p/5914535.html
+ https://www.jianshu.com/p/d39b2e208e72
+ https://blog.csdn.net/gtuu0123/article/details/6025484
+ https://www.jianshu.com/p/d39b2e208e72
+ https://www.jianshu.com/p/9f70a980e40a