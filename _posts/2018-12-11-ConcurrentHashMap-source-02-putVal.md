---
layout: post
title:  ConcurrentHashMap源码分析(02)-putVal()方法
date:   2018-12-11 09:40:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

上一章节，我们对构造方法进行了分析，接下来我们要分析的是元素的插入。在Map接口的方法定义里面，`put()`方法的职责就是插入元素。而`Concurrent.put()`方法又直接调用了`putVal()`方法，那么我们分析的重点将放在`putVal()`上面。

## put()方法

我们先来看看`put()`方法是否如前言所描述那般。根据下面的源码，我们得到了证明。

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

## putVal()方法

先来看看这个方法的源码。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 1. 如果key或者value为null，抛出空指针异常，这点和`HashMap`有所不同
    if (key == null || value == null) throw new NullPointerException();
    // 2. 根据key的hashCode()，重新生成hash
    int hash = spread(key.hashCode());
    int binCount = 0;
    // 3. 这儿是空循环，也就是自旋操作
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 3.1. 如果`table`为null或长度为0，需要对`table`进行初始化。从上一节构造方法的分析来看，构造方法并不会初始化`table`。初始化`table`延迟到`putVal()`方法
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 3.2. 如果hash没有冲突，则使用CAS插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 3.3. 如果hash冲突了，且槽里面第一个元素的hash为MOVED(-1)。则说明槽里面的元素为`ForwardingNode`对象，有其他线程正在扩容。当前线程通过`helpTransfer`进行辅助扩容，扩容完成之后再进行插入操作
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 3.4. 如果hash冲突了，且槽里面第一个元素的hash不为-1，需要对槽里面首元素进行加锁，然后将元素加进去
        else {
            V oldVal = null;
            // 对首元素加锁，相比于jdk7来说，又有了进一步的性能提升
            synchronized (f) {
                // 这儿需要再次判断槽里面的首元素是否为f。因为在添加元素的时候有可能同时又有其他线程在移除元素，这种情况需要自旋来重新插入
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 槽里面的首元素是f，且首元素的hash是非负数，说明槽的数据结构为链表，依次遍历插入即可
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 槽里的首元素是f，且首元素是TreeBin结构，说明槽的数据结构为红黑树，则需要将元素插入到红黑树。红黑树的插入可能会进行左右旋等操作
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // binCount表示槽里面元素的个数，如果个数>=8，则需要将链表结构转换为红黑树结构，以此来提高性能
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 4. 通过`addCount()`方法进行扩容
    addCount(1L, binCount);
    return null;
}
```

我们来分析一下这个方法，该方法做了下面这些事情

1. 如果key或者value为null，抛出空指针异常，这点和`HashMap`有所不同
2. 根据key的hashCode()，重新生成hash
3. 这儿是空循环，也就是自旋操作
    1. 如果`table`为null或长度为0，需要对`table`进行初始化。从上一节构造方法的分析来看，构造方法并不会初始化`table`。初始化`table`延迟到`putVal()`方法
    2. 如果hash没有冲突，则使用CAS插入
    3. 如果hash冲突了，且槽里面第一个元素的hash为MOVED(-1)。则说明槽里面的元素为`ForwardingNode`对象，有其他线程正在扩容。当前线程通过`helpTransfer`进行辅助扩容，扩容完成之后再进行插入操作
    4. 如果hash冲突了，且槽里面第一个元素的hash不为-1，需要对槽里面首元素进行加锁，然后将元素加进去
4. 通过`addCount()`方法进行扩容

为了加深我们对于`putVal()`的理解，我们需要进一步分析该方法内部调用的几个关键方法

1. spread()
2. initTable()
3. tabAt()
4. casTabAt()
5. helpTransfer()
6. addCount()

## 1. spread()

```java
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

这个方法将hash的高位扩展到低位，同时强制最高位为0（可能是为了让hash值保持为非负数吧）。该方法做了下面几件事情

1. 我们将原int叫做A
2. 这儿将原int右移16位得到的int叫做B，则B的高16位则全为0
3. A和B异或，得到C，C的高位和A的高位相同
4. C再和`HASH_BITS`按位与，得到结果D，D的符号位为0

## 2. initTable()

```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    // table为null或者table长度为0，才进行初始化
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl为负数，说明有其他线程正在初始化table，当前线程让出cpu时间片，等待其他线程初始化完成
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // 通过CAS设置sizeCtl为-1成功之后，则说明当前线程抢占到了锁，需要进行真正的初始化操作
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 这儿还需要对table进行二次校验。
                // 因为CAS设置成功，有可能是由于table已经初始化完成了。
                // 如果是这种情况，当前线程不需要再次初始化
                if ((tab = table) == null || tab.length == 0) {
                    // 当前线程通过`new Node[]`的方式进行`table`初始化
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 这儿的`n - (n >>> 2);`写得非常牛逼。
                    // 右移1位，大小折半，右移2位，大小折1/4。
                    // 这儿原始int减去1/4，则为3/4，即：0.75*原int，
                    // 正好是HashMap里面默认的负载因子
                    // 至于为什么要用位移和减法，而不直接使用除法，楼主认为是基于性能的考虑，除法的性能比加减法和位移要低得多。
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc; // 扩容保护
            }
            break;
        }
    }
    return tab;
}
```

该方法初始化`table`，`table`的size记录在`sizeCtl`变量中。该方法采用`自旋`+`CAS`的方式实现了乐观锁，相比于悲观锁，带来了非常大的性能提升。

## 3. tabAt()

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```
该方法目的是取tab数组中，下标为i的元素。那么为啥要这样写呢？ 在理解之前，我们先来看看取数组元素的方法。

首先，我们知道数组元素在内存里面是连续存放的；
其次，数组元素存放的是引用地址，而非具体的值；
最后，数组取元素的方式可以通过数组地址+数组元素偏移量的方式来获取。

`((long)i << ASHIFT) + ABASE`这个表达式中，`(long)i << ASHIFT`表示偏移量，`ABASE`表示数组地址。

另外为啥又要使用`getObjectVolatile()`呢，是因为在并发的情况下，我们需要保证数组元素的内存可见性。

## 4. casTabAt()

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

关键的地方和`tabAt()`相同，不同的在于其功能。该方法用于对数组指定下标的元素进行CAS赋值。

## 5. 扩容方法
`helpTransfer()`和`addCount()`涉及到扩容相关的知识点，本章暂且不表，后续会单独开一节来详细说明。

## 总结

至此我们分析了`putVal()`整体的结构。不过对于`扩容`和`辅助扩容`相关知识点，我们并没有分析，而是一笔带过。原因在于扩容的知识相当得复杂，即使单开一节来分析都稍显不够。为了保证本节的简单些和清晰性，我们留待后续来说明。

即使只是分析了`putVal()`的皮毛，我们也能从中看到Doug Lea对于细节和性能处理方式的周全思虑，楼主阅读之后可谓是受益匪浅~

## 参考链接

+ https://www.jianshu.com/p/39b747c99d32
+ https://blog.csdn.net/fjse51/article/details/55260493
+ https://www.jianshu.com/p/2c1be41f6e59
+ https://www.jianshu.com/p/7094ddf42dd9
+ https://blog.csdn.net/tp7309/article/details/76532366