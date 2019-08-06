---
layout: post
title:  Java定时调度机制 - Timer
date:   2019-01-18 00:10:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 简介

在实现定时调度功能的时候，我们往往会借助于第三方类库来完成，比如：`quartz`、`Spring Schedule`等等。JDK从1.3版本开始，就提供了基于`Timer`的定时调度功能。在`Timer`中，任务的执行是串行的。这种特性在保证了线程安全的情况下，往往带来了一些严重的副作用，比如任务间相互影响、任务执行效率低下等问题。为了解决`Timer`的这些问题，JDK从1.5版本开始，提供了基于`ScheduledExecutorService`的定时调度功能。

本节我们主要分析`Timer`的功能。对于`ScheduledExecutorService`的功能，我们将新开一篇文章来讲解。

## 如何使用

`Timer`需要和`TimerTask`配合使用，才能完成调度功能。`Timer`表示调度器，`TimerTask`表示调度器执行的任务。任务的调度分为两种：一次性调度和循环调度。下面，我们通过一些例子来了解他们是如何使用的。

### 1. 一次性调度

```java
public static void main(String[] args) {
    Timer timer = new Timer();
    TimerTask task = new TimerTask() {
        @Override public void run() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss");
            System.out.println(format.format(scheduledExecutionTime()) + ", called");
        }
    };
    // 延迟一秒，打印一次
    // 打印结果如下：10:58:24, called
    timer.schedule(task, 1000);
}
```

### 2. 循环调度 - schedule()

```java
public static void main(String[] args) {
    Timer timer = new Timer();
    TimerTask task = new TimerTask() {
        @Override public void run() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss");
            System.out.println(format.format(scheduledExecutionTime()) + ", called");
        }
    };
    // 固定时间的调度方式，延迟一秒，之后每隔一秒打印一次
    // 打印结果如下：
    // 11:03:55, called
    // 11:03:56, called
    // 11:03:57, called
    // 11:03:58, called
    // 11:03:59, called
    // ...
    timer.schedule(task, 1000, 1000);
}
```

### 3. 循环调度 - scheduleAtFixedRate()

```java
public static void main(String[] args) {
    Timer timer = new Timer();
    TimerTask task = new TimerTask() {
        @Override public void run() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss");
            System.out.println(format.format(scheduledExecutionTime()) + ", called");
        }
    };
    // 固定速率的调度方式，延迟一秒，之后每隔一秒打印一次
    // 打印结果如下：
    // 11:08:43, called
    // 11:08:44, called
    // 11:08:45, called
    // 11:08:46, called
    // 11:08:47, called
    // ...
    timer.scheduleAtFixedRate(task, 1000, 1000);
}
```

### 4. schedule()和scheduleAtFixedRate()的区别

从2和3的结果来看，他们达到的效果似乎是一样的。既然效果一样，JDK为啥要实现为两个方法呢？他们应该有不一样的地方！

在正常的情况下，他们的效果是一模一样的。而在异常的情况下 - 任务执行的时间比间隔的时间更长，他们是效果是不一样的。

1. `schedule()`方法，任务的`下一次执行时间`是相对于`上一次实际执行完成的时间点` ，因此执行时间会不断延后
2. `scheduleAtFixedRate()`方法，任务的`下一次执行时间`是相对于`上一次开始执行的时间点` ，因此执行时间不会延后
3. 由于`Timer`内部是通过单线程方式实现的，所以这两种方式都不存在线程安全的问题

我们先来看看`schedule()`的异常效果：

```java
public static void main(String[] args) {
    Timer timer = new Timer();
    TimerTask task = new TimerTask() {
        @Override public void run() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(format.format(scheduledExecutionTime()) + ", called");
        }
    };

    timer.schedule(task, 1000, 2000);
    // 执行结果如下：
    // 11:18:56, called
    // 11:18:59, called
    // 11:19:02, called
    // 11:19:05, called
    // 11:19:08, called
    // 11:19:11, called
}
```

接下来我们看看`scheduleAtFixedRate()`的异常效果：

```java
public static void main(String[] args) {
    Timer timer = new Timer();
    TimerTask task = new TimerTask() {
        @Override public void run() {
            SimpleDateFormat format = new SimpleDateFormat("HH:mm:ss");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(format.format(scheduledExecutionTime()) + ", called");
        }
    };

    timer.scheduleAtFixedRate(task, 1000, 2000);
    // 执行结果如下：
    // 11:20:45, called
    // 11:20:47, called
    // 11:20:49, called
    // 11:20:51, called
    // 11:20:53, called
    // 11:20:55, called
}
```

楼主一直相信，实践是检验真理比较好的方式，上面的例子从侧面验证了我们最初的猜想。

但是，这儿引出了另外一个问题。既然`Timer`内部是单线程实现的，在执行间隔为2秒、任务实际执行为3秒的情况下，`scheduleAtFixedRate`是如何做到2秒输出一次的呢？

> 【特别注意】
这儿其实是一个障眼法。需要重点关注的是，打印方法输出的值是通过调用`scheduledExecutionTime()`来生成的，而这个方法并不一定是任务真实执行的时间，而是当前任务应该执行的时间。

## 源码阅读

楼主对于知识的理解是，除了知其然，还需要知其所以然。而阅读源码是打开`知其所以然`大门的一把强有力的钥匙。在JDK中，`Timer`主要由`TimerTask`、`TaskQueue`和`TimerThread`组成。

### 1. TimerTask

`TimerTask`表示任务调度器执行的任务，继承自`Runnable`，其内部维护着任务的状态，一共有4种状态

1. `VIRGIN`，英文名为处女，表示任务还未调度
2. `SCHEDULED`，已经调度，但还未执行
3. `EXECUTED`，对于执行一次的任务，表示已经执行；对于重复执行的任务，该状态无效
4. `CANCELLED`，任务被取消

`TimerTask`还有下面的成员变量

1. nextExecutionTime，下次执行的时间
2. period，任务执行的时间间隔。正数表示固定速率；负数表示固定时延；0表示只执行一次

分析完大致的功能之后，我们来看看其代码。

```java
/**
 * The state of this task, chosen from the constants below.
 */
int state = VIRGIN;

/**
 * This task has not yet been scheduled.
 */
static final int VIRGIN = 0;

/**
 * This task is scheduled for execution.  If it is a non-repeating task,
 * it has not yet been executed.
 */
static final int SCHEDULED   = 1;

/**
 * This non-repeating task has already executed (or is currently
 * executing) and has not been cancelled.
 */
static final int EXECUTED    = 2;

/**
 * This task has been cancelled (with a call to TimerTask.cancel).
 */
static final int CANCELLED   = 3;
```

`TimerTask`有两个操作方法

1. `cancel()` // 取消任务
2. `scheduledExecutionTime()` // 获取任务执行时间

`cancel()`比较简单，主要对当前任务加锁，然后变更状态为已取消。

```java
public boolean cancel() {
    synchronized(lock) {
        boolean result = (state == SCHEDULED);
        state = CANCELLED;
        return result;
    }
}
```

而在`scheduledExecutionTime()`中，任务执行时间是通过下一次执行时间减去间隔时间的方式计算出来的。

```java
public long scheduledExecutionTime() {
    synchronized(lock) {
        return (period < 0 ? nextExecutionTime + period
                           : nextExecutionTime - period);
    }
}
```

### 2. TaskQueue

`TaskQueue`是一个队列，在`Timer`中用于存放任务。其内部是使用【最小堆算法】来实现的，堆顶的任务将最先被执行。由于使用了【最小堆】，`TaskQueue`判断执行时间是否已到的效率极高。我们来看看其内部是怎么实现的。

```java
class TaskQueue {
    /**
     * Priority queue represented as a balanced binary heap: the two children
     * of queue[n] are queue[2*n] and queue[2*n+1].  The priority queue is
     * ordered on the nextExecutionTime field: The TimerTask with the lowest
     * nextExecutionTime is in queue[1] (assuming the queue is nonempty).  For
     * each node n in the heap, and each descendant of n, d,
     * n.nextExecutionTime <= d.nextExecutionTime.
     * 
     * 使用数组来存放任务
     */
    private TimerTask[] queue = new TimerTask[128];

    /**
     * The number of tasks in the priority queue.  (The tasks are stored in
     * queue[1] up to queue[size]).
     * 
     * 用于表示队列中任务的个数，需要注意的是，任务数并不等于数组长度
     */
    private int size = 0;

    /**
     * Returns the number of tasks currently on the queue.
     */
    int size() {
        return size;
    }

    /**
     * Adds a new task to the priority queue.
     * 
     * 往队列添加一个任务
     */
    void add(TimerTask task) {
        // Grow backing store if necessary
        // 在任务数超过数组长度，则通过数组拷贝的方式进行动态扩容
        if (size + 1 == queue.length)
            queue = Arrays.copyOf(queue, 2*queue.length);

        // 将当前任务项放入队列
        queue[++size] = task;
        // 向上调整，重新形成一个最小堆
        fixUp(size);
    }

    /**
     * Return the "head task" of the priority queue.  (The head task is an
     * task with the lowest nextExecutionTime.)
     * 
     * 队列的第一个元素就是最先执行的任务
     */
    TimerTask getMin() {
        return queue[1];
    }

    /**
     * Return the ith task in the priority queue, where i ranges from 1 (the
     * head task, which is returned by getMin) to the number of tasks on the
     * queue, inclusive.
     * 
     * 获取队列指定下标的元素
     */
    TimerTask get(int i) {
        return queue[i];
    }

    /**
     * Remove the head task from the priority queue.
     *
     * 移除堆顶元素，移除之后需要向下调整，使之重新形成最小堆
     */
    void removeMin() {
        queue[1] = queue[size];
        queue[size--] = null;  // Drop extra reference to prevent memory leak
        fixDown(1);
    }

    /**
     * Removes the ith element from queue without regard for maintaining
     * the heap invariant.  Recall that queue is one-based, so
     * 1 <= i <= size.
     *
     * 快速移除指定位置元素，不会重新调整堆
     */
    void quickRemove(int i) {
        assert i <= size;

        queue[i] = queue[size];
        queue[size--] = null;  // Drop extra ref to prevent memory leak
    }

    /**
     * Sets the nextExecutionTime associated with the head task to the
     * specified value, and adjusts priority queue accordingly.
     *
     * 重新调度，向下调整使之重新形成最小堆
     */
    void rescheduleMin(long newTime) {
        queue[1].nextExecutionTime = newTime;
        fixDown(1);
    }

    /**
     * Returns true if the priority queue contains no elements.
     *
     * 队列是否为空
     */
    boolean isEmpty() {
        return size==0;
    }

    /**
     * Removes all elements from the priority queue.
     *
     * 清除队列中的所有元素
     */
    void clear() {
        // Null out task references to prevent memory leak
        for (int i=1; i<=size; i++)
            queue[i] = null;

        size = 0;
    }

    /**
     * Establishes the heap invariant (described above) assuming the heap
     * satisfies the invariant except possibly for the leaf-node indexed by k
     * (which may have a nextExecutionTime less than its parent's).
     *
     * This method functions by "promoting" queue[k] up the hierarchy
     * (by swapping it with its parent) repeatedly until queue[k]'s
     * nextExecutionTime is greater than or equal to that of its parent.
     *
     * 向上调整，使之重新形成最小堆
     */
    private void fixUp(int k) {
        while (k > 1) {
            int j = k >> 1;
            if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }

    /**
     * Establishes the heap invariant (described above) in the subtree
     * rooted at k, which is assumed to satisfy the heap invariant except
     * possibly for node k itself (which may have a nextExecutionTime greater
     * than its children's).
     *
     * This method functions by "demoting" queue[k] down the hierarchy
     * (by swapping it with its smaller child) repeatedly until queue[k]'s
     * nextExecutionTime is less than or equal to those of its children.
     *
     * 向下调整，使之重新形成最小堆
     */
    private void fixDown(int k) {
        int j;
        while ((j = k << 1) <= size && j > 0) {
            if (j < size &&
                queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)
                j++; // j indexes smallest kid
            if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }

    /**
     * Establishes the heap invariant (described above) in the entire tree,
     * assuming nothing about the order of the elements prior to the call.
     */
    void heapify() {
        for (int i = size/2; i >= 1; i--)
            fixDown(i);
    }
}
```

### 3. TimerThread

`TimerThread`作为`Timer`的成员变量，扮演着调度器的校色。我们先来看看它的构造方法，作用主要就是持有任务队列。

```java
TimerThread(TaskQueue queue) {
    this.queue = queue;
}
```

接下来看看`run()`方法，也就是线程执行的入口。

```java
public void run() {
    try {
        mainLoop();
    } finally {
        // Someone killed this Thread, behave as if Timer cancelled
        synchronized(queue) {
            newTasksMayBeScheduled = false;
            queue.clear();  // Eliminate obsolete references
        }
    }
}
```

主逻辑全在`mainLoop()`方法。在`mainLoop`方法执行完之后，会进行资源的清理操作。我们来看看`mainLoop()`方法。

```java
private void mainLoop() {
    // while死循环
    while (true) {
        try {
            TimerTask task;
            boolean taskFired;
            // 对queue进行加锁，保证一个队列里所有的任务都是串行执行的
            synchronized(queue) {
                // Wait for queue to become non-empty
                // 操作1，队列为空，需要等待新任务被调度，这时进行wait操作
                while (queue.isEmpty() && newTasksMayBeScheduled)
                    queue.wait();
                // 这儿再次判断队列是否为空，是因为【操作1】有任务进来了，同时任务又被取消了（进行了`cancel`操作），
                // 这时如果队列再次为空，那么需要退出线程，避免循环被卡死
                if (queue.isEmpty())
                    break; // Queue is empty and will forever remain; die

                // Queue nonempty; look at first evt and do the right thing
                long currentTime, executionTime;
                // 取出队列中的堆顶元素（下次执行时间最小的那个任务）
                task = queue.getMin();
                // 这儿对堆元素进行加锁，是为了保证任务的可见性和原子性
                synchronized(task.lock) {
                    // 取消的任务将不再被执行，需要从队列中移除
                    if (task.state == TimerTask.CANCELLED) {
                        queue.removeMin();
                        continue;  // No action required, poll queue again
                    }
                    // 获取系统当前时间和任务下次执行的时间
                    currentTime = System.currentTimeMillis();
                    executionTime = task.nextExecutionTime;

                    // 任务下次执行的时间 <= 系统当前时间，则执行此任务（设置状态标记`taskFired`为true）
                    if (taskFired = (executionTime<=currentTime)) {
                        // `peroid`为0，表示此任务只需执行一次
                        if (task.period == 0) { // Non-repeating, remove
                            queue.removeMin();
                            task.state = TimerTask.EXECUTED;
                        }
                        // period不为0，表示此任务需要重复执行
                        // 在这儿就体现出了`schedule()`方法和`scheduleAtFixedRate()`的区别
                        else { // Repeating task, reschedule
                            queue.rescheduleMin(
                              task.period<0 ? currentTime   - task.period
                                            : executionTime + task.period);
                        }
                    }
                }
                // 任务没有被触发，队列挂起（带超时时间）
                if (!taskFired) // Task hasn't yet fired; wait
                    queue.wait(executionTime - currentTime);
            }
            // 任务被触发，执行任务。执行完后进入下一轮循环
            if (taskFired)  // Task fired; run it, holding no locks
                task.run();
        } catch(InterruptedException e) {
        }
    }
}
```

### 4. Timer

`Timer`通过构造方法做了下面的事情：

1. 填充`TimerThread`对象的各项属性（比如线程名字、是否守护线程）
2. 启动线程

```java
/**
 * The timer thread.
 */
private final TimerThread thread = new TimerThread(queue);

public Timer(String name, boolean isDaemon) {
    thread.setName(name);
    thread.setDaemon(isDaemon);
    thread.start();
}
```

在`Timer`中，真正的暴露给用户使用的调度方法只有两个，`schedule()`和`scheduleAtFixedRate()`，我们来看看。

```java
public void schedule(TimerTask task, long delay) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");
    sched(task, System.currentTimeMillis()+delay, 0);
}

public void schedule(TimerTask task, Date time) {
    sched(task, time.getTime(), 0);
}

public void schedule(TimerTask task, long delay, long period) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, System.currentTimeMillis()+delay, -period);
}

public void schedule(TimerTask task, Date firstTime, long period) {
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, firstTime.getTime(), -period);
}

public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
    if (delay < 0)
        throw new IllegalArgumentException("Negative delay.");
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, System.currentTimeMillis()+delay, period);
}

public void scheduleAtFixedRate(TimerTask task, Date firstTime,
                                long period) {
    if (period <= 0)
        throw new IllegalArgumentException("Non-positive period.");
    sched(task, firstTime.getTime(), period);
}
```

从上面的代码我们看出下面几点。

1. 这两个方法最终都调用了`sched()`私有方法
2. `schedule()`传入的`period`为负数，`scheduleAtFixedRate()`传入的`period`为正数

接下来我们看看`sched()`方法。

```java
private void sched(TimerTask task, long time, long period) {
    // 1. `time`不能为负数的校验
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");

    // Constrain value of period sufficiently to prevent numeric
    // overflow while still being effectively infinitely large.
    // 2. `period`不能超过`Long.MAX_VALUE >> 1`
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;

    synchronized(queue) {
    	// 3. Timer被取消时，不能被调度
        if (!thread.newTasksMayBeScheduled)
            throw new IllegalStateException("Timer already cancelled.");

        // 4. 对任务加锁，然后设置任务的下次执行时间、执行周期和任务状态，保证任务调度和任务取消是线程安全的
        synchronized(task.lock) {
            if (task.state != TimerTask.VIRGIN)
                throw new IllegalStateException(
                    "Task already scheduled or cancelled");
            task.nextExecutionTime = time;
            task.period = period;
            task.state = TimerTask.SCHEDULED;
        }
        // 5. 将任务添加进队列
        queue.add(task);
        // 6. 队列中如果堆顶元素是当前任务，则唤醒队列，让`TimerThread`可以进行任务调度
        if (queue.getMin() == task)
            queue.notify();
    }
}
```

`sched()`方法经过了下述步骤：

1. `time`不能为负数的校验
2. `period`不能超过`Long.MAX_VALUE >> 1`
3. Timer被取消时，不能被调度
4. 对任务加锁，然后设置任务的下次执行时间、执行周期和任务状态，保证任务调度和任务取消是线程安全的
5. 将任务添加进队列
6. 队列中如果堆顶元素是当前任务，则唤醒队列，让`TimerThread`可以进行任务调度

> 【说明】：我们需要特别关注一下第6点。为什么堆顶元素必须是当前任务时才唤醒队列呢？原因在于堆顶元素所代表的意义，即：堆顶元素表示离当前时间最近的待执行任务！
【例子1】：假如当前时间为1秒，队列里有一个任务A需要在3秒执行，我们新加入的任务B需要在5秒执行。这时，因为`TimerThread`有`wait(timeout)`操作，时间到了会自己唤醒。所以为了性能考虑，不需要在`sched()`操作的时候进行唤醒。
【例子2】：假如当前时间为1秒，队列里有一个任务A需要在3秒执行，我们新加入的任务B需要在2秒执行。这时，如果不在`sched()`中进行唤醒操作，那么任务A将在3秒时执行。而任务B因为需要在2秒执行，已经过了它应该执行的时间，从而出现问题。

任务调度方法`sched()`分析完之后，我们继续分析其他方法。先来看一下`cancel()`，该方法用于取消`Timer`的执行。

```java
 public void cancel() {
    synchronized(queue) {
        thread.newTasksMayBeScheduled = false;
        queue.clear();
        queue.notify();  // In case queue was already empty.
    }
}
```

从上面源码分析来看，该方法做了下面几件事情：

1. 设置`TimerThread`的`newTasksMayBeScheduled`标记为false
2. 清空队列
3. 唤醒队列

有的时候，在一个`Timer`中可能会存在多个`TimerTask`。如果我们只是取消其中几个`TimerTask`，而不是全部，除了对`TimerTask`执行`cancel()`方法调用，还需要对`Timer`进行清理操作。这儿的清理方法就是`purge()`，我们来看看其实现逻辑。

```java
public int purge() {
     int result = 0;

     synchronized(queue) {
         // 1. 遍历所有任务，如果任务为取消状态，则将其从队列中移除，移除数做加一操作
         for (int i = queue.size(); i > 0; i--) {
             if (queue.get(i).state == TimerTask.CANCELLED) {
                 queue.quickRemove(i);
                 result++;
             }
         }
         // 2. 将队列重新形成最小堆
         if (result != 0)
             queue.heapify();
     }

     return result;
 }
```

### 5. 唤醒队列的方法

通过前面源码的分析，我们看到队列的唤醒存在于下面几处：

1. Timer.cancel()
2. Timer.sched()
3. Timer.threadReaper.finalize()

第一点和第二点其实已经分析过了，下面我们来看看第三点。

```java
private final Object threadReaper = new Object() {
    protected void finalize() throws Throwable {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.notify(); // In case queue is empty.
        }
    }
};
```

该方法用于在GC阶段对任务队列进行唤醒，此处往往被读者所遗忘。

那么，我们回过头来想一下，为什么需要这段代码呢？

我们在分析`TimerThread`的时候看到：如果`Timer`创建之后，没有被调度的话，将一直wait，从而陷入`假死状态`。为了避免这种情况，并发大师Doug Lea机智地想到了在`finalize()`中设置状态标记`newTasksMayBeScheduled `，并对任务队列进行唤醒操作（queue.notify()），将`TimerThread`从死循环中解救出来。

## 总结

首先，本文演示了`Timer`是如何使用的，然后分析了调度方法`schedule()`和`scheduleAtFixedRate()`的区别和联系。

然后，为了加深我们对`Timer`的理解，我们通过阅读源码的方式进行了深入的分析。可以看得出，其内部实现得非常巧妙，考虑得也很完善。

但是因为`Timer`串行执行的特性，限制了其在高并发下的运用。后面我们将深入分析高并发、分布式环境下的任务调度是如何实现的，让我们拭目以待吧~

## 参考链接

+ https://www.cnblogs.com/jmcui/p/7519759.html
+ https://www.cnblogs.com/xiaotaoqi/p/6874713.html
+ [[二叉堆(一)之 图文解析 和 C语言的实现](https://www.cnblogs.com/skywang12345/p/3610187.html)
](https://www.cnblogs.com/skywang12345/p/3610187.html)