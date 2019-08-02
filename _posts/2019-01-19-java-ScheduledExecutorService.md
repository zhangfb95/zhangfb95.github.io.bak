---
layout: post
title:  Java定时调度机制 - ScheduledExecutorService
date:   2019-01-19 23:55:00 +0800
categories: 深入理解Java并发
tag: 深入理解Java并发
---

* content
{:toc}

## 前言
通过上一篇文章【[Java定时调度机制 - Timer](https://www.jianshu.com/p/f4c195840159)】的分析，我们知道，Java的定时调度可以通过`Timer&TimerTask`来实现。由于其实现的方式为单线程，因此从JDK1.3发布之后就一直存在一些问题，大致如下：

1. 多个任务之间会相互影响
2. 多个任务的执行是串行的，性能较低

`ScheduledExecutorService`在设计之初就是为了解决`Timer&TimerTask`的这些问题。因为天生就是基于多线程机制，所以任务之间不会相互影响（只要线程数足够。当线程数不足时，有些任务会复用同一个线程）。

除此之外，因为其内部使用的延迟队列，本身就是基于`等待/唤醒`机制实现的，所以CPU并不会一直繁忙。同时，多线程带来的CPU资源复用也能极大地提升性能。

## 如何使用

### 基本作用

因为`ScheduledExecutorService`继承于`ExecutorService`，所以本身支持线程池的所有功能。额外还提供了4种方法，我们来看看其作用。
```java
/**
 * 带延迟时间的调度，只执行一次
 * 调度之后可通过Future.get()阻塞直至任务执行完毕
 */
1. public ScheduledFuture<?> schedule(Runnable command,
                                      long delay, TimeUnit unit);

/**
 * 带延迟时间的调度，只执行一次
 * 调度之后可通过Future.get()阻塞直至任务执行完毕，并且可以获取执行结果
 */
2. public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                          long delay, TimeUnit unit);

/**
 * 带延迟时间的调度，循环执行，固定频率
 */
3. public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                 long initialDelay,
                                                 long period,
                                                 TimeUnit unit);

/**
 * 带延迟时间的调度，循环执行，固定延迟
 */
4. public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                    long initialDelay,
                                                    long delay,
                                                    TimeUnit unit);
```

### 1. schedule Runnable

该方法用于带延迟时间的调度，只执行一次。调度之后可通过`Future.get()`阻塞直至任务执行完毕。我们来看一个例子。

```java
@Test public void test_schedule4Runnable() throws Exception {
    ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();
    ScheduledFuture future = service.schedule(() -> {
        try {
            Thread.sleep(3000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task finish time: " + format(System.currentTimeMillis()));
    }, 1000, TimeUnit.MILLISECONDS);
    System.out.println("schedule finish time: " + format(System.currentTimeMillis()));

    System.out.println("Runnable future's result is: " + future.get() +
                       ", and time is: " + format(System.currentTimeMillis()));
}
```

上述代码达到的效果应该是这样的：延迟执行时间为1秒，任务执行3秒，任务只执行一次，同时通过`Future.get()`阻塞直至任务执行完毕。

我们运行看到的效果的确和我们猜想的一样，如下图所示。

![执行结果](https://upload-images.jianshu.io/upload_images/845143-4b57b453110cc8a8.png?jianshufrom=true)


### 2. schedule Callable

在schedule Runnable的基础上，我们将`Runnable`改为`Callable`来看一下。

```java
@Test public void test_schedule4Callable() throws Exception {
    ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();
    ScheduledFuture<String> future = service.schedule(() -> {
        try {
            Thread.sleep(3000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task finish time: " + format(System.currentTimeMillis()));
        return "success";
    }, 1000, TimeUnit.MILLISECONDS);
    System.out.println("schedule finish time: " + format(System.currentTimeMillis()));

    System.out.println("Callable future's result is: " + future.get() +
            ", and time is: " + format(System.currentTimeMillis()));
}
```

运行看到的结果和`Runnable`基本相同，唯一的区别在于`future.get()`能拿到`Callable`返回的真实结果。

![执行结果](https://upload-images.jianshu.io/upload_images/845143-8c3c2958acca6907.png?jianshufrom=true)

### 3. scheduleAtFixedRate

该方法用于`固定频率`地对一个任务循环执行，我们通过一个例子来看看效果。

```java
@Test public void test_scheduleAtFixedRate() {
    ScheduledExecutorService service = Executors.newScheduledThreadPool(5);
    service.scheduleAtFixedRate(() -> {
        try {
            Thread.sleep(3000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task finish time: " + format(System.currentTimeMillis()));
    }, 1000L, 1000L, TimeUnit.MILLISECONDS);

    System.out.println("schedule finish time: " + format(System.currentTimeMillis()));
    while (true) {
    }
}
```

在这个例子中，任务初始延迟1秒，任务执行3秒，任务执行间隔为1秒。我们来看看执行结果：

![执行结果](https://upload-images.jianshu.io/upload_images/845143-0827cfcc6c06801d.png?jianshufrom=true)

### 4. scheduleWithFixedDelay

该方法用于`固定延迟`地对一个任务循环执行，我们通过一个例子来看看效果。

```java
@Test public void test_scheduleWithFixedDelay() {
    ScheduledExecutorService service = Executors.newScheduledThreadPool(5);
    service.scheduleWithFixedDelay(() -> {
        try {
            Thread.sleep(3000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task finish time: " + format(System.currentTimeMillis()));
    }, 1000L, 1000L, TimeUnit.MILLISECONDS);

    System.out.println("schedule finish time: " + format(System.currentTimeMillis()));
    while (true) {
    }
}
```

在这个例子中，任务初始延迟1秒，任务执行3秒，任务执行间隔为1秒。我们来看看执行结果：

![执行结果](https://upload-images.jianshu.io/upload_images/845143-9cdc36f781858c08.png?jianshufrom=true)

### 5. scheduleAtFixedRate和scheduleWithFixedDelay的区别

既然这两个方法都是对任务循环执行，那么他们又有何区别呢？通过jdk文档我们找到了答案。

![scheduleAtFixedRate - javadoc](https://upload-images.jianshu.io/upload_images/845143-13f897d65d974d33.png?jianshufrom=true)

![scheduleWithFixedDelay - javadoc](https://upload-images.jianshu.io/upload_images/845143-5007960988c65afd.png?jianshufrom=true)

> 直白地讲，`scheduleAtFixedRate()`为固定频率，`scheduleWithFixedDelay()`为固定延迟。固定频率是相对于任务执行的开始时间，而固定延迟是相对于任务执行的结束时间，这就是他们最根本的区别！

另外，从3和4的运行结果也能看出这些差异。

## 源码阅读初体验

一般源码的入口在于构造方法，我们来看看。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

在构造方法中我们看到以下信息：

1. `ScheduledThreadPoolExecutor`构造方法最终调用的是`ThreadPoolExecutor`构造方法
2. 阻塞队列使用的是`DelayedWorkQueue`

上述信息的第2点至关重要，但是限于篇幅，本文将不做深入分析。

接下来我们看看`scheduleWithFixedDelay()`方法，主要做了3件事情：

1. 入参校验，包括空指针、数字范围
2. 将`Runnable`包装成`RunnableScheduledFuture`
3. 延迟执行`RunnableScheduledFuture`

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit) {
    // 1. 入参校验，包括空指针、数字范围
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    // 2. 将Runnable包装成`RunnableScheduledFuture`
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(-delay));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    // 3. 延迟执行`RunnableScheduledFuture`
    delayedExecute(t);
    return t;
}
```

`delayedExecute()`这个方法从字面描述来看是延迟执行的意思，我们深入到这个方法里面去看看。

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    // 1. 线程池运行状态判断
    if (isShutdown())
        reject(task);
    else {
        // 2. 将任务添加到队列
        super.getQueue().add(task);
        // 3. 如果任务添加到队列之后，线程池状态变为非运行状态，
        // 需要将任务从队列移除，同时通过任务的`cancel()`方法来取消任务
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        // 4. 如果任务添加到队列之后，线程池状态是运行状态，需要提前启动线程
        else
            ensurePrestart();
    }
}
```

在线程池状态正常的情况下，最终会调用`ensurePrestart()`方法来完成线程的创建。主要逻辑有两个：

1. 当前线程数未达到核心线程数，则创建核心线程
2. 当前线程数已达到核心线程数，则创建非核心线程，不会将任务放到阻塞队列中，这一点是和普通线程池是不相同的

```java
/**
 * Same as prestartCoreThread except arranges that at least one
 * thread is started even if corePoolSize is 0.
 */
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    // 1. 当前线程数未达到核心线程数，则创建核心线程
    if (wc < corePoolSize)
        addWorker(null, true);
    // 2. 当前线程数已达到核心线程数，则创建非核心线程，
    // 2.1 不会将任务放到阻塞队列中，这一点是和普通线程池是不相同的
    else if (wc == 0)
        addWorker(null, false);
}
```

至此，除了`DelayedWorkQueue`延迟队列的源码还未分析，其他的我们都分析完了。

## 总结

首先，我们了解了`ScheduledExecutorService`的基本作用，然后在此基础上写了一些demo来做验证，得到的结果和基本作用是完全相同的。

然后，我们对其内部的实现原理和源代码做了初步的分析，知道了其和普通线程池是不同的地方在于：`阻塞队列`和`创建线程的方式`。