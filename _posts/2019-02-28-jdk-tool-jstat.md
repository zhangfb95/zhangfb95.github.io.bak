---
layout: post
title:  jdk工具-jstat
date:   2019-02-28 22:21:00 +0800
categories: 深入理解Java核心
tag: java
---

* content
{:toc}

## 简介

`jstat`是jdk工具中比较重要的一个，用于监控jvm的统计信息。它非常轻量，使用也很简单，只是结果字段会比较多。

## 语法

`jstat`的语法如下所示：

```
Usage: jstat -help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```

## 通用命令

从语法我们看出，其包含两个通用命令选项，`-help`和`-options`。

1. `-help`，列出语法
2. `-options`，列出option列表

我们通过运行命令`jstat -options`，可以看到显示效果，如下所示：

![jstat -options显示效果](https://upload-images.jianshu.io/upload_images/845143-1e1477393e6abb21.png)

## 常规命令

+ option
  + -class，显示类加载器的统计信息
  + -compiler，显示JIT(Just-In-Time Compiler，即时编译器)的相关信息
  + -gc，显示与gc相关的堆信息
  + -gccapacity，显示各代的容量及使用情况
  + -gccause，显示垃圾收集相关信息(同-gcutil)，显示最后一次或当前发生的垃圾收集的诱发原因
  + -gcmetacapacity，显示元空间的容量及使用情况（jdk8及以上）
  + -gcnew，显示新生代信息
  + -gcnewcapacity，显示新生代容量及使用情况
  + -gcold，显示老年代信息
  + -gcoldcapacity，显示老年代容量及使用情况
  + -gcutil，显示垃圾收集信息（百分比%）
  + -printcompilation，输出JIT编译的方法信息
+ -t，在输出的信息前加上一个Timestamp列，显示程序运行的时间(s)
+ -h，在周期性输出时，输出多少行后，跟着输出一个表头信息
+ interval，用于指定输出统计数据的周期，单位为毫秒
+ count，用于统计输出多少条数据

接下来，我们通过命令`jstat -class -h3 -t 317860 1000 6`大致看一下执行效果。

![jstat执行效果](https://upload-images.jianshu.io/upload_images/845143-5fa6744b5b2e3b8b.png)

### -class

```
Timestamp       Loaded  Bytes  Unloaded  Bytes     Time
         8825.0   7743 14259.8        2     2.1       3.84
/***/
# Loaded，加载类的数量
# 第一个Bytes，加载类的合计大小（kb）
# Unloaded，卸载类的数量
# 第二个Bytes，卸载类的合计大小（kb）
# Time，加载和卸载的时间总和
```

### -compiler

```
Timestamp       Compiled Failed Invalid   Time   FailedType FailedMethod
         9681.8     6621      0       0     2.39          0
/***/
# Compiled，编译任务执行数量
# Failed，编译任务执行失败数量
# Invalid ，编译任务执行失效数量
# Time ，编译任务消耗时间
# FailedType，最后一个编译失败任务的类型
# FailedMethod，最后一个编译失败任务所在的类及方法
```

### -gc

```
Timestamp        S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
           12.6 17408.0 15872.0  0.0    0.0   155648.0 54279.7   103424.0   25590.1   35416.0 34658.5 4736.0 4500.5      9    0.065   2      0.103    0.168
/***/
# S0C，年轻代中第一个survivor（幸存区）的容量 （kb）
# S1C，年轻代中第二个survivor（幸存区）的容量 (kb)
# S0U，年轻代中第一个survivor（幸存区）目前已使用空间 (kb)
# S1U，年轻代中第二个survivor（幸存区）目前已使用空间 (kb)
# EC，年轻代中Eden（伊甸园）的容量 (kb)
# EU，年轻代中Eden（伊甸园）目前已使用空间 (kb)
# OC，Old代的容量 (kb)
# OU，Old代目前已使用空间 (kb)
# MC，metaspace(元空间)的容量 (kb)
# MU，metaspace(元空间)目前已使用空间 (kb)
# YGC，从应用程序启动到采样时年轻代中gc次数
# YGCT，从应用程序启动到采样时年轻代中gc所用时间(s)
# FGC，从应用程序启动到采样时old代(全gc)gc次数
# FGCT，从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT，从应用程序启动到采样时gc用的总时间(s)
```

### -gccapacity

```
Timestamp        NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC
          168.1  43008.0 686080.0 238592.0 17408.0 15872.0 155648.0    86016.0  1372160.0   103424.0   103424.0      0.0 1079296.0  35416.0      0.0 1048576.0   4736.0      9     2
/***/
# NGCMN，年轻代(young)中初始化(最小)的大小(kb)
# NGCMX，年轻代(young)的最大容量 (kb)
# NGC，年轻代(young)中当前的容量 (kb)
# S0C，年轻代中第一个survivor（幸存区）的容量 (kb)
# S1C，年轻代中第二个survivor（幸存区）的容量 (kb)
# EC，年轻代中Eden（伊甸园）的容量 (kb)
# OGCMN，old代中初始化(最小)的大小 (kb)
# OGCMX，old代的最大容量(kb)
# OGC，old代当前新生成的容量 (kb)
# OC，old代的容量 (kb)
# MCMN，metaspace(元空间)中初始化(最小)的大小 (kb)
# MCMX，metaspace(元空间)的最大容量 (kb)
# MC，metaspace(元空间)当前新生成的容量 (kb)
# CCSMN，最小压缩类空间大小
# CCSMX，最大压缩类空间大小
# CCSC，当前压缩类空间大小
# YGC，从应用程序启动到采样时年轻代中gc次数
# FGC，从应用程序启动到采样时old代(全gc)gc次数
```

### -gccause或-gcutil

```
Timestamp         S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC
          241.6   0.00   0.00  35.27  24.74  97.86  95.03      9    0.065     2    0.103    0.168 Metadata GC Threshold No GC
/***/
# S0，年轻代中第一个survivor（幸存区）已使用的占当前容量百分比
# S1，年轻代中第二个survivor（幸存区）已使用的占当前容量百分比
# E，年轻代中Eden（伊甸园）已使用的占当前容量百分比
# O，old代已使用的占当前容量百分比
# P，perm代已使用的占当前容量百分比
# YGC，从应用程序启动到采样时年轻代中gc次数
# YGCT，从应用程序启动到采样时年轻代中gc所用时间(s)
# FGC，从应用程序启动到采样时old代(全gc)gc次数
# FGCT，从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT，从应用程序启动到采样时gc用的总时间(s)
# LGCC，最后一次或当前正在发生的垃圾回收的诱因
```

### -gcmetacapacity

```
Timestamp          MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT     GCT
          628.8        0.0  1079296.0    35416.0        0.0  1048576.0     4736.0     9     2    0.103    0.168
/***/
# MCMN，最小元数据容量
# MCMX，最大元数据容量
# MC，当前元数据空间大小
# CCSMN，最小压缩类空间大小
# CCSMX，最大压缩类空间大小
# CCSC，当前压缩类空间大小
# YGC，从应用程序启动到采样时年轻代中gc次数
# FGC，从应用程序启动到采样时old代(全gc)gc次数
# FGCT，从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT，从应用程序启动到采样时gc用的总时间(s)
```

### -gcnew

```
Timestamp        S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
          432.5 17408.0 15872.0    0.0    0.0  3  15 17408.0 155648.0  56332.8      9    0.065
/***/
# S0C，年轻代中第一个survivor（幸存区）的容量 (kb)
# S1C，年轻代中第二个survivor（幸存区）的容量 (kb)
# S0U，年轻代中第一个survivor（幸存区）目前已使用空间 (kb)
# S1U，年轻代中第二个survivor（幸存区）目前已使用空间 (kb)
# TT，持有次数限制
# MTT，最大持有次数限制
# DSS，期望的幸存区大小
# EC，年轻代中Eden（伊甸园）的容量 (kb)
# EU ，年轻代中Eden（伊甸园）目前已使用空间 (kb)
# YGC ，从应用程序启动到采样时年轻代中gc次数
# YGCT，从应用程序启动到采样时年轻代中gc所用时间(s)
```

### -gcnewcapacity

```
Timestamp         NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC
          472.6    43008.0   686080.0   238592.0 228352.0  17408.0 228352.0  15872.0   685056.0   155648.0     9     2
/***/
# NGCMN，年轻代(young)中初始化(最小)的大小(kb)
# NGCMX，年轻代(young)的最大容量 (kb)
# NGC，年轻代(young)中当前的容量 (kb)
# S0CMX，年轻代中第一个survivor（幸存区）的最大容量 (kb)
# S0C，年轻代中第一个survivor（幸存区）的容量 (kb)
# S1CMX，年轻代中第二个survivor（幸存区）的最大容量 (kb)
# S1C，年轻代中第二个survivor（幸存区）的容量 (kb)
# ECMX，年轻代中Eden（伊甸园）的最大容量 (kb)
# EC，年轻代中Eden（伊甸园）的容量 (kb)
# YGC，从应用程序启动到采样时年轻代中gc次数
# FGC，从应用程序启动到采样时old代(全gc)gc次数
```

### -gcold 

```
Timestamp          MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT
          526.7  35416.0  34658.5   4736.0   4500.5    103424.0     25590.1      9     2    0.103    0.168
/***/
# MC，metaspace(元空间)的容量 (kb)
# MU，metaspace(元空间)目前已使用空间 (kb)
# CCSC，压缩类空间大小
# CCSU，压缩类空间使用大小
# OC，Old代的容量 (kb)
# OU，Old代目前已使用空间 (kb)
# YGC，从应用程序启动到采样时年轻代中gc次数
# FGC，从应用程序启动到采样时old代(全gc)gc次数
# FGCT，从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT，从应用程序启动到采样时gc用的总时间(s)
```

### -gcoldcapacity

```
Timestamp          OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT
          574.7     86016.0   1372160.0    103424.0    103424.0     9     2    0.103    0.168
/***/
# OGCMN，old代中初始化(最小)的大小 (kb)
# OGCMX，old代的最大容量(kb)
# OGC，old代当前新生成的容量 (kb)
# OC，Old代的容量 (kb)
# YGC，从应用程序启动到采样时年轻代中gc次数
# FGC，从应用程序启动到采样时old代(全gc)gc次数
# FGCT，从应用程序启动到采样时old代(全gc)gc所用时间(s)
# GCT，从应用程序启动到采样时gc用的总时间(s)
```

### -printcompilation

```
Timestamp       Compiled  Size  Type Method
          685.0     4537     41    1 org/apache/catalina/LifecycleEvent <init>
/***/
# Compiled，编译任务的数目
# Size，方法生成的字节码的大小
# Type，编译类型
# Method，类名和方法名用来标识编译的方法。
          类名使用/做为一个命名空间分隔符。
          方法名是给定类中的方法。
          上述格式是由-XX:+PrintComplation选项进行设置的
```

## 常用例子

### 1. gcutil选项

这个例子分析的进程id为21891，采样了7次，样本间隔250毫秒，使用`-gcutil`输出结果。

在这个例子中，我们看到在第3次和第4次采样间发生了一次Young GC。这次GC花费了0.078秒，并且将一些对象从年轻代提升到了老年代，导致的结果就是老年代的空间使用率从66.80%提升到了68.19%。在这次GC之前，幸存区的利用率为97.02%，但是GC之后幸存区的利用率却是91.03%。

```
jstat -gcutil 21891 250 7
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00  97.02  70.31  66.80  95.52  89.14      7    0.300     0    0.000    0.300
  0.00  97.02  86.23  66.80  95.52  89.14      7    0.300     0    0.000    0.300
  0.00  97.02  96.53  66.80  95.52  89.14      7    0.300     0    0.000    0.300
 91.03   0.00   1.98  68.19  95.89  91.24      8    0.378     0    0.000    0.378
 91.03   0.00  15.82  68.19  95.89  91.24      8    0.378     0    0.000    0.378
 91.03   0.00  17.80  68.19  95.89  91.24      8    0.378     0    0.000    0.378
 91.03   0.00  17.80  68.19  95.89  91.24      8    0.378     0    0.000    0.378
```

### 2. 重复的表头信息显示

这个例子分析的进程id为21891，样本间隔250毫秒，使用`-gcnew`输出结果。另外，它使用了`-h3`选项用于每隔3行输出一次表头信息。

另外，在第2次和第3次样本之间发生了一次Young GC，持续时间为0.001秒。这次GC找到了足够多的存活对象，S0中对象的存活次数超过了指定的阈值，tenuring threshold（DSS）。结果就是，这些存活的对象被提升到老年代（老年代的情况在`-gcnew`时不可见）。新生代对象的存活次数由31降低为2。

```
jstat -gcnew -h3 21891 250
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
  64.0   64.0    0.0   31.7 31  31   32.0    512.0    178.6    249    0.203
  64.0   64.0    0.0   31.7 31  31   32.0    512.0    355.5    249    0.203
  64.0   64.0   35.4    0.0  2  31   32.0    512.0     21.9    250    0.204
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
  64.0   64.0   35.4    0.0  2  31   32.0    512.0    245.9    250    0.204
  64.0   64.0   35.4    0.0  2  31   32.0    512.0    421.1    250    0.204
  64.0   64.0    0.0   19.0 31  31   32.0    512.0     84.4    251    0.204
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
  64.0   64.0    0.0   19.0 31  31   32.0    512.0    306.7    251    0.204
```

### 3. 每个样本都包含时间戳

这个例子分析的进程id为21891，样本间隔250毫秒，使用`-t`选项在每条样本的第一列生成一个时间戳，这个时间戳记录了从jvm进程启动到当前时间的秒数。该例子使用`-gcoldcapacity`输出结果。
在这个例子中，我们看到老年代的容量，在1/8Full GC的时间内，已经从11.696kb上升到13.820kb，而老年代最大容量为60.544kb。

```
Timestamp      OGCMN    OGCMX     OGC       OC       YGC   FGC    FGCT    GCT
          150.1   1408.0  60544.0  11696.0  11696.0   194    80    2.874   3.799
          150.4   1408.0  60544.0  13820.0  13820.0   194    81    2.938   3.863
          150.7   1408.0  60544.0  13820.0  13820.0   194    81    2.938   3.863
```


## 参考链接

+ https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html
+ https://docs.oracle.com/javase/7/docs/technotes/tools/share/jstat.html
+ https://blog.csdn.net/it_freshman/article/details/80833323
+ https://www.cnblogs.com/lizhonghua34/p/7307139.html
+ https://www.jianshu.com/p/213710fb9e40
+ https://www.cnblogs.com/davygeek/p/5668747.html