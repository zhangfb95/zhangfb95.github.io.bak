---
layout: post
title: 我的计划
permalink: /plan/
---

* content
{:toc}

# 1. doing

## java并发编程

+ Java线程池 — Executors工厂类 / 线程池对象的参数及工作机制 / 源码剖析

# 2. todo

## java基础

+ java的运行，命令包括：java、javac
+ 基本类型及其包装类型
+ Object的方法
+ String的基本用法，源码阅读
+ private、default、protected和public对类和方法的修饰
+ static关键字
+ 集合框架，Collection（List、Set和Queue）、Map、Collections
+ 异常体系，继承关系，运行时异常和非运行时异常、异常堆栈
+ IO（InputStream、OutputStream、Reader和Writer）
+ NIO（Channel、SelectedKey等）
+ 时区和时间（SimpleDateFormat、DateTimeFormatter、DateFormatter），Date和时区TimeZone
+ 泛型
+ 注解
+ Class、Type
+ 反射
+ JNI

### 时区参考

+ https://www.hollischuang.com/archives/3017
+ https://www.hollischuang.com/archives/3082

## java并发编程

+ 内存模型
+ volatile
+ Happen-Before原则
+ 线程协作类 — Semaphore / CountDownLatch / CyclicBarrier / Exchanger
+ 线程协作工具 — LockSupport
+ CAS原理
+ 锁优化 — 偏向锁 / 轻量级锁 / 重量级锁 / 锁消除 / 锁粗化 / 锁分离 / 如何自己优化锁
+ 管道流 — PipedWriter / PipedReader / PipedInputStream / PipedOutputStream
+ Fork-Join框架
+ 死锁 — 什么是死锁 / 如何检测 / 形成原因及如何解决
+ AQS — CLH锁
+ 线程中断
+ 源码阅读：ConcurrentHashMap — 各个方法 / 如何扩容
+ 源码阅读：LinkedBlockingQueue
+ 源码阅读：CopyOnWriteArrayList
+ 源码阅读：ThreadLocal
+ 源码阅读：AbstractQueuedSynchronizer — 抽象队列同步器，也叫AQS
+ 源码阅读：ConcurrentLinkedQueue
+ 源码阅读：SynchronousQueue
+ 源码阅读：CountDown
+ 源码阅读：CyclicBarrier
+ 源码阅读：Semaphore
+ 源码阅读：Exchanger
+ 源码阅读：ReentrantLock, 公平锁和非公平锁
+ 源码阅读：ReentrantReadWriteLock
+ 源码阅读：Timer
+ 源码阅读：FutureTask
+ 源码阅读：SheduledThreadPoolExecutor
+ 源码阅读：LinkedTransferQueue

## 深入理解netty

+ IO模式，同步和异步，阻塞和非阻塞
+ select / poll / epoll等算法
+ 网卡缓冲区、内核缓冲区和用户缓冲区
+ Reactor模式
+ 源码阅读

### 参考

+ https://blog.csdn.net/davidsguo008/article/details/73556811
+ https://www.jianshu.com/p/1ccbc6a348db
+ http://blog.chinaunix.net/uid-22906954-id-4161625.html

## vcs: git

## service mesh / k8s / docker

## 算法+数据结构

+ 数组
+ 链表
+ 双向链表
+ 树、二叉树、二叉查找树、平衡二叉树、红黑树、B树、B+树、B*树
+ LRU
+ 雪花算法，升级版的[参考链接](https://juejin.im/entry/58e6f6d244d904006d379737)
+ 一致性hash算法
+ 权重负载均衡算法

## 设计模式

+ SOLID原则
+ 代理模式、动态代理、静态代理、正向代理、反向代理。。。
+ 装饰模式

## nginx

## 微服务

## 分布式

+ 高并发和高可用
+ session和分布式session
+ 分布式事务
+ 分布式单点登录

## spring全家桶

## 持久层，mybatis、JPA和Hibernate

## 关系型数据库，Mysql

+ dml
+ ddl
+ 事务、explain、优化、存储引擎、乐观锁、悲观锁、复制算法和主从、切片、冷热库、索引结构
+ 函数索引，like、大于、小于、between in、等索引情况

## redis

+ 常用数据结构
+ 淘汰策略

## 常用类库源码阅读

## 日志框架

logback、slf4j、log4j等

## 消息中间件

kafka / rocketmq / rabbitmq /...

## mongodb

使用场景、语法、内部实现等

## 任务调度

quartz / 分布式任务调度器

## gc 和 jvm

+ 性能调优监控工具
+ jvm参数
+ class字节码
+ 类加载机制
+ jvm源码分析

## zookeeper

## netflix oss

+ feign
+ ribbon
+ eureka
+ hystrix
+ zuul

## spring cloud

## spring boot

## servlet和web容器

## 网络攻击和网络安全

## DNS和域名解析、CDN

## Linux操作系统

+ 常用命令
+ 常用系统性能监控工具，如：top、vmstat等
+ 文件系统、描述符和句柄
+ 内存、磁盘、cpu、网络、IO、时钟

# 3. done

## java并发编程

+ wait / notify
+ Thread过期方法和常用方法 — stop / suspend / resume(过期); interrupt / isInterrupted / static interrupted / join / yield / sleep / holdsLock / setContextClassLoader
+ 锁 — synchronized / ReentrantLock / ReadWriteLock
