---
layout: post
title:  spring async中传递线程上下文变量
date:   2017-09-13 06:00:23 +0800
categories: Java并发和多线程
tag: java
---

* content
{:toc}

默认的`@Async`注释处理，并不会对调用者线程的`线程上下文`做处理，有时为了更方便地定位`流式日志`，我们需要补充这种处理。
在spring的线程任务执行器上进行扩充，我们可以实现拦截处理。

## 实现类

```java
package com.learn.spring.async.config.async;

import org.slf4j.MDC;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.util.concurrent.ListenableFuture;

import java.util.Map;
import java.util.concurrent.Callable;
import java.util.concurrent.Future;

/**
 * @author zhangfb
 */
public class ContextAwareExecutorDecorator extends ThreadPoolTaskExecutor {

    @Override
    public void execute(Runnable command) {
        super.execute(decorateContextAware(command));
    }

    @Override
    public void execute(Runnable task, long startTimeout) {
        super.execute(decorateContextAware(task), startTimeout);
    }

    @Override
    public Future<?> submit(Runnable task) {
        return super.submit(decorateContextAware(task));
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        return super.submit(decorateContextAware(task));
    }

    @Override
    public ListenableFuture<?> submitListenable(Runnable task) {
        return super.submitListenable(decorateContextAware(task));
    }

    @Override
    public <T> ListenableFuture<T> submitListenable(Callable<T> task) {
        return super.submitListenable(decorateContextAware(task));
    }

    private Runnable decorateContextAware(Runnable command) {
        final Map<String, String> originalContextCopy = MDC.getCopyOfContextMap();

        return () -> {
            // copy the current context
            final Map<String, String> localContextCopy = MDC.getCopyOfContextMap();

            // add codes with decorate model
            before(originalContextCopy);
            command.run();
            after(localContextCopy);
        };
    }

    private <T> Callable<T> decorateContextAware(Callable<T> command) {
        final Map<String, String> originalContextCopy = MDC.getCopyOfContextMap();

        return () -> {
            // copy the current context
            final Map<String, String> localContextCopy = MDC.getCopyOfContextMap();

            // add codes with decorate model
            before(originalContextCopy);
            T ret = command.call();
            after(localContextCopy);

            return ret;
        };
    }

    private void before(final Map<String, String> originalContextCopy) {
        MDC.clear();
        if (originalContextCopy != null) {
            // set the desired context that was present at point of calling execute
            MDC.setContextMap(originalContextCopy);
        }
    }

    private void after(final Map<String, String> localContextCopy) {
        MDC.clear();
        if (localContextCopy != null) {
            // reset the context
            MDC.setContextMap(localContextCopy);
        }
    }
}
```

## 配置类

```java
package com.learn.spring.async.config.async;

import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.AsyncConfigurerSupport;
import org.springframework.scheduling.annotation.EnableAsync;

import java.util.concurrent.Executor;

/**
 * @author zhangfb
 */
@Configuration
@EnableAsync
public class AsyncConfig extends AsyncConfigurerSupport {

    @Override
    public Executor getAsyncExecutor() {
        ContextAwareExecutorDecorator decorator = new ContextAwareExecutorDecorator();
        decorator.setCorePoolSize(7);
        decorator.setCorePoolSize(7);
        decorator.setMaxPoolSize(42);
        decorator.setQueueCapacity(11);
        decorator.setThreadNamePrefix("user_def_thread_");
        decorator.initialize();
        return decorator;
    }
}
```