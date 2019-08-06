---
layout: post
title:  jdk的内置锁
date:   2018-10-26 22:56:00 +0800
categories: Java并发和多线程
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

Java并发编程为了保证线程安全，需要维护共享变量的三种特性-原子性、可见性和有序性。而锁可以实现这三种特性，在实际场景中出现的频率非常高，这是因为有些场景我们必须使用锁才能满足需求。但是锁的使用会对性能有所影响，除了某些环境必须使用，否则能不用就不用。

JDK内置有三种锁：

1. `sychronized`关键字
2. `ReentrantLock`可重入锁
3. `ReadWriteLock`读写分离锁

下面我们就来分析一下这三种锁。

## synchronized

`synchronized`有三种用法，其实就是下面代码中的注释说明。

```java
// 1. 代码块，锁对象需要显式指定
public void doSomeThing() {
	synchronized(lock) {
		// do some thing
	}
}

// 2. 对象方法，锁对象不需要显式指定，默认为当前对象，即：this
public synchronized void doSomeThing() {
	// do some thing
}

// 3. 类方法，锁对象不需要显式指定，默认为当前类，即：getClass()
public static synchronized void doSomeThing() {
	// do some thing
}
```

使用`synchronized`时我们需要特别注意锁释放的时机，退出时释放锁。

1. 正常执行完成
2. 抛出异常退出

其内部实现为`monitorenter`和`monitorexit`，而这两条指令是对更底层的jvm指令`lock`和`unlock`的封装。
`synchronized`是jdk最先引入的锁机制，但是因为它存在死锁的风险，所以在jdk1.5及之后的版本引入了其他的锁机制来代替解决死锁问题，同时也为了满足更多的需求而加入了更多的功能支持。

我们以一个例子来看看`synchronized`如何引起死锁的。
> 这儿线程a持有fork1的锁，等待fork2的锁；线程b持有fork2的锁，等待fork1的锁。两个线程谁也获取不到第二个锁，导致出现互相等待的情况，也就是我们通常所说的死锁。

```java
public class DeadLock extends Thread {
    protected Object tool;
    static Object fork1 = new Object();
    static Object fork2 = new Object();
    public DeadLock(Object tool) {
        this.tool = tool;
        if (tool == fork1) {
            this.setName("哲学家A");
        }
        if (tool == fork2) {
            this.setName("哲学家B");
        }
    }
    @Override public void run() {
        if (tool == fork1) {
            synchronized (fork1) {
                try {
                    Thread.sleep(500L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (fork2) {
                    System.out.println("哲学家A开始吃饭了");
                }
            }
        }
        if (tool == fork2) {
            synchronized (fork2) {
                try {
                    Thread.sleep(500L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (fork1) {
                    System.out.println("哲学家B开始吃饭了");
                }
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        DeadLock a = new DeadLock(fork1);
        DeadLock b = new DeadLock(fork2);
        a.start();
        b.start();
        Thread.sleep(1000L);
    }
}
```

## 重入锁 - ReentrantLock

`ReentrantLock`是对`Lock`接口的实现。那么我们首先看看Lock接口里面有哪些方法，分别是做什么用的。

```java
// Lock接口中各个方法的说明
public interface Lock {
	// 阻塞方法，获取锁
    void lock();
	// 阻塞方法，可中断的获取锁
    void lockInterruptibly() throws InterruptedException;
	// 非阻塞方法，尝试获取锁。获取到了返回true，否则返回false
    boolean tryLock();
	// 非阻塞方法，带超时时间的尝试获取锁。在指定时间内获取到了返回true，否则返回false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
	// 释放锁
    void unlock();
	// 实例化Condition，用于线程间通信
    Condition newCondition();
}
```

锁的标准实现分为`ReentrantLock`和`ReadWriteLock`，本小节只讲`ReentrantLock`。那么我们写一个例子来看看`ReentrantLock`是怎么使用的。

```java
/**
 * 1. 进入了多少次锁，则需要退出多少次锁，次数必须相同。
 * 2. 如果进入的次数比退出的次数多，则会产生死锁
 * 3. 如果进入的次数比退出的次数少，则会出现异常java.lang.IllegalMonitorStateException
 * 4. unlock()的调用必须放在finally中，以便保证锁的退出肯定会执行
 */
public class ReenterLock implements Runnable {

    public static ReentrantLock lock = new ReentrantLock();
    public static int i = 0;

    public void run() {
        for (int j = 0; j < 1000000; j++) {
            lock.lock();
            lock.lock();
            try {
                i++;
            } finally {
                lock.unlock();
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ReenterLock rl = new ReenterLock();
        Thread t1 = new Thread(rl);
        Thread t2 = new Thread(rl);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```

`ReentrantLock`为什么不直接叫锁，而要叫重入锁？从上面的demo，我们看得出在一个线程里面可以反复进入多次（`lock()`调用多次，对应的`unlock()`也需要和`lock()`的个数相同），这其实就是叫重入锁的原因了。

`Lock`接口除了`lock()`和`unlock()`之外，还有一些额外方法，比如：`lockInterruptibly`、`tryLock`。他们又有什么用处呢？

### 可中断特性 - lockInterruptibly

先说说`lockInterruptibly()`，字面意思为获取锁的时候可以被中断。其实这就是一种解决死锁的方式，通过对线程执行中断操作来释放锁。我们看一个例子。

```java
public class LockInterruptiblyDemo {

    private static ReentrantLock lock1 = new ReentrantLock();
    private static ReentrantLock lock2 = new ReentrantLock();

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            try {
                lock1.lockInterruptibly();
                Thread.sleep(1000L);
                lock2.lockInterruptibly();
            } catch (InterruptedException e) {
                System.out.println("t1 lock1 interrupted");
            } finally {
                lock1.unlock();
                lock2.unlock();
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            try {
                lock2.lockInterruptibly();
                Thread.sleep(1000L);
                lock1.lockInterruptibly();
            } catch (InterruptedException e) {
                System.out.println("t2 lock2 interrupted");
            } finally {
                lock2.unlock();
                if (lock1.isHeldByCurrentThread()) {
                    lock1.unlock();
                }
            }
        }, "t2");

        t1.start();
        t2.start();
        Thread.sleep(2000L);
        t2.interrupt();
    }
}
```

上面的例子如果我们将main中的`t2.interrupt();`去掉，那么我们通过jps和jstack就能发现t1和t2产生了死锁。而在产生死锁的线程中，我们使用了`lockInterruptibly`，也就是可中断的获取锁。在外部对死锁线程中断的时候，锁就被释放了。

### 非阻塞特性 - tryLock

tryLock，就是尝试获取锁，如果没有获取到，则返回false，获取到了则返回true。我们看一个例子。

```java
public class TryLock implements Runnable {

    public static ReentrantLock lock1 = new ReentrantLock();
    public static ReentrantLock lock2 = new ReentrantLock();
    int lock;

    public TryLock(int lock) {
        this.lock = lock;
    }

    public void run() {
        if (lock == 1) {
            while (true) {
                if (lock1.tryLock()) {
                    try {
                        try {
                            Thread.sleep(500L);
                        } catch (InterruptedException e) {
                        }
                        if (lock2.tryLock()) {
                            try {
                                System.out.println(Thread.currentThread().getId() + ":My Job done");
                                return;
                            } finally {
                                lock2.unlock();
                            }
                        }
                    } finally {
                        lock1.unlock();
                    }
                }
            }
        } else {
            while (true) {
                if (lock2.tryLock()) {
                    try {
                        try {
                            Thread.sleep(500L);
                        } catch (InterruptedException e) {
                        }
                        if (lock1.tryLock()) {
                            try {
                                System.out.println(Thread.currentThread().getId() + ":My Job done");
                                return;
                            } finally {
                                lock1.unlock();
                            }
                        }
                    } finally {
                        lock2.unlock();
                    }
                }
            }
        }
    }

    public static void main(String[] args) {
        TryLock r1 = new TryLock(1);
        TryLock r2 = new TryLock(2);
        Thread t1 = new Thread(r1);
        Thread t2 = new Thread(r2);
        t1.start();
        t2.start();
    }
}
```

这段代码因为死锁可能会执行很久，但是最终会将死锁解决掉，为什么呢？因为`tryLock()`是一个非阻塞的方法，它是通过返回值来告诉当前线程是否获取锁成功，我们可以通过返回结果来做相应的处理。

### 公平锁和非公平锁

+ 什么是公平锁呢？公平锁保证锁的获取是公平的，按照每个线程获取锁的先后顺序来获取锁。
+ 什么是非公平锁呢？非公平锁不像公平锁，它不保证每个线程获取锁的机会必须相同，而是由cpu统一处理。
+ `synchronized`获取的锁只能是非公平的。而重入锁却是可以在构造锁的时候通过传入参数的方式来标记是否公平 - `public ReentrantLock(boolean fair)`。
+ 公平锁是公平的，不会产生饥饿现象，但是性能比非公平锁低，非公平锁比它效率高90+倍。不过，并非每种场景都追求性能，有一些需求场景确实需要使用公平锁来满足。

我们写一个公平锁的demo，看看其效果。

```java
public class FairLock implements Runnable {

    public static ReentrantLock fairLock = new ReentrantLock(true);

    public void run() {
        while (true) {
            try {
                fairLock.lock();
                System.out.println(Thread.currentThread().getName() + " get lock");
            } finally {
                fairLock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        FairLock fl = new FairLock();
        Thread t1 = new Thread(fl, "thread1");
        Thread t2 = new Thread(fl, "thread2");
        t1.start();
        t2.start();
    }
}
```

运行发现，打印的结果是非常均匀的，t1和t2是交叉执行的。结果如下：

```
thread1 get lock
thread2 get lock
thread1 get lock
thread2 get lock
thread1 get lock
thread2 get lock
thread1 get lock
thread2 get lock
thread1 get lock
thread2 get lock
```

### 重入锁相比synchronized，有哪些优势

1. 重入锁可被中断
2. 非阻塞性获取锁或超时等待
3. 支持公平锁

### 重入锁伴生的Condition

synchronized可以和`wait/notify`配合实现线程间通信。`ReentrantLock`也可以和`Condition`配合达到一样的效果，甚至可以实现更多的功能。

Condition怎么获取呢？其实是可以通过`Condition newCondition();`来获取。我们再来看看Condition有哪些方法。

```java
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```

看到这些方法，我们似乎有一种熟悉的感觉。是的，你没猜错，他们是和`Object的wait/notify`相对应，`await对应wait`，`signal对应notify`。但是它能提供更强大的功能，比如忽略中断的`awaitUninterruptibly`、等待直到某个时间的`awaitUntil`，这些功能在Object的wait/notify里面可没有。

和`wait/notify`的使用相似，`await/signal`的使用也必须要先获取到锁，否则会抛异常`IllegalMonitorStateException`。

## 读写锁 - ReadWriteLock

读写锁是分离锁的一种实现方式，写的时候才加锁，读的时候不加锁，那么如果先写后读呢？我们总结一下：

1. 写写 - 加锁
2. 读写 - 加锁
4. 读读 - 不加锁

我们写一个例子看看。

```java
public class ReadWriteLockDemo {

    private static Lock lock = new ReentrantLock();
    private static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private static Lock readLock = readWriteLock.readLock();
    private static Lock writeLock = readWriteLock.writeLock();
    private int value;

    public Object handleRead(Lock lock) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000);
            return value;
        } finally {
            lock.unlock();
        }
    }

    public void handleWrite(Lock lock, int index) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000L);
            value = index;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        final ReadWriteLockDemo demo = new ReadWriteLockDemo();
        Runnable readRunnable = new Runnable() {
            public void run() {
                try {
                    demo.handleRead(readLock);
//                    demo.handleRead(lock);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        Runnable writeRunnable = new Runnable() {
            public void run() {
                try {
                    demo.handleWrite(writeLock, new Random().nextInt());
//                    demo.handleWrite(lock, new Random().nextInt());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        for (int i = 0; i < 18; i++) {
            new Thread(readRunnable).start();
        }

        for (int i = 18; i < 20; i++) {
            new Thread((writeRunnable)).toString();
        }
    }
}
```

这个例子，我们又如何测试读写锁呢？其实比较简单。

1. 如果写的时候加锁，读的时候不加锁，那么最终执行的时间肯定是写锁占用的时间，也就是2秒。
2. 如果不是我们期望的，则会总共执行20秒。

假如我们使用注释掉的代码代替注释前的代码，我们看到总共执行了20秒。

## 总结

最后，我们再来总结一下本文所说的几种锁，即：jdk世界自带的三把锁-synchronized、重入锁和读写锁。

1. 重入锁完全可以代替synchronized，且带来了更多更强大的功能。比如：中断响应、超时等待、公平锁等。重入锁配合Condition来实现线程通信，能提供比wait/notify更强大的功能支持，比如忽略中断的awaitUninterruptibly，比如awaitUntil。
2. 读写锁更是将读和写进行了锁分离，在读多写少的场景能极大地提高程序的性能。
3. 另外，能不用synchronized，尽量不用。在非用锁不可的场景下，也尽可能多地考虑重入锁和读写锁。