---
layout: post
title:  深入理解Tomcat（七）Server和Service
date:   2018-11-21 22:55:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

回顾[【深入理解Tomcat（三）架构及组件】](https://www.jianshu.com/p/2b6359daf5c8)，同时参考tomcat自带的`server.xml`，我们发现在连接器(Connector)和容器(Container)外层主要是`Server`和`Service`。基于由外而内、由浅入深的方式，本文先来分析`Server`和`Service`。

这儿，我们在[【深入理解Tomcat（三）架构及组件】](https://www.jianshu.com/p/2b6359daf5c8)之上增加一张更为完整的整体结构图！

![整体结构图](https://upload-images.jianshu.io/upload_images/845143-b348216a4be35a7d.png)

在[【深入理解Tomcat（二）Lifecycle】](https://www.jianshu.com/p/2a9ffbd00724)中，我们分析了Lifecycle，也知道了每个tomcat组件都有`init()`、`start()`等生命周期方法。因为Lifecycle是基于模板方法模式设计的，`init()`会调用`initInternal()`，而`start()`会调用`startInternal()`。为了降低分析源码的复杂性，我们在对每个组件进行分析的时候，只会关心生命周期接口下的模板方法，而不会重复分析其下的非模板方法！

## Server
 
1. `Catalina.load()`里面调用了`Server.init()`方法。
2. `Catalina.start()`里面调用了`Server.start()`方法。

所以我们从`Server`的`init()`和`start()`着手分析。

在分析之前，我们先看看Server有哪些方法：

```java
/**
 * A <code>Server</code> element represents the entire Catalina
 * servlet container.  Its attributes represent the characteristics of
 * the servlet container as a whole.  A <code>Server</code> may contain
 * one or more <code>Services</code>, and the top level set of naming
 * resources.
 * <p>
 * Normally, an implementation of this interface will also implement
 * <code>Lifecycle</code>, such that when the <code>start()</code> and
 * <code>stop()</code> methods are called, all of the defined
 * <code>Services</code> are also started or stopped.
 * <p>
 * In between, the implementation must open a server socket on the port number
 * specified by the <code>port</code> property.  When a connection is accepted,
 * the first line is read and compared with the specified shutdown command.
 * If the command matches, shutdown of the server is initiated.
 * <p>
 * <strong>NOTE</strong> - The concrete implementation of this class should
 * register the (singleton) instance with the <code>ServerFactory</code>
 * class in its constructor(s).
 *
 * @author Craig R. McClanahan
 */
public interface Server extends Lifecycle {

    // ------------------------------------------------------------- Properties

    public NamingResourcesImpl getGlobalNamingResources();
    public void setGlobalNamingResources
        (NamingResourcesImpl globalNamingResources);
    public javax.naming.Context getGlobalNamingContext();
    public int getPort();
    public void setPort(int port);
    public String getAddress();
    public void setAddress(String address);
    public String getShutdown();
    public void setShutdown(String shutdown);
    public ClassLoader getParentClassLoader();
    public void setParentClassLoader(ClassLoader parent);
    public Catalina getCatalina();
    public void setCatalina(Catalina catalina);
    public File getCatalinaBase();
    public void setCatalinaBase(File catalinaBase);
    public File getCatalinaHome();
    public void setCatalinaHome(File catalinaHome);


    // --------------------------------------------------------- Public Methods

    public void addService(Service service);
    public void await();
    public Service findService(String name);
    public Service[] findServices();
    public void removeService(Service service);
    public Object getNamingToken();
}
```

除了`Server`本身包含的方法`addService(Service service)`和`Service[] findServices()`，我们额外阅读并翻译一下Server的javadoc注释。

> 1. 一个`Server`节点代表整个Catalina servlet容器。它的属性代表这个servlet容器的所有特征。一个`Server`可以包含1个或多个`Service`，以及顶级名字资源。
> 2. 正常情况下，该接口的实现类也必须实现`Lifecycle接口`，因此当`start()`和`stop()`方法被调用的时候，所有定义在其下面每个`Service`的`start()`和`stop()`也必须相应地得到调用。
> 3. 实现类必须在指定的端口号打开一个`server socket`(服务器套接字)。当一个连接被接受，且第一行读取之后，需要比较`特殊的shutdown命令`，如果命令匹配成功，则Server的shutdown操作将被发起。

`Server`的实现类为`StandardServer`，我们分析一下`StandardServer.initInternal()`方法。该方法用于对`Server`进行初始化，关键的地方就是代码最后对services的循环操作，对每个service调用init方法。
【注】：这儿我们只粘贴出这部分代码。

```java
@Override
protected void initInternal() throws LifecycleException {
    super.initInternal();

    // Initialize our defined Services
    for (int i = 0; i < services.length; i++) {
        services[i].init();
    }
}
```

初始化完成之后接着会调用`start()`方法，还是很简单，其实就是调用每个service的`start()`方法。

```java
@Override
protected void startInternal() throws LifecycleException {

    fireLifecycleEvent(CONFIGURE_START_EVENT, null);
    setState(LifecycleState.STARTING);

    globalNamingResources.start();

    // Start our defined Services
    synchronized (servicesLock) {
        for (int i = 0; i < services.length; i++) {
            services[i].start();
        }
    }
}
```

## Service

既然一个Server可以包含多个Service，对一个`Server`的`init()`和`start()`操作，其实就是对`Server`下的每个`Service`进行`init()`和`start()`操作。所以，我们接下来要分析的必然是`Service`。因为`Service`的实现类为`StandardService`，所以我们就分析这个实现类。

我们先来看看`init()`方法。

```java
@Override
protected void initInternal() throws LifecycleException {
    super.initInternal();

    if (engine != null) {
        engine.init();
    }

    // Initialize any Executors
    for (Executor executor : findExecutors()) {
        if (executor instanceof JmxEnabled) {
            ((JmxEnabled) executor).setDomain(getDomain());
        }
        executor.init();
    }

    // Initialize mapper listener
    mapperListener.init();

    // Initialize our defined Connectors
    synchronized (connectorsLock) {
        for (Connector connector : connectors) {
            try {
                connector.init();
            } catch (Exception e) {
                String message = sm.getString(
                        "standardService.connector.initFailed", connector);
                log.error(message, e);

                if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"))
                    throw new LifecycleException(message);
            }
        }
    }
}
```

`Service`的`init()`操作主要做了几件事情：

1. Engine的初始化操作
2. Service线程执行器（tomcat中对`java.util.concurrent.Executor`的实现）的初始化操作
3. MapperListener的初始化操作
4. 对Connectors（可能有多个）的初始化操作

【注】：Engine、MapperListener、Mapper和Connector我们会在后面章节详细分析，本节不做深入讨论。我们这儿只需要知道Mapper的作用，主要是通过请求url快速和精确地找到相应的Wrapper。

接下来我们看看`Service`的`start()`方法

```java
@Override
protected void startInternal() throws LifecycleException {
    if(log.isInfoEnabled())
        log.info(sm.getString("standardService.start.name", this.name));
    setState(LifecycleState.STARTING);

    // Start our defined Container first
    if (engine != null) {
        synchronized (engine) {
            engine.start();
        }
    }

    synchronized (executors) {
        for (Executor executor: executors) {
            executor.start();
        }
    }

    mapperListener.start();

    // Start our defined Connectors second
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            try {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            } catch (Exception e) {
                log.error(sm.getString(
                        "standardService.connector.startFailed",
                        connector), e);
            }
        }
    }
}
```

和`Service`的`init()`类似，分别调用了Engine、Executors、MapperListener和Connectors的`start()`方法。

## 总结

本节，我们重新详细全面地拆分了Tomcat组件的整体架构图，知道了组件的包含关系和数量映射关系。其关系如下：

1. 1个Catalina包含1个Server
2. 1个Server包含多个Service
3. 每个Service对应有下面的组件
    1. 1个Engine
    2. 多个Connector
    3. 一个MapperListener