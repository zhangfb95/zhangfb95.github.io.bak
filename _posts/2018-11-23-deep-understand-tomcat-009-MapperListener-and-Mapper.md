---
layout: post
title:  深入理解Tomcat（九）MapperListener和Mapper
date:   2018-11-23 23:38:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

为了能够快速地通过指定的uri找到对应的wrapper及servlet，tomcat开发人员设计出了两个组件：`MapperListener`和`Mapper`。

`MapperListener`主要作用如下：

1. 通过监听容器的`AFTER_START_EVENT`事件来对容器进行注册；
2. 通过监听容器的`BEFORE_STOP_EVENT`事件来完成对容器的取消注册。

而Mapper作为uri映射到容器的工具，扮演的角色就是一个映射组件。它会缓存所有容器信息（包括容器名称、容器本身、容器层级等等），同时提供映射规则，将一个uri按照映射规则映射到具体的Host、Context和Wrapper，并最终通过Wrapper找到逻辑处理单元Servlet。

> 本小节我们会对`MapperListener`及`Mapper缓存容器信息`这一块的源码进行分析，将映射逻辑放在第十小节。

那么，让我们开始我们的征程吧~

## 入口

在`org.apache.catalina.core.StandardService`类中，我们看到两个关键的字段--`mapper`和`mapperListenter`。其中`mapperListener`依赖`service对象`进行构造。

```java
/**
 * Mapper.
 */
protected final Mapper mapper = new Mapper();
/**
 * Mapper listener.
 */
protected final MapperListener mapperListener = new MapperListener(this);
```

## 1. MapperListener的注册过程

`MapperListener`的构造方法比较简单，仅仅将mapper和service存储到当前对象的相关属性中。

```java
public MapperListener(Service service) {
    this.service = service;
    this.mapper = service.getMapper();
}
```

`MapperListener`在构造完成之后，会调用其`start()`方法，我们来看看主要做了哪些事情？

1. engine容器不存在，则MapperListener也不需要启动
2. 查找默认主机，并设置到mapper的defaultHostName属性中
3. 对容器及下面的所有子容器添加事件监听器
4. 注册engine下面的host、context和wrapper，registerHost会注册host及下面的子容器

```java
public void startInternal() throws LifecycleException {
    setState(LifecycleState.STARTING);

    // 1. engine容器不存在，则MapperListener也不需要启动
    Engine engine = service.getContainer();
    if (engine == null) {
        return;
    }

    // 2. 查找默认主机，并设置到mapper的defaultHostName属性中
    findDefaultHost();

    // 3. 对容器及下面的所有子容器添加事件监听器
    addListeners(engine);

    // 4. 注册engine下面的host、context和wrapper，registerHost会注册host及下面的子容器
    Container[] conHosts = engine.findChildren();
    for (Container conHost : conHosts) {
        Host host = (Host) conHost;
        if (!LifecycleState.NEW.equals(host.getState())) {
            // Registering the host will register the context and wrappers
            registerHost(host);
        }
    }
}
```

接着我们看看方法`findDefaultHost`，主要目的是检查并设置`mapper`的`defaultHostName`属性。

```java
private void findDefaultHost() {
    // 获取engine下面配置的defaultHost属性
    Engine engine = service.getContainer();
    String defaultHost = engine.getDefaultHost();

    boolean found = false;

    // 如果defaultHost属性不为空，则查找hosts下面的所有主机名及别名。
    // 1. 找到了则设置到mapper的defaultHostName属性
    // 2. 没找到则记录警告信息
    if (defaultHost != null && defaultHost.length() >0) {
        Container[] containers = engine.findChildren();

        for (Container container : containers) {
            Host host = (Host) container;
            if (defaultHost.equalsIgnoreCase(host.getName())) {
                found = true;
                break;
            }

            String[] aliases = host.findAliases();
            for (String alias : aliases) {
                if (defaultHost.equalsIgnoreCase(alias)) {
                    found = true;
                    break;
                }
            }
        }
    }

    if(found) {
        mapper.setDefaultHostName(defaultHost);
    } else {
        log.warn(sm.getString("mapperListener.unknownDefaultHost",
                defaultHost, service));
    }
}
```

接下来我们分析`addListeners`，该方法用于`对所有容器设置监听器`。它是一个递归方法！

```java
private void addListeners(Container container) {
    // 对当前容器添加容器监听器和生命周期监听器，也就是当前对象
    container.addContainerListener(this);
    container.addLifecycleListener(this);
    // 对当前容器下的子容器执行addListeners操作
    for (Container child : container.findChildren()) {
        addListeners(child);
    }
}
```

接下来我们分析`registerHost`，该方法用于往`mapper`中注册虚拟主机

```java
/**
 * 注册虚拟主机
 * Register host.
 */
private void registerHost(Host host) {

    String[] aliases = host.findAliases();
    // 往mapper中添加主机
    mapper.addHost(host.getName(), aliases, host);

    // 注册host下的每个context
    for (Container container : host.findChildren()) {
        if (container.getState().isAvailable()) {
            registerContext((Context) container);
        }
    }
    if(log.isDebugEnabled()) {
        log.debug(sm.getString("mapperListener.registerHost",
                host.getName(), domain, service));
    }
}
```

`registerHost`除了注册虚拟主机，额外会调用`registerContext`来注册`context`。我们看看这个方法。该方法完成了以下操作：

1. contextPath如果为斜杠，则统一转换为空字符串
2. 将context下面的每个wrapper都添加到mapper
3. 将context添加到mapper

```java
/**
 * 注册context
 * Register context.
 */
private void registerContext(Context context) {
    // contextPath如果为斜杠，则统一转换为空字符串
    String contextPath = context.getPath();
    if ("/".equals(contextPath)) {
        contextPath = "";
    }
    Host host = (Host)context.getParent();

    WebResourceRoot resources = context.getResources();
    String[] welcomeFiles = context.findWelcomeFiles();
    List<WrapperMappingInfo> wrappers = new ArrayList<>();

    // 将context下面的每个wrapper都添加到mapper
    for (Container container : context.findChildren()) {
        // 准备wrapper信息，以便后续插入mapper
        prepareWrapperMappingInfo(context, (Wrapper) container, wrappers);

        if(log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.registerWrapper",
                    container.getName(), contextPath, service));
        }
    }

    // 将context添加到mapper
    mapper.addContextVersion(host.getName(), host, contextPath,
            context.getWebappVersion(), context, welcomeFiles, resources,
            wrappers);

    if(log.isDebugEnabled()) {
        log.debug(sm.getString("mapperListener.registerContext",
                contextPath, service));
    }
}
```

关键方法为`prepareWrapperMappingInfo`，用于准备注册到mapper下的wrapper，这儿mapper对于wrapper的支持是wrapper的包装对象--`WrapperMappingInfo`。而一个context可能有多个wrapper，所以`WrapperMappingInfo`是一个`list`。我们来分析一下这个list对象的生成方法--`prepareWrapperMappingInfo`。

该方法就是将`映射url`、`wrapper名字`和`资源只读标记`等信息组合成对象添加到wrappers中。

```java
private void prepareWrapperMappingInfo(Context context, Wrapper wrapper,
        List<WrapperMappingInfo> wrappers) {
    String wrapperName = wrapper.getName();
    boolean resourceOnly = context.isResourceOnlyServlet(wrapperName);
    String[] mappings = wrapper.findMappings();
    for (String mapping : mappings) {
        boolean jspWildCard = (wrapperName.equals("jsp")
                               && mapping.endsWith("/*"));
        wrappers.add(new WrapperMappingInfo(mapping, wrapper, jspWildCard,
                resourceOnly));
    }
}
```

## 2. MapperListener的取消注册过程

在tomcat组件中，`start()`的逆向过程为`stop()`。`MapperListener组件`也不例外。我们来分析一下其`stop()`方法，该方法的作用是将当前监听器从容器中移除。

```java
@Override
public void stopInternal() throws LifecycleException {
    setState(LifecycleState.STOPPING);

    Engine engine = service.getContainer();
    if (engine == null) {
        return;
    }
    removeListeners(engine);
}
private void removeListeners(Container container) {
    container.removeContainerListener(this);
    container.removeLifecycleListener(this);
    for (Container child : container.findChildren()) {
        removeListeners(child);
    }
}
```

当每个容器在`stop()`方法被调用的时候，都会触发相应的容器事件。我们看看`ContainerBase`下面触发事件的代码，该方法会调用所有容器监听器的`containerEvent()`方法。

```java
@Override
public void fireContainerEvent(String type, Object data) {
    if (listeners.size() < 1)
        return;

    ContainerEvent event = new ContainerEvent(this, type, data);
    // Note for each uses an iterator internally so this is safe
    for (ContainerListener listener : listeners) {
        listener.containerEvent(event);
    }
}
```

当每个容器在`stop()`方法被调用的时候，都会触发相应的生命周期事件，我们看看`LifecycleBase`下面触发事件的代码，就是调用生命周期监听器的`lifecycleEvent()`方法

```java
protected void fireLifecycleEvent(String type, Object data) {
    LifecycleEvent event = new LifecycleEvent(this, type, data);
    for (LifecycleListener listener : lifecycleListeners) {
        listener.lifecycleEvent(event);
    }
}
```

好了我们已经看到了MapperListener接下来要分析的方法了，即：`containerEvent()`容器方法和`lifecycleEvent()`生命周期方法。

先来看`containerEvent()`，虽然该方法的代码非常得长，但是逻辑却很简单。该方法有非常多的`if-else`（虽然我们推荐使用设计模式代替）。所有的操作都是对mapper缓存的资源进行增删改操作。

```java
@Override
public void containerEvent(ContainerEvent event) {
    if (Container.ADD_CHILD_EVENT.equals(event.getType())) {
        Container child = (Container) event.getData();
        addListeners(child);
        // If child is started then it is too late for life-cycle listener
        // to register the child so register it here
        if (child.getState().isAvailable()) {
            if (child instanceof Host) {
                registerHost((Host) child);
            } else if (child instanceof Context) {
                registerContext((Context) child);
            } else if (child instanceof Wrapper) {
                // Only if the Context has started. If it has not, then it
                // will have its own "after_start" life-cycle event later.
                if (child.getParent().getState().isAvailable()) {
                    registerWrapper((Wrapper) child);
                }
            }
        }
    } else if (Container.REMOVE_CHILD_EVENT.equals(event.getType())) {
        Container child = (Container) event.getData();
        removeListeners(child);
        // No need to unregister - life-cycle listener will handle this when
        // the child stops
    } else if (Host.ADD_ALIAS_EVENT.equals(event.getType())) {
        // Handle dynamically adding host aliases
        mapper.addHostAlias(((Host) event.getSource()).getName(),
                event.getData().toString());
    } else if (Host.REMOVE_ALIAS_EVENT.equals(event.getType())) {
        // Handle dynamically removing host aliases
        mapper.removeHostAlias(event.getData().toString());
    } else if (Wrapper.ADD_MAPPING_EVENT.equals(event.getType())) {
        // Handle dynamically adding wrappers
        Wrapper wrapper = (Wrapper) event.getSource();
        Context context = (Context) wrapper.getParent();
        String contextPath = context.getPath();
        if ("/".equals(contextPath)) {
            contextPath = "";
        }
        String version = context.getWebappVersion();
        String hostName = context.getParent().getName();
        String wrapperName = wrapper.getName();
        String mapping = (String) event.getData();
        boolean jspWildCard = ("jsp".equals(wrapperName)
                && mapping.endsWith("/*"));
        mapper.addWrapper(hostName, contextPath, version, mapping, wrapper,
                jspWildCard, context.isResourceOnlyServlet(wrapperName));
    } else if (Wrapper.REMOVE_MAPPING_EVENT.equals(event.getType())) {
        // Handle dynamically removing wrappers
        Wrapper wrapper = (Wrapper) event.getSource();

        Context context = (Context) wrapper.getParent();
        String contextPath = context.getPath();
        if ("/".equals(contextPath)) {
            contextPath = "";
        }
        String version = context.getWebappVersion();
        String hostName = context.getParent().getName();

        String mapping = (String) event.getData();

        mapper.removeWrapper(hostName, contextPath, version, mapping);
    } else if (Context.ADD_WELCOME_FILE_EVENT.equals(event.getType())) {
        // Handle dynamically adding welcome files
        Context context = (Context) event.getSource();

        String hostName = context.getParent().getName();

        String contextPath = context.getPath();
        if ("/".equals(contextPath)) {
            contextPath = "";
        }

        String welcomeFile = (String) event.getData();

        mapper.addWelcomeFile(hostName, contextPath,
                context.getWebappVersion(), welcomeFile);
    } else if (Context.REMOVE_WELCOME_FILE_EVENT.equals(event.getType())) {
        // Handle dynamically removing welcome files
        Context context = (Context) event.getSource();

        String hostName = context.getParent().getName();

        String contextPath = context.getPath();
        if ("/".equals(contextPath)) {
            contextPath = "";
        }

        String welcomeFile = (String) event.getData();

        mapper.removeWelcomeFile(hostName, contextPath,
                context.getWebappVersion(), welcomeFile);
    } else if (Context.CLEAR_WELCOME_FILES_EVENT.equals(event.getType())) {
        // Handle dynamically clearing welcome files
        Context context = (Context) event.getSource();

        String hostName = context.getParent().getName();

        String contextPath = context.getPath();
        if ("/".equals(contextPath)) {
            contextPath = "";
        }

        mapper.clearWelcomeFiles(hostName, contextPath,
                context.getWebappVersion());
    }
}
```

接下来看看生命周期方法，`lifecycleEvent()`。

```java
@Override
public void lifecycleEvent(LifecycleEvent event) {
    if (event.getType().equals(Lifecycle.AFTER_START_EVENT)) {
        Object obj = event.getSource();
        if (obj instanceof Wrapper) {
            Wrapper w = (Wrapper) obj;
            // Only if the Context has started. If it has not, then it will
            // have its own "after_start" event later.
            if (w.getParent().getState().isAvailable()) {
                registerWrapper(w);
            }
        } else if (obj instanceof Context) {
            Context c = (Context) obj;
            // Only if the Host has started. If it has not, then it will
            // have its own "after_start" event later.
            if (c.getParent().getState().isAvailable()) {
                registerContext(c);
            }
        } else if (obj instanceof Host) {
            registerHost((Host) obj);
        }
    } else if (event.getType().equals(Lifecycle.BEFORE_STOP_EVENT)) {
        Object obj = event.getSource();
        if (obj instanceof Wrapper) {
            unregisterWrapper((Wrapper) obj);
        } else if (obj instanceof Context) {
            unregisterContext((Context) obj);
        } else if (obj instanceof Host) {
            unregisterHost((Host) obj);
        }
    }
}
```

`AFTER_START_EVENT`我们前面已经分析过了，因此接下来我们主要分析分析`BEFORE_STOP_EVENT`。该事件可能完成对Host、Context和Wrapper的取消注册操作。我们分别来看看这3个方法。

```java
/**
 * Unregister host.
 * 对host进行取消注册操作，根据hostname来remove
 */
private void unregisterHost(Host host) {
    String hostname = host.getName();

    mapper.removeHost(hostname);

    if(log.isDebugEnabled()) {
        log.debug(sm.getString("mapperListener.unregisterHost", hostname,
                domain, service));
    }
}

/**
 * Unregister context.
 * 对context取消注册
 */
private void unregisterContext(Context context) {
    String contextPath = context.getPath();
    if ("/".equals(contextPath)) {
        contextPath = "";
    }
    String hostName = context.getParent().getName();

    if (context.getPaused()) {
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.pauseContext",
                    contextPath, service));
        }
        // 暂停的context，不能从mapper中移除，只能在mapper暂停
        mapper.pauseContextVersion(context, hostName, contextPath,
                context.getWebappVersion());
    } else {
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.unregisterContext",
                    contextPath, service));
        }
        // 非暂停的context，需要从mapper中移除
        mapper.removeContextVersion(context, hostName, contextPath,
                context.getWebappVersion());
    }
}

/**
 * Unregister wrapper.
 * 对wrapper取消注册
 */
private void unregisterWrapper(Wrapper wrapper) {
    Context context = ((Context) wrapper.getParent());
    String contextPath = context.getPath();
    String wrapperName = wrapper.getName();

    if ("/".equals(contextPath)) {
        contextPath = "";
    }
    String version = context.getWebappVersion();
    String hostName = context.getParent().getName();

    String[] mappings = wrapper.findMappings();
    // 一个wrapper可能有多个map地址，对每个地址都需要移除操作，所以这儿是一个循环
    for (String mapping : mappings) {
        mapper.removeWrapper(hostName, contextPath, version,  mapping);
    }

    if(log.isDebugEnabled()) {
        log.debug(sm.getString("mapperListener.unregisterWrapper",
                wrapperName, contextPath, service));
    }
}
```

总结下来，`BEFORE_STOP_EVENT`在MapperListener里面有下面的功能：

1. 对host进行取消注册操作，根据hostname来remove
2. 对context取消注册，暂停的context，不能从mapper中移除，只能在mapper暂停
3. 对context取消注册，非暂停的context，需要从mapper中移除
4. 对wrapper取消注册，一个wrapper可能有多个map地址，对每个地址都需要移除操作，所以这儿是一个循环

## 3. 容器在Mapper的表现形式

在Mapper中，所有容器都使用`MapElement`来表示，不同的容器有不同的子类实现，我们来看看类继承层级。

![MapElement类继承层级](https://upload-images.jianshu.io/upload_images/845143-6ff1aa16c6189285.png?jianshufrom=1)

我们从父类`MapElement`开始分析，这是一个protected修饰的抽象类，包含`name`和`object`两个属性。其中`object`是泛型类型。

```java
protected abstract static class MapElement<T> {
    public final String name;
    public final T object;

    public MapElement(String name, T object) {
        this.name = name;
        this.object = object;
    }
}
```

### MappedWrapper

为了减少对`MapElement`子类依赖的说明，我们从`MappedWrapper`开始说明每个子类的用途。

在`MappedWrapper`中，`object`为Wrapper容器。额外多了`jsp通配符标记`和`是否资源标记`两个boolean属性。

```java
protected static class MappedWrapper extends MapElement<Wrapper> {
    public final boolean jspWildCard;
    public final boolean resourceOnly;

    public MappedWrapper(String name, Wrapper wrapper, boolean jspWildCard,
            boolean resourceOnly) {
        super(name, wrapper);
        this.jspWildCard = jspWildCard;
        this.resourceOnly = resourceOnly;
    }
}
```

### `Context`为什么需要两个`MapElement`子类呢？

从类继承层级来看，`Context`容器关于`MapElement`有两个子类`ContextVersion`和`MappedContext`，为什么需要有两个呢？

这儿不重复造轮子，参考下面的博客，我们就能知道原因。

> 简单来说，tomcat从7.x版本开始，在同一个tomcat中运行存在一个应用的多个版本，方便用户进行应用的`热升级`。

1. 多个应用版本是通过session来分配转化的。
2. 假如我们有应用`app`、`app##1`，`app##2`这3个版本，`app`表示最老的版本、`app##1`表示次老的版本，`app##2`表示最新的版本。
3. 在部署`app##1`之前创建的session，在`app##1`部署之后，仍然会请求到`app`，直到session终结。
4. 在`app##1`之后，`app##2`之前创建的session，会请求到`app##1`。
5. 而`app##1`和`app##2`也是有类似的处理方式。

【总结】：对应上述的`app`、`app##1`、`app##2`，tomcat中会用3个`ContextVersion`对象来表示，而`MappedContext`是对这3个`ContextVersion`的数组封装。

参考链接

1. [Tomcat如何部署同一应用的不同版本](https://my.oschina.net/jsan/blog/870927)
2. [[tomcat多版本war应用部署(实例讲解)](https://www.cnblogs.com/ggjucheng/archive/2013/04/16/3024450.html)
](https://www.cnblogs.com/ggjucheng/archive/2013/04/16/3024450.html)

### ContextVersion

分析了`ContextVersion`和`MappedContext`的区别和联系，我们接着看看`ContextVersion`，泛型类型为Context

```java
protected static final class ContextVersion extends MapElement<Context> {
    public final String path; // contextPath，上下文路径
    public final int slashCount; // 上下文路径的斜杠数量
    public final WebResourceRoot resources; // 根web资源
    public String[] welcomeResources; // 欢迎资源列表
    public MappedWrapper defaultWrapper = null; // 默认wrapper
    public MappedWrapper[] exactWrappers = new MappedWrapper[0]; // 准确wrapper列表
    public MappedWrapper[] wildcardWrappers = new MappedWrapper[0]; // 通配符wrapper列表
    public MappedWrapper[] extensionWrappers = new MappedWrapper[0]; // 扩展wrapper列表
    public int nesting = 0; // wrapper嵌套层次，用于表示context下wrapper列表中最大的斜杠数
    private volatile boolean paused; // 暂停标记

    public ContextVersion(String version, String path, int slashCount,
            Context context, WebResourceRoot resources,
            String[] welcomeResources) {
        super(version, context);
        this.path = path;
        this.slashCount = slashCount;
        this.resources = resources;
        this.welcomeResources = welcomeResources;
    }

    public boolean isPaused() {
        return paused;
    }

    public void markPaused() {
        paused = true;
    }
}
```

### MappedContext

一个`MappedContext`是`ContextVersion[]`数组的包装。

```java
protected static final class MappedContext extends MapElement<Void> {
    public volatile ContextVersion[] versions;

    public MappedContext(String name, ContextVersion firstVersion) {
        super(name, null);
        this.versions = new ContextVersion[] { firstVersion };
    }
}
```

### ContextList

`ContextList`是`MappedContext[]`数组的封装。同时提供对`MappedContext`的新增和删除操作。

```java
protected static final class ContextList {

    public final MappedContext[] contexts;
    public final int nesting;

    public ContextList() {
        this(new MappedContext[0], 0);
    }

    private ContextList(MappedContext[] contexts, int nesting) {
        this.contexts = contexts;
        this.nesting = nesting;
    }

    public ContextList addContext(MappedContext mappedContext,
            int slashCount) {
        MappedContext[] newContexts = new MappedContext[contexts.length + 1];
        if (insertMap(contexts, newContexts, mappedContext)) {
            return new ContextList(newContexts, Math.max(nesting,
                    slashCount));
        }
        return null;
    }

    public ContextList removeContext(String path) {
        MappedContext[] newContexts = new MappedContext[contexts.length - 1];
        if (removeMap(contexts, newContexts, path)) {
            int newNesting = 0;
            for (MappedContext context : newContexts) {
                newNesting = Math.max(newNesting, slashCount(context.name));
            }
            return new ContextList(newContexts, newNesting);
        }
        return null;
    }
}
```

### MappedHost

该方法有一些重要的属性和特征。包括：

1. ContextList contextList，上下文列表
2. MappedHost realHost
    + 真实的host，一个主机可能有多个别名。
    + 所有`别名的MappedHost`共享一个`非别名的MappedHost`，并存放在这个属性中
3. List<MappedHost> aliases
    + `别名MappedHost列表`。
    + 为了统一处理和简单使用，这个字段只会在`非别名的MappedHost`才有值
    + `别名的MappedHost`，该属性为null

```java
protected static final class MappedHost extends MapElement<Host> {

    public volatile ContextList contextList;

    /**
     * Link to the "real" MappedHost, shared by all aliases.
     * 真实的host，一个主机可能有多个别名。
     * 所有`别名的MappedHost`共享一个`非别名的MappedHost`，并存放在这个属性中
     */
    private final MappedHost realHost;

    /**
     * Links to all registered aliases, for easy enumeration. This field
     * is available only in the "real" MappedHost. In an alias this field
     * is <code>null</code>.
     * 1. `别名MappedHost列表`。
     * 2. 为了统一处理和简单使用，这个字段只会在`非别名的MappedHost`才有值
     * 3. `别名的MappedHost`，该属性为null
     */
    private final List<MappedHost> aliases;

    /**
     * Constructor used for the primary Host
     *
     * @param name The name of the virtual host
     * @param host The host
     */
    public MappedHost(String name, Host host) {
        super(name, host);
        realHost = this;
        contextList = new ContextList();
        aliases = new CopyOnWriteArrayList<>();
    }

    /**
     * Constructor used for an Alias
     *
     * @param alias    The alias of the virtual host
     * @param realHost The host the alias points to
     */
    public MappedHost(String alias, MappedHost realHost) {
        super(alias, realHost.object);
        this.realHost = realHost;
        this.contextList = realHost.contextList;
        this.aliases = null;
    }

    public boolean isAlias() {
        return realHost != this;
    }

    public MappedHost getRealHost() {
        return realHost;
    }

    public String getRealHostName() {
        return realHost.name;
    }

    public Collection<MappedHost> getAliases() {
        return aliases;
    }

    public void addAlias(MappedHost alias) {
        aliases.add(alias);
    }

    public void addAliases(Collection<? extends MappedHost> c) {
        aliases.addAll(c);
    }

    public void removeAlias(MappedHost alias) {
        aliases.remove(alias);
    }
}
```

## 4. MapElement的新增、删除和查询操作

查找相关方法主要为下面几个：

```java
private static final <T, E extends MapElement<T>> E exactFind(E[] map, CharChunk name)
private static final <T, E extends MapElement<T>> E exactFind(E[] map, String name)
private static final <T, E extends MapElement<T>> E exactFindIgnoreCase(E[] map, CharChunk name)
private static final <T> int find(MapElement<T>[] map, CharChunk name)
private static final <T> int find(MapElement<T>[] map, CharChunk name, int start, int end)
private static final <T> int find(MapElement<T>[] map, String name)
private static final <T> int findIgnoreCase(MapElement<T>[] map, CharChunk name)
private static final <T> int findIgnoreCase(MapElement<T>[] map, CharChunk name, int start, int end)
```

在Mapper里面，查询一共有下面三大类方法：

1. `exactFind(xxx)`，查找并提取出name相同的节点
2. `find`，查找并返回name相同的节点的下标
3. `findIgnoreCase`，查找并返回name相同（忽略大小写）的节点的下标

这3类方法都很相似，这儿我们仅仅分析其中的一个方法。

```java
private static final <T> int find(MapElement<T>[] map, String name) {
    // a表示开始位置，默认为0，表示从第一个开始；b表示结束位置，默认为长度-1
    int a = 0;
    int b = map.length - 1;

    // b == -1表示map里面没有元素，返回-1表示未查询到
    // Special cases: -1 and 0
    if (b == -1) {
        return -1;
    }

    // 名字比map里面的第一个元素还小，返回-1表示未查询到
    if (name.compareTo(map[0].name) < 0) {
        return -1;
    }
    // b == 0表示名字比第一个元素大，但还是未查询到
    if (b == 0) {
        return 0;
    }

    // 其他情况，则使用二分查找法来查询元素下标
    int i = 0;
    while (true) {
        i = (b + a) / 2;
        int result = name.compareTo(map[i].name);
        if (result > 0) {
            a = i;
        } else if (result == 0) {
            return i;
        } else {
            b = i;
        }
        if ((b - a) == 1) {
            int result2 = name.compareTo(map[b].name);
            if (result2 < 0) {
                return a;
            } else {
                return b;
            }
        }
    }
}
```

分析完了查找，我们继续分析删除，只有一个方法。先查到name匹配的节点的下标pos，pos之前和之后的元素都拷贝到一个新数组里面，然后返回true表示删除成功。

```java
/**
 * Insert into the right place in a sorted MapElement array.
 */
private static final <T> boolean removeMap
    (MapElement<T>[] oldMap, MapElement<T>[] newMap, String name) {
    int pos = find(oldMap, name);
    if ((pos != -1) && (name.equals(oldMap[pos].name))) {
        System.arraycopy(oldMap, 0, newMap, 0, pos);
        System.arraycopy(oldMap, pos + 1, newMap, pos,
                         oldMap.length - pos - 1);
        return true;
    }
    return false;
}
```

最后我们来分析新增方法，也只有一个方法，即：`insertMap()`。该方法做如下操作：

1. 通过二分查找的方式查找节点名字在oldMap中最近的位置。为啥是最近的呢？是因为待插入的元素会`将后面的元素往后挤一位`。
2. oldMap里面存在同名的节点，则不会插入，对外返回false
3. 数组拷贝到newMap中，并将新节点插入到查找到的位置。这儿需要注意，插入前后的数组都是有序的！

```java
/**
 * 正确插入元素到一个有序的数组，并且防止重复插入
 *
 * Insert into the right place in a sorted MapElement array, and prevent
 * duplicates.
 */
private static final <T> boolean insertMap
    (MapElement<T>[] oldMap, MapElement<T>[] newMap, MapElement<T> newElement) {
    // 通过二分查找的方式查找节点名字在oldMap中最近的位置
    int pos = find(oldMap, newElement.name);
    // oldMap里面存在同名的节点，则不会插入，对外返回false
    if ((pos != -1) && (newElement.name.equals(oldMap[pos].name))) {
        return false;
    }
    // 数组拷贝到newMap中，并将新节点插入到查找到的位置。
    // 这儿需要注意，插入前后的数组都是有序的！
    System.arraycopy(oldMap, 0, newMap, 0, pos + 1);
    newMap[pos + 1] = newElement;
    System.arraycopy
        (oldMap, pos + 1, newMap, pos + 2, oldMap.length - pos - 1);
    return true;
}
```

## 5. 容器的新增、删除和查询操作

我们先来看看`addHost`方法。主要做下面事情：

1. 数组扩容，长度为原数组长度+1
2. 插入成功之后，将再次判断新节点名字和默认主机名是否一致，如果是，则将新节点设置为默认主机。
3. 插入失败，说明host已经存在于hosts里面了，如果是这种情况，则该方法提供幂等支持。否则需要找出重复的节点，并记录错误日志然后返回
4. 如果host有多个别名，需要针对每个别名生成MappedHost对象，并放入主名MappedHost的aliases列表中

```java
/**
 * Add a new host to the mapper.
 * 添加一个新的host到mapper
 *
 * @param name Virtual host name
 * @param aliases Alias names for the virtual host
 * @param host Host object
 */
public synchronized void addHost(String name, String[] aliases,
                                 Host host) {
    name = renameWildcardHost(name);
    // 数组扩容，长度为原数组长度+1
    MappedHost[] newHosts = new MappedHost[hosts.length + 1];
    MappedHost newHost = new MappedHost(name, host);

    // 插入成功之后，将再次判断新节点名字和默认主机名是否一致，如果是，则将新节点设置为默认主机。
    if (insertMap(hosts, newHosts, newHost)) {
        hosts = newHosts;
        if (newHost.name.equals(defaultHostName)) {
            defaultHost = newHost;
        }
        if (log.isDebugEnabled()) {
            log.debug(sm.getString("mapper.addHost.success", name));
        }
    }

    // 插入失败，说明host已经存在于hosts里面了，如果是这种情况，则该方法提供幂等支持。否则需要找出重复的节点，并记录错误日志然后返回
    else {
        MappedHost duplicate = hosts[find(hosts, name)];
        if (duplicate.object == host) {
            // The host is already registered in the mapper.
            // E.g. it might have been added by addContextVersion()
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("mapper.addHost.sameHost", name));
            }
            newHost = duplicate;
        } else {
            log.error(sm.getString("mapper.duplicateHost", name,
                    duplicate.getRealHostName()));
            // Do not add aliases, as removeHost(hostName) won't be able to
            // remove them
            return;
        }
    }

    // 如果host有多个别名，需要针对每个别名生成MappedHost对象，并放入主名MappedHost的aliases列表中
    List<MappedHost> newAliases = new ArrayList<>(aliases.length);
    for (String alias : aliases) {
        alias = renameWildcardHost(alias);
        MappedHost newAlias = new MappedHost(alias, newHost);
        if (addHostAliasImpl(newAlias)) {
            newAliases.add(newAlias);
        }
    }
    newHost.addAliases(newAliases);
}
```

`Context`和`Wrapper`新增的入口都在`addContextVersion()`，我们从这个方法开始分析。该方法实现得非常复杂，逻辑也比较难懂，如若发现楼主有理解偏差，请大家不吝赐教！

实现的功能大致如下：

1. 重新生成主机名
2. 如果主机不在主机列表中，则添加到主机列表。添加之后仍然找不到，则记录错误日志，并终止对context的添加
3. 如果映射的主机为别名主机，则认为host不存在
4. 如果context下面有wrapper，则将其下的所有wrapper也添加到mapper
5. 在mapper中，contextPath可以看成是context的名字，所以这儿使用contextPath来查找
    1. 没有找到，则说明是首次新增context，版本由方法调用时传入。会做以下操作：
        1. 将其添加到MappedHost下面的context列表
        2. 将其添加到mapper的contextObjectToContextVersionMap中
    2. 找到了，也需要尝试重新插入。因为可能存在不同版本的context。
        1. 若插入contextVersion成功，则版本列表更新为扩容后的version数组
        2. 若没有插入convertVersion成功，则执行"覆盖"的相关操作

```java
public void addContextVersion(String hostName, Host host, String path,
        String version, Context context, String[] welcomeResources,
        WebResourceRoot resources, Collection<WrapperMappingInfo> wrappers) {

    // 重新生成主机名
    hostName = renameWildcardHost(hostName);

    // 如果主机不在主机列表中，则添加到主机列表
    MappedHost mappedHost  = exactFind(hosts, hostName);
    if (mappedHost == null) {
        addHost(hostName, new String[0], host);
        // 添加之后仍然找不到，则记录错误日志，并终止对context的添加
        mappedHost = exactFind(hosts, hostName);
        if (mappedHost == null) {
            log.error("No host found: " + hostName);
            return;
        }
    }
    // 如果映射的主机为别名主机，则认为host不存在
    if (mappedHost.isAlias()) {
        log.error("No host found: " + hostName);
        return;
    }
    // 获取contextPath路径的斜杠数
    int slashCount = slashCount(path);

    synchronized (mappedHost) {
        ContextVersion newContextVersion = new ContextVersion(version,
                path, slashCount, context, resources, welcomeResources);
        // 如果context下面有wrapper，则将其下的所有wrapper也添加到mapper
        if (wrappers != null) {
            addWrappers(newContextVersion, wrappers);
        }

        ContextList contextList = mappedHost.contextList;
        // 在mapper中，contextPath可以看成是context的名字，所以这儿使用contextPath来查找
        MappedContext mappedContext = exactFind(contextList.contexts, path);

        // 没有找到，则说明是首次新增context，版本由方法调用时传入。会做以下操作：
        // 1. 将其添加到MappedHost下面的context列表
        // 2. 将其添加到mapper的contextObjectToContextVersionMap中
        if (mappedContext == null) {
            mappedContext = new MappedContext(path, newContextVersion);
            ContextList newContextList = contextList.addContext(
                    mappedContext, slashCount);
            if (newContextList != null) {
                updateContextList(mappedHost, newContextList);
                contextObjectToContextVersionMap.put(context, newContextVersion);
            }
        }

        // 找到了，也需要尝试重新插入。因为可能存在不同版本的context。
        // 1. 若插入contextVersion成功，则版本列表更新为扩容后的version数组
        // 2. 若没有插入convertVersion成功，则执行"覆盖"的相关操作
        else {
            ContextVersion[] contextVersions = mappedContext.versions;
            ContextVersion[] newContextVersions = new ContextVersion[contextVersions.length + 1];
            if (insertMap(contextVersions, newContextVersions,
                    newContextVersion)) {
                mappedContext.versions = newContextVersions;
                contextObjectToContextVersionMap.put(context, newContextVersion);
            } else {
                // Re-registration after Context.reload()
                // Replace ContextVersion with the new one
                // 找到了且名字相同，则认为是reload重新加载操作，则需要对context进行覆盖操作
                int pos = find(contextVersions, version);
                if (pos >= 0 && contextVersions[pos].name.equals(version)) {
                    contextVersions[pos] = newContextVersion;
                    contextObjectToContextVersionMap.put(context, newContextVersion);
                }
            }
        }
    }
}
```

接下来，我们分析一下新增wrapper的方法`addWrapper()`。该方法也很长，但是做的事情却很清楚。

1. 以"/*"结尾，则认为是通配符wrapper，需要将其加到context的wildcardWrappers列表中
2. 以"*."开头，则认为是扩展wrapper，需要将其加到context的extensionWrappers列表中
3. 路径等于"/"，则认为是默认的wrapper，直接赋值为defaultWrapper
4. 其他情况则认为是精确wrapper，需要将其加到context的exactWrappers列表中

```java
protected void addWrapper(ContextVersion context, String path,
        Wrapper wrapper, boolean jspWildCard, boolean resourceOnly) {

    synchronized (context) {
        // 以"/*"结尾，则认为是通配符wrapper，需要将其加到context的wildcardWrappers列表中
        if (path.endsWith("/*")) {
            // Wildcard wrapper
            String name = path.substring(0, path.length() - 2);
            MappedWrapper newWrapper = new MappedWrapper(name, wrapper,
                    jspWildCard, resourceOnly);
            MappedWrapper[] oldWrappers = context.wildcardWrappers;
            MappedWrapper[] newWrappers = new MappedWrapper[oldWrappers.length + 1];
            if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                context.wildcardWrappers = newWrappers;
                int slashCount = slashCount(newWrapper.name);
                if (slashCount > context.nesting) {
                    context.nesting = slashCount;
                }
            }
        }

        // 以"*."开头，则认为是扩展wrapper，需要将其加到context的extensionWrappers列表中
        else if (path.startsWith("*.")) {
            // Extension wrapper
            String name = path.substring(2);
            MappedWrapper newWrapper = new MappedWrapper(name, wrapper,
                    jspWildCard, resourceOnly);
            MappedWrapper[] oldWrappers = context.extensionWrappers;
            MappedWrapper[] newWrappers =
                new MappedWrapper[oldWrappers.length + 1];
            if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                context.extensionWrappers = newWrappers;
            }
        }

        // 路径等于"/"，则认为是默认的wrapper，直接赋值为defaultWrapper
        else if (path.equals("/")) {
            // Default wrapper
            MappedWrapper newWrapper = new MappedWrapper("", wrapper,
                    jspWildCard, resourceOnly);
            context.defaultWrapper = newWrapper;
        }

        // 其他情况则认为是精确wrapper，需要将其加到context的exactWrappers列表中
        else {
            // Exact wrapper
            final String name;
            if (path.length() == 0) {
                // Special case for the Context Root mapping which is
                // treated as an exact match
                name = "/";
            } else {
                name = path;
            }
            MappedWrapper newWrapper = new MappedWrapper(name, wrapper,
                    jspWildCard, resourceOnly);
            MappedWrapper[] oldWrappers = context.exactWrappers;
            MappedWrapper[] newWrappers = new MappedWrapper[oldWrappers.length + 1];
            if (insertMap(oldWrappers, newWrappers, newWrapper)) {
                context.exactWrappers = newWrappers;
            }
        }
    }
}
```

接下来我们分析一下容器查询的方法-`findContextVersion()`。首先查找host，然后查找context，最后查找contextVersion。

```java
private ContextVersion findContextVersion(String hostName,
        String contextPath, String version, boolean silent) {
    // 查找host
    MappedHost host = exactFind(hosts, hostName);
    if (host == null || host.isAlias()) {
        if (!silent) {
            log.error("No host found: " + hostName);
        }
        return null;
    }
    // 查找context
    MappedContext context = exactFind(host.contextList.contexts,
            contextPath);
    if (context == null) {
        if (!silent) {
            log.error("No context found: " + contextPath);
        }
        return null;
    }
    // 查找contextVersion
    ContextVersion contextVersion = exactFind(context.versions, version);
    if (contextVersion == null) {
        if (!silent) {
            log.error("No context version found: " + contextPath + " "
                    + version);
        }
        return null;
    }
    return contextVersion;
}
```

容器相关的`remove()`方法是`add()`方法的逆向操作。`remove()`方法和`add()`方法比较类似，代码也比较多。限于篇幅，本文不再详细描述！

至此，我们基本分析完了Mapper中容器的新增、删除和查询操作。

## 6. Mapper.map()方法

【注】：该方法虽然隶属于Mapper，但是里面的逻辑非常复杂，我们留待第十小节来分析！

## 总结

本文详细分析了`MapperListener`和`Mapper`。`MapperListener`通过监听容器事件来完成对容器的注册和取消注册。而`Mapper`用于对容器进行缓存和管理，同时提供uri映射的功能。在tomcat中这两个组件非常非常地重要，代码也非常多，功能也很复杂，要想完全弄懂，还是需要深入到代码里面去。

不过，楼主相信本文已经将这两个组件的设计和实现给讲清楚了。代码里面，各种设计模式、数据结构和编程思想满天飞。阅读完了他们的代码，我们很容易发现，大牛的软技能和架构设计的功力可是相当的深厚。感叹惊讶的同时，自己也开阔了眼界，楼主可谓是受益匪浅~
