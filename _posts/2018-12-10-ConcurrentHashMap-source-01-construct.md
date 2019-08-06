---
layout: post
title:  ConcurrentHashMap源码分析(01)-构造方法
date:   2018-12-10 16:50:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

`ConcurrentHashMap`作为并发工具集里面的一员，扮演着极其重要的角色。它支持`HashMap`的绝大多数功能，并且保证线程安全。为了线程安全，它内部的实现用到了锁、CAS和自旋等不同于`HashMap`的操作。

`ConcurrentHashMap`在jdk8中的实现，又有别于jdk7及以前的版本。在jdk7中，`ConcurrentHashMap`的实现是基于Segment分段锁的方式。而jdk8摒弃了分段锁技术，采用了性能更优的方式。

> 【注意】：本系列文章分析的`ConcurrentHashMap`是基于jdk8版本的，若读者需要jdk7及以前版本的，请自行查阅google。

## 重要的成员变量

```java
/**
 * The array of bins. Lazily initialized upon first insertion.
 * Size is always a power of two. Accessed directly by iterators.
 */
transient volatile Node<K,V>[] table;

/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 */
private transient volatile long baseCount;

/**
 * Table initialization and resizing control.  When negative, the
 * table is being initialized or resized: -1 for initialization,
 * else -(1 + the number of active resizing threads).  Otherwise,
 * when table is null, holds the initial table size to use upon
 * creation, or 0 for default. After initialization, holds the
 * next element count value upon which to resize the table.
 */
private transient volatile int sizeCtl;
```

`table`、`baseCount`和`sizeCtl`都是`volatile`类型的，说明了其可见性和有序性。

`table`和HashMap里面的类似，类似一个个的槽，每个槽的hash是不同的，同一个槽的hash相同。如果槽里面的元素个数不超过8个，则槽里面的元素使用链表。如果槽里面的元素个数超过8个，则槽里面的元素将转换为红黑树。

`baseCount`表示容器的大小。

`sizeCtl`用于`table`的初始化和扩容，有几种情况。

1. -1，表示table正在初始化
2. -N，表示有N-1个线程正在进行扩容操作
3. 其他情况
    1. 如果table未初始化，表示table需要初始化的大小
    2. 如果table初始化完成，表示table的容量，默认是table大小的0.75 倍。在源码里面使用的这个公式来算0.75（`n - (n >>> 2)`）

## 构造方法

`ConcurrentHashMap`总共有5个构造方法，我们来看看。

1. ConcurrentHashMap()
2. ConcurrentHashMap(int initialCapacity)
3. ConcurrentHashMap(int initialCapacity, float loadFactor)
4. ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)
5. ConcurrentHashMap(Map<? extends K, ? extends V> m)

其中的1是空方法。3最终调用的是4，传入参数`concurrencyLevel`值为1。而5对于我们来说使用频率比较低，这儿暂时不做分析。

因此，我们真正需要分析的是2和4。

### 1. ConcurrentHashMap(int initialCapacity)

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```

该方法首先会对传入的初始容量大小做校验，不允许超过最大容量。其次会将传入的容量大小进行`2^N`转换。最后将转换后的容量大小设置到`sizeCtl`。

### 2. ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel)

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    // 负载因子和并发级别的校验。
    // 并发级别指的是预估的可能同时修改的线程数量，我们可以根据它来调整我们的初始容量。
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

负载因子和并发级别的校验。并发级别指的是预估的可能同时修改的线程数量，我们可以根据它来调整我们的初始容量。默认的table.size为`(1.0+初始容量)/负载因子`。

## 总结

至此我们对`ConcurrentHashMap`有一定的入门级了解，同时分析完了`ConcurrentHashMap`的构造方法。