---
layout: post
title:  ConcurrentHashMap源码分析(03)-扩容方法
date:   2018-12-11 17:49:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## addCount()

在分析到`putVal()`最后的时候，有调用`addCount()`方法，这个方法又是做什么用的呢？从字面意思来看是增加元素的数量，我们来看一下。

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 这儿是统计ConcurrentHashMap里面节点个数的地方
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
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
    // check就是binCount，该值在`putVal()`里面一定是>=0的，所以这个条件一定会为true
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 这儿是自旋，需同时满足下面的条件
        // 1. 第一个条件是元素个数超过数组的容量
        // 2. 第二个条件是`table`不为null
        // 3. 第三个条件是`table`的长度不能超过最大容量
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // 该判断表示已经有线程在进行扩容操作了
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 该判断表示当前线程是否加入辅助扩容。若失败，则进行自旋操作
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 该判断表示当前线程是否是扩容操作的第一个线程
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

## resizeStamp()

在`addCount()`我们有看到`resizeStamp()`这个方法，我们进一步来分析一下。

```java
/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static int RESIZE_STAMP_BITS = 16;

/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

可以看到这个方法的返回一个与table容量n大小有关的扩容标记，而n是2的幂次，可得知返回值rs对于不同容量大小的table值必然不相同。经过rs << RESIZE_STAMP_SHIFT变为负数后再赋值给sizeCtl，那么在扩容时sizeCtl值的意义便如下表所示：

|高RESIZE_STAMP_BITS位|	低RESIZE_STAMP_SHIFT位|
|---|---|
|扩容标记|	并行扩容线程数|

> 【说明】：`Integer.numberOfLeadingZeros(n)`用于获取当前int从高位到低位第一个1前面0的个数。

## transfer()

接下来进入我们分析的主要地方，`transfer()`，该方法用于扩容。这个方法非常得核心，也非常得复杂，楼主分析下来只对基本的逻辑进行了了解，更深层次的`为什么`却没有深究。

```java
/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // stride表示每个线程处理的槽个数。同时基于性能考虑，stride不能比16小。
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating，这个判断用以初始化操作
        try {
        	// 数组扩容为原来的两倍
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
        	// 数组扩容的时候有可能出现OOME，这时需要将sizeCtl设置为Integer.MAX_VALUE，用以表示这个异常
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // 扩容之后的数组赋值给nextTable
        nextTable = nextTab;
        // transferIndex用以标识扩容总的进度，默认设置为原数组长度。
        // 因为扩容时候的元素迁移是从最末的元素开始的，所以迁移的时候下标是递减的，从下面的`--i`就能看出来了
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 扩容时准备的特殊节点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 用于表示table里面单个元素是否迁移完毕，初始为true是为了进入while循环。
    // 1. true表示还未迁移完
    // 2. false表示已经迁移完
    boolean advance = true;
    // 用于表示table里面的所有元素是否迁移完毕
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 这儿又是自旋，bound表示槽的边界
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            // 倒序迁移老表元素的下标已达到槽的边界，或者整个table已经迁移完毕，说明迁移完成了
            if (--i >= bound || finishing)
                advance = false;
            // 扩容总进度<=0，说明迁移完成了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 1. transferIndex减去每个线程处理的槽个数，将这个值设置为即将迁移的边界
            // 2. 设置待迁移元素的下标
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 1. 个人感觉只有第一个条件可能会满足，后面两个条件应该不会被满足
        // 2. 至于为什么不满足，楼主目前没有发现触发的位置，若有人知晓，可留言告诉我
        // 3. i < 0表示所有元素都迁移完了
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 数组元素所在的位置没有值，则直接设置ForwordNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 数组元素所在位置节点为ForwordNode，表示已经被迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        // 其他情况，需要对元素首节点进行加锁，然后将该元素所在槽里面的每个节点都进行迁移
        else {
            synchronized (f) {
            	// 这儿多判断一次，是否为了防止可能出现的remove()操作
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) { // hash值是非负数，表示节点类型为链表
                    	// runBit有两个作用
                    	// 1. 用于生成lastRun，lastRun及之后节点扩容之后的hash值是一样的
                    	// 2. 通过runBit是否为0，判断lastRun应该放置在高位还是低位。若是0，放在低位；若是1，放在高位
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 再次遍历所有节点，低位节点放在原位置，高位节点放在原位置+原数组长度
                        // 这儿的遍历为了提高性能，只遍历到lastRun为止
                        // 从上面的逻辑我们知道lastRun及后面的元素，其runBit相同
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) { // hash值是负数，且节点类型为TreeBin，说明是红黑树
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

## helpTransfer()

`putVal()`方法里面有一个地方用于辅助扩容，我们再来回顾一下代码。

```java
// ...
else if ((fh = f.hash) == MOVED)
    tab = helpTransfer(tab, f);
else {
// ...
```

我们进一步分析这个方法，以加深我们的理解。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

这个方法和`addCount()`扩容逻辑非常相似，最终都会调用`tansfer()`。

## 总结

1. 扩容由单一线程进行创建并设置nextTable，扩容为原来的两倍
2. 对`ConcurrentHashMap`进行新增或删除的时候，如果判断节点类型为`ForwardingNode`，则说明有线程正在扩容，则当前线程协助一起进行扩容，扩容完成之后再进行新增或删除操作
3. 扩容的时候，元素节点可能需要进行迁移（挪位置），迁移的顺序是倒序的
4. 扩容的时候，为了避免线程切换和资源竞争的性能开销，每个线程最少被分配为16个槽进行迁移
5. 每个槽的迁移是单线程的，但是槽的范围却是多线程的，在没有迁移完成所有槽之前每个线程需要重复获取迁移槽范围，直至所有槽迁移完成
6. 迁移过程中sizeCtl用于记录参与扩容线程的数量，全部迁移完成后sizeCtl更新为新table的扩容阈值

## 参考链接

+ https://blog.csdn.net/tp7309/article/details/76532366