---
layout: post
title:  Spring IOC源码解析（04）AliasRegistry
date:   2020-05-24 23:00:52 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## AliasRegistry

中文翻译为`别名注册器`，也就是对别名进行管理。

```java
public interface AliasRegistry {

	/**
	 * 用于注册一个别名。当别名存在的情况下，将抛出异常。
	 * Given a name, register an alias for it.
	 * @param name the canonical name
	 * @param alias the alias to be registered
	 * @throws IllegalStateException if the alias is already in use
	 * and may not be overridden
	 */
	void registerAlias(String name, String alias);

	/**
	 * 移除别名
	 * Remove the specified alias from this registry.
	 * @param alias the alias to remove
	 * @throws IllegalStateException if no such alias was found
	 */
	void removeAlias(String alias);

	/**
	 * 判断当前传入的字符串是否是一个别名
	 * Determine whether the given name is defined as an alias
	 * (as opposed to the name of an actually registered component).
	 * @param name the name to check
	 * @return whether the given name is an alias
	 */
	boolean isAlias(String name);

	/**
	 * 获取指定名称的所有别名数组。当没有别名的时候，返回空数组
	 * Return the aliases for the given name, if defined.
	 * @param name the name to check for aliases
	 * @return the aliases, or an empty array if none
	 */
	String[] getAliases(String name);
}
```

## SimpleAliasRegistry

`AliasRegistry`是一个接口，定义了别名的操作规范。而`SimpleAliasRegistry`是`AliasRegistry`的实现。

```java
public class SimpleAliasRegistry implements AliasRegistry {

	/** Logger available to subclasses. */
	protected final Log logger = LogFactory.getLog(getClass());

	/** Map from alias to canonical name. */
	// 使用ConcurrentHashMap来存储。额外需要注意以下几点：
	// 1. key为别名，value为别名对应的名称；
	// 2. 别名不允许重复；
	// 3. 别名不允许存在循环；
	// 4. value也可以是别名，这样就代表着存在多重别名。
	//    比如：a的别名是b，b的别名是c，c的别名是d。那么存储如下：
	//    1. d -> c
	//    2. c -> b
	//    3. b -> a
	private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);


	@Override
	public void registerAlias(String name, String alias) {
	    // 别名和正名不允许为空
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
		synchronized (this.aliasMap) {
		    // 别名和正名相同，则直接移除，说明不存在别名
			if (alias.equals(name)) {
				this.aliasMap.remove(alias);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
				}
			}
			else {
				String registeredName = this.aliasMap.get(alias);
				// 别名已经存在的话，有以下几种情况：
				// 1. 如果传入的正名和已注册的正名相同，则直接返回，这儿说明支持幂等
				// 2. 如果传入的正名和已注册的正名不同，如果不允许覆盖，则抛出异常
				// 3. 如果传入的正名和已注册的正名不同，或者允许覆盖，则先检查是否存在循环，如果检查通过，则注册成功。
				if (registeredName != null) {
					if (registeredName.equals(name)) {
						// An existing alias - no need to re-register
						return;
					}
					// 不允许覆盖
					if (!allowAliasOverriding()) {
						throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
								name + "': It is already registered for name '" + registeredName + "'.");
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
								registeredName + "' with new target name '" + name + "'");
					}
				}
				checkForAliasCircle(name, alias);
				this.aliasMap.put(alias, name);
				if (logger.isTraceEnabled()) {
					logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
				}
			}
		}
	}

	/**
	 * 这个方法用于返回是否覆盖别名，因为是protected修饰的，所以允许子类重写
	 * Determine whether alias overriding is allowed.
	 * <p>Default is {@code true}.
	 */
	protected boolean allowAliasOverriding() {
		return true;
	}

	/**
	 * 判断alias是否是name的别名。注意：这儿使用了递归的方式处理别名的传递。
	 * Determine whether the given name has the given alias registered.
	 * @param name the name to check
	 * @param alias the alias to look for
	 * @since 4.2.1
	 */
	public boolean hasAlias(String name, String alias) {
		String registeredName = this.aliasMap.get(alias);
		return ObjectUtils.nullSafeEquals(registeredName, name) || (registeredName != null
				&& hasAlias(name, registeredName));
	}

    /**
     * 移除别名，很简单，直接调用remove方法移掉即可
     */
	@Override
	public void removeAlias(String alias) {
		synchronized (this.aliasMap) {
			String name = this.aliasMap.remove(alias);
			if (name == null) {
				throw new IllegalStateException("No alias '" + alias + "' registered");
			}
		}
	}

    /**
     * 判断指定名称是否是一个别名
     */
	@Override
	public boolean isAlias(String name) {
		return this.aliasMap.containsKey(name);
	}

    /**
     * 获取别名列表，内部使用`retrieveAliases`来实现
     */
	@Override
	public String[] getAliases(String name) {
		List<String> result = new ArrayList<>();
		synchronized (this.aliasMap) {
			retrieveAliases(name, result);
		}
		return StringUtils.toStringArray(result);
	}

	/**
	 * 获取别名列表，结果存放到result参数中，这儿需要注意两点：
	 * 1. 这儿会对aliasMap进行全量遍历，虽然不存在IO操作，但是当数量很多的情况下，效率也会很低；
	 * 2. 同时使用了递归的方式处理别名的传递。
	 * Transitively retrieve all aliases for the given name.
	 * @param name the target name to find aliases for
	 * @param result the resulting aliases list
	 */
	private void retrieveAliases(String name, List<String> result) {
		this.aliasMap.forEach((alias, registeredName) -> {
			if (registeredName.equals(name)) {
				result.add(alias);
				retrieveAliases(alias, result);
			}
		});
	}

	/**
	 * 使用字符串值解析器，对当前别名注册器进行解析转换。
	 * 通常使用的场景在于placeholders，也就是占位符变量替换的场景。
	 * Resolve all alias target names and aliases registered in this
	 * registry, applying the given {@link StringValueResolver} to them.
	 * <p>The value resolver may for example resolve placeholders
	 * in target bean names and even in alias names.
	 * @param valueResolver the StringValueResolver to apply
	 */
	public void resolveAliases(StringValueResolver valueResolver) {
		Assert.notNull(valueResolver, "StringValueResolver must not be null");
		synchronized (this.aliasMap) {
			Map<String, String> aliasCopy = new HashMap<>(this.aliasMap);
			aliasCopy.forEach((alias, registeredName) -> {
				String resolvedAlias = valueResolver.resolveStringValue(alias);
				String resolvedName = valueResolver.resolveStringValue(registeredName);
				// 如果解析之后的正名和解析之后的别名相同，则直接移除。
				if (resolvedAlias == null || resolvedName == null || resolvedAlias.equals(resolvedName)) {
					this.aliasMap.remove(alias);
				}
				// 如果解析之后的别名和解析之前的别名不同，那么需要做以下几件事情：
				// 1. 如果注册器中解析之后的别名对应的正名和解析之后的正名不同，则抛出异常。这是因为存在重复注册的问题。
				// 2. 检查循环依赖
				// 3. 将解析之前的别名移除掉
				// 4. 将解析之后的别名添加进去
				else if (!resolvedAlias.equals(alias)) {
					String existingName = this.aliasMap.get(resolvedAlias);
					if (existingName != null) {
						if (existingName.equals(resolvedName)) {
							// Pointing to existing alias - just remove placeholder
							this.aliasMap.remove(alias);
							return;
						}
						throw new IllegalStateException(
								"Cannot register resolved alias '" + resolvedAlias + "' (original: '" + alias +
								"') for name '" + resolvedName + "': It is already registered for name '" +
								registeredName + "'.");
					}
					checkForAliasCircle(resolvedName, resolvedAlias);
					this.aliasMap.remove(alias);
					this.aliasMap.put(resolvedAlias, resolvedName);
				}
				// 如果已注册的正名和解析之后的正名不同，直接注册进去
				else if (!registeredName.equals(resolvedName)) {
					this.aliasMap.put(alias, resolvedName);
				}
				// 其他情况，直接忽略掉
			});
		}
	}

	/**
	 * 检查循环依赖，通过调用hasAlias方法实现，hasAlias方法内部使用了递归
	 * Check whether the given name points back to the given alias as an alias
	 * in the other direction already, catching a circular reference upfront
	 * and throwing a corresponding IllegalStateException.
	 * @param name the candidate name
	 * @param alias the candidate alias
	 * @see #registerAlias
	 * @see #hasAlias
	 */
	protected void checkForAliasCircle(String name, String alias) {
		if (hasAlias(alias, name)) {
			throw new IllegalStateException("Cannot register alias '" + alias +
					"' for name '" + name + "': Circular reference - '" +
					name + "' is a direct or indirect alias for '" + alias + "' already");
		}
	}

	/**
	 * 根据别名，找到最原始的正名。比如a的别名为b，b的别名为c，c的别名为d。
	 * 当在此方法中传入d，那么返回的是a。
	 * Determine the raw name, resolving aliases to canonical names.
	 * @param name the user-specified name
	 * @return the transformed name
	 */
	public String canonicalName(String name) {
		String canonicalName = name;
		// Handle aliasing...
		String resolvedName;
		do {
			resolvedName = this.aliasMap.get(canonicalName);
			if (resolvedName != null) {
				canonicalName = resolvedName;
			}
		}
		while (resolvedName != null);
		return canonicalName;
	}
}
```