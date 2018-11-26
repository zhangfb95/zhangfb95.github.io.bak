---
layout: post
title:  java线程池陷阱分析
date:   2016-10-12 06:07:32 +0800
categories: java
tag: java
---

* content
{:toc}

公司的一个线上bug，在使用线程池+Future，且并发量较大时（访问量大于线程池数量），出现访问时间超长的问题。

## 问题现象

先看一个现象，下面的例子程序可以执行，但是我们发现只会执行序号为0-19的线程，并且程序不会终止，会在`f.get()`时卡住。

```java
public class MultiThreadTest {

    @Test
    public void test() throws Exception {
        // 声明一个核心连接数和最大连接数都为10的线程池，工作队列大小也为10，拒绝策略使用忽略策略
        ThreadPoolExecutor es = new ThreadPoolExecutor(10, 10, 1, TimeUnit.MINUTES, new ArrayBlockingQueue<Runnable>(10),
                new ThreadPoolExecutor.DiscardPolicy());
        List<Future<Integer>> futures = new ArrayList<>();

        for (int i = 0; i < 30; i++) {
            final int ii = i;
            Future f = es.submit(new Callable<Integer>() {
                public Integer call() {
                    System.out.println("Running task " + ii);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        // ignore
                    }
                    System.out.println("Task " + ii + " ends");
                    return ii;
                }
            });
            futures.add(f);
        }

        for (Future<Integer> f : futures) {
            try {
                System.out.println("Result: " + f.get());
            } catch (Exception e) {
                f.cancel(true);
                e.printStackTrace();
            }
        }
    }
}
```

## 问题分析

我们知道线程池的执行，在某些临界操作时，需要一些特殊处理，比如：线程池满、执行超时、逻辑异常等。
而我们使用的线程数+工作队列长度=20，比实际的请求操作数30小。所以在线程池满的时候，这段代码执行是有问题的。

首先，这儿使用的拒绝策略为`ThreadPoolExecutor.DiscardPolicy`，这个策略的处理逻辑如下：

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardPolicy}.
     */
    public DiscardPolicy() { }

    /**
     * Does nothing, which has the effect of discarding task r.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```

它不会抛出异常，也没有做资源释放相关的工作，仅仅是“忽略”。

其次，我们看`ThreadPoolExecutor:submit`的执行。

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

因为线程池满了，所以最终会执行`reject(command);`操作。
而这个方法会执行拒绝策略的`rejectedExecution`方法，而`ThreadPoolExecutor.DiscardPolicy`没有做任何处理。

```java
private volatile RejectedExecutionHandler handler;

final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
```

第三，我们看看`FutureTask`是怎么处理的。`Future`有`state`表示其状态，这个状态有以下几个。

```java
private volatile int state;

private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

而state默认又是NEW

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

`FutureTask.get()`方法根据`state`的值不同，处理方法又不相同。它在状态为`NEW`和`COMPLETING`时，就一直等待直至完结。
而此时`Future`又不可能完结得了，所以就出现线程死锁的现象了。

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

最后，我们忽略了一个比较重要的地方，`ThreadPoolExecutor:submit`这个方法的声明中，会抛出两种异常：

1. NullPointerException
1. RejectedExecutionException

第一种异常很好理解，当传入的参数对象为空指针时，会抛出。第二种异常恰恰就是我们忽略的地方，它其实是拒绝策略所可能抛出的异常。

默认的拒绝策略一共有以下几种：

1. AbortPolicy // 直接拒绝，抛出RejectedExecutionException
1. CallerRunsPolicy // 再次执行线程的run方法
1. DiscardOldestPolicy // 移除最老的线程，再次执行当前线程的run方法
1. DiscardPolicy // 直接放弃执行，忽略

我们根据实际业务场景不同，选择的策略也是不同的。
我们需要特别注意，**`DiscardOldestPolicy`和`DiscardPolicy`因为没有抛出`RejectedExecutionException`，所以不能和Future一起使用。**

## 总结

针对此问题，我们提供两种解决方案。

一、在拒绝策略中进行`Future`的关闭；

```java
public static class MyOwnPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() {}
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (r instanceof Future) {
            Future f = (Future) r;
            f.cancel(true);
        }
    }
}
```

二、拒绝策略必须抛出`RejectedExecutionException`，对`ThreadPoolExecutor:submit`进行异常捕获。

这儿的demo有两点替换，一是拒绝策略变更为了`AbortPolicy`，二是`ThreadPoolExecutor:submit`有进行异常处理。

```java
public class MultiThreadTest {

    @Test
    public void test() throws Exception {
        // 声明一个核心连接数和最大连接数都为10的线程池，工作队列大小也为10，拒绝策略使用忽略策略
        ThreadPoolExecutor es = new ThreadPoolExecutor(10, 10, 1, TimeUnit.MINUTES, new ArrayBlockingQueue<Runnable>(10),
                new ThreadPoolExecutor.AbortPolicy());
        List<Future<Integer>> futures = new ArrayList<>();

        for (int i = 0; i < 30; i++) {
            final int ii = i;
            Future f = null;
            try {
                f = es.submit(new Callable<Integer>() {
                    public Integer call() {
                        System.out.println("Running task " + ii);
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            // ignore
                        }
                        System.out.println("Task " + ii + " ends");
                        return ii;
                    }
                });
                futures.add(f);
            } catch (Exception e) {
                //TODO exception handle codes is here
                e.printStackTrace();
            }
        }



        for (Future<Integer> f : futures) {
            try {
                System.out.println("Result: " + f.get());
            } catch (Exception e) {
                f.cancel(true);
                e.printStackTrace();
            }
        }
    }
}
```

## 参考文档

[A Concurrent Affair](http://www.concurrentaffair.org/2012/10/27/problems-with-rejectedexecutionhandler-and-futures/)