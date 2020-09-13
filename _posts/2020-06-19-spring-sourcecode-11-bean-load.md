---
layout: post
title:  Spring IOC源码解析（11）Bean加载过程
date:   2020-06-19 19:16:21 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## 前言

前面我们分析了spring ioc边缘化的很多组件，这些组件是分析Bean加载过程的基石。基石可理解成下面的意思：

1. 基石是高层的基础
2. 基石可以有很多种，这些基石相互组合、继承、扩展，从而延伸出更多高级的功能
3. 单独的基石只能完成某一部分相对独立的功能，它不是完整的
4. 基石很重要，但是基石的堆砌更重要（即：架构设计的思维）

Spring IOC最核心的功能在于Bean加载过程，也是最能体现架构设计高度的一块内容。网上有很多这方面的分析文章，在借鉴了这些前辈经验的基础上，楼主还是想将源码阅读的过程记录下来，并最终对其进行分析总结。

## 开篇，一个最简单的入门级例子

Spring IOC容器管理的两个类。

```java
public interface MessageService {
    String getMessage();
}

public class MessageServiceImpl implements MessageService {
    public String getMessage() {
        return "hello world";
    }
}
```

xml配置信息。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-autowire="byName">

    <bean id="messageService" class="com.juconcurrent.learn.spring.MessageServiceImpl"/>
</beans>
```

如何使用？

```java
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
        MessageService messageService = context.getBean(MessageService.class);
        System.out.println(messageService.getMessage());
    }
}
```

输出结果：

```
hello world
```

从上面的例子我们看到，Spring IOC有两个口子供我们分析：

1. ClassPathXmlApplicationContext()构造方法
2. context.getBean()方法

## 构造器

先来看看构造方法，其主要做了以下事情：

1. 配置的位置可转换成多个（即：数组），默认为一个
2. `refresh`参数表示是否自动刷新上下文，默认刷新
3. `parent`表示父context，默认没有

```java
/**
 * Create a new ClassPathXmlApplicationContext, loading the definitions
 * from the given XML file and automatically refreshing the context.
 * @param configLocation resource location
 * @throws BeansException if context creation failed
 */
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
	this(new String[] {configLocation}, true, null);
}

/**
 * Create a new ClassPathXmlApplicationContext with the given parent,
 * loading the definitions from the given XML files.
 * @param configLocations array of resource locations
 * @param refresh whether to automatically refresh the context,
 * loading all bean definitions and creating all singletons.
 * Alternatively, call refresh manually after further configuring the context.
 * @param parent the parent context
 * @throws BeansException if context creation failed
 * @see #refresh()
 */
public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {

	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
		refresh();
	}
}
```

我们一直点`super(parent)`，看到最内部的实现如下：

```java
public abstract class AbstractApplicationContext {
    public AbstractApplicationContext(@Nullable ApplicationContext parent) {
        // 设置解析资源文件的策略类对象，可解析多种类型的资源文件，
        // 默认实现为`PathMatchingResourcePatternResolver`。
    	this();
    	// 设置父容器，在父容器不为空的情况下，父容器的`Environment`即被继承过来。
    	setParent(parent);
    }
    
    public AbstractApplicationContext() {
    	this.resourcePatternResolver = getResourcePatternResolver();
    }
    
    protected ResourcePatternResolver getResourcePatternResolver() {
		return new PathMatchingResourcePatternResolver(this);
	}
    
    public void setParent(@Nullable ApplicationContext parent) {
    	this.parent = parent;
    	if (parent != null) {
    		Environment parentEnvironment = parent.getEnvironment();
    		if (parentEnvironment instanceof ConfigurableEnvironment) {
    			getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
    		}
    	}
    }
}
```

接下来看看`setConfigLocations()`。

```java
/**
 * 设置配置的位置。如果不设置，就是空的。
 * Set the config locations for this application context.
 * <p>If not set, the implementation may use a default as appropriate.
 */
public void setConfigLocations(@Nullable String... locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
		    // 这儿会使用`Environment`进行路径解析，
		    // 默认的实现为`StandardEnvironment`。
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}

protected String resolvePath(String path) {
	return getEnvironment().resolveRequiredPlaceholders(path);
}

public ConfigurableEnvironment getEnvironment() {
	if (this.environment == null) {
		this.environment = createEnvironment();
	}
	return this.environment;
}

protected ConfigurableEnvironment createEnvironment() {
	return new StandardEnvironment();
}
```

## refresh方法

核心的代码在于`refresh()`方法。之所以命名成refresh，而不是init之类的方法，有以下几个原因：

1. 该方法方法可反复调用
2. 在配置变动之后，可通过该方法进行重新加载（也就是热加载）
3. refresh英文名为刷新，正符合当前的语境

接下来，我们分析一下这个方法。

```java
public void refresh() throws BeansException, IllegalStateException {
    // 先来一个锁，避免并发刷新的问题
	synchronized (this.startupShutdownMonitor) {
	    // 为刷新而准备上下文
		// Prepare this context for refreshing.
		prepareRefresh();

        // 告诉子类刷新内部的bean工厂，主要做了以下几件事：
        // 1. 如果存在bean工厂，则先销毁其管理的所有bean，并关闭工厂
        // 2. 创建新的bean工厂
        // 3. 加载BeanDefinitions
		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 准备在上下文中使用这个bean工厂
		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// 允许子类重写此方法，以便增加bean工厂创建后的逻辑。默认什么都不做。
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// 调用bean工厂的后置处理器，这些处理器前面已经注册到了上下文
			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// 注册bean后置处理器，在bean的创建过程中会被调用
			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// 初始化消息源
			// Initialize message source for this context.
			initMessageSource();

			// 初始化上下文中的事件广播器
			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// 初始化其他的特殊Bean
			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// 检查并注册监听器bean
			// Check for listener beans and register them.
			registerListeners();

			// 实例化所有剩下的`non-lazy-init`单例
			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// 完成刷新，发布相应的事件
			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// 在加载过程中出现异常的话，将销毁所有已创建的bean，避免资源悬空
			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// 重置active标记为false
			// Reset 'active' flag.
			cancelRefresh(ex);

			// 将异常传递回调用者
			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// 最后会将中间产生的一些缓存清除掉，这些缓存后期可能永远都不会再被使用。
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

这个方法包含了Spring IOC容器初始化的所有操作步骤。当我们读懂了每一行代码，就意味着我们了解了Spring IOC的所有功能。虽然该方法只有寥寥十多行，但是当下钻到每个方法的内部，我们就会发现其调用层级之深，简直令人发指，不过这并不能阻碍我们的钻研。接下来，我们将深入分析每个方法内部的逻辑。

## prepareRefresh方法

该方法用于准备上下文。

```java
/**
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 */
protected void prepareRefresh() {
	// Switch to active.
	// 设置开始时间，设置关闭标记为false，设置激活标记为true
	this.startupDate = System.currentTimeMillis();
	this.closed.set(false);
	this.active.set(true);

    // 简单记录一下日志
	if (logger.isDebugEnabled()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Refreshing " + this);
		}
		else {
			logger.debug("Refreshing " + getDisplayName());
		}
	}

	// 在上下文环境中，初始化属性源（例如：将带占位符的属性源进行替换），
	// 默认为空实现。
	// Initialize any placeholder property sources in the context environment.
	initPropertySources();

	// 在Environment中，校验所有需要的属性（如果那些标注为必需的属性不存在的话，将抛出异常）
	// Validate that all properties marked as required are resolvable:
	// see ConfigurablePropertyResolver#setRequiredProperties
	getEnvironment().validateRequiredProperties();

	// 存储（或者叫设置）预刷新的应用监听器，需要注意以下两点：
	// 1. 如果`earlyApplicationListeners`不为空，说明用户自定义过，那么将不需要使用`applicationListeners`进行覆盖
	// 2. 如果`earlyApplicationListeners`为空，说明用户未自定义过，那么需要根据`applicationListeners`进行设置
	// Store pre-refresh ApplicationListeners...
	if (this.earlyApplicationListeners == null) {
		this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
	}
	else {
		// Reset local application listeners to pre-refresh state.
		this.applicationListeners.clear();
		this.applicationListeners.addAll(this.earlyApplicationListeners);
	}

	// 设置初始的早期应用事件
	// Allow for the collection of early ApplicationEvents,
	// to be published once the multicaster is available...
	this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

根据上面的源码，我们知道这个方法主要做了以下几件事情：

1. 设置开始时间和激活标记；
2. 校验所有需要的属性
3. 初始化应用监听器及应用事件

## obtainFreshBeanFactory方法

该方法用于获取一个新鲜的bean工厂。那么，何为新鲜？

1. 如果已经存在一个bean工厂的话，需要先将其销毁并关闭
2. 创建一个新的bean工厂

那么其内部具体是怎么实现的呢？我们来分析下`obtainFreshBeanFactory()`方法。

```java
/**
 * 先刷新，后获取。
 * Tell the subclass to refresh the internal bean factory.
 * @return the fresh BeanFactory instance
 * @see #refreshBeanFactory()
 * @see #getBeanFactory()
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	return getBeanFactory();
}
```

`refreshBeanFactory()`方法：

```java
/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
 */
@Override
protected final void refreshBeanFactory() throws BeansException {
    // 如果bean工厂存在，那么销毁所有的单例bean，并且关闭bean工厂
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
	    // 创建bean工厂
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		// 设置序列id
		beanFactory.setSerializationId(getId());
		// 自定义bean工厂
		customizeBeanFactory(beanFactory);
		// 加载BeanDefinitions
		loadBeanDefinitions(beanFactory);
		// 设置bean工厂
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

接下来我们分析下`refreshBeanFactory()`方法内部调用的方法，这些方法除了`loadBeanDefinitions()`比较复杂，其他都很简单。

销毁相关的方法如下：

```java
/**
 * Determine whether this context currently holds a bean factory,
 * i.e. has been refreshed at least once and not been closed yet.
 */
protected final boolean hasBeanFactory() {
	synchronized (this.beanFactoryMonitor) {
		return (this.beanFactory != null);
	}
}

// 从这儿看出只是销毁了所有的单例bean，其他scope的bean并没有被销毁
protected void destroyBeans() {
	getBeanFactory().destroySingletons();
}

// 关闭bean工厂，就是将bean工厂设置为null
protected final void closeBeanFactory() {
	synchronized (this.beanFactoryMonitor) {
		if (this.beanFactory != null) {
			this.beanFactory.setSerializationId(null);
			this.beanFactory = null;
		}
	}
}
```

`createBeanFactory()`方法，内部其实就是一个`DefaultListableBeanFactory`的实例。

```java
protected DefaultListableBeanFactory createBeanFactory() {
	return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
```

`customizeBeanFactory`方法，选择性地设置了两个参数：

1. 允许bean定义被覆盖
2. 允许循环依赖

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
	if (this.allowBeanDefinitionOverriding != null) {
		beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
	}
	if (this.allowCircularReferences != null) {
		beanFactory.setAllowCircularReferences(this.allowCircularReferences);
	}
}
```

由于我们分析的入口是`ClassPathXmlApplicationContext`，因此Spring IOC管理的bean是从xml中加载过来的。我们分析的`loadBeanDefinitions()`方法，其内部应该也是如此。

最后有一个赋值语句`this.beanFactory = beanFactory;`，这说明ApplicationContext和BeanFactory的关系，最终还是组合的关系，这样的好处就是松耦合，并且方便后续扩展和维护。

## loadBeanDefinitions()

```java
/**
 * Loads the bean definitions via an XmlBeanDefinitionReader.
 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 * @see #initBeanDefinitionReader
 * @see #loadBeanDefinitions
 */
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 通过bean工厂，实例化一个`XmlBeanDefinitionReader`，表示xml下BeanDefinition的读取器
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// 配置bean定义读取器，包括：环境对象、资源加载器、资源实体处理器
	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	// 初始化
	initBeanDefinitionReader(beanDefinitionReader);
	// 加载
	loadBeanDefinitions(beanDefinitionReader);
}
```

`initBeanDefinitionReader()`用于初始化操作，主要设置校验开关。

1. 当校验的时候，校验模式设置为自动
2. 当不校验的时候，校验模式设置为“不校验”，同时设置命名空间验证，这样的话，就从侧面进行了验证。

```java
protected void initBeanDefinitionReader(XmlBeanDefinitionReader reader) {
	reader.setValidating(this.validating);
}
/**
 * Set whether to use XML validation. Default is {@code true}.
 * <p>This method switches namespace awareness on if validation is turned off,
 * in order to still process schema namespaces properly in such a scenario.
 * @see #setValidationMode
 * @see #setNamespaceAware
 */
public void setValidating(boolean validating) {
	this.validationMode = (validating ? VALIDATION_AUTO : VALIDATION_NONE);
	this.namespaceAware = !validating;
}
```

重载的`loadBeanDefinitions()`方法传入参数为reader。

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		reader.loadBeanDefinitions(configLocations);
	}
}
```

从上面的代码我们看出，xml配置加载的方式有两种，一种是以通用资源的方式，另一种是以配置位置的方式。配置位置的方式，最终将转换为通用资源的方式。接下来，我们分析下配置位置的方式。

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int count = 0;
	for (String location : locations) {
		count += loadBeanDefinitions(location);
	}
	return count;
}

public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}

/**
 * Load bean definitions from the specified resource location.
 * <p>The location can also be a location pattern, provided that the
 * ResourceLoader of this bean definition reader is a ResourcePatternResolver.
 * @param location the resource location, to be loaded with the ResourceLoader
 * (or ResourcePatternResolver) of this bean definition reader
 * @param actualResources a Set to be filled with the actual Resource objects
 * that have been resolved during the loading process. May be {@code null}
 * to indicate that the caller is not interested in those Resource objects.
 * @return the number of bean definitions found
 * @throws BeanDefinitionStoreException in case of loading or parsing errors
 * @see #getResourceLoader()
 * @see #loadBeanDefinitions(org.springframework.core.io.Resource)
 * @see #loadBeanDefinitions(org.springframework.core.io.Resource[])
 */
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
	ResourceLoader resourceLoader = getResourceLoader();
	if (resourceLoader == null) {
		throw new BeanDefinitionStoreException(
				"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
	}

	if (resourceLoader instanceof ResourcePatternResolver) {
		// Resource pattern matching available.
		try {
		    // 关键的代码在这儿，通过资源加载器将配置位置转换成了通用资源。
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
			int count = loadBeanDefinitions(resources);
			if (actualResources != null) {
				Collections.addAll(actualResources, resources);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
			}
			return count;
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Could not resolve bean definition resource pattern [" + location + "]", ex);
		}
	}
	else {
		// Can only load single resources by absolute URL.
		Resource resource = resourceLoader.getResource(location);
		int count = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
		}
		return count;
	}
}
```

通用资源的BeanDefinition加载过程非常地复杂，因为xml配置支持的标签非常多。限于篇幅，本文不做深入的分析。

在阅读源码的时候，楼主发现`BeanDefinition`有好几个实现类，如下所示。我们着重分析红框内的类。

![图片.png](https://upload-images.jianshu.io/upload_images/845143-799b5217d281f774.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

他们的区别大致是这样的。

+ `AnnotatedBeanDefinition`，包含有注解`@Bean`的Bean定义。一般情况下，通过注解的方式得到Bean定义，类型都是该接口的实现类。
+ `AnnotatedGenericBeanDefinition`，有`@Configuration`注解的类，会生成`AnnotatedGenericBeanDefinition`类型的Bean定义。
+ `ScannedGenericBeanDefinition`，有`@Component`注解的类，会生成`ScannedGenericBeanDefinition`类型的Bean定义。注意其它继承了`@Component`的注解，同`@Component`。
+ `RootBeanDefinition`、`ChildBeanDefinition`、`GenericBeanDefinition`，这几个类均继承了`AbstractBeanDefinition`。从官方文档里，我们得到了以下一些信息：

> `RootBeanDefinition`是最常用的实现类，它对应一般性的`<bean>`元素标签，`GenericBeanDefinition`是自2.5以后新加入的bean文件配置属性定义类，是一站式服务类。在配置文件中可以定义`父<bean>`和`子<bean>`，`父<bean>`用`RootBeanDefinition`表示，而`子<bean>`用`ChildBeanDefiniton`表示，而没有`父<bean>`的`<bean>`就使用`RootBeanDefinition`表示。`AbstractBeanDefinition`对两者共同的类信息进行了抽象。

## prepareBeanFactory方法

用于设置一些bean工厂相关的信息。例如：

1. bean的类加载器、表达式处理、属性编辑器注册器
2. 手动添加了一些bean后置处理器
3. 指定某些依赖接口不被spring ioc容器所管理
4. 手动注入一些关键的bean，例如bean工厂、应用上下文、环境相关的bean等等

```java
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * @param beanFactory the BeanFactory to configure
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// Tell the internal bean factory to use the context's class loader etc.
	// 设置bean的类加载器
	beanFactory.setBeanClassLoader(getClassLoader());
	// 设置bean的表达式处理器
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	// 设置属性编辑器注册器
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// Configure the bean factory with context callbacks.
	// 添加bean后置处理器，用于给管理的bean设置“应用上下文”
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	// 忽略依赖接口，凡是继承这些接口的类实例将不会被spring ioc容器管理
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// BeanFactory interface not registered as resolvable type in a plain factory.
	// MessageSource registered (and found for autowiring) as a bean.
	// 手动注入`BeanFactory`、`ResourceLoader`、`ApplicationEventPublisher`和`ApplicationContext`。
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
	// 添加bean后置处理器，用于探测内部bean
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
	// 添加bean后置处理器，用于AOP运行时织入
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// Register default environment beans.
	// 注册一些默认的环境相关的单例bean
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```

## postProcessBeanFactory方法

该方法用于bean工厂加载后的后置处理，默认为空实现。

```java
/**
 * Modify the application context's internal bean factory after its standard
 * initialization. All bean definitions will have been loaded, but no beans
 * will have been instantiated yet. This allows for registering special
 * BeanPostProcessors etc in certain ApplicationContext implementations.
 * @param beanFactory the bean factory used by the application context
 */
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```

## invokeBeanFactoryPostProcessors方法

该方法用于调用bean工厂的后置处理器（已注册的）。

```java
/**
 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before singleton instantiation.
 */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	// 运行时切面织入相关的配置
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
```

`PostProcessorRegistrationDelegate`是一个工具类，其下的`invokeBeanFactoryPostProcessors`方法非常复杂。为啥会这么复杂，主要是由于以下几个方面：

1. `BeanFactoryPostProcessor`可以被配置；
2. `BeanFactoryPostProcessor`有一个特殊的子类`BeanDefinitionRegistryPostProcessor`，其用途就是方便`BeanFactoryPostProcessor`可以灵活地配置；
3. `BeanFactoryPostProcessor`可以定义`优先级`、`顺序性`，以及`无顺序性`。

接下来，我们具体看看源代码。

```java
// 该类是final修饰，说明其不可继承。同时其构造器是私有的，说明不能被实例化。因此其就是一个静态工具类。
final class PostProcessorRegistrationDelegate {

	private PostProcessorRegistrationDelegate() {
	}

	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();

		// 因为beanFactory的默认实现为`DefaultListableBeanFactory`，所以本身就继承了`BeanDefinitionRegistry`。
		// 这儿，我们仅分析if语句块就可以了。
		if (beanFactory instanceof BeanDefinitionRegistry) {
		    // bean工厂就是`BeanDefinitionRegistry`
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			// 分离出标准的后置处理器，以及注册器后置处理器
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			// 1. 工厂bean不会被实例化
			// 2. 优先处理注册器后置处理器
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			// 第一步，处理实现了`PriorityOrdered`接口的`BeanDefinitionRegistryPostProcessor`，包括：
			// 1. 排序
			// 2. 添加到`registryProcessors`，方便后续统一处理
			// 3. 调用`postProcessBeanDefinitionRegistry`方法
			// 4. 清理currentRegistryProcessors，方便后续复用
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			// 第二步，处理实现了`Ordered`接口的`BeanDefinitionRegistryPostProcessor`，处理步骤和第一步相同
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
			// 最终，调用所有除了实现`PriorityOrdered`和`Ordered`的`BeanDefinitionRegistryPostProcessors`，
			// 处理步骤和第一步相同
			// 注意，这儿使用了while循环，而不是if，楼主也不知道是为什么
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			// OK，上面的都做完了，就可以开始调用bean工厂的后置处理器了
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// 除了上面的bean工厂后置处理器，我们还可以自己配置一些bean工厂后置处理器，例如：
		// 1. 在xml中通过<bean>的方式
		// 2. 在java config中通过@Bean的方式
		// 3. 在自动装配中通过@Componet的方式
		// 因此，在这儿我们需要对这些后置处理器进行处理。
		// 同样地，根据是否是`PriorityOrdered`、`Ordered`，会有特殊的处理。
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
}
```

## registerBeanPostProcessors方法

该方法用于注册bean后置处理器，最终调用的仍然是`PostProcessorRegistrationDelegate`中的工具方法，我们来看一下。

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	// Register BeanPostProcessorChecker that logs an info message when
	// a bean is created during BeanPostProcessor instantiation, i.e. when
	// a bean is not eligible for getting processed by all BeanPostProcessors.
	// 这儿会做两件事情：
	// 1. 统计bean后置处理器的个数，格式是根据以下地方进行统计的
	//     1. bean工厂内部的
	//     2. 随后添加的一个检查bean后置处理器
	//     3. 用户自己定义的
	// 2. 添加一个bean后置处理器，用于检查目的，当检查不通过时，将记录info级别的日志
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// Separate between BeanPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	// 分离出`PriorityOrdered`、`Ordered`和剩下的bean后置处理器
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, register the BeanPostProcessors that implement PriorityOrdered.
	// 第一步，排序并重新注册到bean工厂（PriorityOrdered）
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// Next, register the BeanPostProcessors that implement Ordered.
	// 第二步，排序并重新注册到bean工厂（Ordered）
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);

	// Now, register all regular BeanPostProcessors.
	// 第三步，注册剩余的到bean工厂
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

	// Finally, re-register all internal BeanPostProcessors.
	// 最后一步，重新注册内部的到bean工厂，目的是将他们排序到最后位置
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);

	// Re-register post-processor for detecting inner beans as ApplicationListeners,
	// moving it to the end of the processor chain (for picking up proxies etc).
	// 添加一个事件监听的bean后置处理器，但是不纳入bean后置处理器个数的统计
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

## initMessageSource方法

该方法用于初始化消息源。所谓消息源，是为了支持国际化而服务的，本文我们不会过多地对此进行分析。

## initApplicationEventMulticaster方法

该方法用于初始化应用事件广播器。

```java
/**
 * Initialize the ApplicationEventMulticaster.
 * Uses SimpleApplicationEventMulticaster if none defined in the context.
 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
 */
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	// 如果bean工厂包含应用事件广播器，那么就使用它
	// 否则，使用`SimpleApplicationEventMulticaster`创建一个默认的应用事件广播器
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isTraceEnabled()) {
			logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
					"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
		}
	}
}
```

## onRefresh方法

该方法用于初始化其他特殊的bean，默认为空实现

## registerListeners方法

用于注册应用监听器到应用事件广播器中。

```java
/**
 * Add beans that implement ApplicationListener as listeners.
 * Doesn't affect other listeners, which can be added without being beans.
 */
protected void registerListeners() {
	// Register statically specified listeners first.
	// 注册静态的监听器
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let post-processors apply to them!
	// 注册自定义的监听器
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// Publish early application events now that we finally have a multicaster...
	// 对预设置的事件进行广播
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
```

## finishBeanFactoryInitialization方法

用于实例化所有剩余的非懒加载的单例bean。

```java
/**
 * Finish the initialization of this context's bean factory,
 * initializing all remaining singleton beans.
 */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	// 初始化转换服务
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	// 如果没有嵌套值处理器，那么注册一个默认的嵌套值处理器
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	// 初始化运行时织入的bean
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	// 停止临时的类加载器
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	// 冻结配置（即：配置不可再被修改）
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	// 调用工厂方法`preInstantiateSingletons`，预实例化所有的单例bean。
	// 该方法非常关键，其内部解决循环依赖问题的思路非常有意思。
	beanFactory.preInstantiateSingletons();
}
```

## preInstantiateSingletons方法

该方法用于初始化所有的单例bean，其最最核心和关键的内容在于使用三级缓存解决循环依赖的问题。

那么，Spring为什么能解决循环依赖的问题呢？网上有很多这方面的分析，各种前因后果，细细道来冗长而繁琐。

> 为了便于理解，楼主直接说结果。能解决的根本原因在于`引用传递`。对象初始化之后，其引用就已经确定下来了，后期也不会再变化，因此引用可以被其他地方使用。我们可以将初始化的过程分成两个阶段：
1. 构造器创建对象（创建a对象，传播b对象）
2. 设置依赖关系（给a对象设置b对象引用，给b对象设置a对象引用）

在此基础上，Spring衍生出了三级缓存，其仅支持单例的循环依赖。

1. 不能解决的情况：
    1. 构造器注入循环依赖
    2. `prototype` field属性注入（或者setter方法注入）循环依赖
2. 能解决的情况：
    1. field属性注入（或者setter方法注入）循环依赖

好了，步入正轨，我们开始分析这个方法。

```java
public void preInstantiateSingletons() throws BeansException {
    // 简单记录一下跟踪日志
	if (logger.isTraceEnabled()) {
		logger.trace("Pre-instantiating singletons in " + this);
	}

	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	// bean定义的名称，就是bean的名称
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
	    // 合并bean定义
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		// 只有非抽象的、单例的、非懒加载的bean定义，才会实例化成bean，并注册到bean工厂
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
		    // 1. 如果是工厂bean，同时为预加载的话，那么直接调用`getBean`方法加载bean
		    // 2. 如果不是工厂bean，直接调用`getBean`方法加载bean
			if (isFactoryBean(beanName)) {
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}

	// Trigger post-initialization callback for all applicable beans...
	// bean加载之后，如果bean为`SmartInitializingSingleton`类型，那么将触发方法`afterSingletonsInstantiated()`
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```

`getBean()`方法至关重要，用于获取一个bean。其调用的是`doGetBean()`方法，根据名称获取一个bean。

```java
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```

接下来，我们分析一下`doGetBean()`方法。

```java
/**
 * Return an instance, which may be shared or independent, of the specified bean.
 * @param name the name of the bean to retrieve
 * @param requiredType the required type of the bean to retrieve
 * @param args arguments to use when creating a bean instance using explicit arguments
 * (only applied when creating a new instance as opposed to retrieving an existing one)
 * @param typeCheckOnly whether the instance is obtained for a type check,
 * not for actual use
 * @return an instance of the bean
 * @throws BeansException if the bean could not be created
 */
@SuppressWarnings("unchecked")
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
		@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 获取bean最初的名称，因为传入的bean名称可能是一个别名，也可能是一个工厂bean的名称
	final String beanName = transformedBeanName(name);
	Object bean;

	// Eagerly check singleton cache for manually registered singletons.
	// 这儿尝试从三级缓存获取单例对象
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isTraceEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		// 从三级缓存获取到之后，将尝试将bean手动注册到bean工厂
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		// 如果三级缓存没获取到，且bean为prototype类型，那么抛出异常
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		// 如果当前bean工厂不包含bean的定义，则委托给父工厂进行get处理
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else if (requiredType != null) {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
			else {
				return (T) parentBeanFactory.getBean(nameToLookup);
			}
		}

        // 如果不仅仅是类型检查，那么将beanName放入已创建的bean名称集合里面
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
			// 这就是上面方法的实现
			/*
			protected void markBeanAsCreated(String beanName) {
                if (!this.alreadyCreated.contains(beanName)) {
                	synchronized (this.mergedBeanDefinitions) {
                		if (!this.alreadyCreated.contains(beanName)) {
                			// Let the bean definition get re-merged now that we're actually creating
                			// the bean... just in case some of its metadata changed in the meantime.
                			clearMergedBeanDefinition(beanName);
                			this.alreadyCreated.add(beanName);
                		}
                	}
                }
            }
            */
		}

		try {
		    // 获取合并的bean定义，并检查（这儿判断是否为abstract）
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			// 这儿会判断是否存在构造器的循环依赖，如果存在则抛出异常，否则的话就先注册依赖关系，再get这个bean
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					registerDependentBean(dep, beanName);
					try {
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}

			// Create bean instance.
			// 创建单例
			if (mbd.isSingleton()) {
			    // 创建单例的时候，会传入一个ObjectFactory的对象。首先它是一个工厂，然后他会创一个对象。
				sharedInstance = getSingleton(beanName, () -> {
					try {
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						// Explicitly remove instance from singleton cache: It might have been put there
						// eagerly by the creation process, to allow for circular reference resolution.
						// Also remove any beans that received a temporary reference to the bean.
						destroySingleton(beanName);
						throw ex;
					}
				});
				// 获取到bean的时候，为啥不直接返回，原因在于获取到的bean可能是一个工厂bean，工厂bean又可能是工厂bean的工厂bean。
				// 因此，这个方法就是解决这个问题而存在的。
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}
            
            // 创建prototype
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

            // 创建其他Scope
			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	// 如果类型不匹配，则需要通过转换器进行转换。当转换不成功，则抛出异常。
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			if (logger.isTraceEnabled()) {
				logger.trace("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```

重点来了！

在分析完`doGetBean()`方法后，我们发现不管是当前bean，还是依赖的bean，最终调用的都是`getSingleton()`方法。该方法有好几个重载方法，我们依次来分析。

```java
/** Cache of singleton objects: bean name to bean instance. */
// 一级缓存，缓存已实例化的bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of early singleton objects: bean name to bean instance. */
// 二级缓存，缓存`early`（可提前暴露）的bean
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

/** Cache of singleton factories: bean name to ObjectFactory. */
// 三级缓存，缓存对象工厂对象
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Set of registered singletons, containing the bean names in registration order. */
// 已注册的单例bean名称
private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

/** Names of beans that are currently in creation. */
// 当前正在创建的单例bean名称
private final Set<String> singletonsCurrentlyInCreation =
		Collections.newSetFromMap(new ConcurrentHashMap<>(16));

// 调用的是下面那个方法，且参数`allowEarlyReference`为true
public Object getSingleton(String beanName) {
	return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 1. 先从一级缓存`singletonObjects`中去获取。（如果获取到就直接return）
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
		    // 2. 如果获取不到或者对象正在创建中（isSingletonCurrentlyInCreation()），那就再从二级缓存earlySingletonObjects中获取。（如果获取到就直接return）
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
			    // 3. 如果还是获取不到，且允许`singletonFactories`（`allowEarlyReference = true`）通过`getObject()`获取。
			    // 就从三级缓存`singletonFactory.getObject()`获取。
			    // （如果获取到了就从`singletonFactories`中移除，并且放进`earlySingletonObjects`。
			    // 其实也就是从三级缓存移动（是剪切、不是复制哦~）到了二级缓存）
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return singletonObject;
}

/**
 * 相比较于上个方法，多了错误校验、异常处理、外置化`ObjectFactory`创建对象的操作。
 * Return the (raw) singleton object registered under the given name,
 * creating and registering a new one if none registered yet.
 * @param beanName the name of the bean
 * @param singletonFactory the ObjectFactory to lazily create the singleton
 * with, if necessary
 * @return the registered singleton object
 */
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
			try {
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName);
			}
			if (newSingleton) {
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
```

简单总结一下，Spring解决循环依赖的三级缓存逻辑如下：

> 1. 先从一级缓存`singletonObjects`中去获取。（如果获取到就直接return）
> 2. 如果获取不到或者对象正在创建中（isSingletonCurrentlyInCreation()），那就再从二级缓存earlySingletonObjects中获取。（如果获取到就直接return）
> 3. 如果还是获取不到，且允许`singletonFactories`（`allowEarlyReference = true`）通过`getObject()`获取。就从三级缓存`singletonFactory.getObject()`获取。（如果获取到了就从`singletonFactories`中移除，并且放进`earlySingletonObjects`。其实也就是从三级缓存移动（是剪切、不是复制哦~）到了二级缓存）。

## 生命周期

在分析`finishRefresh()`方法之前，我们有必要澄清一下，何为生命周期？

所谓生命周期，其实是一系列的回调。包括：

1. Initialization callbacks（初始化回调）
2. Destruction callbacks（销毁回调）

要与容器的bean生命周期管理交互，即容器在启动后和容器在销毁前对每个bean执行操作，有如下三种方法:

1. 实现Spring框架的`InitializingBean`和`DisposableBean`接口。容器为前者调用`afterPropertiesSet()`方法，为后者调用`destroy()`方法，以允许bean在初始化和销毁的时候执行某些操作。

```java
public class HelloLifeCycle implements InitializingBean, DisposableBean {
    
    public void afterPropertiesSet() throws Exception {
        System.out.println("afterPropertiesSet 启动");
    }

    public void destroy() throws Exception {
        System.out.println("DisposableBean 停止");
    }
}
```

2. 在xml配置中，bean在定义的时候，指定初始化方法和销毁方法。

```xml
 <bean id="helloLifeCycle" class="com.hzways.life.cycle.HelloLifeCycle" init-method="init3" destroy-method="destroy3"/>
```

3. JSR-250中，`@PostConstruct`和`@PreDestroy`注解，可用于bean的初始化后和销毁前操作。
  + `@PostConstruct`注解用于方法上，该方法在初始化完成之后被执行。
  + `@PreDestroy`注解用于方法上，该方法在销毁前被执行。

```java
@Service
public class HelloLifeCycle {

    @PostConstruct
    private void init2() {
        System.out.println("PostConstruct 启动");
    }

    @PreDestroy
    private void destroy2() {
        System.out.println("PreDestroy 停止");
    }
}
```

## finishRefresh方法

该方法主要调用了生命周期处理器的`onRefresh()`方法。

```java
/**
 * Finish the refresh of this context, invoking the LifecycleProcessor's
 * onRefresh() method and publishing the
 * {@link org.springframework.context.event.ContextRefreshedEvent}.
 */
protected void finishRefresh() {
	// Clear context-level resource caches (such as ASM metadata from scanning).
	// 清除上下文级别的资源缓存
	clearResourceCaches();

	// Initialize lifecycle processor for this context.
	// 初始化生命周期处理器
	initLifecycleProcessor();

	// Propagate refresh to lifecycle processor first.
	// 传播刷新操作到声明周期处理器
	getLifecycleProcessor().onRefresh();

	// Publish the final event.
	// 发布事件
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	// MBean管理相关
	LiveBeansView.registerApplicationContext(this);
}
```

这儿我们着重关注两个方法：`initLifecycleProcessor()`和`getLifecycleProcessor().onRefresh()`。

## initLifecycleProcessor方法

该方法很简单。

1. 首先尝试从bean工厂获取生命周期处理器；
2. 如果bean工厂未获取到，初始化一个默认的生命周期处理器(`DefaultLifecycleProcessor`)，并注册到bean工厂；
3. 在前面两部的基础上，设置处理到`AbstractApplicationContext`的成员变量`lifecycleProcessor`上，供后续使用。

```java
/**
 * Initialize the LifecycleProcessor.
 * Uses DefaultLifecycleProcessor if none defined in the context.
 * @see org.springframework.context.support.DefaultLifecycleProcessor
 */
protected void initLifecycleProcessor() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
		this.lifecycleProcessor =
				beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
		}
	}
	else {
		DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
		defaultProcessor.setBeanFactory(beanFactory);
		this.lifecycleProcessor = defaultProcessor;
		beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
		if (logger.isTraceEnabled()) {
			logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
					"[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
		}
	}
}
```

## onRefresh方法

分析该方法前，先看看`getLifecycleProcessor()`，其实就是检查并返回成员变量`lifecycleProcessor`。

```java
LifecycleProcessor getLifecycleProcessor() throws IllegalStateException {
	if (this.lifecycleProcessor == null) {
		throw new IllegalStateException("LifecycleProcessor not initialized - " +
				"call 'refresh' before invoking lifecycle methods via the context: " + this);
	}
	return this.lifecycleProcessor;
}
```

```java
public void onRefresh() {
	startBeans(true);
	this.running = true;
}
```

`onRefresh()`方法调用了`startBeans`方法，之后更新了运行状态。

```java
private void startBeans(boolean autoStartupOnly) {
    // 获取所有Bean工厂管理的`Lifecycle`对象
	Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
	Map<Integer, LifecycleGroup> phases = new HashMap<>();
	// 根据`phase`进行分组
	lifecycleBeans.forEach((beanName, bean) -> {
		if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
			int phase = getPhase(bean);
			LifecycleGroup group = phases.get(phase);
			if (group == null) {
				group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
				phases.put(phase, group);
			}
			group.add(beanName, bean);
		}
	});
	// 根据分组，调用对象的`start()`方法
	if (!phases.isEmpty()) {
		List<Integer> keys = new ArrayList<>(phases.keySet());
		Collections.sort(keys);
		for (Integer key : keys) {
			phases.get(key).start();
		}
	}
}
```

`startBeans()`方法做了以下事情：

1. 首先获取所有Bean工厂管理的`Lifecycle`对象；
2. 对`Lifecycle`对象根据`phase`（阶段的意思，其实是一个int值）进行分组。相同`phase`的对象划到一个组；
3. 根据`phase`的值大小，升序调用其`Lifecycle`对象的`start()`方法。

`getLifecycleBeans()`方法用于获取对象。

```java
/**
 * Retrieve all applicable Lifecycle beans: all singletons that have already been created,
 * as well as all SmartLifecycle beans (even if they are marked as lazy-init).
 * @return the Map of applicable beans, with bean names as keys and bean instances as values
 */
protected Map<String, Lifecycle> getLifecycleBeans() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	Map<String, Lifecycle> beans = new LinkedHashMap<>();
	// 获取所有类型为`Lifecycle`的bean名称
	String[] beanNames = beanFactory.getBeanNamesForType(Lifecycle.class, false, false);
	for (String beanName : beanNames) {
		String beanNameToRegister = BeanFactoryUtils.transformedBeanName(beanName);
		boolean isFactoryBean = beanFactory.isFactoryBean(beanNameToRegister);
		String beanNameToCheck = (isFactoryBean ? BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
		// 这儿对bean加了一些限制条件
		// 1. 如果是单例的话，不能是工厂bean，且类型必须为`Lifecycle`
		// 2. 如果不是单例的话，类型必须为`SmartLifecycle`
		if ((beanFactory.containsSingleton(beanNameToRegister) &&
				(!isFactoryBean || matchesBeanType(Lifecycle.class, beanNameToCheck, beanFactory))) ||
				matchesBeanType(SmartLifecycle.class, beanNameToCheck, beanFactory)) {
			Object bean = beanFactory.getBean(beanNameToCheck);
			if (bean != this && bean instanceof Lifecycle) {
				beans.put(beanNameToRegister, (Lifecycle) bean);
			}
		}
	}
	return beans;
}
```

最关键的地方在于`start()`方法的调用。

```java
public void start() {
	if (this.members.isEmpty()) {
		return;
	}
	if (logger.isDebugEnabled()) {
		logger.debug("Starting beans in phase " + this.phase);
	}
	// 对分组内的bean按照名称进行排序，并最终调用了`doStart()`方法
	Collections.sort(this.members);
	for (LifecycleGroupMember member : this.members) {
		doStart(this.lifecycleBeans, member.name, this.autoStartupOnly);
	}
}

/**
 * Start the specified bean as part of the given set of Lifecycle beans,
 * making sure that any beans that it depends on are started first.
 * @param lifecycleBeans a Map with bean name as key and Lifecycle instance as value
 * @param beanName the name of the bean to start
 */
private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
	Lifecycle bean = lifecycleBeans.remove(beanName);
	if (bean != null && bean != this) {
		String[] dependenciesForBean = getBeanFactory().getDependenciesForBean(beanName);
		for (String dependency : dependenciesForBean) {
		    // 这儿有递归调用。
		    // 需要特别注意的地方在于，如果一个`Lifecycle`依赖另外一个`Lifecycle`，则会优先调用被依赖的`Lifecycle`。
			doStart(lifecycleBeans, dependency, autoStartupOnly);
		}
		if (!bean.isRunning() &&
				(!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle) bean).isAutoStartup())) {
			if (logger.isTraceEnabled()) {
				logger.trace("Starting bean '" + beanName + "' of type [" + bean.getClass().getName() + "]");
			}
			try {
			    // 庐山真面目，最终调到了`Lifecycle`的`start`方法。
				bean.start();
			}
			catch (Throwable ex) {
				throw new ApplicationContextException("Failed to start bean '" + beanName + "'", ex);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Successfully started bean '" + beanName + "'");
			}
		}
	}
}
```

至此，`onRefresh()`方法分析完成。

## destroyBeans方法

该方法用于销毁bean，内部调用的是bean工厂的`destroySingletons()`方法。【注意】：这儿仅销毁了单例bean！

## 总结

我们从源码层面分析了bean加载的整个过程，每个步骤各司其职，环环相扣，非常地精彩。其中最显眼的地方在于Spring对bean循环依赖的解决方案（三级缓存）。

借鉴于网上的一张图，bean加载的过程如下所示：

![](https://upload-images.jianshu.io/upload_images/460263-74d88a767a80843a.png)

而这张图标绿的地方，单独解释如下：

+ `获取 BeanName`，对传入的 name 进行解析，转化为可以从 Map 中获取到`BeanDefinition`的 bean name。
+ `合并 Bean 定义`，对父类的定义进行合并和覆盖，如果父类还有父类，会进行递归合并，以获取完整的`Bean`定义信息。
+ `实例化`，使用构造或者工厂方法创建`Bean`实例。
属性填充，寻找并且注入依赖，依赖的`Bean`还会递归调用`getBean`方法获取。
+ `初始化`，调用自定义的初始化方法。
+ `获取最终的 Bean`，如果是`FactoryBean`需要调用`getObject`方法，如果需要类型转换调用`TypeConverter`进行转化。

除此之外，我们还需要关注一下bean的生命周期，这是贯穿Spring IOC的一条主线。

1. 实例化bean
2. 设置属性值
3. 调用`BeanNameAware.setBeanName()`方法
4. 调用`BeanFactoryAware.setBeanFactory()`方法
5. 调用`BeanPostProcessor.postProcessBeforeInitialization()`方法
    1. 调用`ApplicationContextAware.setApplicationContext()`方法
    2. 调用`@PostConstruct`所在的方法
    3. 其他...
6. 调用`InitializingBean.afterPropertiesSet()`方法
7. 调用init方法
8. 调用`BeanPostProcessor.postProcessAfterInitialization()`方法
9. 调用`DisposableBean.destroy()`方法
10. 调用destory方法

## 参考文档

1. https://blog.csdn.net/nuomizhende45/article/details/81158383
2. https://www.jianshu.com/p/9ea61d204559
3. https://www.cnblogs.com/lqmblog/p/8592817.html
4. https://www.jianshu.com/p/e4ca039a2272
5. https://www.jianshu.com/p/a6a03d94d6f7
6. https://www.jianshu.com/p/43b65ed2e166
7. https://blog.csdn.net/f641385712/article/details/92801300