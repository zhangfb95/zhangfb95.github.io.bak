---
layout: post
title:  Spring IOC源码解析（05）BeanDefinitionRegistry
date:   2020-05-26 13:31:07 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## BeanDefinitionRegistry

在阐释`BeanDefinitionRegistry`之前，我们有必要简单提一下`BeanDefinition`。`BeanDefinition`是spring容器中bean的定义，这些定义可以是xml配置，可以是注解配置，还可以是别的。

`BeanDefinitionRegistry`是一个接口，中文名为`BeanDefinition注册器`，继承了`AliasRegistry`接口，同时增加了对`BeanDefinition`的管理。

因此，我们可以说，这个接口包含以下几方面的功能：

1. `BeanDefinition`管理的功能
2. 包含对别名管理的功能

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

	/**
	 * 注册BeanDefinition
	 * Register a new bean definition with this registry.
	 * Must support RootBeanDefinition and ChildBeanDefinition.
	 * @param beanName the name of the bean instance to register
	 * @param beanDefinition definition of the bean instance to register
	 * @throws BeanDefinitionStoreException if the BeanDefinition is invalid
	 * @throws BeanDefinitionOverrideException if there is already a BeanDefinition
	 * for the specified bean name and we are not allowed to override it
	 * @see GenericBeanDefinition
	 * @see RootBeanDefinition
	 * @see ChildBeanDefinition
	 */
	void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;

	/**
	 * 移除BeanDefinition
	 * Remove the BeanDefinition for the given name.
	 * @param beanName the name of the bean instance to register
	 * @throws NoSuchBeanDefinitionException if there is no such bean definition
	 */
	void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	/**
	 * 根据bean名称获取BeanDefinition
	 * Return the BeanDefinition for the given bean name.
	 * @param beanName name of the bean to find a definition for
	 * @return the BeanDefinition for the given name (never {@code null})
	 * @throws NoSuchBeanDefinitionException if there is no such bean definition
	 */
	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	/**
	 * 判断是否包含bean名称
	 * Check if this registry contains a bean definition with the given name.
	 * @param beanName the name of the bean to look for
	 * @return if this registry contains a bean definition with the given name
	 */
	boolean containsBeanDefinition(String beanName);

	/**
	 * 获取BeanDefinition的所有名称数组
	 * Return the names of all beans defined in this registry.
	 * @return the names of all beans defined in this registry,
	 * or an empty array if none defined
	 */
	String[] getBeanDefinitionNames();

	/**
	 * 获取BeanDefinition的数量
	 * Return the number of beans defined in the registry.
	 * @return the number of beans defined in the registry
	 */
	int getBeanDefinitionCount();

	/**
	 * 判断beanName对应的BeanDefinition是否被使用了
	 * Determine whether the given bean name is already in use within this registry,
	 * i.e. whether there is a local bean or alias registered under this name.
	 * @param beanName the name to check
	 * @return whether the given bean name is already in use
	 */
	boolean isBeanNameInUse(String beanName);
}
```

## DefaultListableBeanFactory中的实现

`DefaultListableBeanFactory`是BeanFactory的具体实现，内容繁多且负责，这儿仅仅分析实现`BeanDefinitionRegistry`接口的那些方法。

#### 涉及到的字段信息

```java
// BeanDefinition容器
/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);

// BeanDefinition名称列表
/** List of bean definition names, in registration order. */
private volatile List<String> beanDefinitionNames = new ArrayList<>(256);

// 手动单例名称列表
/** List of names of manually registered singletons, in registration order. */
private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);

// 冻结名称列表
/** Cached array of bean definition names in case of frozen configuration. */
@Nullable
private volatile String[] frozenBeanDefinitionNames;

// 冻结开关，默认关闭
/** Whether bean definition metadata may be cached for all beans. */
private volatile boolean configurationFrozen = false;
```

#### registerBeanDefinition

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
		throws BeanDefinitionStoreException {

    // beanName不允许为空，beanDefinition不允许为null
	Assert.hasText(beanName, "Bean name must not be empty");
	Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    // 如果是AbstractBeanDefinition类型，则需要调用`validate()`校验一下正确性
	if (beanDefinition instanceof AbstractBeanDefinition) {
		try {
			((AbstractBeanDefinition) beanDefinition).validate();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
					"Validation of bean definition failed", ex);
		}
	}

    // 尝试从BeanDefinition容器中查找
	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
	
	// BeanDefinition容器中存在，则做了以下事情：
	// 1. 如果不允许重复注册，则抛出异常
	// 2. 条件满足的情况下，记录一帕拉日志
	// 3. 注册到BeanDefinition容器
	if (existingDefinition != null) {
		if (!isAllowBeanDefinitionOverriding()) {
			throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
		}
		else if (existingDefinition.getRole() < beanDefinition.getRole()) {
			// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
			if (logger.isInfoEnabled()) {
				logger.info("Overriding user-defined bean definition for bean '" + beanName +
						"' with a framework-generated bean definition: replacing [" +
						existingDefinition + "] with [" + beanDefinition + "]");
			}
		}
		else if (!beanDefinition.equals(existingDefinition)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
	
	// BeanDefinition容器中不存在
	// 1. 如果bean创建已经开始，则需要在锁环境下完成注册操作
	// 2. 如果bean创建还未开始，则不需要在所环境下操作，因为这个时候所有的操作都是在一个线程内完成
	// 3. 清空冻结名称列表，目的是为了后续查找方便
	else {
		if (hasBeanCreationStarted()) {
			// Cannot modify startup-time collection elements anymore (for stable iteration)
			synchronized (this.beanDefinitionMap) {
				this.beanDefinitionMap.put(beanName, beanDefinition);
				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
				updatedDefinitions.addAll(this.beanDefinitionNames);
				updatedDefinitions.add(beanName);
				this.beanDefinitionNames = updatedDefinitions;
				removeManualSingletonName(beanName);
			}
		}
		else {
			// Still in startup registration phase
			this.beanDefinitionMap.put(beanName, beanDefinition);
			this.beanDefinitionNames.add(beanName);
			removeManualSingletonName(beanName);
		}
		this.frozenBeanDefinitionNames = null;
	}

    // 如果BeanDefinition容器中存在，或者当前BeanDefinition是一个单例定义，则需要重置BeanDefinition
	if (existingDefinition != null || containsSingleton(beanName)) {
		resetBeanDefinition(beanName);
	}
}
```

#### resetBeanDefinition

```java
/**
 * 重置BeanDefinition，包括beanName被移除或替换的场景。
 * Reset all bean definition caches for the given bean,
 * including the caches of beans that are derived from it.
 * <p>Called after an existing bean definition has been replaced or removed,
 * triggering {@link #clearMergedBeanDefinition}, {@link #destroySingleton}
 * and {@link MergedBeanDefinitionPostProcessor#resetBeanDefinition} on the
 * given bean and on all bean definitions that have the given bean as parent.
 * @param beanName the name of the bean to reset
 * @see #registerBeanDefinition
 * @see #removeBeanDefinition
 */
protected void resetBeanDefinition(String beanName) {
	// Remove the merged bean definition for the given bean, if already created.
	// 清除合并的BeanDefinition
	clearMergedBeanDefinition(beanName);

	// Remove corresponding bean from singleton cache, if any. Shouldn't usually
	// be necessary, rather just meant for overriding a context's default beans
	// (e.g. the default StaticMessageSource in a StaticApplicationContext).
	// 销毁单例
	destroySingleton(beanName);

	// Notify all post-processors that the specified bean definition has been reset.
	// 调用MergedBeanDefinitionPostProcessor进行回调处理
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		if (processor instanceof MergedBeanDefinitionPostProcessor) {
			((MergedBeanDefinitionPostProcessor) processor).resetBeanDefinition(beanName);
		}
	}

	// Reset all bean definitions that have the given bean as parent (recursively).
	for (String bdName : this.beanDefinitionNames) {
		if (!beanName.equals(bdName)) {
			BeanDefinition bd = this.beanDefinitionMap.get(bdName);
			// Ensure bd is non-null due to potential concurrent modification
			// of the beanDefinitionMap.
			// 父名称和当前名称相同的情况下，需要递归调用此方法
			if (bd != null && beanName.equals(bd.getParentName())) {
				resetBeanDefinition(bdName);
			}
		}
	}
}
```

#### removeBeanDefinition

```java
public void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	Assert.hasText(beanName, "'beanName' must not be empty");

    // 移除不存在的BeanDefinition，会抛出异常
	BeanDefinition bd = this.beanDefinitionMap.remove(beanName);
	if (bd == null) {
		if (logger.isTraceEnabled()) {
			logger.trace("No bean named '" + beanName + "' found in " + this);
		}
		throw new NoSuchBeanDefinitionException(beanName);
	}

    // 和注册逻辑一样，Bean创建开始了，则需要在锁环境下进行移除操作，同时额外执行以下操作：
    // 1. 清空冻结名称列表
    // 2. 重置BeanDefinition
	if (hasBeanCreationStarted()) {
		// Cannot modify startup-time collection elements anymore (for stable iteration)
		synchronized (this.beanDefinitionMap) {
			List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames);
			updatedDefinitions.remove(beanName);
			this.beanDefinitionNames = updatedDefinitions;
		}
	}
	else {
		// Still in startup registration phase
		this.beanDefinitionNames.remove(beanName);
	}
	this.frozenBeanDefinitionNames = null;

	resetBeanDefinition(beanName);
}
```

#### getBeanDefinition && containsBeanDefinition && getBeanDefinitionCount

很简单！

```java
public BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException {
	BeanDefinition bd = this.beanDefinitionMap.get(beanName);
	if (bd == null) {
		if (logger.isTraceEnabled()) {
			logger.trace("No bean named '" + beanName + "' found in " + this);
		}
		throw new NoSuchBeanDefinitionException(beanName);
	}
	return bd;
}

public boolean containsBeanDefinition(String beanName) {
	Assert.notNull(beanName, "Bean name must not be null");
	return this.beanDefinitionMap.containsKey(beanName);
}

public int getBeanDefinitionCount() {
	return this.beanDefinitionMap.size();
}
```

#### getBeanDefinitionNames

首先从`frozenBeanDefinitionNames`获取，如果为null，再从`beanDefinitionNames`获取。

【注意】：为了线程安全，这儿使用了数组的`clone()`操作。

```java
public String[] getBeanDefinitionNames() {
	String[] frozenNames = this.frozenBeanDefinitionNames;
	if (frozenNames != null) {
		return frozenNames.clone();
	}
	else {
		return StringUtils.toStringArray(this.beanDefinitionNames);
	}
}
```

#### isBeanNameInUse

这个方法在`AbstractBeanFactory`类里面。beanName是否在使用，有3种情况：

1. 是一个别名
2. 是一个本地Bean的名称
3. 通过这个名字创建了一个内部bean（也叫依赖bean）

```java
public boolean isBeanNameInUse(String beanName) {
	return isAlias(beanName) || containsLocalBean(beanName) || hasDependentBean(beanName);
}
```

## 总结

至此，我们粗略地对`BeanDefinitionRegistry`及其实现`DefaultListableBeanFactory`进行了分析。相对来说，比较简单，主要是对`ConcurrentHashMap`这种集合对象进行增删改查操作。