---
layout: post
title:  Java线程池源码分析
date:   2018-12-28 23:57:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

在上一篇文章【[Java线程池的使用](https://www.jianshu.com/p/7ab4ae9443b9)】中，我们分析了线程池的用法。但那仅仅是用法，关于线程池内部是如何实现的，我们却没有深入分析。本着`知其然，知其所以然`的想法，楼主将尝试深入到线程池源码去一窥究竟。

在jdk里面，线程池最重要的实现是`ThreadPoolExecutor`。因此，我们分析的重点就是这个类，主要包括线程池状态、线程池变量、构造方法、提交任务等内容。

## 线程池状态

线程池可以包含多个线程，线程池本身又维护着自己的状态。在`ThreadPoolExecutor`中，线程池状态和worker数量都是由ctl变量控制。

那么，ctl变量又是如何控制线程池状态的呢？我们先来看看ctl相关的源码。

```java
// 1. `ctl`，可以看做一个int类型的数字，高3位表示线程池状态，低29位表示worker数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 2. `COUNT_BITS`，`Integer.SIZE`为32，所以`COUNT_BITS`为29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 3. `CAPACITY`，线程池允许的最大线程数。1左移29位，然后减1，即为 2^29 - 1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 4. 线程池有5种状态，按大小排序如下：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// 5. `runStateOf()`，获取线程池状态，通过按位与操作，低29位将全部变成0
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 6. `workerCountOf()`，获取线程池worker数量，通过按位与操作，高3位将全部变成0
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 7. `ctlOf()`，根据线程池状态和线程池worker数量，生成ctl值
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */
// 8. `runStateLessThan()`，线程池状态小于xx
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}
// 9. `runStateAtLeast()`，线程池状态大于等于xx
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
```

在上面的源码中，每个常量和方法都很关键。这儿，我们对每个关键点都进行一下解释。

1. `ctl`，可以看做一个int类型的数字，高3位表示线程池状态，低29位表示线程的数量
2. `COUNT_BITS`，`Integer.SIZE`为32，所以`COUNT_BITS`为29
3. `CAPACITY`，线程池允许的最大线程数。1左移29位，然后减1，即为 2^29 - 1
4. 线程池有5种状态，按大小排序如下：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED
5. `runStateOf()`，获取线程池状态，通过按位与操作，低29位将全部变成0
6. `workerCountOf()`，获取线程池worker数量，通过按位与操作，高3位将全部变成0
7. `ctlOf()`，根据线程池状态和线程池worker数量，生成ctl值
8. `runStateLessThan()`，线程池状态小于xx
9. `runStateAtLeast()`，线程池状态大于等于xx

为了加深对这段代码的理解，我们将常量对应的二进制数以表格的形式列出来，如下所示。

| 常量 | 二进制数 |
| --- | --- |
| RUNNING | 11100000000000000000000000000000 |
| SHUTDOWN | 00000000000000000000000000000000 |
| STOP | 00100000000000000000000000000000 |
| TIDYING | 01000000000000000000000000000000 |
| TERMINATED | 01100000000000000000000000000000 |
| CAPACITY | 00011111111111111111111111111111 |
| ~CAPACITY | 11100000000000000000000000000000 |

## 构造方法

`ThreadPoolExecutor`有多个重载构造方法，但是他们最终调用了同一个构造方法。该方法非常简单，主要做了两件事情：

1. 对输入参数进行校验并设置到成员变量
2. 根据输入参数`unit`和`keepAliveTime`，将存活时间转换为纳秒存到变量`keepAliveTime `中

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    // 基本类型参数校验
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    // 空指针校验
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    // 根据传入参数`unit`和`keepAliveTime`，将存活时间转换为纳秒存到变量`keepAliveTime `中
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

## 使用线程池执行任务 - execute()

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    // worker数量比核心线程数小，直接创建worker执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // worker数量超过核心线程数，任务直接进入队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 线程池状态不是RUNNING状态，说明执行过shutdown命令，需要对新加入的任务执行reject()操作。
        // 这儿为什么需要recheck，是因为任务入队列前后，线程池的状态可能会发生变化。
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 这儿为什么需要判断0值，主要是在线程池构造方法中，核心线程数允许为0
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果线程池不是运行状态，或者任务进入队列失败，则尝试创建worker执行任务。
    // 这儿有3点需要注意：
    // 1. 线程池不是运行状态时，addWorker内部会判断线程池状态
    // 2. addWorker第2个参数表示是否创建核心线程
    // 3. addWorker返回false，则说明任务执行失败，需要执行reject操作
    else if (!addWorker(command, false))
        reject(command);
}
```

我们已经分析完了`execute()`方法，其内部调用了一些关键方法，如下所示：

1. addWorker() // 创建worker
2. reject() // 拒绝任务

由于`reject()`较为简单，我们先分析它，然后分析`addWorker()`。

## 拒绝任务 - reject()

```java
private volatile RejectedExecutionHandler handler;

final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

只是调用了拒绝策略的回调方法。

## 添加worker - addWorker()

这个方法可以简单地分成两个部分，一部分通过`乐观锁+自旋`的方式增加worker的数量，另一部分通过`悲观锁`的方式创建worker（一个worker可以简单地被认为是线程池里的一个线程）。

先来看第一部分。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 外层自旋
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 这个条件写得比较难懂，我对其进行了调整，和下面的条件等价
        // (rs > SHUTDOWN) || 
        // (rs == SHUTDOWN && firstTask != null) || 
        // (rs == SHUTDOWN && workQueue.isEmpty())
        // 1. 线程池状态大于SHUTDOWN时，直接返回false
        // 2. 线程池状态等于SHUTDOWN，且firstTask不为null，直接返回false
        // 3. 线程池状态等于SHUTDOWN，且队列为空，直接返回false
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        // 内层自旋
        for (;;) {
            int wc = workerCountOf(c);
            // worker数量超过容量，直接返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 使用CAS的方式增加worker数量。
            // 若增加成功，则直接跳出外层循环进入到第二部分
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 线程池状态发生变化，对外层循环进行自旋
            if (runStateOf(c) != rs)
                continue retry;
            // 其他情况，直接内层循环进行自旋即可
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // ...
}
```

接下来我们分析第二部分。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    // ...
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // worker的添加必须是串行的，因此需要加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 这儿需要重新检查线程池状态
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // worker已经调用过了start()方法，则不再创建worker
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // worker创建并添加到workers成功
                    workers.add(w);
                    // 更新`largestPoolSize`变量
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 启动worker线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // worker线程启动失败，说明线程池状态发生了变化（关闭操作被执行），需要进行shutdown相关操作
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

通过对第一部分和第二部分的分析，我们知道了线程池中的线程是封装在`Worker`中执行的。下一步，我们就从`Worker`进行分析。

## 线程池执行单元 - Worker

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 这儿是Worker的关键所在，使用了线程工厂创建了一个线程。传入的参数为当前worker
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // 省略AQS相关的代码...
}
```

上述代码为`Worker`的基本实现（省略了AQS相关的代码）。我们看到`Worker`实现了两个接口，`Runnable`和`AbstractQueuedSynchronizer`。

关于AQS相关的知识，我们在[AbstractQueuedSynchronizer(AQS)源码解析](https://www.jianshu.com/p/a053b93f5bd0)做了比较全面的说明。因此，这儿我们更多的是关心`run()`方法的实现。

`run()`方法其实调用了`ThreadPoolExecutor.runWorker()`方法，且把当前`Worker`对象传递了进去。

## 核心线程执行逻辑 - runWorker()

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 调用unlock()是为了让外部可以中断
    w.unlock(); // allow interrupts
    // 这个变量用于判断是否进入过自旋（while循环）
    boolean completedAbruptly = true;
    try {
        // 这儿是自旋
        // 1. 如果firstTask不为null，则执行firstTask；
        // 2. 如果firstTask为null，则调用getTask()从队列获取任务。
        // 3. 阻塞队列的特性就是：当队列为空时，当前线程会被阻塞等待
        while (task != null || (task = getTask()) != null) {
            // 这儿对worker进行加锁，是为了达到下面的目的
            // 1. 降低锁范围，提升性能
            // 2. 保证每个worker执行的任务是串行的
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 如果线程池正在停止，则对当前线程进行中断操作
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            // 执行任务，且在执行前后通过`beforeExecute()`和`afterExecute()`来扩展其功能。
            // 这两个方法在当前类里面为空实现。
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                // 帮助gc
                task = null;
                // 已完成任务数加一 
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 自旋操作被退出，说明线程池正在结束
        processWorkerExit(w, completedAbruptly);
    }
}
```

该方法是线程池执行任务的关键所在。虽然是核心功能，但是本身的逻辑却非常简单和高效。该方法主要通过自旋、阻塞队列的阻塞特性和worker加锁的方式来完成任务的执行。除此之外，使用了`finally机制`来释放资源（处理线程池结束的后续操作）。

## worker结束处理逻辑 - processWorkerExit

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 这个变量用于表示是否进入过自旋。
    // 1. 如果没有进入过，该值为false
    // 2. 进入过，该值为true
    // 只有进入过自旋，worker的数量才需要减一
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    // 通过全局锁的方式移除worker
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    // 尝试终止线程池
    tryTerminate();

    int c = ctl.get();
    // 如果线程池状态为`SHUTDOWN`或`RUNNING`，
    // 则通过调用`addWorker()`来创建线程，辅助完成对阻塞队列中任务的处理。
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

该方法主要通过全局锁的方式来移除当前worker，同时如果线程池状态是`SHUTDOWN`或`RUNNING`时，则通过调用`addWorker()`来创建线程，辅助完成对阻塞队列中任务的处理。

## 线程池的退出

线程池退出的方式有两种，一种为优雅型，一种为强迫型。

1. 优雅型会等待所有已提交到线程池的任务执行完成之后才结束线程池。
2. 强迫型会对当前正在执行的任务进行中断操作，并将阻塞队列里面的任务以`List<Runnable>`的方式返回给调用方，由调用方做后续处理。

### 优雅型退出 - shutdown()

```java
public void shutdown() {
    // 使用全局锁来操作线程池终止
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 权限校验
        checkShutdownAccess();
        // 通过CAS的方式变更线程池状态为`SHUTDOWN`
        advanceRunState(SHUTDOWN);
        // 中断空闲的worker，这也是和`shutdownNow()`不同的地方
        interruptIdleWorkers();
        // 留给ScheduledThreadPoolExecutor处理
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

关键的地方为`interruptIdleWorkers()`，该方法用于中断空闲的worker。我们来看看这个方法，然后对其进行补充说明。

```java
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}

// 1. `onlyOne`用于控制是否只循环一次，在`shutdown()`里面，该值为false，因此是遍历整个worker列表
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // 2. 对每个worker，会尝试去获取worker锁。
            //    如果获取到了，那么说明该worker是空闲的，可以进行中断操作
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

`interruptIdleWorkers()`做了两件事情：

1. `onlyOne`用于控制是否只循环一次，在`shutdown()`里面，该值为false，因此是遍历整个worker列表
2. 对每个worker，会尝试去获取worker锁。如果获取到了，那么说明该worker是空闲的，可以进行中断操作

### 强迫型退出 - shutdownNow()

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    // 使用全局锁来操作线程池终止
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 权限校验
        checkShutdownAccess();
        // 通过CAS的方式变更线程池状态为`STOP`
        advanceRunState(STOP);
        // 中断所有的worker，这也是和`shutdown()`不同的地方
        interruptWorkers();
        // 将阻塞队列里面的任务排出到tasks
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

该方法和`shutdown()`不同的地方在于对worker的处理方式。`shutdownNow()`处理得非常得强势，调用了`interruptWorkers()`方法。我们来看看此方法，然后作出说明。

```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 1. 对所有worker调用`interruptIfStarted()`方法
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}

private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    void interruptIfStarted() {
        Thread t;
        // 2.线程池状态为`非RUNNING`时，直接对worker包装的线程进行中断操作
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

该方法做了两件事情：

1. 对所有worker调用`interruptIfStarted()`方法
2. 线程池状态为`非RUNNING`时，直接对worker包装的线程进行中断操作

### 线程池状态变更 - advanceRunState()

在`shutdown()`和`shutdownNow()`中，我们都看到了对`advanceRunState()`方法的调用。该方法用于线程安全地变更线程池状态，那么它是如何做到线程安全的呢？

```java
private void advanceRunState(int targetState) {
    // 1. 自旋操作
    for (;;) {
        // 2. 通过CAS的方式保证线程池状态变更的线程安全性
        int c = ctl.get();
        if (runStateAtLeast(c, targetState) ||
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
            break;
    }
}
```

> PS：我们可能很好奇，`advanceRunState()`方法为什么要用CAS呢？在`shutdown()`和`shutdownNow()`里面不是有加锁吗？
请注意，外层方法的确有加锁，但是和worker线程所使用的锁并不是同一个，因此需要通过自旋+CAS的方式来额外保证线程安全！

### 阻塞队列任务排出 - drainQueue()

```java
private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    q.drainTo(taskList);
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```

该方法通过两个阶段来保证导出任务的完整性。

1. 调用阻塞队列的`drainTo()`方法
2. 遍历阻塞队列剩余的元素，通过`remove()&add()`的方式来导出元素。

在`drainTo()`的过程中可能有新元素进来，这也是需要两个阶段来完成导出的原因。

## 带返回结果的任务提交 - submit()

```java
 public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    // ftask既是Runnable，也是Future
    RunnableFuture<T> ftask = newTaskFor(task);
    // 执行任务
    execute(ftask);
    return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable {
    return new FutureTask<T>(callable);
}
```

`submit()`内部调用的仍然是`execute()`方法，只是调完之后返回了`Future`对象，我们可以通过这个对象的`get()`方法来拿到最终执行的结果。

到这里，我们必须打住，Future相关的知识点不在本节的讨论范围内。

因为`Future`和`FutureTask`非常重要，楼主计划单独写一篇博客来做阐述。`Future`相关的东西比较多，而且复杂。为了不影响本问的初衷，至此暂且略过。

## 总结

本文通过对Java线程池代码的分析，使得我们知道了其大概的实现方式。

在Java线程池出现之前，如果我们想异步执行一个任务，往往是先创建一个线程，然后执行任务，执行完之后就扔掉了，最后等待GC对创建线程的回收。而这样处理的后果，往往是非常昂贵的内存和CPU开销。

Java线程池比较彻底地解决了这个问题！其内部实现合理地运用了自旋、CAS、乐观锁、悲观锁、recheck重测、状态机、原子变量等等各种并发技术，充分有效地发挥了CPU的多核计算能力。由此可见并发大师Doug Lea设计和编码的功力之深，简直是前无古人，后无来者？

我们将Java线程池的分析放在这一篇博客中，其实有点管中窥豹之嫌。很多的细节，我们往往是一笔代过。但是，大致的结构和功能却是分析到了的。

最后，楼主希望本文能对读者有所帮助！

## 参考链接

+ https://zhidao.baidu.com/question/148760603.html
+ https://www.jianshu.com/p/7db405f1d6e8