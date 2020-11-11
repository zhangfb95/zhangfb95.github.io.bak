---
layout: post
title:  Spring IOC源码解析（06）SingletonBeanRegistry
date:   2020-05-28 23:03:15 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## 前言

`SingletonBeanRegistry`是单例Bean注册器接口，用于管理单例Bean。

## UML图

![类继承关系](https://upload-images.jianshu.io/upload_images/845143-b7015206d5920c5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## SingletonBeanRegistry

所有接口方法都定义得很清晰、很明了。而`SingletonBeanRegistry`的主要实现类为`DefaultSingletonBeanRegistry`。而`FactoryBeanRegistrySupport`又增加了对工厂Bean的支持。

```java
public interface SingletonBeanRegistry {

	/**
	 * 注册一个已存在的单例Bean
	 */
	void registerSingleton(String beanName, Object singletonObject);

	/**
	 * 获取一个单例Bean
	 */
	@Nullable
	Object getSingleton(String beanName);

	/**
	 * 根据bean名称，判断是否包含此单例Bean
	 */
	boolean containsSingleton(String beanName);

	/**
	 * 获取所有单例Bean的名称数组
	 */
	String[] getSingletonNames();

	/**
	 * 获取所有单例Bean的数量
	 */
	int getSingletonCount();

	/**
	 * 获取互斥锁，主要用在外部加锁场景。
	 * Return the singleton mutex used by this registry (for external collaborators).
	 * @return the mutex object (never {@code null})
	 * @since 4.2
	 */
	Object getSingletonMutex();
}
```

## DefaultSingletonBeanRegistry

字段定义如下：

```java

// 单例bean对象的缓存，beanName -> bean实例
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 对象工厂的缓存，beanName -> 对象工厂
/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

// 预加载的单例bean对象
/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

// 已注册的单例bean名称
/** Set of registered singletons, containing the bean names in registration order. */
private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

// 可销毁的bean对象
/** Disposable bean instances: bean name to disposable instance. */
private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

// 被包含的bean map，外部bean名称 -> 内部bean名称列表
/** Map between containing bean names: bean name to Set of bean names that the bean contains. */
private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

// 被依赖的bean map，被依赖的bean名称 -> 依赖bean名称集合
/** Map between dependent bean names: bean name to Set of dependent bean names. */
private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

// 依赖的bean map，依赖的bean名称 -> 被依赖bean名称集合
/** Map between depending bean names: bean name to Set of bean names for the bean's dependencies. */
private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
```

#### registerSingleton()

```java
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    // 断言非空
	Assert.notNull(beanName, "Bean name must not be null");
	Assert.notNull(singletonObject, "Singleton object must not be null");
	// 在锁环境下，首先判断对象是否存在，存在的话就报错；不存在的话就调用`addSingleton`方法进行添加。
	synchronized (this.singletonObjects) {
		Object oldObject = this.singletonObjects.get(beanName);
		if (oldObject != null) {
			throw new IllegalStateException("Could not register object [" + singletonObject +
					"] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
		}
		addSingleton(beanName, singletonObject);
	}
}

/**
 * 1. 先写入单例bean缓存；
 * 2. 再从对象工厂缓存移除（目的是保证唯一）；
 * 3. 同时移除预加载的单例bean缓存；
 * 4. 最后将bean名称添加到bean名称集合里面。
 * Add the given singleton object to the singleton cache of this factory.
 * <p>To be called for eager registration of singletons.
 * @param beanName the name of the bean
 * @param singletonObject the singleton object
 */
protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```

#### getSingleton()

```java
public Object getSingleton(String beanName) {
	return getSingleton(beanName, true);
}

/**
 * 参数`allowEarlyReference`用于判断在bean还没有初始化完毕的情况下，是否可以被获取。
 *
 * 执行过程如下：
 * 1. 尝试从单例bean缓存获取，获取到的话，直接返回
 * 2. 在第1步未获取到的情况下，如果bean正在创建的过程中，则尝试从预加载的单例bean缓存获取，获取到的话，直接返回
 * 3. 在第2步未获取到的情况下，如果allowEarlyReference`为true，则尝试从对象工厂缓存获取，未获取到的话，返回null
 * 4. 在第3步获取到的情况下，尝试从对象工厂中获取对象，并注册到预加载的单例bean缓存中，同时从单例bean缓存移除
 * 
 * Return the (raw) singleton object registered under the given name.
 * <p>Checks already instantiated singletons and also allows for an early
 * reference to a currently created singleton (resolving a circular reference).
 * @param beanName the name of the bean to look for
 * @param allowEarlyReference whether early references should be created or not
 * @return the registered singleton object, or {@code null} if none found
 */
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
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
```

#### containsSingleton()等等

`containsSingleton()`、`getSingletonNames()`、`getSingletonCount()`和`getSingletonMutex()`，这几个方法的代码都非常简单。

```java
@Override
public boolean containsSingleton(String beanName) {
	return this.singletonObjects.containsKey(beanName);
}

@Override
public String[] getSingletonNames() {
	synchronized (this.singletonObjects) {
		return StringUtils.toStringArray(this.registeredSingletons);
	}
}

@Override
public int getSingletonCount() {
	synchronized (this.singletonObjects) {
		return this.registeredSingletons.size();
	}
}

/**
 * Exposes the singleton mutex to subclasses and external collaborators.
 * <p>Subclasses should synchronize on the given Object if they perform
 * any sort of extended singleton creation phase. In particular, subclasses
 * should <i>not</i> have their own mutexes involved in singleton creation,
 * to avoid the potential for deadlocks in lazy-init situations.
 */
@Override
public final Object getSingletonMutex() {
	return this.singletonObjects;
}
```

## FactoryBeanRegistrySupport

```java
public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry {

    // 通过工厂bean创建的bean对象，将缓存到这个字段中。
	/** Cache of singleton objects created by FactoryBeans: FactoryBean name to object. */
	private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);


	/**
	 * 获取工厂Bean即将创建的bean的类型
	 * Determine the type for the given FactoryBean.
	 * @param factoryBean the FactoryBean instance to check
	 * @return the FactoryBean's object type,
	 * or {@code null} if the type cannot be determined yet
	 */
	@Nullable
	protected Class<?> getTypeForFactoryBean(final FactoryBean<?> factoryBean) {
		try {
			if (System.getSecurityManager() != null) {
				return AccessController.doPrivileged((PrivilegedAction<Class<?>>)
						factoryBean::getObjectType, getAccessControlContext());
			}
			else {
				return factoryBean.getObjectType();
			}
		}
		catch (Throwable ex) {
			// Thrown from the FactoryBean's getObjectType implementation.
			logger.info("FactoryBean threw exception from getObjectType, despite the contract saying " +
					"that it should return null if the type of its object cannot be determined yet", ex);
			return null;
		}
	}

	/**
	 * 从缓存中获取工厂bean创建的bean对象
	 * Obtain an object to expose from the given FactoryBean, if available
	 * in cached form. Quick check for minimal synchronization.
	 * @param beanName the name of the bean
	 * @return the object obtained from the FactoryBean,
	 * or {@code null} if not available
	 */
	@Nullable
	protected Object getCachedObjectForFactoryBean(String beanName) {
		return this.factoryBeanObjectCache.get(beanName);
	}

	/**
	 * 这个方法是当前类里面最核心，也最复杂的一个。其主要目的在于：
	 * 1. 根据工厂bean，创建名称为`beanName`的bean对象。
	 * 2. 同时，如果`shouldPostProcess`为true，则调用BeanPostProcessor的创建后逻辑代码。
	 * 
	 * 该方法的主要逻辑如下：
	 * 1. 如果工厂bean为单例并且已经创建了名称为beanName的bean对象
	 * 2. 尝试从缓存中获取，获取到了就直接返回
	 * 3. 在第2步未获取到的情况下，调用方法`doGetObjectFromFactoryBean()`创建bean对象，
	 *    同时和缓存里面的对象进行比对，相同则直接返回
	 * 4. 在不同的情况下，如果postProcess开关打开，则调用`postProcessObjectFromFactoryBean`方法执行创建后逻辑代码。
	 *    该段代码主要由子类实现。
	 * 5. 在第4步执行之后，将创建好的bean对象缓存起来
	 * 6. 在第1步不满足的情况下，说明创建的bean的scope不是单例的，逻辑如下：
	 *    1. 那么每调用一次就创建一个新的对象，
	 *    2. 如果postProcess开关打开，则调用`postProcessObjectFromFactoryBean`方法执行创建后逻辑代码。
	 *    3. 【注意】：因为不是单例的，这儿并没有将创建的对象缓存起来
	 * Obtain an object to expose from the given FactoryBean.
	 * @param factory the FactoryBean instance
	 * @param beanName the name of the bean
	 * @param shouldPostProcess whether the bean is subject to post-processing
	 * @return the object obtained from the FactoryBean
	 * @throws BeanCreationException if FactoryBean object creation failed
	 * @see org.springframework.beans.factory.FactoryBean#getObject()
	 */
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							beforeSingletonCreation(beanName);
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								afterSingletonCreation(beanName);
							}
						}
						if (containsSingleton(beanName)) {
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}

	/**
	 * 从工厂bean获取bean实例的地方
	 * 1. 调用`factory.getObject()`获取bean实例
	 * 2. 当创建失败的情况下，如果bean实例为单例，那么直接抛出异常，否则返回类型为`NullBean`的对象，避免空指针的情况。
	 * Obtain an object to expose from the given FactoryBean.
	 * @param factory the FactoryBean instance
	 * @param beanName the name of the bean
	 * @return the object obtained from the FactoryBean
	 * @throws BeanCreationException if FactoryBean object creation failed
	 * @see org.springframework.beans.factory.FactoryBean#getObject()
	 */
	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}

	/**
	 * 这个方法在当前类中为空实现，由子类具体实现，子类为`AbstractAutowireCapableBeanFactory`。
	 * Post-process the given object that has been obtained from the FactoryBean.
	 * The resulting object will get exposed for bean references.
	 * <p>The default implementation simply returns the given object as-is.
	 * Subclasses may override this, for example, to apply post-processors.
	 * @param object the object obtained from the FactoryBean.
	 * @param beanName the name of the bean
	 * @return the object to expose
	 * @throws org.springframework.beans.BeansException if any post-processing failed
	 */
	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) throws BeansException {
		return object;
	}

	/**
	 * 如果当前bean实例是一个工厂bean，则直接返回bean实例，否则抛出异常。
	 * Get a FactoryBean for the given bean if possible.
	 * @param beanName the name of the bean
	 * @param beanInstance the corresponding bean instance
	 * @return the bean instance as FactoryBean
	 * @throws BeansException if the given bean cannot be exposed as a FactoryBean
	 */
	protected FactoryBean<?> getFactoryBean(String beanName, Object beanInstance) throws BeansException {
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanCreationException(beanName,
					"Bean instance of type [" + beanInstance.getClass() + "] is not a FactoryBean");
		}
		return (FactoryBean<?>) beanInstance;
	}

	/**
	 * 在锁环境下移除单例，也会从工厂bean缓存中移除。
	 * Overridden to clear the FactoryBean object cache as well.
	 */
	@Override
	protected void removeSingleton(String beanName) {
		synchronized (getSingletonMutex()) {
			super.removeSingleton(beanName);
			this.factoryBeanObjectCache.remove(beanName);
		}
	}

	/**
	 * 清空单例缓存。
	 * Overridden to clear the FactoryBean object cache as well.
	 */
	@Override
	protected void clearSingletonCache() {
		synchronized (getSingletonMutex()) {
			super.clearSingletonCache();
			this.factoryBeanObjectCache.clear();
		}
	}

	/**
	 * 不知道有什么用
	 * Return the security context for this bean factory. If a security manager
	 * is set, interaction with the user code will be executed using the privileged
	 * of the security context returned by this method.
	 * @see AccessController#getContext()
	 */
	protected AccessControlContext getAccessControlContext() {
		return AccessController.getContext();
	}
}
```

#### registerDisposableBean()

```java
/**
 * 注册可销毁的bean
 * Add the given bean to the list of disposable beans in this registry.
 * <p>Disposable beans usually correspond to registered singletons,
 * matching the bean name but potentially being a different instance
 * (for example, a DisposableBean adapter for a singleton that does not
 * naturally implement Spring's DisposableBean interface).
 * @param beanName the name of the bean
 * @param bean the bean instance
 */
public void registerDisposableBean(String beanName, DisposableBean bean) {
	synchronized (this.disposableBeans) {
		this.disposableBeans.put(beanName, bean);
	}
}

/**
 * 注册包含的bean，也就是外部bean和内部bean。
 * 他们也一定有依赖关系，所以内部还会调用`registerDependentBean()`方法
 * Register a containment relationship between two beans,
 * e.g. between an inner bean and its containing outer bean.
 * <p>Also registers the containing bean as dependent on the contained bean
 * in terms of destruction order.
 * @param containedBeanName the name of the contained (inner) bean
 * @param containingBeanName the name of the containing (outer) bean
 * @see #registerDependentBean
 */
public void registerContainedBean(String containedBeanName, String containingBeanName) {
	synchronized (this.containedBeanMap) {
		Set<String> containedBeans =
				this.containedBeanMap.computeIfAbsent(containingBeanName, k -> new LinkedHashSet<>(8));
		if (!containedBeans.add(containedBeanName)) {
			return;
		}
	}
	registerDependentBean(containedBeanName, containingBeanName);
}

/**
 * 注册依赖bean，通过`canonicalName`去除别名。
 * Register a dependent bean for the given bean,
 * to be destroyed before the given bean is destroyed.
 * @param beanName the name of the bean
 * @param dependentBeanName the name of the dependent bean
 */
public void registerDependentBean(String beanName, String dependentBeanName) {
	String canonicalName = canonicalName(beanName);

	synchronized (this.dependentBeanMap) {
		Set<String> dependentBeans =
				this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
		if (!dependentBeans.add(dependentBeanName)) {
			return;
		}
	}

	synchronized (this.dependenciesForBeanMap) {
		Set<String> dependenciesForBean =
				this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
		dependenciesForBean.add(canonicalName);
	}
}
```

#### isDependent()

这个方法用于判断两个bean之间是否有依赖关系。

```java
/**
 * beanName表示被依赖的bean，dependentBeanName表示依赖的bean。
 * 
 * Determine whether the specified dependent bean has been registered as
 * dependent on the given bean or on any of its transitive dependencies.
 * @param beanName the name of the bean to check
 * @param dependentBeanName the name of the dependent bean
 * @since 4.0
 */
protected boolean isDependent(String beanName, String dependentBeanName) {
	synchronized (this.dependentBeanMap) {
		return isDependent(beanName, dependentBeanName, null);
	}
}

private boolean isDependent(String beanName, String dependentBeanName, @Nullable Set<String> alreadySeen) {
    // 从被依赖的bean map里面开始查找，
    // 当已经查看过的bean集合里面包含被依赖的bean时，说明已经查找完了，没有找到
	if (alreadySeen != null && alreadySeen.contains(beanName)) {
		return false;
	}
	
	// 剔除别名，如果被依赖的bean map里面不包含的话，说明没有找到
	String canonicalName = canonicalName(beanName);
	Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
	if (dependentBeans == null) {
		return false;
	}
	
	// 剔除别名，如果被依赖的bean map里面包含的话，说明已经找到
	if (dependentBeans.contains(dependentBeanName)) {
		return true;
	}
	
	// 判断是否存在传递依赖
	for (String transitiveDependency : dependentBeans) {
		if (alreadySeen == null) {
			alreadySeen = new HashSet<>();
		}
		alreadySeen.add(beanName);
		if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
			return true;
		}
	}
	return false;
}
```

## FactoryBean

`FactoryBean`，顾名思义，首先它表示一个管理bean的工厂bean，其次可以使用它获取一个bean实例。

```java
public interface FactoryBean<T> {

	/**
	 * 工厂bean对象类型
	 * The name of an attribute that can be
	 * {@link org.springframework.core.AttributeAccessor#setAttribute set} on a
	 * {@link org.springframework.beans.factory.config.BeanDefinition} so that
	 * factory beans can signal their object type when it can't be deduced from
	 * the factory bean class.
	 * @since 5.2
	 */
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";


	/**
	 * 获取工厂bean管理的bean实例
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * <p>As with a {@link BeanFactory}, this allows support for both the
	 * Singleton and Prototype design pattern.
	 * <p>If this FactoryBean is not fully initialized yet at the time of
	 * the call (for example because it is involved in a circular reference),
	 * throw a corresponding {@link FactoryBeanNotInitializedException}.
	 * <p>As of Spring 2.0, FactoryBeans are allowed to return {@code null}
	 * objects. The factory will consider this as normal value to be used; it
	 * will not throw a FactoryBeanNotInitializedException in this case anymore.
	 * FactoryBean implementations are encouraged to throw
	 * FactoryBeanNotInitializedException themselves now, as appropriate.
	 * @return an instance of the bean (can be {@code null})
	 * @throws Exception in case of creation errors
	 * @see FactoryBeanNotInitializedException
	 */
	@Nullable
	T getObject() throws Exception;

	/**
	 * 获取工厂bean管理的bean的类型
	 * Return the type of object that this FactoryBean creates,
	 * or {@code null} if not known in advance.
	 * <p>This allows one to check for specific types of beans without
	 * instantiating objects, for example on autowiring.
	 * <p>In the case of implementations that are creating a singleton object,
	 * this method should try to avoid singleton creation as far as possible;
	 * it should rather estimate the type in advance.
	 * For prototypes, returning a meaningful type here is advisable too.
	 * <p>This method can be called <i>before</i> this FactoryBean has
	 * been fully initialized. It must not rely on state created during
	 * initialization; of course, it can still use such state if available.
	 * <p><b>NOTE:</b> Autowiring will simply ignore FactoryBeans that return
	 * {@code null} here. Therefore it is highly recommended to implement
	 * this method properly, using the current state of the FactoryBean.
	 * @return the type of object that this FactoryBean creates,
	 * or {@code null} if not known at the time of the call
	 * @see ListableBeanFactory#getBeansOfType
	 */
	@Nullable
	Class<?> getObjectType();

	/**
	 * 当前工厂bean管理的bean实例是否一个单例
	 * Is the object managed by this factory a singleton? That is,
	 * will {@link #getObject()} always return the same object
	 * (a reference that can be cached)?
	 * <p><b>NOTE:</b> If a FactoryBean indicates to hold a singleton object,
	 * the object returned from {@code getObject()} might get cached
	 * by the owning BeanFactory. Hence, do not return {@code true}
	 * unless the FactoryBean always exposes the same reference.
	 * <p>The singleton status of the FactoryBean itself will generally
	 * be provided by the owning BeanFactory; usually, it has to be
	 * defined as singleton there.
	 * <p><b>NOTE:</b> This method returning {@code false} does not
	 * necessarily indicate that returned objects are independent instances.
	 * An implementation of the extended {@link SmartFactoryBean} interface
	 * may explicitly indicate independent instances through its
	 * {@link SmartFactoryBean#isPrototype()} method. Plain {@link FactoryBean}
	 * implementations which do not implement this extended interface are
	 * simply assumed to always return independent instances if the
	 * {@code isSingleton()} implementation returns {@code false}.
	 * <p>The default implementation returns {@code true}, since a
	 * {@code FactoryBean} typically manages a singleton instance.
	 * @return whether the exposed object is a singleton
	 * @see #getObject()
	 * @see SmartFactoryBean#isPrototype()
	 */
	default boolean isSingleton() {
		return true;
	}
}
```

## 总结

综上所述。

首先，我们从`SingletonBeanRegistry`分析了单例bean操作的方法定义。

其次，我们从具体实现类`DefaultSingletonBeanRegistry`分析了单例bean具体操作的方法实现。

最后，我们延伸出了工厂bean注册器的实现，即：`FactoryBeanRegistrySupport`。