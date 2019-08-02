---
layout: post
title:  wait/notify详解
date:   2018-10-26 22:53:00 +0800
categories: 深入理解Java并发
tag: 深入理解Java并发
---

* content
{:toc}

## 前言

今天我们来分析一下Object的`wait/notify`方法。虽然明面上是两个方法，但是通过查看Object源码，我们发现`wait/notify`相关的方法其实一共有5个。而Object总共才12个方法，由此可以看出其重要性。先来看看jdk中的源码。

```java
public final native void notify();
public final native void notifyAll();
public final native void wait(long timeout) throws InterruptedException;
public final void wait(long timeout, int nanos) throws InterruptedException {
	if (timeout < 0) { throw new IllegalArgumentException("timeout value is negative"); }
	if (nanos < 0 || nanos > 999999) { throw new IllegalArgumentException("nanosecond timeout value out of range");}
	if (nanos >= 500000 || (nanos != 0 && timeout == 0)) {
		timeout++;
	}
	wait(timeout);
}
public final void wait() throws InterruptedException {
	wait(0);
}
```

其中有3个native方法（用c语言实现），另外两个wait方法也是间接调用了`wait(long timeout)`方法。

1. `wait(long timeout, int nanos)`，该方法其实并非真正能精确到纳秒，而是通过纳秒四舍五入成微秒的方式来代替
2. `wait(0)`，方法参数0，是无限等待的意思，如果没有notify，那么就会一直等待下去

既然`wait/notify`这么重要，它又是拿来做什么呢？其实我们可以使用它们来`实现线程间通信`。那么什么又是线程通信呢？线程通信的英文名为`Thread Signaling`，即：线程间互相发送信号的意思，通俗地将就是一个线程等待其他线程发送信号。

假如没有`wait/notify`，我们怎么实现类似`发布订阅功能`呢？自然而然地我们想到使用状态标记的方式。伪代码如下：

```java
// 生产者
void produce() {
	修改条件使其满足
	doSomething()
}

// 消费者
void consume() {
	while (条件不满足) {
		Thread.sleep(1000L)
	}
	doSomeThing();
}
```

这种方式存在一些问题

1. 因为使用了`sleep()`让线程休眠，所以处理可能存在`不及时`的情况
2. 如果`sleep()`休眠时间过短，cpu的消耗就会过多

这两种情况是相互矛盾的，一个问题缓解了，另外一个必然就会加重。所以java自带的`wait/notify`真是非常的及时和实用。

## 如何使用

我们写一段demo来看看。

```java
public class WaitNotify {

    private final static Object lock = new Object();

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override public void run() {
                System.out.println("线程 A 等待拿锁");
                synchronized (lock) {
                    try {
                        System.out.println("线程 A 拿到锁了");
                        TimeUnit.SECONDS.sleep(1L);
                        System.out.println("线程 A 开始等待并放弃锁");
                        lock.wait();
                        System.out.println("被通知可以继续执行 则 继续运行至结束");
                    } catch (InterruptedException e) {
                    }
                }
            }
        }, "线程 A").start();

        try {
            TimeUnit.SECONDS.sleep(1L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(new Runnable() {
            @Override public void run() {
                System.out.println("线程 B 等待锁");
                synchronized (lock) {
                    System.out.println("线程 B 拿到锁了");
                    try {
                        TimeUnit.SECONDS.sleep(3L);
                    } catch (InterruptedException e) {
                    }
                    lock.notify();
                    System.out.println("线程 B 随机通知 Lock 对象的某个线程");
                }
            }
        }, "线程 B").start();
    }
}
```

执行结果如下

```
线程 A 等待拿锁
线程 A 拿到锁了
线程 A 开始等待并放弃锁
线程 B 等待锁
线程 B 拿到锁了
线程 B 随机通知 Lock 对象的某个线程
被通知可以继续执行 则 继续运行至结束
```

我们分析一下代码

1. 线程A和B会抢占lock对象的锁
2. 因为main主线程有`sleep()`休眠1秒的操作，所以线程A会先拿到锁。
3. 线程A执行到`lock.wait()`时，会释放锁，并将当前线程状态设置为`WAITING`状态
4. 线程B获得锁，当执行到`lock.notify()`时，会通知所有等待队列中的线程，然后继续执行`lock.notify()`之后的代码，执行完成之后释放锁
5. 锁被释放后，等待队列中的线程会抢占锁，抢到锁的线程A会把线程状态变更为`RUNNABLE`，然后继续执行`wait()`之后的代码

> 这儿需要特别注意，B线程的`notify()`执行之后，会唤醒A线程，但是A线程并不会马上执行（因为锁还未释放），A线程`wait()`方法后面的逻辑仍然需要先拿到锁才会执行！

## `wait()/notify()`为什么必须在synchroized中

表面上，如果`wait()`和`notify()`不放在synchroized中，执行时会抛出异常`IllegalMonitorStateException`，即：非法的监视器状态异常。

那么为什么会是这样的呢？主要原因还是因为存在`竟态条件`！这儿先科普一下，什么是`竟态条件`？

> 当两个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。

`wait()`的调用必然是在等待某种条件的满足，而条件的满足又是另外一个线程在更改条件之后进行了`notify()`通知，所以我们期望达到如下的效果：

```java
// 生产者B
void produce() {
	修改条件使其满足
	notify();
	// do something
}

// 消费者A
void consume() {
	while (条件不满足) {
		wait();
	}
	// do something
}
```

如果不加`synchronized`，可能出现下面的情况

1. A进入`while循环`后，但还没有执行`wait()`方法
2. B更新了条件，通过`notify()`唤醒等待队列中的线程。这时因为A还没有执行到`wait()`方法，所以A不会被唤醒。
3. A执行到`wait()`了，如果除了A之后没有其他线程的`notify()`通知到A，那么A会一直保持在`WAITING`状态而得不到后续的执行

那么如果我们加上`synchronized`后，变成下面这种情况，又可不可以呢？

```java
// 生产者B
void produce() {
	synchronized(obj_a) {
		修改条件使其满足
		obj_b.notify();
		// do something
	}
}

// 消费者A
void consume() {
	synchronized(obj_a) {
		while (条件不满足) {
			obj_b.wait();
		}
		// do something
	}
}
```

表面上看貌似可以，实际上却是行不通的。因为`wait()`在线程状态变更之前必然会释放锁，而sychronized可以嵌套包含，这时`wait()`就不知道应该释放哪个锁了。所以最理所当然地就是把`wait()`所属对象当做释放的锁，所以`JUC`的作者就使用了这样一种很直观，又很合理的方式。

最终的呈现如下：

```java
// 生产者B
void produce() {
	synchronized(obj_a) {
		修改条件使其满足
		obj_a.notify();
		// do something
	}
}

// 消费者A
void consume() {
	synchronized(obj_a) {
		while (条件不满足) {
			obj_a.wait();
		}
		// do something
	}
}
```

## 实现原理

我们来看看，使用这两个方法的顺序一般是什么样的？

1. 使用`wait()`、`notify()`、`notifyAll()`时，必须先对调用对象加锁
2. 调用`wait()`方法后，线程状态由`RUNNABLE`变更为`WAITING`，并将当前线程放置到对象的等待队列
3. `notify()`或`notifyAll()`调用后，等待线程依然不会从`wait()`返回，需要调用`notify()`或`notifyAll()`的线程释放锁之后，等待线程才有机会从`wait()`返回
4. `notify()`方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而`notifyAll()`则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由`WAITING`变为`BLOCKED`
5. 从`wait()`方法返回的前提是获得了被调用对象的锁

从上述细节可以看到，等待/通知机制依托于同步机制，其目的就是确保等待线程从`wait()`方法返回后能够感知到通知线程对变量做出的修改。

该图描述了上面的步骤：

![image.png](https://upload-images.jianshu.io/upload_images/845143-37cc7bdab827feda.png?jianshufrom=1)


## 总结

至此，`wait()`和`notify()`的讲解就告一段落了，如果还需要了解更底层的原理，必须去看jdk的源码。jdk的源码是非常复杂的，不要轻易去看。里面非常多的细节和扩展，往往会掩盖抽象和原理。不过为了兴趣，也为了自己在java领域更加的深入，不管jdk源码多么多么得复杂，我迟早要介入了解的。

## 参考链接

1. [http://thinkinjava.cn/2018/01/并发编程之-wait-notify-方法剖析](http://thinkinjava.cn/2018/01/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B9%8B-wait-notify-%E6%96%B9%E6%B3%95%E5%89%96%E6%9E%90/)
2. [https://blog.csdn.net/lengxiao1993/article/details/52296220](https://blog.csdn.net/lengxiao1993/article/details/52296220)
3. [https://docs.oracle.com/javase/tutorial/essential/concurrency/guardmeth.html](https://docs.oracle.com/javase/tutorial/essential/concurrency/guardmeth.html)
4. [http://www.xyzws.com/javafaq/why-wait-notify-notifyall-must-be-called-inside-a-synchronized-method-block/127](http://www.xyzws.com/javafaq/why-wait-notify-notifyall-must-be-called-inside-a-synchronized-method-block/127)
5. [https://stackoverflow.com/questions/2779484/why-must-wait-always-be-in-synchronized-block](https://stackoverflow.com/questions/2779484/why-must-wait-always-be-in-synchronized-block)
6. [http://ifeve.com/thread-signaling/](http://ifeve.com/thread-signaling/)