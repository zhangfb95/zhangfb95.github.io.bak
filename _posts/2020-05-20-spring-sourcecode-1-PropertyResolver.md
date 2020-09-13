---
layout: post
title:  Spring IOC源码解析（01）PropertyResolver
date:   2020-05-20 19:51:07 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## 前言

`PropertyResolver`，英文翻译为属性解析器。也就是说，我们可以通过获取属性、设置属性、替换变量等操作。

## 特别说明

在分析源码的过程中，我们有的时候想要方便地查看接口、类的继承关系。在Intellij IDEA中，我们可以通过以下步骤来完成。

1. 选中需要查看的接口或类
2. Navigate -> Type Hirerarchy
3. 在Hirerarchy窗口中，点击Subtypes Hirerarchy
4. 在Hirerarchy窗口中，点击Expand All
5. 在Hirerarchy窗口中，通过快捷键Ctrl+A选中需要查看的接口或类
6. 在Hirerarchy窗口中，鼠标右键 -> Diagrams -> Show Diagrams -> Java Class Diagrams
7. 通过以上步骤，就能看到选中接口或类的继承关系了。

## 继承关系图

![继承关系图](https://upload-images.jianshu.io/upload_images/845143-dc314e6606d7463e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## PropertyResolver源码分析

所有方法都是围绕着获取属性以及替换变量而展开的。

```java
public interface PropertyResolver {

	/**
	 * 是否包含指定属性，即获取的属性是否为null
	 */
	boolean containsProperty(String key);

	/**
	 * 获取属性
	 */
	@Nullable
	String getProperty(String key);

	/**
	 * 获取属性，当属性未获取到，或者为null时，返回默认值
	 */
	String getProperty(String key, String defaultValue);

	/**
	 * 获取特定类型的属性
	 */
	@Nullable
	<T> T getProperty(String key, Class<T> targetType);

	/**
	 * 获取特定类型的属性，当属性为获取到，或者为null时，返回默认值
	 */
	<T> T getProperty(String key, Class<T> targetType, T defaultValue);

	/**
	 * 获取属性，当属性未获取到，或者为null时，抛出异常
	 */
	String getRequiredProperty(String key) throws IllegalStateException;

	/**
	 * 获取特定类型的属性，当属性未获取到，或者为null时，抛出异常
	 */
	<T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;

	/**
	 * 替换指定的文本中的变量
	 */
	String resolvePlaceholders(String text);

	/**
	 * 替换指定的文本中的变量，如果未替换完，则抛出异常
	 */
	String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;
}
```

## ConfigurablePropertyResolver

`ConfigurablePropertyResolver`在`PropertyResolver`的基础上，增加了可配置的操作。包括：

1. 获取或设置转换服务
2. 设置占位符前缀
3. 设置占位符后缀
4. 设置属性值分隔符
5. 其他

```java
public interface ConfigurablePropertyResolver extends PropertyResolver {

	/**
	 * 获取转换服务
	 */
	ConfigurableConversionService getConversionService();

	/**
	 * 设置转换服务
	 */
	void setConversionService(ConfigurableConversionService conversionService);

	/**
	 * 设置占位符前缀
	 */
	void setPlaceholderPrefix(String placeholderPrefix);

	/**
	 * 设置占位符后缀
	 */
	void setPlaceholderSuffix(String placeholderSuffix);

	/**
	 * 设置属性值分隔符
	 */
	void setValueSeparator(@Nullable String valueSeparator);

	/**
	 * 设置是否忽略未解决的变量
	 * @since 3.2
	 */
	void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);

	/**
	 * 设置必须解决的变量，即：哪些变量必须要存在
	 */
	void setRequiredProperties(String... requiredProperties);

	/**
	 * 校验需要的变量是否存在，如果不存在，则抛出异常
	 */
	void validateRequiredProperties() throws MissingRequiredPropertiesException;
}
```

## AbstractPropertyResolver

`AbstractPropertyResolver`为抽象的属性解析器，其包含大部分的逻辑实现。

```java
public abstract class AbstractPropertyResolver implements ConfigurablePropertyResolver {

    // 日志对象
	protected final Log logger = LogFactory.getLog(getClass());

    // 转换服务，字段默认为null，但是获取方法中将其设置为了DefaultConversionService
	@Nullable
	private volatile ConfigurableConversionService conversionService;

    // 非严格模式下的属性占位符辅助类
	@Nullable
	private PropertyPlaceholderHelper nonStrictHelper;

    // 严格模式下的属性占位符辅助类
	@Nullable
	private PropertyPlaceholderHelper strictHelper;
	
    // 此属性用于判断是使用严格模式，还是非严格模式
	private boolean ignoreUnresolvableNestedPlaceholders = false;
    
    // 占位符前缀。不设置的话，有默认值
	private String placeholderPrefix = SystemPropertyUtils.PLACEHOLDER_PREFIX;
    
    // 占位符后缀。不设置的话，有默认值
	private String placeholderSuffix = SystemPropertyUtils.PLACEHOLDER_SUFFIX;
    
    // 值分隔符。不设置的话，有默认值
	@Nullable
	private String valueSeparator = SystemPropertyUtils.VALUE_SEPARATOR;
    
    // 必须的属性
	private final Set<String> requiredProperties = new LinkedHashSet<>();

	@Override
	public ConfigurableConversionService getConversionService() {
		// Need to provide an independent DefaultConversionService, not the
		// shared DefaultConversionService used by PropertySourcesPropertyResolver.
		// 以懒加载的方式设置转换服务
		ConfigurableConversionService cs = this.conversionService;
		if (cs == null) {
			synchronized (this) {
				cs = this.conversionService;
				if (cs == null) {
					cs = new DefaultConversionService();
					this.conversionService = cs;
				}
			}
		}
		return cs;
	}

	@Override
	public void setConversionService(ConfigurableConversionService conversionService) {
		Assert.notNull(conversionService, "ConversionService must not be null");
		this.conversionService = conversionService;
	}

	/**
	 * 设置占位符前缀标识符
	 */
	@Override
	public void setPlaceholderPrefix(String placeholderPrefix) {
		Assert.notNull(placeholderPrefix, "'placeholderPrefix' must not be null");
		this.placeholderPrefix = placeholderPrefix;
	}

	/**
	 * 设置占位符后缀标识符
	 */
	@Override
	public void setPlaceholderSuffix(String placeholderSuffix) {
		Assert.notNull(placeholderSuffix, "'placeholderSuffix' must not be null");
		this.placeholderSuffix = placeholderSuffix;
	}

	/**
	 * 设置值分隔符
	 */
	@Override
	public void setValueSeparator(@Nullable String valueSeparator) {
		this.valueSeparator = valueSeparator;
	}

	/**
	 * 设置是否忽略未解决的变量
	 * @since 3.2
	 */
	@Override
	public void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders) {
		this.ignoreUnresolvableNestedPlaceholders = ignoreUnresolvableNestedPlaceholders;
	}

    /**
     * 设置必须存在的属性
     */
	@Override
	public void setRequiredProperties(String... requiredProperties) {
		Collections.addAll(this.requiredProperties, requiredProperties);
	}
    
    /**
     * 校验必须存在的属性。其内部是通过遍历的方式来实现的，只要有一个不存在，就抛出异常
     */
	@Override
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}

    /**
     * 是否包含属性。是通过属性值是否为null来判断的
     */
	@Override
	public boolean containsProperty(String key) {
		return (getProperty(key) != null);
	}

    /**
     * 获取属性
     */
	@Override
	@Nullable
	public String getProperty(String key) {
		return getProperty(key, String.class);
	}

    /**
     * 获取属性。未获取到时，返回默认值
     */
	@Override
	public String getProperty(String key, String defaultValue) {
		String value = getProperty(key);
		return (value != null ? value : defaultValue);
	}

    /**
     * 获取属性。未获取到时，返回默认值
     */
	@Override
	public <T> T getProperty(String key, Class<T> targetType, T defaultValue) {
		T value = getProperty(key, targetType);
		return (value != null ? value : defaultValue);
	}

    /**
     * 获取属性。未获取到时，抛出异常
     */
	@Override
	public String getRequiredProperty(String key) throws IllegalStateException {
		String value = getProperty(key);
		if (value == null) {
			throw new IllegalStateException("Required key '" + key + "' not found");
		}
		return value;
	}

    /**
     * 获取属性。未获取到时，抛出异常
     */
	@Override
	public <T> T getRequiredProperty(String key, Class<T> valueType) throws IllegalStateException {
		T value = getProperty(key, valueType);
		if (value == null) {
			throw new IllegalStateException("Required key '" + key + "' not found");
		}
		return value;
	}

    /**
     * 使用非严格模式的辅助类，替换变量
     */
	@Override
	public String resolvePlaceholders(String text) {
		if (this.nonStrictHelper == null) {
			this.nonStrictHelper = createPlaceholderHelper(true);
		}
		return doResolvePlaceholders(text, this.nonStrictHelper);
	}

    /**
     * 使用严格模式的辅助类，替换变量
     */
	@Override
	public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
		if (this.strictHelper == null) {
			this.strictHelper = createPlaceholderHelper(false);
		}
		return doResolvePlaceholders(text, this.strictHelper);
	}

	/**
	 * 替换变量
	 * @since 3.2
	 * @see #setIgnoreUnresolvableNestedPlaceholders
	 */
	protected String resolveNestedPlaceholders(String value) {
		return (this.ignoreUnresolvableNestedPlaceholders ?
				resolvePlaceholders(value) : resolveRequiredPlaceholders(value));
	}

    /**
     * 创建辅助类
     */
	private PropertyPlaceholderHelper createPlaceholderHelper(boolean ignoreUnresolvablePlaceholders) {
		return new PropertyPlaceholderHelper(this.placeholderPrefix, this.placeholderSuffix,
				this.valueSeparator, ignoreUnresolvablePlaceholders);
	}
	
    /**
     * 使用辅助类替换变量
     */
	private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
		return helper.replacePlaceholders(text, this::getPropertyAsRawString);
	}

	/**
	 * 转换值到特定类型。当类型不匹配时需通过转换服务进行转换
	 * @since 4.3.5
	 */
	@SuppressWarnings("unchecked")
	@Nullable
	protected <T> T convertValueIfNecessary(Object value, @Nullable Class<T> targetType) {
		if (targetType == null) {
			return (T) value;
		}
		ConversionService conversionServiceToUse = this.conversionService;
		if (conversionServiceToUse == null) {
			// Avoid initialization of shared DefaultConversionService if
			// no standard type conversion is needed in the first place...
			if (ClassUtils.isAssignableValue(targetType, value)) {
				return (T) value;
			}
			conversionServiceToUse = DefaultConversionService.getSharedInstance();
		}
		return conversionServiceToUse.convert(value, targetType);
	}


	/**
	 * 获取原始类型的属性，即：不做任何变量转换
	 */
	@Nullable
	protected abstract String getPropertyAsRawString(String key);
}
```

## PropertySourcesPropertyResolver

`PropertySourcesPropertyResolver`为属性解析器的最终实现，所有的操作都是围绕`PropertySources`来展开的，该类表示“属性源列表对象”。

```java
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {

    // 属性源列表对象，我们将在下一章进行分析
	@Nullable
	private final PropertySources propertySources;


	/**
	 * 通过`PropertySources`进行构造
	 * @param propertySources the set of {@link PropertySource} objects to use
	 */
	public PropertySourcesPropertyResolver(@Nullable PropertySources propertySources) {
		this.propertySources = propertySources;
	}

    /**
     * 是否包含此属性
     */
	@Override
	public boolean containsProperty(String key) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				if (propertySource.containsProperty(key)) {
					return true;
				}
			}
		}
		return false;
	}

	@Override
	@Nullable
	public String getProperty(String key) {
		return getProperty(key, String.class, true);
	}

	@Override
	@Nullable
	public <T> T getProperty(String key, Class<T> targetValueType) {
		return getProperty(key, targetValueType, true);
	}

	@Override
	@Nullable
	protected String getPropertyAsRawString(String key) {
		return getProperty(key, String.class, false);
	}

    /**
     * 获取属性，并将属性进行变量替换
     */
	@Nullable
	protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				if (logger.isTraceEnabled()) {
					logger.trace("Searching for key '" + key + "' in PropertySource '" +
							propertySource.getName() + "'");
				}
				Object value = propertySource.getProperty(key);
				if (value != null) {
					if (resolveNestedPlaceholders && value instanceof String) {
						value = resolveNestedPlaceholders((String) value);
					}
					logKeyFound(key, propertySource, value);
					return convertValueIfNecessary(value, targetValueType);
				}
			}
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Could not find key '" + key + "' in any property source");
		}
		return null;
	}

	/**
	 * 记录一下日志
	 * @since 4.3.1
	 */
	protected void logKeyFound(String key, PropertySource<?> propertySource, Object value) {
		if (logger.isDebugEnabled()) {
			logger.debug("Found key '" + key + "' in PropertySource '" + propertySource.getName() +
					"' with value of type " + value.getClass().getSimpleName());
		}
	}
}
```

## Environment

`Environment`在`PropertyResolver`的基础上，增加了对profile的支持。

```java
public interface Environment extends PropertyResolver {

	/**
	 * 获取激活的profile数组
	 */
	String[] getActiveProfiles();

	/**
	 * 获取默认的profile数组
	 */
	String[] getDefaultProfiles();

	/**
	 * 是否接受传入的profile数组
	 * 当传入的profile数组中的某一个匹配上，则返回true，否则返回false
	 */
	@Deprecated
	boolean acceptsProfiles(String... profiles);

	/**
	 * 同上一个方法，只是类型由字符串数组变更为了Profiles对象
	 */
	boolean acceptsProfiles(Profiles profiles);
}
```

## ConfigurableEnvironment

`ConfigurableEnvironment`在`Environment`和`ConfigurablePropertyResolver`的基础上，增加了对profile的修改操作。

`addActiveProfile()`和`setActiveProfiles()`的区别在于，`addActiveProfile()`是追加，`setActiveProfiles()`是替换。

```java
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {

	/**
	 * 设置激活的profile
	 */
	void setActiveProfiles(String... profiles);

	/**
	 * 添加激活的profile，和setActiveProfiles的区别在于，一个是追加，一个是替换。
	 */
	void addActiveProfile(String profile);

	/**
	 * 设置默认的profile
	 */
	void setDefaultProfiles(String... profiles);

	/**
	 *获取可变的属性源列表对象
	 */
	MutablePropertySources getPropertySources();

	/**
	 * 获取系统属性，即：System.getProperties()
	 */
	Map<String, Object> getSystemProperties();

	/**
	 * 获取环境属性，即：System.getEnv();
	 */
	Map<String, Object> getSystemEnvironment();

	/**
	 * 合并另外一个ConfigurableEnvironment到当前ConfigurableEnvironment
	 */
	void merge(ConfigurableEnvironment parent);
}
```

## AbstractEnvironment

`AbstractEnvironment`包含`Environment`几乎所有的实现。

```java
public abstract class AbstractEnvironment implements ConfigurableEnvironment {

	/**
	 * 一些常量，使用场景暂时未知。
	 * @see #suppressGetenvAccess()
	 */
	public static final String IGNORE_GETENV_PROPERTY_NAME = "spring.getenv.ignore";

	/**
	 * 常量，表示激活的profile列表
	 * @see ConfigurableEnvironment#setActiveProfiles
	 */
	public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";

	/**
	 * 常量，表示默认的profile列表
	 * @see ConfigurableEnvironment#setDefaultProfiles
	 */
	public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";

	/**
	 * 常量，“default”
	 */
	protected static final String RESERVED_DEFAULT_PROFILE_NAME = "default";

    // 日志对象
	protected final Log logger = LogFactory.getLog(getClass());

    // 激活的profile列表
	private final Set<String> activeProfiles = new LinkedHashSet<>();

    // 默认的profile列表
	private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

    // 可变的属性源列表
	private final MutablePropertySources propertySources = new MutablePropertySources();

    // 属性解析器，使用组合的方式来实现的
	private final ConfigurablePropertyResolver propertyResolver =
			new PropertySourcesPropertyResolver(this.propertySources);

	/**
	 * 一个简单的构造器
	 * @see #customizePropertySources(MutablePropertySources)
	 */
	public AbstractEnvironment() {
		customizePropertySources(this.propertySources);
	}

	/**
	 * 自定义属性源，这儿可通过`StandardEnvironment`的实现来看
	 */
	protected void customizePropertySources(MutablePropertySources propertySources) {
	}

	/**
	 * 获取默认的profile集合对象
	 */
	protected Set<String> getReservedDefaultProfiles() {
		return Collections.singleton(RESERVED_DEFAULT_PROFILE_NAME);
	}


	//---------------------------------------------------------------------
	// Implementation of ConfigurableEnvironment interface
	//---------------------------------------------------------------------

    /**
     * 先获取集合对象，再转换为数组
     */
	@Override
	public String[] getActiveProfiles() {
		return StringUtils.toStringArray(doGetActiveProfiles());
	}

	/**
	 * 获取激活的profile列表，如果未获取到，则说明未初始化加载。
	 * 初始化加载，主要从属性源列表对象中获取。
	 */
	protected Set<String> doGetActiveProfiles() {
		synchronized (this.activeProfiles) {
			if (this.activeProfiles.isEmpty()) {
				String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.activeProfiles;
		}
	}

    /**
     * 设置激活的profile列表，设置前需要前置校验
     * 1. profile不能为空
     * 2. 不能以叹号开头
     */
	@Override
	public void setActiveProfiles(String... profiles) {
		Assert.notNull(profiles, "Profile array must not be null");
		if (logger.isDebugEnabled()) {
			logger.debug("Activating profiles " + Arrays.asList(profiles));
		}
		synchronized (this.activeProfiles) {
			this.activeProfiles.clear();
			for (String profile : profiles) {
				validateProfile(profile);
				this.activeProfiles.add(profile);
			}
		}
	}

    /**
     * 添加激活的profile
     */
	@Override
	public void addActiveProfile(String profile) {
		if (logger.isDebugEnabled()) {
			logger.debug("Activating profile '" + profile + "'");
		}
		validateProfile(profile);
		doGetActiveProfiles();
		synchronized (this.activeProfiles) {
			this.activeProfiles.add(profile);
		}
	}

    /**
     * 获取默认的profile列表，和激活的profile操作类似
     */
	@Override
	public String[] getDefaultProfiles() {
		return StringUtils.toStringArray(doGetDefaultProfiles());
	}

	/**
	 * 类似doGetActiveProfiles
	 */
	protected Set<String> doGetDefaultProfiles() {
		synchronized (this.defaultProfiles) {
			if (this.defaultProfiles.equals(getReservedDefaultProfiles())) {
				String profiles = getProperty(DEFAULT_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					setDefaultProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.defaultProfiles;
		}
	}

	/**
	 * 类似setActiveProfiles
	 */
	@Override
	public void setDefaultProfiles(String... profiles) {
		Assert.notNull(profiles, "Profile array must not be null");
		synchronized (this.defaultProfiles) {
			this.defaultProfiles.clear();
			for (String profile : profiles) {
				validateProfile(profile);
				this.defaultProfiles.add(profile);
			}
		}
	}

    /**
     * 判断profile是否匹配，只要有一个匹配即返回true。
     * 特别注意：叹号表示“非”
     */
	@Override
	@Deprecated
	public boolean acceptsProfiles(String... profiles) {
		Assert.notEmpty(profiles, "Must specify at least one profile");
		for (String profile : profiles) {
			if (StringUtils.hasLength(profile) && profile.charAt(0) == '!') {
				if (!isProfileActive(profile.substring(1))) {
					return true;
				}
			}
			else if (isProfileActive(profile)) {
				return true;
			}
		}
		return false;
	}

    /**
     * 使用Profiles类的函数式匹配方式替换for遍历的方式
     */
	@Override
	public boolean acceptsProfiles(Profiles profiles) {
		Assert.notNull(profiles, "Profiles must not be null");
		return profiles.matches(this::isProfileActive);
	}

	/**
	 * 当前profile是否在激活的profile列表中
	 */
	protected boolean isProfileActive(String profile) {
		validateProfile(profile);
		Set<String> currentActiveProfiles = doGetActiveProfiles();
		return (currentActiveProfiles.contains(profile) ||
				(currentActiveProfiles.isEmpty() && doGetDefaultProfiles().contains(profile)));
	}

	/**
	 * 校验profile，包括空和叹号
	 */
	protected void validateProfile(String profile) {
		if (!StringUtils.hasText(profile)) {
			throw new IllegalArgumentException("Invalid profile [" + profile + "]: must contain text");
		}
		if (profile.charAt(0) == '!') {
			throw new IllegalArgumentException("Invalid profile [" + profile + "]: must not begin with ! operator");
		}
	}

    /**
     * 获取可变的属性源列表对象
     */
	@Override
	public MutablePropertySources getPropertySources() {
		return this.propertySources;
	}

    /**
     * 获取系统属性，优先从System.getProperties()获取；
     * 获取不到的话，再通过懒加载的方式，从System.getProperty(attributeName)获取。
     */
	@Override
	@SuppressWarnings({"rawtypes", "unchecked"})
	public Map<String, Object> getSystemProperties() {
		try {
			return (Map) System.getProperties();
		}
		catch (AccessControlException ex) {
			return (Map) new ReadOnlySystemAttributesMap() {
				@Override
				@Nullable
				protected String getSystemAttribute(String attributeName) {
					try {
						return System.getProperty(attributeName);
					}
					catch (AccessControlException ex) {
						if (logger.isInfoEnabled()) {
							logger.info("Caught AccessControlException when accessing system property '" +
									attributeName + "'; its value will be returned [null]. Reason: " + ex.getMessage());
						}
						return null;
					}
				}
			};
		}
	}

    /**
     * 获取环境属性，优先从System.getenv()获取；
     * 获取不到的话，再通过懒加载的方式，从System.getenv(attributeName)获取。
     */
	@Override
	@SuppressWarnings({"rawtypes", "unchecked"})
	public Map<String, Object> getSystemEnvironment() {
		if (suppressGetenvAccess()) {
			return Collections.emptyMap();
		}
		try {
			return (Map) System.getenv();
		}
		catch (AccessControlException ex) {
			return (Map) new ReadOnlySystemAttributesMap() {
				@Override
				@Nullable
				protected String getSystemAttribute(String attributeName) {
					try {
						return System.getenv(attributeName);
					}
					catch (AccessControlException ex) {
						if (logger.isInfoEnabled()) {
							logger.info("Caught AccessControlException when accessing system environment variable '" +
									attributeName + "'; its value will be returned [null]. Reason: " + ex.getMessage());
						}
						return null;
					}
				}
			};
		}
	}

	/**
	 * 通过spring.properties，获取参数
	 */
	protected boolean suppressGetenvAccess() {
		return SpringProperties.getFlag(IGNORE_GETENV_PROPERTY_NAME);
	}

    /**
     * 合并操作。主要合并以下属性：
     * 1. propertySources
     * 2. activeProfiles
     * 3. defaultProfiles
     */
	@Override
	public void merge(ConfigurableEnvironment parent) {
		for (PropertySource<?> ps : parent.getPropertySources()) {
			if (!this.propertySources.contains(ps.getName())) {
				this.propertySources.addLast(ps);
			}
		}
		String[] parentActiveProfiles = parent.getActiveProfiles();
		if (!ObjectUtils.isEmpty(parentActiveProfiles)) {
			synchronized (this.activeProfiles) {
				Collections.addAll(this.activeProfiles, parentActiveProfiles);
			}
		}
		String[] parentDefaultProfiles = parent.getDefaultProfiles();
		if (!ObjectUtils.isEmpty(parentDefaultProfiles)) {
			synchronized (this.defaultProfiles) {
				this.defaultProfiles.remove(RESERVED_DEFAULT_PROFILE_NAME);
				Collections.addAll(this.defaultProfiles, parentDefaultProfiles);
			}
		}
	}


	//---------------------------------------------------------------------
	// Implementation of ConfigurablePropertyResolver interface
	//---------------------------------------------------------------------

	@Override
	public ConfigurableConversionService getConversionService() {
		return this.propertyResolver.getConversionService();
	}

	@Override
	public void setConversionService(ConfigurableConversionService conversionService) {
		this.propertyResolver.setConversionService(conversionService);
	}

	@Override
	public void setPlaceholderPrefix(String placeholderPrefix) {
		this.propertyResolver.setPlaceholderPrefix(placeholderPrefix);
	}

	@Override
	public void setPlaceholderSuffix(String placeholderSuffix) {
		this.propertyResolver.setPlaceholderSuffix(placeholderSuffix);
	}

	@Override
	public void setValueSeparator(@Nullable String valueSeparator) {
		this.propertyResolver.setValueSeparator(valueSeparator);
	}

	@Override
	public void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders) {
		this.propertyResolver.setIgnoreUnresolvableNestedPlaceholders(ignoreUnresolvableNestedPlaceholders);
	}

	@Override
	public void setRequiredProperties(String... requiredProperties) {
		this.propertyResolver.setRequiredProperties(requiredProperties);
	}

	@Override
	public void validateRequiredProperties() throws MissingRequiredPropertiesException {
		this.propertyResolver.validateRequiredProperties();
	}


	//---------------------------------------------------------------------
	// Implementation of PropertyResolver interface
	//---------------------------------------------------------------------

	@Override
	public boolean containsProperty(String key) {
		return this.propertyResolver.containsProperty(key);
	}

	@Override
	@Nullable
	public String getProperty(String key) {
		return this.propertyResolver.getProperty(key);
	}

	@Override
	public String getProperty(String key, String defaultValue) {
		return this.propertyResolver.getProperty(key, defaultValue);
	}

	@Override
	@Nullable
	public <T> T getProperty(String key, Class<T> targetType) {
		return this.propertyResolver.getProperty(key, targetType);
	}

	@Override
	public <T> T getProperty(String key, Class<T> targetType, T defaultValue) {
		return this.propertyResolver.getProperty(key, targetType, defaultValue);
	}

	@Override
	public String getRequiredProperty(String key) throws IllegalStateException {
		return this.propertyResolver.getRequiredProperty(key);
	}

	@Override
	public <T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException {
		return this.propertyResolver.getRequiredProperty(key, targetType);
	}

	@Override
	public String resolvePlaceholders(String text) {
		return this.propertyResolver.resolvePlaceholders(text);
	}

	@Override
	public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
		return this.propertyResolver.resolveRequiredPlaceholders(text);
	}


	@Override
	public String toString() {
		return getClass().getSimpleName() + " {activeProfiles=" + this.activeProfiles +
				", defaultProfiles=" + this.defaultProfiles + ", propertySources=" + this.propertySources + "}";
	}
}
```

## StandardEnvironment

`StandardEnvironment`主要扩展了系统属性和环境属性。

```java
public class StandardEnvironment extends AbstractEnvironment {

	/** System environment property source name: {@value}. */
	public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

	/** JVM system properties property source name: {@value}. */
	public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";


	/**
	 * 优先从System.getProperties()获取。
	 * 获取不到，再从System.getEnv()获取。
	 */
	@Override
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(
				new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
		propertySources.addLast(
				new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
	}
}
```

## PropertyPlaceholderHelper

`PropertyPlaceholderHelper`，属性占位符辅助类，是最终完成变量替换的地方。

真正替换占位符变量的方法，包含以下3个重要特性：

1. 通过递归+遍历的方式，解决变量循环引用的问题，如：${${var}Name}
2. 通过Set集合的方式，避免变量循环引用的问题
3. 通过值分隔符，在变量替换失败的情况下，返回默认值

同时，我们引申出了函数接口的两个好处：

1. 提高方法复用程度
2. 简化编码难度

```java
public class PropertyPlaceholderHelper {

    // 日志对象
	private static final Log logger = LogFactory.getLog(PropertyPlaceholderHelper.class);

	private static final Map<String, String> wellKnownSimplePrefixes = new HashMap<>(4);

	static {
		wellKnownSimplePrefixes.put("}", "{");
		wellKnownSimplePrefixes.put("]", "[");
		wellKnownSimplePrefixes.put(")", "(");
	}

    // 占位符前缀
	private final String placeholderPrefix;

    // 占位符后缀
	private final String placeholderSuffix;

	private final String simplePrefix;

    // 值分隔符，分隔符前为变量名，分隔符后为默认值
	@Nullable
	private final String valueSeparator;

    // 是否忽略未解决的占位符变量
	private final boolean ignoreUnresolvablePlaceholders;


	/**
	 * 通过指定的占位符前缀和后缀创建辅助类
	 */
	public PropertyPlaceholderHelper(String placeholderPrefix, String placeholderSuffix) {
		this(placeholderPrefix, placeholderSuffix, null, true);
	}

	/**
	 * 全量参数创建辅助类
	 */
	public PropertyPlaceholderHelper(String placeholderPrefix, String placeholderSuffix,
			@Nullable String valueSeparator, boolean ignoreUnresolvablePlaceholders) {

		Assert.notNull(placeholderPrefix, "'placeholderPrefix' must not be null");
		Assert.notNull(placeholderSuffix, "'placeholderSuffix' must not be null");
		this.placeholderPrefix = placeholderPrefix;
		this.placeholderSuffix = placeholderSuffix;
		String simplePrefixForSuffix = wellKnownSimplePrefixes.get(this.placeholderSuffix);
		if (simplePrefixForSuffix != null && this.placeholderPrefix.endsWith(simplePrefixForSuffix)) {
			this.simplePrefix = simplePrefixForSuffix;
		}
		else {
			this.simplePrefix = this.placeholderPrefix;
		}
		this.valueSeparator = valueSeparator;
		this.ignoreUnresolvablePlaceholders = ignoreUnresolvablePlaceholders;
	}


	/**
	 * 通过变量列表，替换文本的占位符变量
	 */
	public String replacePlaceholders(String value, final Properties properties) {
		Assert.notNull(properties, "'properties' must not be null");
		return replacePlaceholders(value, properties::getProperty);
	}

	/**
	 * 替换占位符变量
	 */
	public String replacePlaceholders(String value, PlaceholderResolver placeholderResolver) {
		Assert.notNull(value, "'value' must not be null");
		return parseStringValue(value, placeholderResolver, new HashSet<>());
	}

    /**
     * 真正替换占位符变量的方法
     * 1. 通过递归+遍历的方式，解决变量循环引用的问题，如：${${var}Name}
     * 2. 通过Set集合的方式，避免变量循环引用的问题
     * 3. 通过值分隔符，在变量替换失败的情况下，返回默认值
     */
	protected String parseStringValue(
			String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

		StringBuilder result = new StringBuilder(value);

		int startIndex = value.indexOf(this.placeholderPrefix);
		while (startIndex != -1) {
			int endIndex = findPlaceholderEndIndex(result, startIndex);
			if (endIndex != -1) {
				String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
				String originalPlaceholder = placeholder;
				if (!visitedPlaceholders.add(originalPlaceholder)) {
					throw new IllegalArgumentException(
							"Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
				}
				// Recursive invocation, parsing placeholders contained in the placeholder key.
				placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
				// Now obtain the value for the fully resolved key...
				String propVal = placeholderResolver.resolvePlaceholder(placeholder);
				if (propVal == null && this.valueSeparator != null) {
					int separatorIndex = placeholder.indexOf(this.valueSeparator);
					if (separatorIndex != -1) {
						String actualPlaceholder = placeholder.substring(0, separatorIndex);
						String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
						propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
						if (propVal == null) {
							propVal = defaultValue;
						}
					}
				}
				if (propVal != null) {
					// Recursive invocation, parsing placeholders contained in the
					// previously resolved placeholder value.
					propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
					result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
					if (logger.isTraceEnabled()) {
						logger.trace("Resolved placeholder '" + placeholder + "'");
					}
					startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
				}
				else if (this.ignoreUnresolvablePlaceholders) {
					// Proceed with unprocessed value.
					startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
				}
				else {
					throw new IllegalArgumentException("Could not resolve placeholder '" +
							placeholder + "'" + " in value \"" + value + "\"");
				}
				visitedPlaceholders.remove(originalPlaceholder);
			}
			else {
				startIndex = -1;
			}
		}

		return result.toString();
	}

	private int findPlaceholderEndIndex(CharSequence buf, int startIndex) {
		int index = startIndex + this.placeholderPrefix.length();
		int withinNestedPlaceholder = 0;
		while (index < buf.length()) {
			if (StringUtils.substringMatch(buf, index, this.placeholderSuffix)) {
				if (withinNestedPlaceholder > 0) {
					withinNestedPlaceholder--;
					index = index + this.placeholderSuffix.length();
				}
				else {
					return index;
				}
			}
			else if (StringUtils.substringMatch(buf, index, this.simplePrefix)) {
				withinNestedPlaceholder++;
				index = index + this.simplePrefix.length();
			}
			else {
				index++;
			}
		}
		return -1;
	}


	/**
	 * 函数接口，有两个好处：
	 * 1. 提高方法复用程度
	 * 2. 简化编码难度
	 * Strategy interface used to resolve replacement values for placeholders contained in Strings.
	 */
	@FunctionalInterface
	public interface PlaceholderResolver {

		/**
		 * Resolve the supplied placeholder name to the replacement value.
		 * @param placeholderName the name of the placeholder to resolve
		 * @return the replacement value, or {@code null} if no replacement is to be made
		 */
		@Nullable
		String resolvePlaceholder(String placeholderName);
	}
}
```

## 总结

属性解析器，主要功能是通过属性源列表对象，读取或获取属性。同时可根据这些属性，对文本中的变量进行替换操作。

属性解析器中最至关重要的是`Environment`，其包含了对profile的操作。例如：

1. 设置profile
2. 获取profile
3. profile是否满足

profile，就是环境的代名词。在互联网企业中，环境是很重要的，大体上，我们可以将环境拆分为开发环境、测试环境、灰度环境和线上环境。

profile又分为激活的profile和默认的profile。为啥会有这两个呢？主要是考虑到在未获取到激活的profile时，我们还可以通过默认的profile来操作。这种兼容性在mock环境和单元测试环境下特别有用。