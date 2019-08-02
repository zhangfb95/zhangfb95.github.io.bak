---
layout: post
title:  深入理解Tomcat（11）拾遗-关闭钩子
date:   2018-11-29 22:37:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

Tomcat的功能是由一个个的组件堆砌和架构出来的。几乎每个组件都是`生命周期组件`。何为`生命周期组件`呢？就是可以人为地对组件实施生命周期方法。那么生命周期方法又有哪些呢？我们查看生命周期接口`Lifecycle`得知以下方法。

1. `init()` // 初始化
2. `start()` // 启动
3. `stop()` // 停止
4. `destroy()` // 销毁

通过这些方法，我们可以掌控组件的生与死。`init()`和`start()`在启动的时候调用，而`stop()`和`destroy()`在停止的时候调用，他们互为`逆操作`。我们在分析tomcat的各个组件的时候，更多地关注了启动过程，却往往忽略了停止过程。

那么tomcat是通过哪种机制来停止组件的呢？这么多组件又是按照怎样的顺序来停止的呢？这就是本章需要关注的技术点--`关闭钩子`。

## jdk中的关闭钩子

在java进程运行的过程中，我们开启了很多资源，而这些资源在java进程停止的时候必须得到清理和释放，以避免资源的浪费。那么，我们对java进程进行停止操作（比如`kill -15`），java进程是否有一种机制来感知到，然后做一些处理呢？

> 答案是可以~！

我们可以java进程启动完成之前，通过`Runtime.getRuntime().addShutdownHook(shutdownHook);`方法注册一个`关闭钩子`。`关闭钩子`是一个特殊的`线程`，可以人为编码实现。java进程在停止之前，会调用已注册`关闭钩子`的`run()`方法。

我们写一个例子来看看效果。

```java
public class ShutdownHookDemo {

    public static void main(String[] args) throws Exception {
        System.out.println("process is running");

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("process is down. now, you can close the related resources.");
        }));

        Thread.sleep(Long.MAX_VALUE);
    }
}
```

当我们停止进程之后，得到如下输出。

![停止进程效果图](https://upload-images.jianshu.io/upload_images/845143-7b9df9d9fd459b6d.png?jianshufrom=true)

由此可以验证，java自带的`关闭钩子`是可以生效的。

> 【警告】例外的情况就是，当我们对java进程执行`kill -9`命令的时候，`关闭钩子`并不会被执行，这一点请大家一定注意！

## tomcat中关闭钩子的运用

tomcat中`关闭钩子`是在`Catalina.start()`方法中声明的，我们来看一看。

```java
public void start() {
    if (getServer() == null) {
        load();
    }

    if (getServer() == null) {
        log.fatal("Cannot start server. Server instance is not configured.");
        return;
    }

    long t1 = System.nanoTime();

    // Start the new server
    try {
        getServer().start();
    } catch (LifecycleException e) {
        log.fatal(sm.getString("catalina.serverStartFail"), e);
        try {
            getServer().destroy();
        } catch (LifecycleException e1) {
            log.debug("destroy() failed for failed Server ", e1);
        }
        return;
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {
        log.info("Server startup in " + ((t2 - t1) / 1000000) + " ms");
    }

    // Register shutdown hook
    // 注册关闭钩子
    if (useShutdownHook) {
        if (shutdownHook == null) {
            shutdownHook = new CatalinaShutdownHook();
        }
        Runtime.getRuntime().addShutdownHook(shutdownHook);

        // If JULI is being used, disable JULI's shutdown hook since
        // shutdown hooks run in parallel and log messages may be lost
        // if JULI's hook completes before the CatalinaShutdownHook()
        LogManager logManager = LogManager.getLogManager();
        if (logManager instanceof ClassLoaderLogManager) {
            ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                    false);
        }
    }

    if (await) {
        await();
        stop();
    }
}
```

关键代码就三行，为了突出重点，我们单独将这3行代码粘贴出来看看，是不是和我们前面写的例子很相似？

```java
if (shutdownHook == null) {
    shutdownHook = new CatalinaShutdownHook();
}
Runtime.getRuntime().addShutdownHook(shutdownHook);
```

接下来我们分析一下`CatalinaShutdownHook`，关键代码就一行`Catalina.this.stop();`，停止Catalina框架，代码也是非常的简单和简洁。

```java
protected class CatalinaShutdownHook extends Thread {

    @Override
    public void run() {
        try {
            if (getServer() != null) {
                Catalina.this.stop();
            }
        } catch (Throwable ex) {
            ExceptionUtils.handleThrowable(ex);
            log.error(sm.getString("catalina.shutdownHookFail"), ex);
        } finally {
            // If JULI is used, shut JULI down *after* the server shuts down
            // so log messages aren't lost
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).shutdown();
            }
        }
    }
}
```

## 总结

1. 首先，我们抛出了为什么要有`关闭钩子`，是因为我们需要在java进程关闭的时候进行一些资源的清理和释放操作
2. 其次，我们自行编写了一个`关闭钩子`的例子来看看其用法
3. 最后，我们通过分析tomcat中`关闭钩子`的源码，加深了我们队`关闭钩子`的理解