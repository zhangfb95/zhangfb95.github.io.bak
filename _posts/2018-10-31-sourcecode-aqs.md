---
layout: post
title:  AbstractQueuedSynchronizer(AQS)源码解析
date:   2018-10-31 09:53:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

jdk1.6之前synchronized性能一直较为低下；jdk1.6之后，java开发团队对synchronized进行了大量的优化，其性能也提升了很多。但是与`Lock`系列锁相比，缺少了获取/释放锁的可操作性，可中断和超时处理等特性。另外，它只支持独占锁，不支持共享锁。

说到`Lock`，不得不说其依赖的`AQS`，它是非常非常重要的一个锁抽象。

那么AQS又是什么呢？
`AQS`，英文名为`AbstractQueuedSynchronizer`，也叫队列同步器，是构建同步锁或其他并发框架的基础，Doug Lea希望它能够成为实现大部分同步需求的基础和抽象。`AQS`解决了锁实现的大部分细节，如同步状态的获取、FIFO队列。

AQS的使用主要是通过继承的方式，然后继承者被同步器所组合依赖。

## 主要方法

```java
int getState() // 返回同步状态的当前值
void setState(int newState) // 设置同步状态
boolean compareAndSetState(int expect, int update) // 使用CAS设置当前同步状态
void acquire(int arg) // 以独占的方式获取同步状态
void acquireInterruptibly(int arg) throws InterruptedException // 以独占的方式获取同步状态（可中断）
void acquireShared(int arg) // 以共享的方式获取同步状态
void acquireSharedInterruptibly(int arg) throws InterruptedException // 以共享的方式获取同步状态（可中断）
int tryAcquireShared(int arg) // 以共享的方式获取同步状态（非阻塞，需要继承者实现）
boolean tryAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException // 以共享的方式获取同步状态（带超时）
boolean isHeldExclusively() // 当前线程是否持有同步状态
boolean release(int arg) // 释放同步状态（独占式）
boolean releaseShared(int arg) // 释放同步状态（共享式）
boolean tryAcquire(int arg) // 以独占的方式获取同步状态（非阻塞，需要继承者实现）
boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException // 以独占的方式获取同步状态（带超时）
boolean tryRelease(int arg) // 以独占的方式释放同步状态（非阻塞，需要继承者实现）
boolean tryReleaseShared(int arg)  // 以共享的方式释放同步状态（非阻塞，需要继承者实现）
```

## CLH队列

CLH是一个FIFO(先进先出的队列)，AQS依赖其来完成同步状态的管理。当前线程如果获取同步状态失败，AQS会将当前线程及等待状态等组装成一个Node，并加入到CLH队列，同时会阻塞当前线程，当同步状态释放时，会将首节点唤醒，并使其重新尝试获取同步状态。

我们来看一下CLH中的一个Node的信息有哪些

```java
static final class Node {
    // 标识节点是否共享mode
    static final Node SHARED = new Node();
    // 标识节点是否独占mode
    static final Node EXCLUSIVE = null;

    /** 取消状态，线程被中断 */
    static final int CANCELLED =  1;
    /** 通知状态，继任节点需要被唤醒 */
    static final int SIGNAL    = -1;
    /** 带条件登台状态 */
    static final int CONDITION = -2;
    /** 下一次获取同步状态（共享）应该无条件传播 */
    static final int PROPAGATE = -3;

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     */
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * Returns previous node, or throws NullPointerException if null.
     * Use when predecessor cannot be null.  The null check could
     * be elided, but is present to help the VM.
     *
     * @return the predecessor of this node
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

### CLH同步队列结构图

![image.png](https://upload-images.jianshu.io/upload_images/845143-99a7ee88395e6330.png)

### 入队列

我们看看AQS中入队列的代码

```java
private Node addWaiter(Node mode) {
    // 新建一个节点，mode可以是独占式，也可以是共享式
    Node node = new Node(Thread.currentThread(), mode);

    // 下面这段代码会尝试添加一个尾节点
    // 1. 将旧的tail节点赋值为pred
    // 2. 旧的tail尾节点不为null，则new节点的prev指向旧的tail节点
    // 3. 使用CAS设置new节点为tail节点
    // 4. 如果设置新节点为tail节点成功之后，则将旧的tail节点的next指向new节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 在tail节点为null或者将新节点设置为tail节点失败时，通过自旋的方式再次设置tail节点。
    // 这儿需要特别注意，为什么设置tail节点会有失败的可能，因为addWaiter方法可能会同时被多个线程调用
    enq(node);
    return node;
}
```

在初次尝试如队列失败后，会调用`enq`方法，我们进一步分析此方法的代码

```java
private Node enq(final Node node) {
    // for死循环，其实就是自旋操作
    for (;;) {
        Node t = tail;
        // tail == null表示是第一次添加节点
        if (t == null) { // Must initialize
            // 1. CAS设置head节点可能会失败。如果失败，表示其他线程已经设置head节点成功了
            // 2. 这儿设置了一个空的head节点，后面我们会分析一下为啥需要设置一个空的head节点
            // 3. 在设置head节点成功之后，仍然需要再次执行下面的else逻辑进行尾节点的设置
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 这儿其实和AddWaiter的逻辑是一样的，通过CAS设置尾节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

如队列的整个过程图如下：
![image.png](https://upload-images.jianshu.io/upload_images/845143-c0972835c149ddc1.png)

## 同步状态获取与释放

AQS内部使用了`模板方法模式`来设计，子类通过继承的方式实现`抽象方法`（不完全是抽象方法，其实是一个普通的方法，父类默认实现为throwException的方式）。主要的抽象方法有下面几个

1. 独占式获取同步状态
2. 独占式释放同步状态
3. 共享式获取同步状态
4. 共享式释放同步状态
5. 当前线程是否获取到了同步状态

### 独占式获取

什么叫独占式呢？其实就是同一时刻至多只能有一个线程获取到同步状态。

我们看看独占式获取同步状态的代码，`acquire(int arg) `：

```java
public final void acquire(int arg) {
    // 1. 如果没有获取到同步状态，则新创建一个独占式节点，并将其添加到同步队列。
    // 2. 否则认为已经获取到同步状态成功了，当前线程就可以执行自身的逻辑了。
    // 3. 该方法不会响应中断，但是如果在执行的过程中有过中断操作，则会设置当前线程的中断状态为true
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

我们前面已经分析过了`addWaiter`方法，下面我们分析一下`acquireQueued`方法。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false; // 是否有过中断
        // 这儿有自旋操作
        for (;;) {
            // 如果当前节点的前任是head节点，则尝试获取同步状态，若获取成功，则执行下面操作：
            // 1. 设置当前节点为head节点
            // 2. 前任节点的next指向null，那么前任节点就没有依赖存活的对象了，可以更快得到GC
            // 3. 返回是否有过中断的标记
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 当前任节点不是head节点，或者获取同步状态失败。则会通过`park操作`让`当前线程等待`，让出CPU。我们后面会分析这两个方法，这儿暂时跳过
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 如果失败过，则会对当前节点进行同步状态的取消获取操作
        if (failed)
            cancelAcquire(node);
    }
}
```

acquire方法的流程图如下

![image.png](https://upload-images.jianshu.io/upload_images/845143-75bbb2b9e77d527a.png)

我们接着看一下acquireInterruptibly方法，该方法在获取同步状态的时候，可以响应中断

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {
    // 线程中断了，直接抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 如果尝试获取同步状态失败，则执行doAcquireInterruptibly方法
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

接着，我们看看doAcquireInterruptibly方法，该方法和`acquireQueued`基本一样，唯一的两点差别如下：
1. 方法声明上添加了`throws InterruptedException`
2. 在park的时候，如果检查到中断，则抛出异常`throw new InterruptedException()`

```java
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

接着我们看看超时获取同步状态的方法，`tryAcquireNanos `

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    // 有过中断，直接抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取同步状态失败时，则执行方法doAcquireNanos
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

我们接着分析一下`doAcquireNanos`

```java
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    // 如果超时纳秒数不是正数，则直接返回未获取到同步状态
    if (nanosTimeout <= 0L)
        return false;
    // 当前时间+超时时间=获取同步状态的截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    // 创建一个独占式的节点
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        // 自旋操作
        for (;;) {
            // 这儿和acquireQueued中相应的代码一模一样
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 这儿再次判断是否超过截止时间
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            // 和acquireQueued不同的地方有两点
            // 1. park方法为带毫秒的阻塞（超过指定时间则不再wait，继续执行）
            // 2. 如果离截止时间不超过1微妙，为了性能考虑，不再parkNanos了，而是for死循环。即使使用parkNanos也不准确。spinForTimeoutThreshold为1000（1000纳秒==1微秒）
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            // 同中断的方法一样，超时也会响应中断
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

整体流程图如下：
![image.png](https://upload-images.jianshu.io/upload_images/845143-be9128b6302bd990.png)

独占式释放同步状态方法只有一个——`release`

```java
public final boolean release(int arg) {
    // 尝试释放同步状态，若没成功，则返回false
    if (tryRelease(arg)) {
        Node h = head;
        // head节点不为null，且waitStatus不是初始状态，则unpark继任节点
        if (h != null && h.waitStatus != 0)
           // 这儿会唤醒继任节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

### 共享式

我们接着分析共享式获取同步状态方法

```java
public final void acquireShared(int arg) {
    // tryAcquireShared返回值>=0，表示获取成功了，否则需要自旋地获取
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 自旋操作
        for (;;) {
            final Node p = node.predecessor();
            // 前任节点为head节点，尝试获取共享的同步状态
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 前任节点不是head节点，需要park操作
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

## 阻塞和唤醒线程

不管是共享式，还是独占式，都会调用下面的逻辑，我们需要分析其中关键的两个方法

```java
if (shouldParkAfterFailedAcquire(p, node) &&
    parkAndCheckInterrupt())
    interrupted = true;
```

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 前驱节点为SIGNAL，表示需要等待，直接返回true
    if (ws == Node.SIGNAL)
        return true;
    // 前驱节点的状态>0，表示为CANCELED，表示为已中断的线程，需要从同步队列中移除
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    // 前驱节点状态不为SINGNAL和CANCELED（则为CONDITION和PROPAGATE），则需要将前驱节点的状态设置为SIGNAL
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

接着我们分析一下方法`parkAndCheckInterrupt`，很简单，直接通过`park`方式使当前线程变为WAITING状态，然后返回线程的中断标记

```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

线程释放同步状态后，需要唤醒继任节点，我们分析一下释放同步状态的逻辑

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
           // 唤醒继任节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    // 节点状态为负数，则设置为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    // 后继节点为null或者其状态 > 0 (超时或者被中断了)
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从尾节点往前找未被中断的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## 总结

至此，我们分析完了整个AQS的几个关键的方法，包括入队列、获取同步状态（独占式和共享式）、释放同步状态（独占式和共享式）。
