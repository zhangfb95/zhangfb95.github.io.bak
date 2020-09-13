---
layout: post
title:  Spring IOC源码解析（10）AbstractBeanFactory
date:   2020-06-16 00:18:47 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## 前言

`AbstractBeanFactory`是bean工厂最核心的实现。我们首先分析其字段，再分析其方法。

## 字段信息

```
// 父类bean工厂
/** Parent bean factory, for bean inheritance support. */
@Nullable
private BeanFactory parentBeanFactory;

// bean类加载器
/** ClassLoader to resolve bean class names with, if necessary. */
@Nullable
private ClassLoader beanClassLoader = ClassUtils.getDefaultClassLoader();

// bean临时类加载器
/** ClassLoader to temporarily resolve bean class names with, if necessary. */
@Nullable
private ClassLoader tempClassLoader;

// 是否缓存bean元数据
/** Whether to cache bean metadata or rather reobtain it for every access. */
private boolean cacheBeanMetadata = true;

// bean表达式解析器
/** Resolution strategy for expressions in bean definition values. */
@Nullable
private BeanExpressionResolver beanExpressionResolver;

// 转换服务
/** Spring ConversionService to use instead of PropertyEditors. */
@Nullable
private ConversionService conversionService;

// 自定义属性编辑器注册器
/** Custom PropertyEditorRegistrars to apply to the beans of this factory. */
private final Set<PropertyEditorRegistrar> propertyEditorRegistrars = new LinkedHashSet<>(4);

// 自定义属性编辑器
/** Custom PropertyEditors to apply to the beans of this factory. */
private final Map<Class<?>, Class<? extends PropertyEditor>> customEditors = new HashMap<>(4);

// 自定义类型转换器
/** A custom TypeConverter to use, overriding the default PropertyEditor mechanism. */
@Nullable
private TypeConverter typeConverter;

// 为嵌入的值（如注解属性）添加字符串解析器
/** String resolvers to apply e.g. to annotation attribute values. */
private final List<StringValueResolver> embeddedValueResolvers = new CopyOnWriteArrayList<>();

// bean后置处理器
/** BeanPostProcessors to apply in createBean. */
private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();

// InstantiationAwareBeanPostProcessors是否已注册
/** Indicates whether any InstantiationAwareBeanPostProcessors have been registered. */
private volatile boolean hasInstantiationAwareBeanPostProcessors;

// DestructionAwareBeanPostProcessors是否已注册
/** Indicates whether any DestructionAwareBeanPostProcessors have been registered. */
private volatile boolean hasDestructionAwareBeanPostProcessors;

// bean作用域
/** Map from scope identifier String to corresponding Scope. */
private final Map<String, Scope> scopes = new LinkedHashMap<>(8);

// 安全作用上下文提供者
/** Security context used when running with a SecurityManager. */
@Nullable
private SecurityContextProvider securityContextProvider;

// 合并后的根BeanDefinition
/** Map from bean name to merged RootBeanDefinition. */
private final Map<String, RootBeanDefinition> mergedBeanDefinitions = new ConcurrentHashMap<>(256);

// bean是否已创建
/** Names of beans that have already been created at least once. */
private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));

// 正在创建的prototype类型的bean，之所以使用ThreadLocal，是为了避免多线程创建的冲突
/** Names of beans that are currently in creation. */
private final ThreadLocal<Object> prototypesCurrentlyInCreation =
		new NamedThreadLocal<>("Prototype beans currently in creation");
```

## AbstractBeanFactory()

构造函数，父类beanFactory可作为参数传入

```java
/**
 * 创建一个不带父类beanFactory的实例
 * Create a new AbstractBeanFactory.
 */
public AbstractBeanFactory() {
}

/**
 * 创建一个带父类beanFactory的实例
 * Create a new AbstractBeanFactory with the given parent.
 * @param parentBeanFactory parent bean factory, or {@code null} if none
 * @see #getBean
 */
public AbstractBeanFactory(@Nullable BeanFactory parentBeanFactory) {
	this.parentBeanFactory = parentBeanFactory;
}
```

## getBean()

该方法有4个重载方法，分别为：

1. 根据名称获取bean
2. 根据名称和匹配的类型，获取bean
3. 根据名称和构造时需要的参数，获取bean
4. 根据名称、匹配的类型，以及构造时需要的参数，获取bean

底层最终调用的是`doGetBean()`方法。

```java
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}

@Override
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
	return doGetBean(name, requiredType, null, false);
}

@Override
public Object getBean(String name, Object... args) throws BeansException {
	return doGetBean(name, null, args, false);
}

/**
 * Return an instance, which may be shared or independent, of the specified bean.
 * @param name the name of the bean to retrieve
 * @param requiredType the required type of the bean to retrieve
 * @param args arguments to use when creating a bean instance using explicit arguments
 * (only applied when creating a new instance as opposed to retrieving an existing one)
 * @return an instance of the bean
 * @throws BeansException if the bean could not be created
 */
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
		throws BeansException {

	return doGetBean(name, requiredType, args, false);
}
```

## doGetBean()

该方法比较长，也是该类中比较核心的一个实现方法。包括注释在内，总共超过250行。虽然行数多，但是阅读起来却非常容易。

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

    // 获取beanName，总共可能有三种形式
    // 1. 一个是原始的beanName
    // 2. 一个是加了&的（工厂bean）
    // 3. 一个是别名
	final String beanName = transformedBeanName(name);
	Object bean;

    // 首先检查是否已存在于创建的单例对象中
    // 1. 如果存在，且没有构造参数，进入这个方法
    // 2. 否则的话，往else走，也就是说不获取bean，而是直接创建bean
	// Eagerly check singleton cache for manually registered singletons.
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
		// 1. 如果是普通bean，直接返回
		// 2. 如果是FactoryBean，返回他的getObject
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		// 判断prototype是否正在创建中，如果是的话，说明存在循环依赖，直接抛出异常。
		// 只有singleton类型，才会尝试解决循环依赖问题。
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		// 父beanFactory存在，本地没有当前beanName，则从父容器取
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
        
        // 如果不仅仅是检查类型的话，则将bean名称放入alreadyCreated，说明已经标记为已创建的
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
		    // 获取BeanDefinition
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			// 抽象类检查，如果是抽象类，则抛出异常
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			// 如果有依赖的情况，先初始化依赖的bean
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
				    // 检查是否循环依赖，a依赖b，b依赖a。包括传递的依赖，比如a依赖b，b依赖c，c依赖a
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					// 注册依赖关系
					registerDependentBean(dep, beanName);
					try {
					    // 通过调用getBean方法，初始化依赖的bean
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}

			// Create bean instance.
			// 如果是单例，即：singleton
			if (mbd.isSingleton()) {
			    // 这儿和prototype比较大的区别，在于它通过FactoryObject+单例缓存的方式，从而避免重复创建对象
				sharedInstance = getSingleton(beanName, () -> {
					try {
					    // 创建bean
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
				// 如果是普通bean，直接返回，是FactoryBean，返回他的getObject
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

            // 如果是多例，即：prototype，则每次都会直接创建bean
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
				    // 加入prototypesCurrentlyInCreation，说明当前正在创建
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
				    // 从prototypesCurrentlyInCreation移除，说明已经创建完成
					afterPrototypeCreation(beanName);
				}
				// 如果是普通bean，直接返回，是FactoryBean，返回他的getObject
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

            // 既不是singleton，也不是prototype，那么直接走Scope的方式来创建bean
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
	// 类型检查，当类型不匹配时，将通过TypeConverter进行类型转换
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

## containsBean()

```java
public boolean containsBean(String name) {
	String beanName = transformedBeanName(name);
	// 已创建单例bean或者包括bean的定义
	if (containsSingleton(beanName) || containsBeanDefinition(beanName)) {
	    // 如果不是&开头的，直接返回
	    // 如果是&开头的，判断是否是FactoryBean，没有找到从父类找
		return (!BeanFactoryUtils.isFactoryDereference(name) || isFactoryBean(name));
	}
	// Not found -> check parent.
    // 当前容器没找到，去父类找
	BeanFactory parentBeanFactory = getParentBeanFactory();
	return (parentBeanFactory != null && parentBeanFactory.containsBean(originalBeanName(name)));
}
```

## isSingleton()

判断是否为单例，这儿的name包括bean的名称，也包括工厂bean的名称。
当然如果当前beanFactory未找到的话，则会尝试从父类beanFactory去查找。

```java
public boolean isSingleton(String name) throws NoSuchBeanDefinitionException {
	String beanName = transformedBeanName(name);

    // 从singleton缓存中获取到的对象不为空，说明已经实例化了
	Object beanInstance = getSingleton(beanName, false);
	if (beanInstance != null) {
	    // 工厂bean的话，需要通过工厂bean的isSingleton方法来判断
		if (beanInstance instanceof FactoryBean) {
			return (BeanFactoryUtils.isFactoryDereference(name) || ((FactoryBean<?>) beanInstance).isSingleton());
		}
		else {
			return !BeanFactoryUtils.isFactoryDereference(name);
		}
	}

	// No singleton instance found -> check bean definition.
	// 如果当前容器还没实例化，那么去父类找
	BeanFactory parentBeanFactory = getParentBeanFactory();
	if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
		// No bean definition found in this factory -> delegate to parent.
		return parentBeanFactory.isSingleton(originalBeanName(name));
	}

    // 父类beanFactory不存在，或者当前beanFactory包含bean的定义
    // 则尝试获取合并后的BeanDefinition
	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

	// In case of FactoryBean, return singleton status of created object if not a dereference.
	if (mbd.isSingleton()) {
	    // 如果定义了是单例，判断是否是FactoryBean，如果是FactoryBean，name就要是&开头
		if (isFactoryBean(beanName, mbd)) {
			if (BeanFactoryUtils.isFactoryDereference(name)) {
				return true;
			}
			FactoryBean<?> factoryBean = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
			return factoryBean.isSingleton();
		}
		else {
			return !BeanFactoryUtils.isFactoryDereference(name);
		}
	}
	else {
		return false;
	}
}
```

## isPrototype()

`isPrototype()`和`isSingleton()`非常类似，最终都是根据`RootBeanDefinition`的方法进行判断的，只是最终调用的方法不同罢了。

```java
public boolean isPrototype(String name) throws NoSuchBeanDefinitionException {
	String beanName = transformedBeanName(name);

	BeanFactory parentBeanFactory = getParentBeanFactory();
	if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
		// No bean definition found in this factory -> delegate to parent.
		return parentBeanFactory.isPrototype(originalBeanName(name));
	}

	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
	if (mbd.isPrototype()) {
		// In case of FactoryBean, return singleton status of created object if not a dereference.
		return (!BeanFactoryUtils.isFactoryDereference(name) || isFactoryBean(beanName, mbd));
	}

	// Singleton or scoped - not a prototype.
	// However, FactoryBean may still produce a prototype object...
	if (BeanFactoryUtils.isFactoryDereference(name)) {
		return false;
	}
	if (isFactoryBean(beanName, mbd)) {
		final FactoryBean<?> fb = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
		if (System.getSecurityManager() != null) {
			return AccessController.doPrivileged((PrivilegedAction<Boolean>) () ->
					((fb instanceof SmartFactoryBean && ((SmartFactoryBean<?>) fb).isPrototype()) || !fb.isSingleton()),
					getAccessControlContext());
		}
		else {
			return ((fb instanceof SmartFactoryBean && ((SmartFactoryBean<?>) fb).isPrototype()) ||
					!fb.isSingleton());
		}
	}
	else {
		return false;
	}
}
```

## isTypeMatch()

`isTypeMatch`用于判断名称和类型是否匹配。当然，这儿的类型就包括继承的类型。

```java
public boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException {
	return isTypeMatch(name, typeToMatch, true);
}

protected boolean isTypeMatch(String name, ResolvableType typeToMatch, boolean allowFactoryBeanInit)
			throws NoSuchBeanDefinitionException {

	String beanName = transformedBeanName(name);
	// 判断是否工厂bean标识
	boolean isFactoryDereference = BeanFactoryUtils.isFactoryDereference(name);

	// Check manually registered singletons.
	// 手动检查已注册的单例
	Object beanInstance = getSingleton(beanName, false);
	// 说明是一个已注册的单例
	if (beanInstance != null && beanInstance.getClass() != NullBean.class) {
		if (beanInstance instanceof FactoryBean) {.
	        // 如果是工厂bean，则获取工厂bean能生产的bean的类型，再判断其继承关系
			if (!isFactoryDereference) {
				Class<?> type = getTypeForFactoryBean((FactoryBean<?>) beanInstance);
				return (type != null && typeToMatch.isAssignableFrom(type));
			}
			// 如果是工厂bean，但名称也工厂bean的名称，则判断是否为类型匹配的实例
			else {
				return typeToMatch.isInstance(beanInstance);
			}
		}
		// 普通bean，且没有&开头
		else if (!isFactoryDereference) {
			if (typeToMatch.isInstance(beanInstance)) {
				// Direct match for exposed instance?
				return true;
			}
			// 泛型且有beanName的定义
			else if (typeToMatch.hasGenerics() && containsBeanDefinition(beanName)) {
				// Generics potentially only match on the target class, not on the proxy...
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				Class<?> targetType = mbd.getTargetType();
				if (targetType != null && targetType != ClassUtils.getUserClass(beanInstance)) {
					// Check raw class match as well, making sure it's exposed on the proxy.
					Class<?> classToMatch = typeToMatch.resolve();
					if (classToMatch != null && !classToMatch.isInstance(beanInstance)) {
						return false;
					}
					if (typeToMatch.isAssignableFrom(targetType)) {
						return true;
					}
				}
				ResolvableType resolvableType = mbd.targetType;
				if (resolvableType == null) {
					resolvableType = mbd.factoryMethodReturnType;
				}
				return (resolvableType != null && typeToMatch.isAssignableFrom(resolvableType));
			}
		}
		return false;
	}
	// 已经创建了，但是没有bean的定义，说明是空的注册进来，类型一定不匹配
	else if (containsSingleton(beanName) && !containsBeanDefinition(beanName)) {
		// null instance registered
		return false;
	}

	// No singleton instance found -> check bean definition.
	// 从父类找
	BeanFactory parentBeanFactory = getParentBeanFactory();
	if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
		// No bean definition found in this factory -> delegate to parent.
		return parentBeanFactory.isTypeMatch(originalBeanName(name), typeToMatch);
	}

	// Retrieve corresponding bean definition.
	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
	BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();

	// Setup the types that we want to match against
	Class<?> classToMatch = typeToMatch.resolve();
	if (classToMatch == null) {
		classToMatch = FactoryBean.class;
	}
	Class<?>[] typesToMatch = (FactoryBean.class == classToMatch ?
			new Class<?>[] {classToMatch} : new Class<?>[] {FactoryBean.class, classToMatch});


	// Attempt to predict the bean type
	Class<?> predictedType = null;

	// We're looking for a regular reference but we're a factory bean that has
	// a decorated bean definition. The target bean should be the same type
	// as FactoryBean would ultimately return.
	if (!isFactoryDereference && dbd != null && isFactoryBean(beanName, mbd)) {
		// We should only attempt if the user explicitly set lazy-init to true
		// and we know the merged bean definition is for a factory bean.
		if (!mbd.isLazyInit() || allowFactoryBeanInit) {
			RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
			Class<?> targetType = predictBeanType(dbd.getBeanName(), tbd, typesToMatch);
			if (targetType != null && !FactoryBean.class.isAssignableFrom(targetType)) {
				predictedType = targetType;
			}
		}
	}

	// If we couldn't use the target type, try regular prediction.
	if (predictedType == null) {
		predictedType = predictBeanType(beanName, mbd, typesToMatch);
		if (predictedType == null) {
			return false;
		}
	}

	// Attempt to get the actual ResolvableType for the bean.
	ResolvableType beanType = null;

	// If it's a FactoryBean, we want to look at what it creates, not the factory class.
	if (FactoryBean.class.isAssignableFrom(predictedType)) {
		if (beanInstance == null && !isFactoryDereference) {
			beanType = getTypeForFactoryBean(beanName, mbd, allowFactoryBeanInit);
			predictedType = beanType.resolve();
			if (predictedType == null) {
				return false;
			}
		}
	}
	else if (isFactoryDereference) {
		// Special case: A SmartInstantiationAwareBeanPostProcessor returned a non-FactoryBean
		// type but we nevertheless are being asked to dereference a FactoryBean...
		// Let's check the original bean class and proceed with it if it is a FactoryBean.
		predictedType = predictBeanType(beanName, mbd, FactoryBean.class);
		if (predictedType == null || !FactoryBean.class.isAssignableFrom(predictedType)) {
			return false;
		}
	}

	// We don't have an exact type but if bean definition target type or the factory
	// method return type matches the predicted type then we can use that.
	if (beanType == null) {
		ResolvableType definedType = mbd.targetType;
		if (definedType == null) {
			definedType = mbd.factoryMethodReturnType;
		}
		if (definedType != null && definedType.resolve() == predictedType) {
			beanType = definedType;
		}
	}

	// If we have a bean type use it so that generics are considered
	if (beanType != null) {
		return typeToMatch.isAssignableFrom(beanType);
	}

	// If we don't have a bean type, fallback to the predicted type
	return typeToMatch.isAssignableFrom(predictedType);
}
```

## getType()

`getType()`用于根据名称获取bean的类型。

```java
public Class<?> getType(String name) throws NoSuchBeanDefinitionException {
	return getType(name, true);
}

public Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException {
	String beanName = transformedBeanName(name);

	// Check manually registered singletons.
	// 手动从已注册的单例bean中获取
	Object beanInstance = getSingleton(beanName, false);
	if (beanInstance != null && beanInstance.getClass() != NullBean.class) {
	    // 如果是FactoryBean，且没有&，说明是取FactoryBean的getObjectType类型
		if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
			return getTypeForFactoryBean((FactoryBean<?>) beanInstance);
		}
		// 否则直接返回实例的class
		else {
			return beanInstance.getClass();
		}
	}

	// No singleton instance found -> check bean definition.
	// 尝试从父类beanFactory获取
	BeanFactory parentBeanFactory = getParentBeanFactory();
	if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
		// No bean definition found in this factory -> delegate to parent.
		return parentBeanFactory.getType(originalBeanName(name));
	}

    // 都没有实例化，则从定义里查找
	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

	// Check decorated bean definition, if any: We assume it'll be easier
	// to determine the decorated bean's type than the proxy's type.
	BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
	if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
		RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
		Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd);
		if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
			return targetClass;
		}
	}

	Class<?> beanClass = predictBeanType(beanName, mbd);

	// Check bean class whether we're dealing with a FactoryBean.
	if (beanClass != null && FactoryBean.class.isAssignableFrom(beanClass)) {
		if (!BeanFactoryUtils.isFactoryDereference(name)) {
			// If it's a FactoryBean, we want to look at what it creates, not at the factory class.
			return getTypeForFactoryBean(beanName, mbd, allowFactoryBeanInit).resolve();
		}
		else {
			return beanClass;
		}
	}
	else {
		return (!BeanFactoryUtils.isFactoryDereference(name) ? beanClass : null);
	}
}
```

## getAliases()

`getAliases()`用于获取别名数组。该方法除了支持beanFactory获取之外，最主要的方法仍然是调用了`SimpleAliasRegistry`类的获取别名方法。

```java
public String[] getAliases(String name) {
	String beanName = transformedBeanName(name);
	List<String> aliases = new ArrayList<>();
	boolean factoryPrefix = name.startsWith(FACTORY_BEAN_PREFIX);
	String fullBeanName = beanName;
	if (factoryPrefix) {
		fullBeanName = FACTORY_BEAN_PREFIX + beanName;
	}
	if (!fullBeanName.equals(name)) {
		aliases.add(fullBeanName);
	}
	String[] retrievedAliases = super.getAliases(beanName);
	String prefix = factoryPrefix ? FACTORY_BEAN_PREFIX : "";
	for (String retrievedAlias : retrievedAliases) {
		String alias = prefix + retrievedAlias;
		if (!alias.equals(name)) {
			aliases.add(alias);
		}
	}
	// 没实例化过且没有bean的定义，从父类查找
	if (!containsSingleton(beanName) && !containsBeanDefinition(beanName)) {
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null) {
			aliases.addAll(Arrays.asList(parentBeanFactory.getAliases(fullBeanName)));
		}
	}
	return StringUtils.toStringArray(aliases);
}
```

## get/setParentBeanFactory()

用于获取或设置父类beanFactory，重复设置不同的父类beanFactory将抛出异常。

```java
public BeanFactory getParentBeanFactory() {
	return this.parentBeanFactory;
}
public void setParentBeanFactory(@Nullable BeanFactory parentBeanFactory) {
	if (this.parentBeanFactory != null && this.parentBeanFactory != parentBeanFactory) {
		throw new IllegalStateException("Already associated with parent BeanFactory: " + this.parentBeanFactory);
	}
	this.parentBeanFactory = parentBeanFactory;
}
```

## containsLocalBean()

当前beanFactory是否有指定的bean。

```java
public boolean containsLocalBean(String name) {
	String beanName = transformedBeanName(name);
	return ((containsSingleton(beanName) || containsBeanDefinition(beanName)) &&
			(!BeanFactoryUtils.isFactoryDereference(name) || isFactoryBean(beanName)));
}
```

## is/setCacheBeanMetadata()

用于判断或设置是否缓存bean的元数据，默认为true。如果为false，那么每次创建bean都要从类加载器获取信息。

```java
public boolean isCacheBeanMetadata() {
	return this.cacheBeanMetadata;
}
public void setCacheBeanMetadata(boolean cacheBeanMetadata) {
	this.cacheBeanMetadata = cacheBeanMetadata;
}
```

## getTypeConverter()

用于获取类型转换器。

```java
public void setTypeConverter(TypeConverter typeConverter) {
	this.typeConverter = typeConverter;
}
protected TypeConverter getCustomTypeConverter() {
	return this.typeConverter;
}
public TypeConverter getTypeConverter() {
    // 如果自定义类型转换器存在，则直接返回
	TypeConverter customConverter = getCustomTypeConverter();
	if (customConverter != null) {
		return customConverter;
	}
	// 如果自定义类型转换器不存，则自动生成一个简单的类型转换器，并注册上去。
	else {
		// Build default TypeConverter, registering custom editors.
		SimpleTypeConverter typeConverter = new SimpleTypeConverter();
		typeConverter.setConversionService(getConversionService());
		registerCustomEditors(typeConverter);
		return typeConverter;
	}
}
```

## resolveEmbeddedValue()

`resolveEmbeddedValue()`用于为嵌入的值（如注解属性）添加字符串解析器。

```java
public String resolveEmbeddedValue(@Nullable String value) {
    // 值为null，直接返回null
	if (value == null) {
		return null;
	}
	// 如果存在多个字符串解析，依次进行解析，当解析结果为null或者所有解析器都解析完毕，即返回结果。
	String result = value;
	for (StringValueResolver resolver : this.embeddedValueResolvers) {
		result = resolver.resolveStringValue(result);
		if (result == null) {
			return null;
		}
	}
	return result;
}
```

## addBeanPostProcessor()

```java
public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
	Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
	// Remove from old position, if any
	// 移除旧的
	this.beanPostProcessors.remove(beanPostProcessor);
	// Track whether it is instantiation/destruction aware
	// 如果是下面两个类型，标志位就设置true
	if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
		this.hasInstantiationAwareBeanPostProcessors = true;
	}
	if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
		this.hasDestructionAwareBeanPostProcessors = true;
	}
	// Add to end of list
	this.beanPostProcessors.add(beanPostProcessor);
}
```

## registerScope()

`registerScope()`用于注册作用域。

```java
public void registerScope(String scopeName, Scope scope) {
	Assert.notNull(scopeName, "Scope identifier must not be null");
	Assert.notNull(scope, "Scope must not be null");
	// singleton和prototype不需要注册，是spring ioc内置的
	if (SCOPE_SINGLETON.equals(scopeName) || SCOPE_PROTOTYPE.equals(scopeName)) {
		throw new IllegalArgumentException("Cannot replace existing scopes 'singleton' and 'prototype'");
	}
	// 如果注册前和注册后的scope不相同，则记录一个debug日志。
	Scope previous = this.scopes.put(scopeName, scope);
	if (previous != null && previous != scope) {
		if (logger.isDebugEnabled()) {
			logger.debug("Replacing scope '" + scopeName + "' from [" + previous + "] to [" + scope + "]");
		}
	}
	else {
		if (logger.isTraceEnabled()) {
			logger.trace("Registering scope '" + scopeName + "' with implementation [" + scope + "]");
		}
	}
}
```

## copyConfigurationFrom()

拷贝配置。拷贝的内容包括所有标准配置设置，以及beanPostProcessors、作用域和特定于工厂的内部设置。不应该包含任何实际bean定义的元数据，例如：BeanDefinition对象和bean名称别名。

```java
public void copyConfigurationFrom(ConfigurableBeanFactory otherFactory) {
	Assert.notNull(otherFactory, "BeanFactory must not be null");
	setBeanClassLoader(otherFactory.getBeanClassLoader());
	setCacheBeanMetadata(otherFactory.isCacheBeanMetadata());
	setBeanExpressionResolver(otherFactory.getBeanExpressionResolver());
	setConversionService(otherFactory.getConversionService());
	if (otherFactory instanceof AbstractBeanFactory) {
		AbstractBeanFactory otherAbstractFactory = (AbstractBeanFactory) otherFactory;
		this.propertyEditorRegistrars.addAll(otherAbstractFactory.propertyEditorRegistrars);
		this.customEditors.putAll(otherAbstractFactory.customEditors);
		this.typeConverter = otherAbstractFactory.typeConverter;
		this.beanPostProcessors.addAll(otherAbstractFactory.beanPostProcessors);
		this.hasInstantiationAwareBeanPostProcessors = this.hasInstantiationAwareBeanPostProcessors ||
				otherAbstractFactory.hasInstantiationAwareBeanPostProcessors;
		this.hasDestructionAwareBeanPostProcessors = this.hasDestructionAwareBeanPostProcessors ||
				otherAbstractFactory.hasDestructionAwareBeanPostProcessors;
		this.scopes.putAll(otherAbstractFactory.scopes);
		this.securityContextProvider = otherAbstractFactory.securityContextProvider;
	}
	else {
		setTypeConverter(otherFactory.getTypeConverter());
		String[] otherScopeNames = otherFactory.getRegisteredScopeNames();
		for (String scopeName : otherScopeNames) {
			this.scopes.put(scopeName, otherFactory.getRegisteredScope(scopeName));
		}
	}
}
```

## getMergedBeanDefinition()

合并bean的定义，包括父类BeanFactory继承下来的。

```java
public BeanDefinition getMergedBeanDefinition(String name) throws BeansException {
	String beanName = transformedBeanName(name);
	// Efficiently check whether bean definition exists in this factory.
	// 如果当前容器没有，且父类是ConfigurableBeanFactory类型，去父类找
	if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {
		return ((ConfigurableBeanFactory) getParentBeanFactory()).getMergedBeanDefinition(beanName);
	}
	// Resolve merged bean definition locally.
	return getMergedLocalBeanDefinition(beanName);
}
```

## isFactoryBean()

```java
public boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException {
	String beanName = transformedBeanName(name);
	// 获取单例bean对象
	Object beanInstance = getSingleton(beanName, false);
	// 单例bean对象找着了，则直接判断是否工厂bean
	if (beanInstance != null) {
		return (beanInstance instanceof FactoryBean);
	}
	// 单例bean没找着，当前容器不包含bean定义，如果父类beanFactory为ConfigurableBeanFactory，则从父类找
	// 否则，仍然先找到合并的本地bean定义，再在其基础上判断是否工厂bean
	// No singleton instance found -> check bean definition.
	if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {
		// No bean definition found in this factory -> delegate to parent.
		return ((ConfigurableBeanFactory) getParentBeanFactory()).isFactoryBean(name);
	}
	return isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName));
}
```

## isActuallyInCreation()

判断当前是否正在创建过程中，包括单例和多例。

```java
public boolean isActuallyInCreation(String beanName) {
	return (isSingletonCurrentlyInCreation(beanName) || isPrototypeCurrentlyInCreation(beanName));
}
public boolean isSingletonCurrentlyInCreation(String beanName) {
	return this.singletonsCurrentlyInCreation.contains(beanName);
}
protected boolean isPrototypeCurrentlyInCreation(String beanName) {
    // 当前线程变量有值，说明有在创建多例
	Object curVal = this.prototypesCurrentlyInCreation.get();
	// 如果相等，或者在set中，说明创建的多例有包括指定的bean
	return (curVal != null &&
			(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}
```

## beforePrototypeCreation()

多例创建前，把beanName加入到`prototypesCurrentlyInCreation`，如果是string，说明已创建了一个，需要转换为set。

```java
/**
 * Callback before prototype creation.
 * <p>The default implementation register the prototype as currently in creation.
 * @param beanName the name of the prototype about to be created
 * @see #isPrototypeCurrentlyInCreation
 */
@SuppressWarnings("unchecked")
protected void beforePrototypeCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	if (curVal == null) {
		this.prototypesCurrentlyInCreation.set(beanName);
	}
	else if (curVal instanceof String) {
		Set<String> beanNameSet = new HashSet<>(2);
		beanNameSet.add((String) curVal);
		beanNameSet.add(beanName);
		this.prototypesCurrentlyInCreation.set(beanNameSet);
	}
	else {
		Set<String> beanNameSet = (Set<String>) curVal;
		beanNameSet.add(beanName);
	}
}
```

## afterPrototypeCreation()

多例创建后，需要从`prototypesCurrentlyInCreation`中移除掉。

```java
/**
 * Callback after prototype creation.
 * <p>The default implementation marks the prototype as not in creation anymore.
 * @param beanName the name of the prototype that has been created
 * @see #isPrototypeCurrentlyInCreation
 */
@SuppressWarnings("unchecked")
protected void afterPrototypeCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	if (curVal instanceof String) {
		this.prototypesCurrentlyInCreation.remove();
	}
	else if (curVal instanceof Set) {
		Set<String> beanNameSet = (Set<String>) curVal;
		beanNameSet.remove(beanName);
		if (beanNameSet.isEmpty()) {
			this.prototypesCurrentlyInCreation.remove();
		}
	}
}
```

## destroyBean()

用于销毁bean对象。内部通过创建`DisposableBeanAdapter`对象，并调用其`destroy()`方法完成。

```java
public void destroyBean(String beanName, Object beanInstance) {
	destroyBean(beanName, beanInstance, getMergedLocalBeanDefinition(beanName));
}

/**
 * Destroy the given bean instance (usually a prototype instance
 * obtained from this factory) according to the given bean definition.
 * @param beanName the name of the bean definition
 * @param bean the bean instance to destroy
 * @param mbd the merged bean definition
 */
protected void destroyBean(String beanName, Object bean, RootBeanDefinition mbd) {
	new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), getAccessControlContext()).destroy();
}

public void destroyScopedBean(String beanName) {
	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
	// 单例和多例不能销毁，这是spring ioc内置处理的。
	if (mbd.isSingleton() || mbd.isPrototype()) {
		throw new IllegalArgumentException(
				"Bean name '" + beanName + "' does not correspond to an object in a mutable scope");
	}
	// 如果没有对应作用域，那么不能删除
	String scopeName = mbd.getScope();
	Scope scope = this.scopes.get(scopeName);
	if (scope == null) {
		throw new IllegalStateException("No Scope SPI registered for scope name '" + scopeName + "'");
	}
	// 如果已初始化，就调用销毁的方法
	Object bean = scope.remove(beanName);
	if (bean != null) {
		destroyBean(beanName, bean, mbd);
	}
}
```

## transformedBeanName()

`transformedBeanName()`和`canonicalName()`两个方法一起完成逻辑。

其中，`canonicalName()`用于获取最原始的bean名称，如果当前名称是别名，会一直递归转换，直到不是别名位置。

`transformedBeanName()`用于转换工厂bean名称的前缀`&`。

```java
/**
 * Return the bean name, stripping out the factory dereference prefix if necessary,
 * and resolving aliases to canonical names.
 * @param name the user-specified name
 * @return the transformed bean name
 */
protected String transformedBeanName(String name) {
	return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}
```

## originalBeanName()

该方法用于获取bean的原始名称，如果名称以`&`打头，说明是工厂bean，需要再清楚`&`之后再还原回来。

```java
/**
 * Determine the original bean name, resolving locally defined aliases to canonical names.
 * @param name the user-specified name
 * @return the original bean name
 */
protected String originalBeanName(String name) {
	String beanName = transformedBeanName(name);
	if (name.startsWith(FACTORY_BEAN_PREFIX)) {
		beanName = FACTORY_BEAN_PREFIX + beanName;
	}
	return beanName;
}
```

## registerCustomEditors()

`registerCustomEditors()`用于注册自定义属性编辑器。

```java
/**
 * Initialize the given PropertyEditorRegistry with the custom editors
 * that have been registered with this BeanFactory.
 * <p>To be called for BeanWrappers that will create and populate bean
 * instances, and for SimpleTypeConverter used for constructor argument
 * and factory method type conversion.
 * @param registry the PropertyEditorRegistry to initialize
 */
protected void registerCustomEditors(PropertyEditorRegistry registry) {
	PropertyEditorRegistrySupport registrySupport =
			(registry instanceof PropertyEditorRegistrySupport ? (PropertyEditorRegistrySupport) registry : null);
	if (registrySupport != null) {
		registrySupport.useConfigValueEditors();
	}
	// 每个登记员，都需要对登记器进行登记
	if (!this.propertyEditorRegistrars.isEmpty()) {
		for (PropertyEditorRegistrar registrar : this.propertyEditorRegistrars) {
			try {
				registrar.registerCustomEditors(registry);
			}
			catch (BeanCreationException ex) {
				Throwable rootCause = ex.getMostSpecificCause();
				if (rootCause instanceof BeanCurrentlyInCreationException) {
					BeanCreationException bce = (BeanCreationException) rootCause;
					String bceBeanName = bce.getBeanName();
					if (bceBeanName != null && isCurrentlyInCreation(bceBeanName)) {
						if (logger.isDebugEnabled()) {
							logger.debug("PropertyEditorRegistrar [" + registrar.getClass().getName() +
									"] failed because it tried to obtain currently created bean '" +
									ex.getBeanName() + "': " + ex.getMessage());
						}
						onSuppressedException(ex);
						continue;
					}
				}
				throw ex;
			}
		}
	}
	// 自定义编辑器不为空，登记器会对每个自定义编辑器进行注册。
	if (!this.customEditors.isEmpty()) {
		this.customEditors.forEach((requiredType, editorClass) ->
				registry.registerCustomEditor(requiredType, BeanUtils.instantiateClass(editorClass)));
	}
}
```

## 总结

`AbstractBeanFactory`包含大量的可复用的方法，这些方法底层多是使用了递归、回调、容器（列表、map、set等）、原型拷贝等常用手段达到高度抽象的效果。同时针对工厂bean、bean定义对象有大量的运用到，为后续功能的增强，奠定了厚重的基础。