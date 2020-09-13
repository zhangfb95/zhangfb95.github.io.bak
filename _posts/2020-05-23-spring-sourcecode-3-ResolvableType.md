---
layout: post
title:  Spring IOC源码解析（03）ResolvableType
date:   2020-05-23 10:33:29 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## 前言

在JDK原生类库里面，`Type`总共有5种类型，分别是：原始类型、参数化类型、泛型数组类型、类型变量和通配符类型。这些类型相互间串起来操作的时候，就会出现一些问题。例如：

1. 代码重复和繁长；
2. 对JDK的Type不熟悉的话，也容易出错。

而坏味道的代码，总是容易让一些有洁癖的人无法忍受，并最终处理掉。最终，Spring里面的`ResolvableType`在`Type`的基础上，封装了我们常用的一些操作，使得我们对Java类型的操作变得更加简单。

我们来看看ResolvableType提供的 [javadoc文档](https://docs.spring.io/spring-framework/docs/5.2.5.RELEASE/javadoc-api/org/springframework/core/ResolvableType.html)，其描述信息如下：

```java
/**
 * Encapsulates a Java {@link java.lang.reflect.Type}, providing access to
 * {@link #getSuperType() supertypes}, {@link #getInterfaces() interfaces}, and
 * {@link #getGeneric(int...) generic parameters} along with the ability to ultimately
 * {@link #resolve() resolve} to a {@link java.lang.Class}.
 *
 * <p>{@code ResolvableTypes} may be obtained from {@link #forField(Field) fields},
 * {@link #forMethodParameter(Method, int) method parameters},
 * {@link #forMethodReturnType(Method) method returns} or
 * {@link #forClass(Class) classes}. Most methods on this class will themselves return
 * {@link ResolvableType ResolvableTypes}, allowing easy navigation. For example:
 * <pre class="code">
 * private HashMap&lt;Integer, List&lt;String&gt;&gt; myMap;
 *
 * public void example() {
 *     ResolvableType t = ResolvableType.forField(getClass().getDeclaredField("myMap"));
 *     t.getSuperType(); // AbstractMap&lt;Integer, List&lt;String&gt;&gt;
 *     t.asMap(); // Map&lt;Integer, List&lt;String&gt;&gt;
 *     t.getGeneric(0).resolve(); // Integer
 *     t.getGeneric(1).resolve(); // List
 *     t.getGeneric(1); // List&lt;String&gt;
 *     t.resolveGeneric(1, 0); // String
 * }
 * </pre>
 */
```

中文翻译如下：

> `ResolvableType`是对Java类型的封装，提供了对父类（`getSuperType()`）、接口（`getInterfaces()`）、泛型参数（`getGeneric(int...)`）的访问，最底层的实现为`resolve()`方法。
> `ResolvableType`可以通过`forField(Field)`、`forMethodParameter(Method, int)`、`forMethodReturnType(Method)`和`forClass(Class)`进行构造。这个类的大部分方法返回的都是`ResolvableType`类型，同时允许用户很容易地进行导航操作。例如：

```java
private HashMap<Integer, List<String>> myMap;

public void example() {
    ResolvableType t = ResolvableType.forField(getClass().getDeclaredField("myMap"));
    t.getSuperType(); // AbstractMap<Integer, List<String>>
    t.asMap(); // Map<Integer, List<String>>
    t.getGeneric(0).resolve(); // Integer
    t.getGeneric(1).resolve(); // List
    t.getGeneric(1); // List<String>
    t.resolveGeneric(1, 0); // String
}
```

## 源码解析

#### 构造方法

```java
/**
 * Private constructor used to create a new {@link ResolvableType} for cache key purposes,
 * with no upfront resolution.
 */
private ResolvableType(
		Type type, @Nullable TypeProvider typeProvider, @Nullable VariableResolver variableResolver) {

	this.type = type;
	this.typeProvider = typeProvider;
	this.variableResolver = variableResolver;
	this.componentType = null;
	this.hash = calculateHashCode();
	this.resolved = null;
}

/**
 * Private constructor used to create a new {@link ResolvableType} for cache value purposes,
 * with upfront resolution and a pre-calculated hash.
 * @since 4.2
 */
private ResolvableType(Type type, @Nullable TypeProvider typeProvider,
		@Nullable VariableResolver variableResolver, @Nullable Integer hash) {

	this.type = type;
	this.typeProvider = typeProvider;
	this.variableResolver = variableResolver;
	this.componentType = null;
	this.hash = hash;
	this.resolved = resolveClass();
}

/**
 * Private constructor used to create a new {@link ResolvableType} for uncached purposes,
 * with upfront resolution but lazily calculated hash.
 */
private ResolvableType(Type type, @Nullable TypeProvider typeProvider,
		@Nullable VariableResolver variableResolver, @Nullable ResolvableType componentType) {

	this.type = type;
	this.typeProvider = typeProvider;
	this.variableResolver = variableResolver;
	this.componentType = componentType;
	this.hash = null;
	this.resolved = resolveClass();
}

/**
 * Private constructor used to create a new {@link ResolvableType} on a {@link Class} basis.
 * Avoids all {@code instanceof} checks in order to create a straight {@link Class} wrapper.
 * @since 4.2
 */
private ResolvableType(@Nullable Class<?> clazz) {
	this.resolved = (clazz != null ? clazz : Object.class);
	this.type = this.resolved;
	this.typeProvider = null;
	this.variableResolver = null;
	this.componentType = null;
	this.hash = null;
}
```

从源码看，所有的构造方法都是私有的。因此，我们不能直接通过构造方法创建实例。

这些构造方法，绝大多数操作都只是赋值。唯一有点区别的地方在于，当`hash`或者`componentType`参数有传入的时候，会调用`resolveClass()`方法进行类处理操作。

#### forClass

`forClass`用于通过Class类型的对象构造ResolvableType类型。

```java
public static ResolvableType forClass(@Nullable Class<?> clazz) {
	return new ResolvableType(clazz);
}
```

很简单，调用私有构造方法直接就构建了。

```java
public static ResolvableType forRawClass(@Nullable Class<?> clazz) {
	return new ResolvableType(clazz) {
		@Override
		public ResolvableType[] getGenerics() {
			return EMPTY_TYPES_ARRAY;
		}
		@Override
		public boolean isAssignableFrom(Class<?> other) {
			return (clazz == null || ClassUtils.isAssignable(clazz, other));
		}
		@Override
		public boolean isAssignableFrom(ResolvableType other) {
			Class<?> otherClass = other.resolve();
			return (otherClass != null && (clazz == null || ClassUtils.isAssignable(clazz, otherClass)));
		}
	};
}
```

和`forClass(@Nullable Class<?> clazz)`的逻辑基本相同，唯一的区别就是重写了三个方法。这是为什么呢？原因在于：

1. 对于基本数据类型来说，泛型参数的对象数组一定是空数组；
2. 对于基本数据类型来说，继承关系的比较逻辑是固定不变的。

```java
/**
 * Return a {@link ResolvableType} for the specified base type
 * (interface or base class) with a given implementation class.
 * For example: {@code ResolvableType.forClass(List.class, MyArrayList.class)}.
 * @param baseType the base type (must not be {@code null})
 * @param implementationClass the implementation class
 * @return a {@link ResolvableType} for the specified base type backed by the
 * given implementation class
 * @see #forClass(Class)
 * @see #forClassWithGenerics(Class, Class...)
 */
public static ResolvableType forClass(Class<?> baseType, Class<?> implementationClass) {
	Assert.notNull(baseType, "Base type must not be null");
	ResolvableType asType = forType(implementationClass).as(baseType);
	return (asType == NONE ? forType(baseType) : asType);
}
```

`implementationClass`是`baseType`的子类，这个方法主要获取`baseType`上定义的泛型。

```java
public static ResolvableType forInstance(Object instance) {
	Assert.notNull(instance, "Instance must not be null");
	if (instance instanceof ResolvableTypeProvider) {
		ResolvableType type = ((ResolvableTypeProvider) instance).getResolvableType();
		if (type != null) {
			return type;
		}
	}
	return ResolvableType.forClass(instance.getClass());
}
```

底层调用的仍然是`forClass()`方法，只是在调用之前进行了类型甄别和适配转换。

#### forConstructorParameter

```java
/**
 * Return a {@link ResolvableType} for the specified {@link Constructor} parameter.
 * @param constructor the source constructor (must not be {@code null})
 * @param parameterIndex the parameter index
 * @return a {@link ResolvableType} for the specified constructor parameter
 * @see #forConstructorParameter(Constructor, int, Class)
 */
public static ResolvableType forConstructorParameter(Constructor<?> constructor, int parameterIndex) {
	Assert.notNull(constructor, "Constructor must not be null");
	return forMethodParameter(new MethodParameter(constructor, parameterIndex));
}

/**
 * Return a {@link ResolvableType} for the specified {@link Constructor} parameter
 * with a given implementation. Use this variant when the class that declares the
 * constructor includes generic parameter variables that are satisfied by the
 * implementation class.
 * @param constructor the source constructor (must not be {@code null})
 * @param parameterIndex the parameter index
 * @param implementationClass the implementation class
 * @return a {@link ResolvableType} for the specified constructor parameter
 * @see #forConstructorParameter(Constructor, int)
 */
public static ResolvableType forConstructorParameter(Constructor<?> constructor, int parameterIndex,
		Class<?> implementationClass) {

	Assert.notNull(constructor, "Constructor must not be null");
	MethodParameter methodParameter = new MethodParameter(constructor, parameterIndex, implementationClass);
	return forMethodParameter(methodParameter);
}
```

构造方法和普通方法大体相同，唯一区别在于，构造方法没有返回值。因此，`forConstructorParameter()`方法的底层调用仍然是`forMethodParameter()`方法。关于`forMethodParameter()`方法，我们稍后分析。

#### forField

```java
/**
 * 通过Field构造，内部是使用`FieldTypeProvider`来辅助转换的。
 * Return a {@link ResolvableType} for the specified {@link Field}.
 * @param field the source field
 * @return a {@link ResolvableType} for the specified field
 * @see #forField(Field, Class)
 */
public static ResolvableType forField(Field field) {
	Assert.notNull(field, "Field must not be null");
	return forType(null, new FieldTypeProvider(field), null);
}

/**
 * 同上，只是多了forType().as()的操作。
 * Return a {@link ResolvableType} for the specified {@link Field} with a given
 * implementation.
 * <p>Use this variant when the class that declares the field includes generic
 * parameter variables that are satisfied by the implementation class.
 * @param field the source field
 * @param implementationClass the implementation class
 * @return a {@link ResolvableType} for the specified field
 * @see #forField(Field)
 */
public static ResolvableType forField(Field field, Class<?> implementationClass) {
	Assert.notNull(field, "Field must not be null");
	ResolvableType owner = forType(implementationClass).as(field.getDeclaringClass());
	return forType(null, new FieldTypeProvider(field), owner.asVariableResolver());
}

/**
 * 同上，只是实现类型由Class变更为了ResolvableType
 * Return a {@link ResolvableType} for the specified {@link Field} with a given
 * implementation.
 * <p>Use this variant when the class that declares the field includes generic
 * parameter variables that are satisfied by the implementation type.
 * @param field the source field
 * @param implementationType the implementation type
 * @return a {@link ResolvableType} for the specified field
 * @see #forField(Field)
 */
public static ResolvableType forField(Field field, @Nullable ResolvableType implementationType) {
	Assert.notNull(field, "Field must not be null");
	ResolvableType owner = (implementationType != null ? implementationType : NONE);
	owner = owner.as(field.getDeclaringClass());
	return forType(null, new FieldTypeProvider(field), owner.asVariableResolver());
}

/**
 * 同上，只是增加了嵌套级数。
 * 例如：A<B<C>>。
 * 1. 当嵌套级数为1时，为B<C>；
 * 2. 当嵌套级数为2时，为C；
 * 3. 当嵌套级数为3时，为ResolvableType$EmptyType
 * Return a {@link ResolvableType} for the specified {@link Field} with the
 * given nesting level.
 * @param field the source field
 * @param nestingLevel the nesting level (1 for the outer level; 2 for a nested
 * generic type; etc)
 * @see #forField(Field)
 */
public static ResolvableType forField(Field field, int nestingLevel) {
	Assert.notNull(field, "Field must not be null");
	return forType(null, new FieldTypeProvider(field), null).getNested(nestingLevel);
}

/**
 * 这是更底层的实现，同时支持嵌套级数和实现类。
 * Return a {@link ResolvableType} for the specified {@link Field} with a given
 * implementation and the given nesting level.
 * <p>Use this variant when the class that declares the field includes generic
 * parameter variables that are satisfied by the implementation class.
 * @param field the source field
 * @param nestingLevel the nesting level (1 for the outer level; 2 for a nested
 * generic type; etc)
 * @param implementationClass the implementation class
 * @return a {@link ResolvableType} for the specified field
 * @see #forField(Field)
 */
public static ResolvableType forField(Field field, int nestingLevel, @Nullable Class<?> implementationClass) {
	Assert.notNull(field, "Field must not be null");
	ResolvableType owner = forType(implementationClass).as(field.getDeclaringClass());
	return forType(null, new FieldTypeProvider(field), owner.asVariableResolver()).getNested(nestingLevel);
}
```

#### forMethodReturnType

```java
/**
 * Return a {@link ResolvableType} for the specified {@link Method} return type.
 * @param method the source for the method return type
 * @return a {@link ResolvableType} for the specified method return
 * @see #forMethodReturnType(Method, Class)
 */
public static ResolvableType forMethodReturnType(Method method) {
	Assert.notNull(method, "Method must not be null");
	return forMethodParameter(new MethodParameter(method, -1));
}

/**
 * Return a {@link ResolvableType} for the specified {@link Method} return type.
 * Use this variant when the class that declares the method includes generic
 * parameter variables that are satisfied by the implementation class.
 * @param method the source for the method return type
 * @param implementationClass the implementation class
 * @return a {@link ResolvableType} for the specified method return
 * @see #forMethodReturnType(Method)
 */
public static ResolvableType forMethodReturnType(Method method, Class<?> implementationClass) {
	Assert.notNull(method, "Method must not be null");
	MethodParameter methodParameter = new MethodParameter(method, -1, implementationClass);
	return forMethodParameter(methodParameter);
}
```

最终调用的是`forMethodParameter()`方法，只是传入的下标为-1。

#### forMethodParameter

```java
/**
 * Return a {@link ResolvableType} for the specified {@link Method} parameter.
 * @param method the source method (must not be {@code null})
 * @param parameterIndex the parameter index
 * @return a {@link ResolvableType} for the specified method parameter
 * @see #forMethodParameter(Method, int, Class)
 * @see #forMethodParameter(MethodParameter)
 */
public static ResolvableType forMethodParameter(Method method, int parameterIndex) {
	Assert.notNull(method, "Method must not be null");
	return forMethodParameter(new MethodParameter(method, parameterIndex));
}

/**
 * Return a {@link ResolvableType} for the specified {@link Method} parameter with a
 * given implementation. Use this variant when the class that declares the method
 * includes generic parameter variables that are satisfied by the implementation class.
 * @param method the source method (must not be {@code null})
 * @param parameterIndex the parameter index
 * @param implementationClass the implementation class
 * @return a {@link ResolvableType} for the specified method parameter
 * @see #forMethodParameter(Method, int, Class)
 * @see #forMethodParameter(MethodParameter)
 */
public static ResolvableType forMethodParameter(Method method, int parameterIndex, Class<?> implementationClass) {
	Assert.notNull(method, "Method must not be null");
	MethodParameter methodParameter = new MethodParameter(method, parameterIndex, implementationClass);
	return forMethodParameter(methodParameter);
}

/**
 * Return a {@link ResolvableType} for the specified {@link MethodParameter}.
 * @param methodParameter the source method parameter (must not be {@code null})
 * @return a {@link ResolvableType} for the specified method parameter
 * @see #forMethodParameter(Method, int)
 */
public static ResolvableType forMethodParameter(MethodParameter methodParameter) {
	return forMethodParameter(methodParameter, (Type) null);
}

/**
 * Return a {@link ResolvableType} for the specified {@link MethodParameter} with a
 * given implementation type. Use this variant when the class that declares the method
 * includes generic parameter variables that are satisfied by the implementation type.
 * @param methodParameter the source method parameter (must not be {@code null})
 * @param implementationType the implementation type
 * @return a {@link ResolvableType} for the specified method parameter
 * @see #forMethodParameter(MethodParameter)
 */
public static ResolvableType forMethodParameter(MethodParameter methodParameter,
		@Nullable ResolvableType implementationType) {

	Assert.notNull(methodParameter, "MethodParameter must not be null");
	implementationType = (implementationType != null ? implementationType :
			forType(methodParameter.getContainingClass()));
	ResolvableType owner = implementationType.as(methodParameter.getDeclaringClass());
	return forType(null, new MethodParameterTypeProvider(methodParameter), owner.asVariableResolver()).
			getNested(methodParameter.getNestingLevel(), methodParameter.typeIndexesPerLevel);
}

/**
 * Return a {@link ResolvableType} for the specified {@link MethodParameter},
 * overriding the target type to resolve with a specific given type.
 * @param methodParameter the source method parameter (must not be {@code null})
 * @param targetType the type to resolve (a part of the method parameter's type)
 * @return a {@link ResolvableType} for the specified method parameter
 * @see #forMethodParameter(Method, int)
 */
public static ResolvableType forMethodParameter(MethodParameter methodParameter, @Nullable Type targetType) {
	Assert.notNull(methodParameter, "MethodParameter must not be null");
	return forMethodParameter(methodParameter, targetType, methodParameter.getNestingLevel());
}

/**
 * Return a {@link ResolvableType} for the specified {@link MethodParameter} at
 * a specific nesting level, overriding the target type to resolve with a specific
 * given type.
 * @param methodParameter the source method parameter (must not be {@code null})
 * @param targetType the type to resolve (a part of the method parameter's type)
 * @param nestingLevel the nesting level to use
 * @return a {@link ResolvableType} for the specified method parameter
 * @since 5.2
 * @see #forMethodParameter(Method, int)
 */
static ResolvableType forMethodParameter(
		MethodParameter methodParameter, @Nullable Type targetType, int nestingLevel) {

	ResolvableType owner = forType(methodParameter.getContainingClass()).as(methodParameter.getDeclaringClass());
	return forType(targetType, new MethodParameterTypeProvider(methodParameter), owner.asVariableResolver()).
			getNested(nestingLevel, methodParameter.typeIndexesPerLevel);
}
```

和`forField()`的参数字段类型大体相同，只是多了下标和目标类型的支持而已。

#### forType

除了`ResolvableType`的构造方法，上面的所有方法，最终都调用了`forType`方法。因此，这个方法非常关键。

```java
/**
 * Return a {@link ResolvableType} for the specified {@link Type} backed by a given
 * {@link VariableResolver}.
 * @param type the source type or {@code null}
 * @param typeProvider the type provider or {@code null}
 * @param variableResolver the variable resolver or {@code null}
 * @return a {@link ResolvableType} for the specified {@link Type} and {@link VariableResolver}
 */
static ResolvableType forType(
		@Nullable Type type, @Nullable TypeProvider typeProvider, @Nullable VariableResolver variableResolver) {
    // 当类型为null且类型提供者不为空时，重新生成type。
    // 这是为什么呢？其实是为了更好地支持序列化操作。
	if (type == null && typeProvider != null) {
		type = SerializableTypeWrapper.forTypeProvider(typeProvider);
	}
	// 类型为空时，直接映射为空对象返回
	if (type == null) {
		return NONE;
	}

    // 简单的Class类型，也可以直接构造出来
	// For simple Class references, build the wrapper right away -
	// no expensive resolution necessary, so not worth caching...
	if (type instanceof Class) {
		return new ResolvableType(type, typeProvider, variableResolver, (ResolvableType) null);
	}

	// Purge empty entries on access since we don't have a clean-up thread or the like.
	cache.purgeUnreferencedEntries();

	// Check the cache - we may have a ResolvableType which has been resolved before...
	// 这儿先尝试用缓存获取，如果没获取到，则构造一次，存储到缓存，并返回resolved字段。
	// resolved字段在哪生成的呢？其实是在ResolvableType构造方法。
	// ResolvableType构造方法又调用了resolveClass方法。
	ResolvableType resultType = new ResolvableType(type, typeProvider, variableResolver);
	ResolvableType cachedType = cache.get(resultType);
	if (cachedType == null) {
		cachedType = new ResolvableType(type, typeProvider, variableResolver, resultType.hash);
		cache.put(cachedType, cachedType);
	}
	resultType.resolved = cachedType.resolved;
	return resultType;
}
```

#### SerializableTypeWrapper.forTypeProvider

在`forType()`中有一个关键的地方，在于`SerializableTypeWrapper.forTypeProvider`的调用，它主要提供了序列化功能的支持。

```java
private static final Class<?>[] SUPPORTED_SERIALIZABLE_TYPES = {
		GenericArrayType.class, ParameterizedType.class, TypeVariable.class, WildcardType.class};

static Type forTypeProvider(TypeProvider provider) {
	Type providedType = provider.getType();
	if (providedType == null || providedType instanceof Serializable) {
		// No serializable type wrapping necessary (e.g. for java.lang.Class)
		return providedType;
	}
	if (GraalDetector.inImageCode() || !Serializable.class.isAssignableFrom(Class.class)) {
		// Let's skip any wrapping attempts if types are generally not serializable in
		// the current runtime environment (even java.lang.Class itself, e.g. on Graal)
		return providedType;
	}

	// Obtain a serializable type proxy for the given provider...
	Type cached = cache.get(providedType);
	if (cached != null) {
		return cached;
	}
	for (Class<?> type : SUPPORTED_SERIALIZABLE_TYPES) {
		if (type.isInstance(providedType)) {
			ClassLoader classLoader = provider.getClass().getClassLoader();
			Class<?>[] interfaces = new Class<?>[] {type, SerializableTypeProxy.class, Serializable.class};
			InvocationHandler handler = new TypeProxyInvocationHandler(provider);
			cached = (Type) Proxy.newProxyInstance(classLoader, interfaces, handler);
			cache.put(providedType, cached);
			return cached;
		}
	}
	throw new IllegalArgumentException("Unsupported Type class: " + providedType.getClass().getName());
}
```

其主要对非Class的类型，通过JDK动态代理的方式，对序列化功能（通过实现Serializable接口）进行了增强。除此之外，还增加了对`equals`、`hashCode`、`getTypeProvider`等的增强实现。

```java
private static class TypeProxyInvocationHandler implements InvocationHandler, Serializable {

	private final TypeProvider provider;

	public TypeProxyInvocationHandler(TypeProvider provider) {
		this.provider = provider;
	}

	@Override
	@Nullable
	public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
	    // equals动态代理
		if (method.getName().equals("equals") && args != null) {
			Object other = args[0];
			// Unwrap proxies for speed
			// 解包的目的是为了提高性能
			if (other instanceof Type) {
				other = unwrap((Type) other);
			}
			return ObjectUtils.nullSafeEquals(this.provider.getType(), other);
		}
		// hashCode动态代理
		else if (method.getName().equals("hashCode")) {
			return ObjectUtils.nullSafeHashCode(this.provider.getType());
		}
		// getTypeProvider动态代理
		else if (method.getName().equals("getTypeProvider")) {
			return this.provider;
		}
		
        // 返回类型为Type的动态代理，最终调用的是provider的实现
		if (Type.class == method.getReturnType() && args == null) {
			return forTypeProvider(new MethodInvokeTypeProvider(this.provider, method, -1));
		}
		// 返回类型为Type数组的动态代理，最终调用的是provider的实现
		else if (Type[].class == method.getReturnType() && args == null) {
			Type[] result = new Type[((Type[]) method.invoke(this.provider.getType())).length];
			for (int i = 0; i < result.length; i++) {
				result[i] = forTypeProvider(new MethodInvokeTypeProvider(this.provider, method, i));
			}
			return result;
		}

		try {
			return method.invoke(this.provider.getType(), args);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
}
```

#### resolveClass

```java
@Nullable
private Class<?> resolveClass() {
    // 空类型直接返回
	if (this.type == EmptyType.INSTANCE) {
		return null;
	}
	// Class对象，也直接返回
	if (this.type instanceof Class) {
		return (Class<?>) this.type;
	}
	// 泛型数组类型，通过getComponentType()来转换生成
	if (this.type instanceof GenericArrayType) {
		Class<?> resolvedComponent = getComponentType().resolve();
		return (resolvedComponent != null ? Array.newInstance(resolvedComponent, 0).getClass() : null);
	}
	// 底层调用了resolveType()
	return resolveType().resolve();
}
```

#### resolveType

```java
/**
 * Resolve this type by a single level, returning the resolved value or {@link #NONE}.
 * <p>Note: The returned {@link ResolvableType} should only be used as an intermediary
 * as it cannot be serialized.
 */
ResolvableType resolveType() {
    // 参数化类型，通过`getRawType()`获取原始类型
	if (this.type instanceof ParameterizedType) {
		return forType(((ParameterizedType) this.type).getRawType(), this.variableResolver);
	}
	// 通配符类型，通过上界和下届获取类型
	if (this.type instanceof WildcardType) {
		Type resolved = resolveBounds(((WildcardType) this.type).getUpperBounds());
		if (resolved == null) {
			resolved = resolveBounds(((WildcardType) this.type).getLowerBounds());
		}
		return forType(resolved, this.variableResolver);
	}
	// 类型变量类型，当类型变量处理器不为空，则直接由其处理；否则通过其上界来获取
	if (this.type instanceof TypeVariable) {
		TypeVariable<?> variable = (TypeVariable<?>) this.type;
		// Try default variable resolution
		if (this.variableResolver != null) {
			ResolvableType resolved = this.variableResolver.resolveVariable(variable);
			if (resolved != null) {
				return resolved;
			}
		}
		// Fallback to bounds
		return forType(resolveBounds(variable.getBounds()), this.variableResolver);
	}
	return NONE;
}
```

这个方法对参数化类型、通配符类型、类型变量进行了适配和转换。最终又调回了`forType()`方法。这样就出现了循环调用的情况，感觉会出问题，如下所示：

```
forType() -> ResolvableType() -> resolveClass() -> resolveType() -> forType() -> ...
```

其实，这样调用是有原因的。原因在于，参数化类型、通配符类型、类型变量，都是可以相互嵌套的。例如：`List<? extends List<String>>`，就是通配符嵌套参数化类型。

#### 其他方法

其他的方法含义，可以参考官方提供的测试类，[`ResolvableTypeTests`](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/test/java/org/springframework/core/ResolvableTypeTests.java)。其覆盖场景非常广泛，简简单单的一个测试类，代码行数就达到了1300+行，简直令人发指！

## 总结

本文翻译了部分Spring官方提供的javadoc，同时对内部的核心实现方法进行了详细而全面的分析，基本上了解了Spring是如何对java Type进行封装的。当然，分析源码的过程中，我们看到其实现非常精彩。

但是，对于一些细节，楼主并没有深入分析。楼主觉得，如果进行全面的分析，浪费篇章的同时，也可能出现阐释不清楚的情况。