---
layout: post
title:  深入理解Tomcat（八）Container
date:   2018-11-22 19:14:00 +0800
categories: Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

在Tomcat中，容器（Container）主要包括四种，Engine、Host、Context和Wrapper。在详细分析tomcat源码之前，我们先来看看容器的类继承层级是怎样的。

![容器类继承层次图](https://upload-images.jianshu.io/upload_images/845143-65ec2c102b86affe.png)

这儿，我们先说明一下容器的包含关系。Engine包含多个Host，Host包含多个Context，Context包含多个Wrapper，每个Wrapper对应一个Servlet。我们如何来理解各个容器呢？Tomcat又为什么要把容器分成这4种呢？

1. Engine，我们可以看成是容器对外提供功能的入口，每个Engine是Host的集合，用于管理各个Host。
2. Host，我们可以看成`虚拟主机`，一个tomcat可以支持多个虚拟主机。
3. Context，又叫做上下文容器，我们可以看成`应用服务`，每个Host里面可以运行多个应用服务。同一个Host里面不同的Context，其contextPath必须不同，默认Context的contextPath为空格("")或斜杠(/)。
4. Wrapper，是Servlet的抽象和包装，每个Context可以有多个Wrapper，用于支持不同的Servlet。另外，每个JSP其实也是一个个的Servlet。

### 什么是虚拟主机

http协议从1.1开始，支持在请求头里面添加Host字段用来表示请求的域名。DNS域名解析的时候，可以将不同的域名解析到同一个ip或者主机。

假如我们需要在一个tomcat里面同时支持三个域名：

+ http://www.ramki.com
+ http://www.krishnan.com
+ http://www.blog.ramki.com

其轮廓图如下所示：

![一个tomcat多个域名](https://upload-images.jianshu.io/upload_images/845143-51594604c13bf484.png)

要想在tomcat里面支持多域名，我们需要在`server.xml`文件里面的`Engine标签`下面添加多个`Host标签`，如下所示：

```xml
<Host name="www.ramki.com" appbase="ramki_webapps" />
<Host name="www.krishnan.com" appbase="krishnan_webapps" /> 
<Host name="www.blog.ramki.com" appbase="blog_webapps" /> 
```

其中name表示域名，appbase表示虚拟主机的目录。
当我们在浏览器输入`http://www.ramki.com `之后，相应域名将请求到tomcat。tomcat通过读取并搜索`server.xml`，找到www.ramki.com对应的虚拟主机Host，然后就使用查找到的Host来处理请求。

![请求流程](https://upload-images.jianshu.io/upload_images/845143-1c86297ff31ba8b7.png)

在浏览器请求的时候，请求头信息如下，这儿我们重点关注Host header。

```
GET / HTTP/1.1 
Host: www.ramki.com 
Proxy-Connection: keep-alive 
User-Agent: Mozilla/5.0 (Windows NT 6.2) AppleWebKit/535.11 (KHTML, like Gecko) Chrome/17.0.963.56 Safari/535.11 
Accept: text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Encoding: gzip,deflate,sdch 
Accept-Language: en-US,en;q=0.8 
```

### 什么是contextPath
我们在上面提到：`同一个Host里面不同的Context，其contextPath必须不同，默认Context的contextPath为空格("")或斜杠(/)`，那么我们如何理解这句话呢？

还是以例子来说明，假如我们在`www.ramki.com`映射的Host下配置3个Context子容器，其contextPath分别如下：

1. AContext为：/
2. BContext为：user
3. CContext为：manager

如果我们请求`www.ramki.com/simple`（域名后面不是user，也不是manager），Host将请求转发到AContext；
请求`www.ramki.com/user/add`，Host将请求转发到BContext；
请求`www.ramki.com/manager/view`，Host将请求转发到CContext。

### 接下来做什么

通过上面的讲述，我们知道了每个容器的作用，接下来我们要做的就是分析每个容器的源码实现。

## Engine

Engine的标准实现为`org.apache.catalina.core.StandardEngine`。我们先来看看构造函数。其主要职责为：使用默认的基础阀门创建标准Engine组件。

```java
/**
 * Create a new StandardEngine component with the default basic Valve.
 */
public StandardEngine() {
    super();
    pipeline.setBasic(new StandardEngineValve());
    /* Set the jmvRoute using the system property jvmRoute */
    try {
        setJvmRoute(System.getProperty("jvmRoute"));
    } catch(Exception ex) {
        log.warn(sm.getString("standardEngine.jvmRouteFail"));
    }
    // By default, the engine will hold the reloading thread
    backgroundProcessorDelay = 10;
}
```

接下来我们看看`StandardEngineValve`的`invoke()`方法。该方法主要是选择合适的Host，然后调用Host中pipeline的第一个`Valve的invoke()`方法。

```java
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Select the Host to be used for this Request
    Host host = request.getHost();
    if (host == null) {
        response.sendError
            (HttpServletResponse.SC_BAD_REQUEST,
             sm.getString("standardEngine.noHost",
                          request.getServerName()));
        return;
    }
    if (request.isAsyncSupported()) {
        request.setAsyncSupported(host.getPipeline().isAsyncSupported());
    }

    // Ask this Host to process this request
    host.getPipeline().getFirst().invoke(request, response);
}
```

其实Engine主要的功能就上面的两个方法。额外还有一些细小的功能，我们放在Engine拾遗里面分析。

### Engine拾遗

`addChild`只会让Host作为其子容器添加到子容器列表中，非Host在添加子容器的时候会抛异常！

```java
@Override
public void addChild(Container child) {
    if (!(child instanceof Host))
        throw new IllegalArgumentException
            (sm.getString("standardEngine.notHost"));
    super.addChild(child);
}
```

因为Engine为tomcat中的顶级容器，所以没有父容器了，当调用`setParent`时将抛出异常。

```java
@Override
public void setParent(Container container) {
    throw new IllegalArgumentException
        (sm.getString("standardEngine.notParent"));
}
```

`getParentClassLoader`和`Service`中同名的该方法一样，先判断父类加载器是否存在，存在的话直接返回。因为`Engine`肯定属于某一个`Service`，所以service不会为null，在父类加载器不存在的情况下，会获取`Service`的parentClassLoader。

```java
@Override
public ClassLoader getParentClassLoader() {
    if (parentClassLoader != null)
        return (parentClassLoader);
    if (service != null) {
        return (service.getParentClassLoader());
    }
    return (ClassLoader.getSystemClassLoader());
}
```

## Host

分析Host的时候，我们从Host的构造函数入手，该方法主要是设置基础阀门。

```java
public StandardHost() {
    super();
    pipeline.setBasic(new StandardHostValve());
}
```

接下来我们看看基础阀门`StandardHostValve`的`invoke()`方法，该方法比较长。

```java
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Select the Context to be used for this Request
    Context context = request.getContext();
    if (context == null) {
        response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
             sm.getString("standardHost.noContext"));
        return;
    }

    if (request.isAsyncSupported()) {
        request.setAsyncSupported(context.getPipeline().isAsyncSupported());
    }

    boolean asyncAtStart = request.isAsync();
    boolean asyncDispatching = request.isAsyncDispatching();

    try {
        context.bind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);

        if (!asyncAtStart && !context.fireRequestInitEvent(request.getRequest())) {
            // Don't fire listeners during async processing (the listener
            // fired for the request that called startAsync()).
            // If a request init listener throws an exception, the request
            // is aborted.
            return;
        }

        // Ask this Context to process this request. Requests that are in
        // async mode and are not being dispatched to this resource must be
        // in error and have been routed here to check for application
        // defined error pages.
        try {
            if (!asyncAtStart || asyncDispatching) {
                context.getPipeline().getFirst().invoke(request, response);
            } else {
                // Make sure this request/response is here because an error
                // report is required.
                if (!response.isErrorReportRequired()) {
                    throw new IllegalStateException(sm.getString("standardHost.asyncStateError"));
                }
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            container.getLogger().error("Exception Processing " + request.getRequestURI(), t);
            // If a new error occurred while trying to report a previous
            // error allow the original error to be reported.
            if (!response.isErrorReportRequired()) {
                request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, t);
                throwable(request, response, t);
            }
        }

        // Now that the request/response pair is back under container
        // control lift the suspension so that the error handling can
        // complete and/or the container can flush any remaining data
        response.setSuspended(false);

        Throwable t = (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);

        // Protect against NPEs if the context was destroyed during a
        // long running request.
        if (!context.getState().isAvailable()) {
            return;
        }

        // Look for (and render if found) an application level error page
        if (response.isErrorReportRequired()) {
            if (t != null) {
                throwable(request, response, t);
            } else {
                status(request, response);
            }
        }

        if (!request.isAsync() && !asyncAtStart) {
            context.fireRequestDestroyEvent(request.getRequest());
        }
    } finally {
        // Access a session (if present) to update last accessed time, based
        // on a strict interpretation of the specification
        if (ACCESS_SESSION) {
            request.getSession(false);
        }

        context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
    }
}
```

`invoke()`方法关键的地方就一行-`context.getPipeline().getFirst().invoke(request, response);`，正是调用Context 中pipeline的第一个`Valve的invoke()`方法。作用和`Engine`下基础阀门的`invoke()`调用类似。

## Context

接下来我们分析一下Context的实现`org.apache.catalina.core.StandardContext`。

先来看看构造方法，该方法用于设置`Context.pipeline`的基础阀门。

```java
public StandardContext() {
    super();
    pipeline.setBasic(new StandardContextValve());
    broadcaster = new NotificationBroadcasterSupport();
    // Set defaults
    if (!Globals.STRICT_SERVLET_COMPLIANCE) {
        // Strict servlet compliance requires all extension mapped servlets
        // to be checked against welcome files
        resourceOnlyServlets.add("jsp");
    }
}
```

接下来我们分析一下`StandardContextValve`的`invoke()`方法。在Wrapper不为null的时候，调用下面的代码：`wrapper.getPipeline().getFirst().invoke(request, response);`。

```java
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Disallow any direct access to resources under WEB-INF or META-INF
    MessageBytes requestPathMB = request.getRequestPathMB();
    if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
            || (requestPathMB.equalsIgnoreCase("/META-INF"))
            || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
            || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // Select the Wrapper to be used for this Request
    Wrapper wrapper = request.getWrapper();
    if (wrapper == null || wrapper.isUnavailable()) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // Acknowledge the request
    try {
        response.sendAcknowledgement();
    } catch (IOException ioe) {
        container.getLogger().error(sm.getString(
                "standardContextValve.acknowledgeException"), ioe);
        request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ioe);
        response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        return;
    }

    if (request.isAsyncSupported()) {
        request.setAsyncSupported(wrapper.getPipeline().isAsyncSupported());
    }
    wrapper.getPipeline().getFirst().invoke(request, response);
}
```

## Wrapper

Wrapper是一个Servlet的包装，我们先来看看构造方法。主要作用就是设置基础阀门`StandardWrapperValve`。

```java
public StandardWrapper() {
    super();
    swValve=new StandardWrapperValve();
    pipeline.setBasic(swValve);
    broadcaster = new NotificationBroadcasterSupport();
}
```

接下来我们看看`StandardWrapperValve`的`invoke()`方法。该方法非常长，已经超过了200行。

> 前方高能！

```java
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Initialize local variables we may need
    boolean unavailable = false;
    Throwable throwable = null;
    // This should be a Request attribute...
    long t1=System.currentTimeMillis();
    requestCount.incrementAndGet();
    StandardWrapper wrapper = (StandardWrapper) getContainer();
    Servlet servlet = null;
    Context context = (Context) wrapper.getParent();

    // Check for the application being marked unavailable
    if (!context.getState().isAvailable()) {
        response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                       sm.getString("standardContext.isUnavailable"));
        unavailable = true;
    }

    // Check for the servlet being marked unavailable
    if (!unavailable && wrapper.isUnavailable()) {
        container.getLogger().info(sm.getString("standardWrapper.isUnavailable",
                wrapper.getName()));
        long available = wrapper.getAvailable();
        if ((available > 0L) && (available < Long.MAX_VALUE)) {
            response.setDateHeader("Retry-After", available);
            response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                    sm.getString("standardWrapper.isUnavailable",
                            wrapper.getName()));
        } else if (available == Long.MAX_VALUE) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    sm.getString("standardWrapper.notFound",
                            wrapper.getName()));
        }
        unavailable = true;
    }

    // Allocate a servlet instance to process this request
    try {
        // 关键点1：这儿调用Wrapper的allocate()方法分配一个Servlet实例
        if (!unavailable) {
            servlet = wrapper.allocate();
        }
    } catch (UnavailableException e) {
        container.getLogger().error(
                sm.getString("standardWrapper.allocateException",
                        wrapper.getName()), e);
        long available = wrapper.getAvailable();
        if ((available > 0L) && (available < Long.MAX_VALUE)) {
            response.setDateHeader("Retry-After", available);
            response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                       sm.getString("standardWrapper.isUnavailable",
                                    wrapper.getName()));
        } else if (available == Long.MAX_VALUE) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                       sm.getString("standardWrapper.notFound",
                                    wrapper.getName()));
        }
    } catch (ServletException e) {
        container.getLogger().error(sm.getString("standardWrapper.allocateException",
                         wrapper.getName()), StandardWrapper.getRootCause(e));
        throwable = e;
        exception(request, response, e);
    } catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        container.getLogger().error(sm.getString("standardWrapper.allocateException",
                         wrapper.getName()), e);
        throwable = e;
        exception(request, response, e);
        servlet = null;
    }

    MessageBytes requestPathMB = request.getRequestPathMB();
    DispatcherType dispatcherType = DispatcherType.REQUEST;
    if (request.getDispatcherType()==DispatcherType.ASYNC) dispatcherType = DispatcherType.ASYNC;
    request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
    request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
            requestPathMB);
    // Create the filter chain for this request
    // 关键点2，创建过滤器链，类似于Pipeline的功能
    ApplicationFilterChain filterChain =
            ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

    // Call the filter chain for this request
    // NOTE: This also calls the servlet's service() method
    try {
        if ((servlet != null) && (filterChain != null)) {
            // Swallow output if needed
            if (context.getSwallowOutput()) {
                try {
                    SystemLogHandler.startCapture();
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else {
                        // 关键点3，调用过滤器链的doFilter，最终会调用到Servlet的service方法
                        filterChain.doFilter(request.getRequest(),
                                response.getResponse());
                    }
                } finally {
                    String log = SystemLogHandler.stopCapture();
                    if (log != null && log.length() > 0) {
                        context.getLogger().info(log);
                    }
                }
            } else {
                if (request.isAsyncDispatching()) {
                    request.getAsyncContextInternal().doInternalDispatch();
                } else {
                    // 关键点3，调用过滤器链的doFilter，最终会调用到Servlet的service方法
                    filterChain.doFilter
                        (request.getRequest(), response.getResponse());
                }
            }

        }
    } catch (ClientAbortException e) {
        throwable = e;
        exception(request, response, e);
    } catch (IOException e) {
        container.getLogger().error(sm.getString(
                "standardWrapper.serviceException", wrapper.getName(),
                context.getName()), e);
        throwable = e;
        exception(request, response, e);
    } catch (UnavailableException e) {
        container.getLogger().error(sm.getString(
                "standardWrapper.serviceException", wrapper.getName(),
                context.getName()), e);
        //            throwable = e;
        //            exception(request, response, e);
        wrapper.unavailable(e);
        long available = wrapper.getAvailable();
        if ((available > 0L) && (available < Long.MAX_VALUE)) {
            response.setDateHeader("Retry-After", available);
            response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                       sm.getString("standardWrapper.isUnavailable",
                                    wrapper.getName()));
        } else if (available == Long.MAX_VALUE) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                        sm.getString("standardWrapper.notFound",
                                    wrapper.getName()));
        }
        // Do not save exception in 'throwable', because we
        // do not want to do exception(request, response, e) processing
    } catch (ServletException e) {
        Throwable rootCause = StandardWrapper.getRootCause(e);
        if (!(rootCause instanceof ClientAbortException)) {
            container.getLogger().error(sm.getString(
                    "standardWrapper.serviceExceptionRoot",
                    wrapper.getName(), context.getName(), e.getMessage()),
                    rootCause);
        }
        throwable = e;
        exception(request, response, e);
    } catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        container.getLogger().error(sm.getString(
                "standardWrapper.serviceException", wrapper.getName(),
                context.getName()), e);
        throwable = e;
        exception(request, response, e);
    }

    // Release the filter chain (if any) for this request
    // 关键点4，释放掉过滤器链及其相关资源
    if (filterChain != null) {
        filterChain.release();
    }

    // 关键点5，释放掉Servlet及相关资源
    // Deallocate the allocated servlet instance
    try {
        if (servlet != null) {
            wrapper.deallocate(servlet);
        }
    } catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        container.getLogger().error(sm.getString("standardWrapper.deallocateException",
                         wrapper.getName()), e);
        if (throwable == null) {
            throwable = e;
            exception(request, response, e);
        }
    }

    // If this servlet has been marked permanently unavailable,
    // unload it and release this instance
    // 关键点6，如果servlet被标记为永远不可达，则需要卸载掉它，并释放这个servlet实例
    try {
        if ((servlet != null) &&
            (wrapper.getAvailable() == Long.MAX_VALUE)) {
            wrapper.unload();
        }
    } catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        container.getLogger().error(sm.getString("standardWrapper.unloadException",
                         wrapper.getName()), e);
        if (throwable == null) {
            throwable = e;
            exception(request, response, e);
        }
    }
    long t2=System.currentTimeMillis();

    long time=t2-t1;
    processingTime += time;
    if( time > maxTime) maxTime=time;
    if( time < minTime) minTime=time;
}
```

通过阅读源码，我们发现了几个关键点。现罗列如下，后面我们会逐一分析这些关键点相关的源码。

1. 关键点1：这儿调用Wrapper的allocate()方法分配一个Servlet实例
2. 关键点2，创建过滤器链，类似于Pipeline的功能
3. 关键点3，调用过滤器链的doFilter，最终会调用到Servlet的service方法
4. 关键点4，释放掉过滤器链及其相关资源
5. 关键点5，释放掉Servlet及相关资源
6. 关键点6，如果servlet被标记为永远不可达，则需要卸载掉它，并释放这个servlet实例

### 关键点1 - Wrapper分配Servlet实例

我们来分析一下Wrapper.allocate()方法

```java
@Override
public Servlet allocate() throws ServletException {

    // If we are currently unloading this servlet, throw an exception
    // 卸载过程中，不能分配Servlet
    if (unloading) {
        throw new ServletException(sm.getString("standardWrapper.unloading", getName()));
    }

    boolean newInstance = false;

    // If not SingleThreadedModel, return the same instance every time
    // 如果Wrapper没有实现SingleThreadedModel，则每次都会返回同一个Servlet
    if (!singleThreadModel) {
        // Load and initialize our instance if necessary
        // 实例为null或者实例还未初始化，使用synchronized来保证并发时的原子性
        if (instance == null || !instanceInitialized) {
            synchronized (this) {
                if (instance == null) {
                    try {
                        if (log.isDebugEnabled()) {
                            log.debug("Allocating non-STM instance");
                        }

                        // Note: We don't know if the Servlet implements
                        // SingleThreadModel until we have loaded it.
                        // 加载Servlet
                        instance = loadServlet();
                        newInstance = true;
                        if (!singleThreadModel) {
                            // For non-STM, increment here to prevent a race
                            // condition with unload. Bug 43683, test case
                            // #3
                            countAllocated.incrementAndGet();
                        }
                    } catch (ServletException e) {
                        throw e;
                    } catch (Throwable e) {
                        ExceptionUtils.handleThrowable(e);
                        throw new ServletException(sm.getString("standardWrapper.allocate"), e);
                    }
                }
                // 初始化Servlet
                if (!instanceInitialized) {
                    initServlet(instance);
                }
            }
        }

        if (singleThreadModel) {
            if (newInstance) {
                // Have to do this outside of the sync above to prevent a
                // possible deadlock
                synchronized (instancePool) {
                    instancePool.push(instance);
                    nInstances++;
                }
            }
        }
        // 非单线程模型，直接返回已经创建的Servlet，也就是说，这种情况下只会创建一个Servlet
        else {
            if (log.isTraceEnabled()) {
                log.trace("  Returning non-STM instance");
            }
            // For new instances, count will have been incremented at the
            // time of creation
            if (!newInstance) {
                countAllocated.incrementAndGet();
            }
            return instance;
        }
    }

    // 如果是单线程模式，则使用servlet对象池技术来加载多个Servlet
    synchronized (instancePool) {
        while (countAllocated.get() >= nInstances) {
            // Allocate a new instance if possible, or else wait
            if (nInstances < maxInstances) {
                try {
                    instancePool.push(loadServlet());
                    nInstances++;
                } catch (ServletException e) {
                    throw e;
                } catch (Throwable e) {
                    ExceptionUtils.handleThrowable(e);
                    throw new ServletException(sm.getString("standardWrapper.allocate"), e);
                }
            } else {
                try {
                    instancePool.wait();
                } catch (InterruptedException e) {
                    // Ignore
                }
            }
        }
        if (log.isTraceEnabled()) {
            log.trace("  Returning allocated STM instance");
        }
        countAllocated.incrementAndGet();
        return instancePool.pop();
    }
}
```

总结下来，注意以下几点即可：

1. 卸载过程中，不能分配Servlet
2. 如果不是单线程模式，则每次都会返回同一个Servlet（默认Servlet实现方式）
3. `Servlet`实例为`null`或者`Servlet`实例还`未初始化`，使用synchronized来保证并发时的原子性
4. 如果是单线程模式，则使用servlet对象池技术来加载多个Servlet

接下来我们看看`loadServlet()`方法

```java
public synchronized Servlet loadServlet() throws ServletException {

    // Nothing to do if we already have an instance or an instance pool
    if (!singleThreadModel && (instance != null))
        return instance;

    PrintStream out = System.out;
    if (swallowOutput) {
        SystemLogHandler.startCapture();
    }

    Servlet servlet;
    try {
        long t1=System.currentTimeMillis();
        // Complain if no servlet class has been specified
        if (servletClass == null) {
            unavailable(null);
            throw new ServletException
                (sm.getString("standardWrapper.notClass", getName()));
        }

        // 关键的地方，就是通过实例管理器，创建Servlet实例，而实例管理器是通过特殊的类加载器来加载给定的类
        InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();
        try {
            servlet = (Servlet) instanceManager.newInstance(servletClass);
        } catch (ClassCastException e) {
            unavailable(null);
            // Restore the context ClassLoader
            throw new ServletException
                (sm.getString("standardWrapper.notServlet", servletClass), e);
        } catch (Throwable e) {
            e = ExceptionUtils.unwrapInvocationTargetException(e);
            ExceptionUtils.handleThrowable(e);
            unavailable(null);

            // Added extra log statement for Bugzilla 36630:
            // https://bz.apache.org/bugzilla/show_bug.cgi?id=36630
            if(log.isDebugEnabled()) {
                log.debug(sm.getString("standardWrapper.instantiate", servletClass), e);
            }

            // Restore the context ClassLoader
            throw new ServletException
                (sm.getString("standardWrapper.instantiate", servletClass), e);
        }

        if (multipartConfigElement == null) {
            MultipartConfig annotation =
                    servlet.getClass().getAnnotation(MultipartConfig.class);
            if (annotation != null) {
                multipartConfigElement =
                        new MultipartConfigElement(annotation);
            }
        }

        // Special handling for ContainerServlet instances
        // Note: The InstanceManager checks if the application is permitted
        //       to load ContainerServlets
        if (servlet instanceof ContainerServlet) {
            ((ContainerServlet) servlet).setWrapper(this);
        }

        classLoadTime=(int) (System.currentTimeMillis() -t1);

        if (servlet instanceof SingleThreadModel) {
            if (instancePool == null) {
                instancePool = new Stack<>();
            }
            singleThreadModel = true;
        }

        // 调用Servlet的init方法
        initServlet(servlet);

        fireContainerEvent("load", this);

        loadTime=System.currentTimeMillis() -t1;
    } finally {
        if (swallowOutput) {
            String log = SystemLogHandler.stopCapture();
            if (log != null && log.length() > 0) {
                if (getServletContext() != null) {
                    getServletContext().log(log);
                } else {
                    out.println(log);
                }
            }
        }
    }
    return servlet;
}
```

关键的地方有两个：
1. 通过实例管理器，创建Servlet实例，而实例管理器是通过特殊的类加载器来加载给定的类
2. 调用Servlet的init方法

### 关键点2 - 创建过滤器链

创建过滤器链是调用的`org.apache.catalina.core.ApplicationFilterFactory`的`createFilterChain()`方法。我们来分析一下这个方法。该方法需要注意的地方已经在代码的comments里面说明了。

```java
public static ApplicationFilterChain createFilterChain(ServletRequest request,
        Wrapper wrapper, Servlet servlet) {

    // If there is no servlet to execute, return null
    if (servlet == null)
        return null;

    // Create and initialize a filter chain object
    // 1. 如果加密打开了，则可能会多次调用这个方法
    // 2. 为了避免重复生成filterChain对象，所以会将filterChain对象放在Request里面进行缓存
    ApplicationFilterChain filterChain = null;
    if (request instanceof Request) {
        Request req = (Request) request;
        if (Globals.IS_SECURITY_ENABLED) {
            // Security: Do not recycle
            filterChain = new ApplicationFilterChain();
        } else {
            filterChain = (ApplicationFilterChain) req.getFilterChain();
            if (filterChain == null) {
                filterChain = new ApplicationFilterChain();
                req.setFilterChain(filterChain);
            }
        }
    } else {
        // Request dispatcher in use
        filterChain = new ApplicationFilterChain();
    }

    filterChain.setServlet(servlet);
    filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());

    // Acquire the filter mappings for this Context
    StandardContext context = (StandardContext) wrapper.getParent();
    // 从这儿看出过滤器链对象里面的元素是根据Context里面的filterMaps来生成的
    FilterMap filterMaps[] = context.findFilterMaps();

    // If there are no filter mappings, we are done
    if ((filterMaps == null) || (filterMaps.length == 0))
        return (filterChain);

    // Acquire the information we will need to match filter mappings
    DispatcherType dispatcher =
            (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

    String requestPath = null;
    Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
    if (attribute != null){
        requestPath = attribute.toString();
    }

    String servletName = wrapper.getName();

    // Add the relevant path-mapped filters to this filter chain
    // 类型和路径都匹配的情况下，将context.filterConfig放到过滤器链里面
    for (int i = 0; i < filterMaps.length; i++) {
        if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
            continue;
        }
        if (!matchFiltersURL(filterMaps[i], requestPath))
            continue;
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
            context.findFilterConfig(filterMaps[i].getFilterName());
        if (filterConfig == null) {
            // FIXME - log configuration problem
            continue;
        }
        filterChain.addFilter(filterConfig);
    }

    // Add filters that match on servlet name second
    // 类型和servlet名称都匹配的情况下，将context.filterConfig放到过滤器链里面
    for (int i = 0; i < filterMaps.length; i++) {
        if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
            continue;
        }
        if (!matchFiltersServlet(filterMaps[i], servletName))
            continue;
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
            context.findFilterConfig(filterMaps[i].getFilterName());
        if (filterConfig == null) {
            // FIXME - log configuration problem
            continue;
        }
        filterChain.addFilter(filterConfig);
    }

    // Return the completed filter chain
    return filterChain;
}
```

### 关键点3 - 调用过滤器链的doFilter

```java
// 1. `doFilter`方法会对异常进行重新抛出，也就是将检查性异常变更为运行时异常
// 2. `doFilter`如果访问授权开关打开，则会有访问鉴权
// 3. `doFilter`最终会调用internalDoFilter方法
@Override
public void doFilter(ServletRequest request, ServletResponse response)
    throws IOException, ServletException {

    if( Globals.IS_SECURITY_ENABLED ) {
        final ServletRequest req = request;
        final ServletResponse res = response;
        try {
            java.security.AccessController.doPrivileged(
                new java.security.PrivilegedExceptionAction<Void>() {
                    @Override
                    public Void run()
                        throws ServletException, IOException {
                        internalDoFilter(req,res);
                        return null;
                    }
                }
            );
        } catch( PrivilegedActionException pe) {
            Exception e = pe.getException();
            if (e instanceof ServletException)
                throw (ServletException) e;
            else if (e instanceof IOException)
                throw (IOException) e;
            else if (e instanceof RuntimeException)
                throw (RuntimeException) e;
            else
                throw new ServletException(e.getMessage(), e);
        }
    } else {
        internalDoFilter(request,response);
    }
}

// 1. `internalDoFilter`方法通过pos和n来调用过滤器链里面的每个过滤器。pos表示当前的过滤器下标，n表示总的过滤器数量
// 2. `internalDoFilter`方法最终会调用servlet.service()方法
private void internalDoFilter(ServletRequest request,
                              ServletResponse response)
    throws IOException, ServletException {

    // Call the next filter if there is one
    if (pos < n) {
        ApplicationFilterConfig filterConfig = filters[pos++];
        try {
            Filter filter = filterConfig.getFilter();

            if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                    filterConfig.getFilterDef().getAsyncSupported())) {
                request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
            }
            if( Globals.IS_SECURITY_ENABLED ) {
                final ServletRequest req = request;
                final ServletResponse res = response;
                Principal principal =
                    ((HttpServletRequest) req).getUserPrincipal();

                Object[] args = new Object[]{req, res, this};
                SecurityUtil.doAsPrivilege ("doFilter", filter, classType, args, principal);
            } else {
                filter.doFilter(request, response, this);
            }
        } catch (IOException | ServletException | RuntimeException e) {
            throw e;
        } catch (Throwable e) {
            e = ExceptionUtils.unwrapInvocationTargetException(e);
            ExceptionUtils.handleThrowable(e);
            throw new ServletException(sm.getString("filterChain.filter"), e);
        }
        return;
    }

    // We fell off the end of the chain -- call the servlet instance
    try {
        if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
            lastServicedRequest.set(request);
            lastServicedResponse.set(response);
        }

        if (request.isAsyncSupported() && !servletSupportsAsync) {
            request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                    Boolean.FALSE);
        }
        // Use potentially wrapped request from this point
        if ((request instanceof HttpServletRequest) &&
                (response instanceof HttpServletResponse) &&
                Globals.IS_SECURITY_ENABLED ) {
            final ServletRequest req = request;
            final ServletResponse res = response;
            Principal principal =
                ((HttpServletRequest) req).getUserPrincipal();
            Object[] args = new Object[]{req, res};
            SecurityUtil.doAsPrivilege("service",
                                       servlet,
                                       classTypeUsedInService,
                                       args,
                                       principal);
        } else {
            servlet.service(request, response);
        }
    } catch (IOException | ServletException | RuntimeException e) {
        throw e;
    } catch (Throwable e) {
        e = ExceptionUtils.unwrapInvocationTargetException(e);
        ExceptionUtils.handleThrowable(e);
        throw new ServletException(sm.getString("filterChain.servlet"), e);
    } finally {
        if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
            lastServicedRequest.set(null);
            lastServicedResponse.set(null);
        }
    }
}
```

1. `doFilter`方法会对异常进行重新抛出，也就是将检查性异常变更为运行时异常
2. `doFilter`如果访问授权开关打开，则会有访问鉴权
3. `doFilter`最终会调用internalDoFilter方法
4. `internalDoFilter`方法通过pos和n来调用过滤器链里面的每个过滤器。pos表示当前的过滤器下标，n表示总的过滤器数量
5. `internalDoFilter`方法最终会调用servlet.service()方法

### 关键点4 - 释放掉过滤器链及其相关资源

调用的`FilterChain.release()`方法，我们来分析一下，逻辑较之前的代码简单很多。

1. 过滤器置为null，help gc
2. n和pos置为0
3. servlet置为null

```java
void release() {
    for (int i = 0; i < n; i++) {
        filters[i] = null;
    }
    n = 0;
    pos = 0;
    servlet = null;
    servletSupportsAsync = false;
}
```

### 关键点5 - 释放掉Servlet及相关资源

调用的是`StandardWrapper`的`deallocate()`方法，我们来分析一下。

1. 如果是非单线程模型，直接返回。
2. 如果是单线程模型，则将Servlet对象放回对象池。

```java
@Override
public void deallocate(Servlet servlet) throws ServletException {
    // If not SingleThreadModel, no action is required
    if (!singleThreadModel) {
        countAllocated.decrementAndGet();
        return;
    }

    // Unlock and free this instance
    synchronized (instancePool) {
        countAllocated.decrementAndGet();
        instancePool.push(servlet);
        instancePool.notify();
    }
}
```

### 关键点6 - 卸载servlet实例

该逻辑调用的是StandardWrapper的unload()方法，该方法非常得长！

> 前方高能！

```java
@Override
public synchronized void unload() throws ServletException {

    // Nothing to do if we have never loaded the instance
    // 如果实例未加载，那么什么事情都不做
    if (!singleThreadModel && (instance == null))
        return;
    // 设置正在卸载标记为true
    unloading = true;

    // Loaf a while if the current instance is allocated
    // (possibly more than once if non-STM)
    // 当前有实例已经分配了，则需要等待分配的实例完成处理。所以这儿会有线程休眠操作
    if (countAllocated.get() > 0) {
        int nRetries = 0;
        long delay = unloadDelay / 20;
        while ((nRetries < 21) && (countAllocated.get() > 0)) {
            if ((nRetries % 10) == 0) {
                log.info(sm.getString("standardWrapper.waiting",
                                      countAllocated.toString(),
                                      getName()));
            }
            try {
                Thread.sleep(delay);
            } catch (InterruptedException e) {
                // Ignore
            }
            nRetries++;
        }
    }

    if (instanceInitialized) {
        PrintStream out = System.out;
        if (swallowOutput) {
            SystemLogHandler.startCapture();
        }

        // Call the servlet destroy() method
        // 如果为非单线程模型，则调用servlet的destroy方法
        try {
            if( Globals.IS_SECURITY_ENABLED) {
                try {
                    SecurityUtil.doAsPrivilege("destroy", instance);
                } finally {
                    SecurityUtil.remove(instance);
                }
            } else {
                instance.destroy();
            }

        } catch (Throwable t) {
            t = ExceptionUtils.unwrapInvocationTargetException(t);
            ExceptionUtils.handleThrowable(t);
            instance = null;
            instancePool = null;
            nInstances = 0;
            fireContainerEvent("unload", this);
            unloading = false;
            throw new ServletException
                (sm.getString("standardWrapper.destroyException", getName()),
                 t);
        } finally {
            // Annotation processing
            if (!((Context) getParent()).getIgnoreAnnotations()) {
                try {
                    ((Context)getParent()).getInstanceManager().destroyInstance(instance);
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("standardWrapper.destroyInstance", getName()), t);
                }
            }
            // Write captured output
            if (swallowOutput) {
                String log = SystemLogHandler.stopCapture();
                if (log != null && log.length() > 0) {
                    if (getServletContext() != null) {
                        getServletContext().log(log);
                    } else {
                        out.println(log);
                    }
                }
            }
        }
    }

    // Deregister the destroyed instance
    // 将销毁的servlet从jmx中unregister操作
    instance = null;
    instanceInitialized = false;

    if (isJspServlet && jspMonitorON != null ) {
        Registry.getRegistry(null, null).unregisterComponent(jspMonitorON);
    }

    // 如果是单线程模型，需要将servlet进行出栈(pop）操作，并调用servlet的destory方法
    if (singleThreadModel && (instancePool != null)) {
        try {
            while (!instancePool.isEmpty()) {
                Servlet s = instancePool.pop();
                if (Globals.IS_SECURITY_ENABLED) {
                    try {
                        SecurityUtil.doAsPrivilege("destroy", s);
                    } finally {
                        SecurityUtil.remove(s);
                    }
                } else {
                    s.destroy();
                }
                // Annotation processing
                if (!((Context) getParent()).getIgnoreAnnotations()) {
                   ((StandardContext)getParent()).getInstanceManager().destroyInstance(s);
                }
            }
        } catch (Throwable t) {
            t = ExceptionUtils.unwrapInvocationTargetException(t);
            ExceptionUtils.handleThrowable(t);
            instancePool = null;
            nInstances = 0;
            unloading = false;
            fireContainerEvent("unload", this);
            throw new ServletException
                (sm.getString("standardWrapper.destroyException",
                              getName()), t);
        }
        instancePool = null;
        nInstances = 0;
    }

    singleThreadModel = false;

    // 卸载完成，卸载标记置为false
    unloading = false;
    // 发送容器unload事件
    fireContainerEvent("unload", this);
}
```

总结一下该方法的逻辑：

1. 如果实例未加载，那么什么事情都不做
2. 设置正在卸载标记为true
3. 当前有实例已经分配了，则需要等待分配的实例完成处理。所以这儿会有线程休眠操作
4. 如果为非单线程模型，则调用servlet的destroy方法
5. 将销毁的servlet从jmx中unregister操作
6. 如果是单线程模型，需要将servlet进行出栈(pop）操作，并调用servlet的destory方法
7. 卸载完成，卸载标记置为false
8. 发送容器unload事件

## 总结

在tomcat中，容器(Container)分4种，Engine、Host、Context和Wrapper。Engine是容器的入口，Host是虚拟主机，Context是应用服务，Wrapper为具体的Servlet。

从外层的Engine到内层的Wrapper，我们逐层进行了详细分析。尤其是对Wrapper进行了详细分析，包括Servlet的加载、过滤器链的生成、过滤器链的调用等等各项复杂的设计和代码。

本文主要对Container进行了分析，关于其他的组件我们只是一笔带过。

至此，楼主已经完全清楚了容器的初始化、加载和运行的机制。那么，你呢？是否也完全明白了呢？

## 参考链接

https://www.ramkitech.com/2012/02/understanding-virtual-host-concept-in.html