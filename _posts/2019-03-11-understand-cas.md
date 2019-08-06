---
layout: post
title:  CAS的实现原理
date:   2019-03-11 23:30:00 +0800
categories: Java并发和多线程
tag: java
---

* content
{:toc}

## 前言

有的时候，为了保证业务逻辑的原子性，我们需要用到锁机制。如下所示：

```java
public class Counter {
    private int counter;

    public synchronized void add(int number) {
        counter = counter + number;
    }
}
```

通过对这段代码的分析，我们发现：

1. 如果不加锁的话，多线程修改变量`counter`将会出现线程安全问题；
2. 如果加锁的话，锁的开销（如：获取锁、阻塞等待、释放锁等）将会非常高。

那么，有没有不加锁、同时又保证线程安全的方法呢？答案是：有的！

我们仔细分析一下，`add()`方法的逻辑其实可以拆分为3步：

1. 读取number
2. 将number和counter相加
3. 将第2步的结果赋值给counter

如果不使用锁，上述操作并非一个原子操作。这儿我们假设初始number为1，counter也为1，多个线程可能并发地读取到number，然后都将其和counter相加，最后都将相加的结果赋值给counter，counter就变成了2。有一个线程的执行逻辑被莫名奇妙地吞掉了！

问题出在哪儿呢？其实是在第3步。如果有一种机制，可以判断当前变量的值已经被其他线程修改了，那么只需要让线程进行简单的重试就可以了。我们将上述代码进行重构，如下所示：

```java
public class Counter {
    private int counter;

    public void add(int number) {
        // 第一步
        int tmp = counter;

        // 第二步
        int targetValue = tmp + number;

        // 第三步
        while (!compareAndSwap(counter, targetValue)) {
            // continue
        }
    }

    private boolean compareAndSwap(int sourceValue, int targetValue) {
        if (this.counter == sourceValue) {
            this.counter = targetValue;
            return true;
        }
        return false;
    }
}
```

在上述代码中，我们先将counter做了备份，然后对备份和number进行求和，最后在`compareAndSwap()`方法中对原始值进行比较，如果相同则更新并返回true，如果不同则返回false，等待下一次的循环。

最后，还有一个问题，`compareAndSwap()`方法并不具备原子性，也没有使用锁，怎么保证线程安全呢？其实这儿是保证不了线程安全的，楼主只是编写了CAS的伪代码，然后借此引出CAS的概念而已。

## 1. CAS的概念

CAS，英文为Compare And Swap，中文意思为：比较并交换，它是一种无锁原子操作算法。大致过程是这样的：CAS包含三个参数，V、E和N。V表示待更新的变量，E是预期值，N表示新值。仅仅当V的值为E，才将V的值变更为N，否则不做任何处理。V的值不为E，必然是因为其他线程已经对V进行了修改。

CAS的返回结果为一个boolean值，true表示更新成功，false表示更新失败。

用CAS会使得代码变得更复杂一些，但是因为其天生的乐观特性（总是认为绝大多数情况能更新成功），所以天生地对线程竞争具有免疫性。同时，其优越的性能也比锁要高效很多。

那么，CAS是如何实现的呢？它是如何将两个操作（比较和交换）变成一个操作的呢？这就要从CAS的底层实现说起了。

## 2. CAS的底层实现

CAS的底层其实是通过一条硬件指令来实现的。理论上我们可以使用同步锁的方式来达到相同的效果，但是这并不是我们想要的效果（性能高、线程安全）。因此我们只能从硬件指令上着手考虑。通过硬件指令，我们可以让看起来多步的操作只需要一条指令就可以完成。

这样的指令有下面几种：

1. 测试并设置（Tetst-and-Set）
2. 获取并增加（Fetch-and-Increment）
3. 交换（Swap）
4. 比较并交换（Compare-and-Swap）
5. 加载链接/条件存储（Load-Linked/Store-Conditional）

其中前3种在20世纪已经实现，且大部分处理器都支持，而后2种是在21世纪现代处理器中新增的。同时，这两条指令的目的和功能是类似的，在IA64，x86 指令集中由`cmpxchg`指令完成CAS功能，在sparc-TSO也有`casa`指令实现，而在ARM和PowerPC架构下，则需要使用一对`ldrex/strex`指令来完成 LL/SC 的功能。

CPU实现原子指令有两种方式：

1. 锁总线
2. 锁缓存

所谓锁总线，其实是线程在总线上输出`LOCK#信号`，其他线程在检测到此信号时将被阻塞住，保证同一时间只能由一个线程独享内存，这种方式由于其开销太大，所以才会有锁缓存的出现。

所谓锁缓存，是在内存区域被缓存在缓存行时，并且在Lock期间被锁住，如果其他线程执行锁操作回写到内存时，处理器不在总线上输出`LOCK#信号`，而是修改内部的内存地址，并允许它的缓存一致性算法来保证原子性，因为缓存一致性机制会阻止同时修改两个以上处理器缓存的内存区域数据（这里和`volatile`的可见性原理相同），当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。

这儿需要注意两点：

1. 如果操作跨多个缓存行时，还是会锁总线
2. 有一些处理器是不支持锁缓存的

## 3. CAS在Java中的实现

JDK1.5及以上版本，在`java.util.concurrent.atomic`中提供了很多原子类，其所有方法都具有原子性。

![juc.atomic包](https://upload-images.jianshu.io/upload_images/845143-9cb89b3cd2e1ace9.png)

那么，其用法又是怎样的呢？

```java
public class Counter {
    private AtomicInteger counter = new AtomicInteger();

    public void add() {
        counter.incrementAndGet();
    }

    public static void main(String[] args) throws Exception {
        Counter counter = new Counter();

        Thread[] threads = new Thread[1000];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(counter::add);
        }

        for (Thread thread : threads) {
            thread.start();
            thread.join();
        }

        System.out.println(counter.counter.get());
    }
}
```

在这个例子中，我们开启了1000个线程，对共享变量进行加一操作。如果结果正常的话，那么输出结果一定是1000。事实上，运行的结果和我们推断的一样。

这儿有一个关键的方法`AtomicInteger.incrementAndGet()`，我们跟进源码去看看。

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();

public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

该方法调用的是`Unsafe`实例的`getAndAddInt()`方法。那么我们先来看看Unsafe实例的构造方法，Unsafe是一个单例，且不能被JDK之外的代码初始化，这也是其叫做`un safe`的原因啦。

```java
@CallerSensitive
public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
        throw new SecurityException("Unsafe");
    } else {
        return theUnsafe;
    }
}
```

再来看看`getAndAddInt()`方法，这个方法最核心的还是`while`语句里面的`compareAndSwapInt()`方法。之所以要用`while`，根本原因在于失败时需要通过自旋进行重试。

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

## 4. CAS有什么问题

CAS非常厉害，不光性能高，而且线程安全，那么它又是不是万能的呢？

其实它也是有缺陷的，最典型的就是ABA问题。

举个例子，有一个变量var，其初始值为a，有一个线程先将其变更为b，然后又变更为a，这种情况CAS是感知不到的。事实上，这个变量是确实是变更过了的。这种情况对于基本类型没有任何问题，但是对于引用类型，我们往往会比较关心中间的变更情况。

怎么解决ABA问题呢？最简单的方式就是通过时间戳。变量每变更一次，时间戳就递增一次，这样就能有效解决ABA问题，`AtomicReferenceFieldUpdater`其实就是这种理论的实践。虽然ABA问题解决了，但是因为需要维护时间戳，其性能也受到了一定影响，在实际使用的时候需要综合评估和权衡。

## 总结

首先，我们通过一个普通的例子引出CAS的概念。

然后通过`是什么`、`怎么用`和`有什么问题`的方式对CAS进行了全方位的说明。

在JDK里，CAS扮演着非常重要的角色。从CPU都单独为其设计一条指令来进行支持，就能看出其重要性了。我们可以这样说：没有CAS，就没有JDK的并发编程库！

## 参考文档

+ https://www.cnblogs.com/javalyy/p/8882172.html
+ https://www.jianshu.com/p/a3b2b0c94ebf