---
layout: post
title:  Thread的过期方法和常用方法
date:   2018-10-26 22:59:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

`Thread`类里面有很多方法，大多数在jdk早期版本就已经存在了。有些方法，他们的命名都是非常简单易懂的，但是因为某些问题已经废弃掉了。有些方法却是很常用的。下面我们简单罗列一下：

1. stop() // 过期方法，停止线程的执行
2. suspend() // 过期方法，暂停线程的执行
3. resume() // 过期方法，恢复线程的执行
4. interrupt() // 常用方法，中断线程
5. boolean isInterrupted() // 常用方法，返回线程是否被中断
6. static boolean interrupted() // 常用方法，返回线程是否被中断，同时清空中断标记
7. join() // 常用方法，等待线程结束
8. yield() // 常用方法，让出cpu时间片
9. sleep() // 常用方法，线程休眠
10. holdsLock(Object obj) // 常用方法，线程是否持有锁
11. setContextClassLoader() // 常用方法，设置线程上下文类加载器

## 过期方法 - stop()

```java
/**
 * @deprecated This method is inherently unsafe.  Stopping a thread with
 *       Thread.stop causes it to unlock all of the monitors that it
 *       has locked (as a natural consequence of the unchecked
 *       <code>ThreadDeath</code> exception propagating up the stack).  If
 *       any of the objects previously protected by these monitors were in
 *       an inconsistent state, the damaged objects become visible to
 *       other threads, potentially resulting in arbitrary behavior.  Many
 *       uses of <code>stop</code> should be replaced by code that simply
 *       modifies some variable to indicate that the target thread should
 *       stop running.  The target thread should check this variable
 *       regularly, and return from its run method in an orderly fashion
 *       if the variable indicates that it is to stop running.  If the
 *       target thread waits for long periods (on a condition variable,
 *       for example), the <code>interrupt</code> method should be used to
 *       interrupt the wait.
 *       For more information, see
 *       <a href="{@docRoot}/../technotes/guides/concurrency/threadPrimitiveDeprecation.html">Why
 *       are Thread.stop, Thread.suspend and Thread.resume Deprecated?</a>.
 */
@Deprecated
public final void stop() {
	SecurityManager security = System.getSecurityManager();
	if (security != null) {
		checkAccess();
		if (this != Thread.currentThread()) {
			security.checkPermission(SecurityConstants.STOP_THREAD_PERMISSION);
		}
	}
	// A zero status value corresponds to "NEW", it can't change to
	// not-NEW because we hold the lock.
	if (threadStatus != 0) {
		resume(); // Wake up thread if it was suspended; no-op otherwise
	}

	// The VM can handle all thread states
	stop0(new ThreadDeath());
}
```
翻译一下废弃的原因：

```
该方法本质上是不安全的。调用此方法会释放所有持有的监视器的锁（作为沿堆栈向上传播的未检查 ThreadDeath 异常的一个自然后果）。
如果这些监视器对象保护的数据存在不一致的状态，这些不一致的数据将对其他线程将变得可见，会导致任何可能的行为。
stop的许多使用都应由只修改某些变量以指示目标线程应该停止运行的代码来取代。目标线程应定期检查该变量，并且如果该变量指示它要停止运行，则从其运行方法依次返回。
如果目标线程等待很长时间（例如基于一个条件变量），则应使用 interrupt 方法来中断该等待。
```

其实原因里面说得很明白了。
1. 废弃原因为`锁对象保护的数据在调用stop方法的时候可能存在不一致的情况`。
2. 解决方法为`使用基于变量的方式来来stop线程，变量由外部传入`，该变量通常是`volatile boolean`类型，且在循环里面所使用。

## 过期方法 - suspend() / resume()

我们使用音乐播放器播放音乐时，可以对正在播放的音乐`暂停`，对暂停的音乐`恢复`或`继续`播放。这两个方法分别对应到`暂停`和`恢复`，他们通常是成对出现的。我们看看源码上废弃的原因。

```java
/**
 * @deprecated   This method has been deprecated, as it is
 *   inherently deadlock-prone.  If the target thread holds a lock on the
 *   monitor protecting a critical system resource when it is suspended, no
 *   thread can access this resource until the target thread is resumed. If
 *   the thread that would resume the target thread attempts to lock this
 *   monitor prior to calling <code>resume</code>, deadlock results.  Such
 *   deadlocks typically manifest themselves as "frozen" processes.
 *   For more information, see
 *   <a href="{@docRoot}/../technotes/guides/concurrency/threadPrimitiveDeprecation.html">Why
 *   are Thread.stop, Thread.suspend and Thread.resume Deprecated?</a>.
 */
@Deprecated
public final void suspend() {
    checkAccess();
    suspend0();
}
```

```java
/**
 * @deprecated This method exists solely for use with {@link #suspend},
 *     which has been deprecated because it is deadlock-prone.
 *     For more information, see
 *     <a href="{@docRoot}/../technotes/guides/concurrency/threadPrimitiveDeprecation.html">Why
 *     are Thread.stop, Thread.suspend and Thread.resume Deprecated?</a>.
 */
@Deprecated
public final void resume() {
    checkAccess();
    resume0();
}
```

这两个方法混合使用时，存在死锁的风险。
1. 如果目标线程持有锁，调用suspend之后，不会释放锁。
2. 如果resume方法先于suspend方法调用，就会导致死锁

为什么会这样呢，我们写一个demo来看看。

```java
public class BadSuspend {

    public static Object u = new Object();

    static ChangeObjectThread t1 = new ChangeObjectThread("t1");
    static ChangeObjectThread t2 = new ChangeObjectThread("t2");

    public static class ChangeObjectThread extends Thread {

        public ChangeObjectThread(String name) {
            super(name);
        }

        @Override public void run() {
            synchronized (u) {
                System.out.println("in " + getName());
                Thread.currentThread().suspend();
            }
        }
    }

    public static void main(String[] args) throws Exception {
		// t1的start和t1的resume之间因为有sleep方法，所以resume会比suspend后执行，因此也不会出现死锁的问题
		// t2的start和t2的resume之间没有sleep方法，通常情况resume会比suspend先执行，而suspend并不会释放锁，也不会被恢复，从而出现死锁
		// t2死锁的时候我通过jps和jstack查看线程的状态，居然是Runnable，非常的坑爹！
		// t2的临时解决方法是：在t2.resume之前加入sleep
        t1.start();
        Thread.sleep(100L);
        t2.start();
        t1.resume();
        // Thread.sleep(100L);
        t2.resume();
        t1.join();
        t2.join();
    }
}
```

![image.png](https://upload-images.jianshu.io/upload_images/845143-2b5a325a548255eb.png)


## 常用方法 - interrupt() / boolean isInterrupted() / static boolean interrupted()

这三个方法都和线程中断有关，中断可以代替stop来停止线程，但是停止线程的过程又由用户自己来控制，并不像stop那样暴力。

1. interrupt() // 中断线程，这儿的中断只是给线程打一个中断的标记，并不会真正地使线程退出。线程的退出仍然由用户来控制
2. boolean isInterrupted() // 判断线程是否是中断的，该方法可以重复调用，而不会清除掉线程的中断标记
3. static boolean interrupted() // 判断线程是否是中断的，同时清除线程的中断标记（这点和boolean isInterrupted()有所不同）

还是写一些demo来看看效果吧。

```java
public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		for (; ; ) {
		}
	});

	t1.start();
	Thread.sleep(1000);
	t1.interrupt();
	System.out.println("main stopped");
}
```

我们直接写一个for循环，main线程会调用interrupt，但是我们发现t1线程并没有退出，这是因为interrupt只会给线程设置一个中断标记，并不会真正地中断线程，而是需要用户自己来选择，在合适的时机来做处理。我们实际开发中不要这样写！

修改后的代码如下

```java
public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		for (; ; ) {
			if (Thread.currentThread().isInterrupted()) {
				break;
			}
		}
		System.out.println("ended");
	});

	t1.start();
	Thread.sleep(1000);
	t1.interrupt();
	System.out.println("main stopped");
}
```

我们另外再看看`interrupted`方法。通过以下的方式，我们可以看出`interrupted会清除中断标记`。

```java
public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		for (; ; ) {
			if (Thread.interrupted()) {
				System.out.println("is interrupted: " + Thread.currentThread().isInterrupted());
				break;
			}
		}
		System.out.println("ended");
	});

	t1.start();
	Thread.sleep(1000);
	t1.interrupt();
	System.out.println("main stopped");
}
```

另外，在我们的代码中如果有调用`sleep`或者`wait`方法，这两个方法在中断的时候会清除中断标记，同时抛出`InterruptedException`异常。还是以实际demo为例。

```java
public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
		for (; ; ) {
			try {
				Thread.sleep(1000L);
			} catch (InterruptedException e) {
				e.printStackTrace();
				System.out.println("interrupted status: " + Thread.currentThread().isInterrupted());
				break;
			}
		}
	});

	t1.start();
	Thread.sleep(1000);
	t1.interrupt();
	System.out.println("main stopped");
}
```

## 常用方法 - join()

```java
/**
 * Waits for this thread to die.
 *
 * <p> An invocation of this method behaves in exactly the same
 * way as the invocation
 *
 * <blockquote>
 * {@linkplain #join(long) join}{@code (0)}
 * </blockquote>
 *
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public final void join() throws InterruptedException {
    join(0);
}
```

字面意思很简单，等待线程直到死，够直接的哈！
这个方法一般在一个线程需要另一个线程结果的时候使用。那么这个方法内部又是怎么实现的呢？我们分析一下源码。

```java
public final synchronized void join(long millis)
throws InterruptedException {
	// 获取当前时间，设置为base
    long base = System.currentTimeMillis();
    long now = 0;
	
	// 如果等待的毫秒数是负数，则传入参数有问题，抛出异常
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

	// 传入的参数为0，则一直循环判断当前线程是否是活着，活着的话就wait(0)，直到另外一个线程notify当前线程
    if (millis == 0) {
		// isAlive()就是线程start之后，且还没有死亡
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
			// 带超时时间的wait
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

源码还是蛮简单的，内部实现是通过`wait()`和`wait(long timeout)`来实现的。我们还是写一个demo来验证一下这个方法的用法吧。

```java
public class JoinMain {

    public volatile static int i = 0;

    public static class AddThread extends Thread {

        @Override public void run() {
            for (i = 0; i < 1000000; i++) {
            }
        }
    }

    public static void main(String[] args) throws Exception {
        AddThread at = new AddThread();
        at.start();
        at.join();
        System.out.println(i);
    }
}
```

这段代码最终打印的结果一定是1000000！

## 常用方法 - yield()

这个方法比较简单，就是让出cpu时间片，让其他线程执行。

```java
/**
 * A hint to the scheduler that the current thread is willing to yield
 * its current use of a processor. The scheduler is free to ignore this
 * hint.
 *
 * <p> Yield is a heuristic attempt to improve relative progression
 * between threads that would otherwise over-utilise a CPU. Its use
 * should be combined with detailed profiling and benchmarking to
 * ensure that it actually has the desired effect.
 *
 * <p> It is rarely appropriate to use this method. It may be useful
 * for debugging or testing purposes, where it may help to reproduce
 * bugs due to race conditions. It may also be useful when designing
 * concurrency control constructs such as the ones in the
 * {@link java.util.concurrent.locks} package.
 */
public static native void yield();
```

## 常用方法 - sleep()

该方法的意思为`使线程休眠一段时间`，休眠的线程，其状态为`TIMED_WAITING`。该方法在Thread里面有两个重载方法

1. public static native void sleep(long millis) throws InterruptedException;
2. public static void sleep(long millis, int nanos) throws InterruptedException;

我们剖析一下源码

```java
/**
 * 1. 让当前线程休眠一定时间（暂停执行）
 * 2. 线程不会释放锁
 * 3. 纳秒其实并不是真正的纳秒休眠，而是将纳秒进行四舍五入
 * 
 * Causes the currently executing thread to sleep (temporarily cease
 * execution) for the specified number of milliseconds plus the specified
 * number of nanoseconds, subject to the precision and accuracy of system
 * timers and schedulers. The thread does not lose ownership of any
 * monitors.
 *
 * @param  millis
 *         the length of time to sleep in milliseconds
 *
 * @param  nanos
 *         {@code 0-999999} additional nanoseconds to sleep
 *
 * @throws  IllegalArgumentException
 *          if the value of {@code millis} is negative, or the value of
 *          {@code nanos} is not in the range {@code 0-999999}
 *
 * // InterruptedException，如果有其他线程中断当前线程，则抛出此异常。异常抛出之后，当前线程的中断标记会被清除。
 * @throws  InterruptedException
 *          if any thread has interrupted the current thread. The
 *          <i>interrupted status</i> of the current thread is
 *          cleared when this exception is thrown.
 */
public static void sleep(long millis, int nanos) throws InterruptedException {
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }

    sleep(millis);
}
```

## 常用方法 - holdsLock()

```java
/**
 * 当前线程持有指定对象的监视器锁，才会返回true，否则返回false。
 *
 * Returns <tt>true</tt> if and only if the current thread holds the
 * monitor lock on the specified object.
 *
 * <p>This method is designed to allow a program to assert that
 * the current thread already holds a specified lock:
 * <pre>
 *     assert Thread.holdsLock(obj);
 * </pre>
 *
 * @param  obj the object on which to test lock ownership
 * @throws NullPointerException if obj is <tt>null</tt>
 * @return <tt>true</tt> if the current thread holds the monitor lock on
 *         the specified object.
 * @since 1.4
 */
public static native boolean holdsLock(Object obj);
```

## 常用方法 - setContextClassLoader()

这个方法在很多框架代码里经常出现。为什么呢？

1. 类加载机制里面的双亲委派机制
2. 什么时候会违背双亲委派机制
3. 这个方法如何解决未被双亲委派机制场景下的问题

我们会单独写一篇博客来说明一下类加载机制。