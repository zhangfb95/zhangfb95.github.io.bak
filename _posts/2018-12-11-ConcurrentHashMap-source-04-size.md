---
layout: post
title:  ConcurrentHashMap源码分析(04)-size()方法
date:   2018-12-11 19:34:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

`HashMap.size()`的代码非常简单，直接返回成员变量size即可。可是在`ConcurrentHashMap`里面，是否也是这样呢？答案是否定的，因为我们需要考虑并发的情况。若我们继续沿用`HashMap.size()`的思想，并发的效率可以想象得到，将变得非常得低效。

那么`ConcurrentHashMap.size()`又是怎么实现的呢？我们这一小节将进行分析~

## size()

这个方法内部调用`sumCount()`方法，但是`sumCount()`返回的是long类型的，而`size()`返回的是int类型的，需要进行一定的适配。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```

## sumCount()

这个方法用于返回节点的个数，有用到两个关键的成员变量

1. `baseCount` -- 基本的计数器，当没有竞争的时候，使用这个值
2. `counterCells` -- 数组结构，当有竞争的时候使用它来进行计算

`sumCount()`实现的逻辑如下：
1. 首先判断`counterCells`是否为null，为null说明没有竞争，直接返回`baseCount`。
2. 如果`counterCells`不为null，说明有竞争，通过遍历`counterCells`数组，将所有元素的`value`进行求和。

```java
/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 * 基本的计数器，当没有竞争的时候，使用这个值
 */
private transient volatile long baseCount;

/**
 * Table of counter cells. When non-null, size is a power of 2.
 */
private transient volatile CounterCell[] counterCells;

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

既然竞争的时候需要用到`counterCells`，那么它又是什么时候生成的呢？在前面的分析我们看到了`counterCells`的影子，那就是在`addCount()`方法中，那么接下来我们就再来分析一下关于`counterCells`的代码。

## addCount()方法再分析

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 1. 初始的时候，counterCells一定为null，第一个条件此时将不会被满足
    // 2. 在第一个条件不满足的时候，需要判断第二个条件。
    // 3. 第二个条件在存在竞争的情况下将返回false，也就是需要进入到if里面
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 这儿有三个或条件，只需要满足其一，就可以进入到if
        // 1. counterCells为null
        // 2. counterCells为空数组
        // 3. counterCells指定位置元素为null
        // 4. CAS设置counterCell.value失败的时候
        // 这几种情况将进入到`fullAddCount()`进行自旋操作
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 省略掉扩容的代码...
}
```

## fullAddCount()方法

这个方法非常得复杂，代码也很长，如果深入进去将陷入到细节中去。在此我们只需要知道这个方法的用途，那就是通过自旋的方式新增或更新`counterCells`，使用`counterCells`来统计节点个数将不会得到准确的结果。

## CounterCell类

既然竞争的情况下是围绕`counterCells`来展开的，那么我们需要看看这个成员变量所属的类型，`CounterCell`。

```java
/**
 * A padded cell for distributing counts.  Adapted from LongAdder
 * and Striped64.  See their internal docs for explanation.
 * 改编自LongAdder和Striped64
 */
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

这个类有几个需要注意的地方

1. `value`使用`volatile`修饰，可保证其可见性
2. `CounterCell`使用`@sun.misc.Contended`修饰

`@sun.misc.Contended`这个注释需要明确说明一下。如果类上面使用该注释，标识需要防止`伪共享`。

那么什么又是`伪共享`呢？借鉴一下网上大神的阐述。

> 【说明】：伪共享(false sharing)。先引用对伪共享的解释
缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，
一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，
如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

JDK 8版本之前没有这个注解，Doug Lea使用拼接来解决这个问题，把缓存行加满，让缓存之间的修改互不影响。

## mappingCount()

我们额外分析一下这个方法，这是jdk8新加的一个方法，内部同样调用了`sumCount()`，和`size()`有同样的效果，但是它的返回值却是long，可以避免精度的损失。同时这个方法也是被推荐用来替代`size()`。

```java
/**
 * Returns the number of mappings. This method should be used
 * instead of {@link #size} because a ConcurrentHashMap may
 * contain more mappings than can be represented as an int. The
 * value returned is an estimate; the actual count may differ if
 * there are concurrent insertions or removals.
 *
 * @return the number of mappings
 * @since 1.8
 */
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}
```

## 总结

期初，我们还以为`size()`会很简单，但是一通分析下来却发现它并不是想象得那么简单，尤其是`addCount()`内部调用的`fullAddCount()`方法，其复杂度不下于[扩容方法](https://www.jianshu.com/p/f7d0b8c23b3d)。

为了省事，本节并没有对这个方法进行分析，后续有时间再来分析。