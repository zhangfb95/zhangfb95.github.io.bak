---
layout: post
title:  jdk工具-jinfo
date:   2019-02-28 23:18:00 +0800
categories: 深入理解Java核心
tag: java
---

* content
{:toc}

## 简介

`jinfo`主要用于打印配置信息，包括命令行参数、系统变量。极少数的情况下，我们可以用其来修改命令行参数。

## 语法

```
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
```

最主要的语法只有一个`jinfo [option] <pid>`。pid表示Java进程id，而对于option，我们将逐一来进行分析。

### 1. -flag <name>

用于打印虚拟机标记参数的值，name表示虚拟机标记参数的名称。

```
C:\Users\zhangfubing>jinfo -flag PrintGC 21768
-XX:-PrintGC
```

### 2. -flag [+|-]<name>

用于开启或关闭虚拟机标记参数。+表示开启，-表示关闭。

![演示效果](https://upload-images.jianshu.io/upload_images/845143-df0d0e4e484cfb01.png)

### 3. -flag <name>=<value>

用于设置虚拟机标记参数，但并不是每个参数都可以被动态修改的。

![演示效果](https://upload-images.jianshu.io/upload_images/845143-aff657dc7bc7815c.png)

### 4. -flags

打印虚拟机参数。什么是虚拟机参数呢？如`-XX:NewSize,-XX:OldSize`等就是虚拟机参数。

![演示效果](https://upload-images.jianshu.io/upload_images/845143-44e63a4656ba7669.png)

### 5. -sysprops

打印系统参数。

![演示效果](https://upload-images.jianshu.io/upload_images/845143-146000066a075a53.png)

### 6. <no option>

不带任何选项时，会同时打印虚拟机参数和系统参数。

![演示效果](https://upload-images.jianshu.io/upload_images/845143-5d4d4d436129a2a2.png)

### 7. -h | -help

打印帮助信息。

## 总结

`jinfo`可用于打印和动态修改虚拟机参数，也可以打印系统参数。功能强大，但使用方式却很简单。另外，相对于`jstat`、`jstack`来说，`jinfo`的用法要简单很多。

## 参考链接

+ https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jinfo.html
+ https://blog.csdn.net/it_freshman/article/details/80833323