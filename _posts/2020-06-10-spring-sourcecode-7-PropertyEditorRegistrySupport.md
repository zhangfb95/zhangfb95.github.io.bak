---
layout: post
title:  Spring IOC源码解析（07）PropertyEditorRegistrySupport
date:   2020-06-10 22:37:47 +0800
categories: Spring
tags: Spring
---

* content
{:toc}

## 前言

在讲解`PropertyEditorRegistrySupport`之前，我们有比较对jdk内部自带的`PropertyEditor`进行说明。

`PropertyEditor`，意为属性编辑器，是JavaBean规范中定义的接口，最初的目的是为IDE的设计开发做准备的，它可以很方便地让IDE以可视化的方式设置JavaBean属性。但是在Spring中，尤其是IOC的xml配置，我们往往需要将一个字符串灵活地转换为各种其他类型。因此，Spring借鉴了`PropertyEditor`接口，在其基础上做了增强和扩展，同时自带了一些标准的实现，让我们可以比较快速而灵活地进行使用。

## PropertyEditor

`PropertyEditor`是属性编辑器的接口，其主要方法如下：

```java
// 设置属性值
void setValue(Object value);

// 获取属性值
Object getValue();

// 获取属性的初始值字符串
String getJavaInitializationString();

// 将数值已一个字符串表示
String getAsText();

// 用一个字符串去更新属性值
void setAsText(String text) throws java.lang.IllegalArgumentException;

// 返回有效属性值的可罗列值，例如boolean的有效值为true和false。缺省为null，即：不可罗列的。
String[] getTags();

// 其他...
```

## PropertyEditorSupport

`PropertyEditorSupport`是`PropertyEditor`的标准实现，大部分逻辑都在此类里面。

```java
public class PropertyEditorSupport implements PropertyEditor {

    /**
     * 构造一个对象，源为自己
     * Constructs a <code>PropertyEditorSupport</code> object.
     *
     * @since 1.5
     */
    public PropertyEditorSupport() {
        setSource(this);
    }

    /**
     * 构造一个对象，源不为自己，而是外部传入。<br/>
     * 所谓源就是在事件触发的时候可追溯的一个对象。
     * Constructs a <code>PropertyEditorSupport</code> object.
     *
     * @param source the source used for event firing
     * @since 1.5
     */
    public PropertyEditorSupport(Object source) {
        if (source == null) {
           throw new NullPointerException();
        }
        setSource(source);
    }

    /**
     * 获取触发事件对应的源
     * Returns the bean that is used as the
     * source of events. If the source has not
     * been explicitly set then this instance of
     * <code>PropertyEditorSupport</code> is returned.
     *
     * @return the source object or this instance
     * @since 1.5
     */
    public Object getSource() {
        return source;
    }

    /**
     * 设置源
     * Sets the source bean.
     * <p>
     * The source bean is used as the source of events
     * for the property changes. This source should be used for information
     * purposes only and should not be modified by the PropertyEditor.
     *
     * @param source source object to be used for events
     * @since 1.5
     */
    public void setSource(Object source) {
        this.source = source;
    }

    /**
     * 设置属性值，同时触发属性变更的事件
     * Set (or change) the object that is to be edited.
     *
     * @param value The new target object to be edited.  Note that this
     *     object should not be modified by the PropertyEditor, rather
     *     the PropertyEditor should create a new object to hold any
     *     modified value.
     */
    public void setValue(Object value) {
        this.value = value;
        firePropertyChange();
    }

    /**
     * 获取属性值
     * Gets the value of the property.
     *
     * @return The value of the property.
     */
    public Object getValue() {
        return value;
    }

    //----------------------------------------------------------------------

    /**
     * 是否可绘制的，默认为false
     * Determines whether the class will honor the paintValue method.
     *
     * @return  True if the class will honor the paintValue method.
     */

    public boolean isPaintable() {
        return false;
    }

    /**
     * 绘制属性，默认为空实现
     * Paint a representation of the value into a given area of screen
     * real estate.  Note that the propertyEditor is responsible for doing
     * its own clipping so that it fits into the given rectangle.
     * <p>
     * If the PropertyEditor doesn't honor paint requests (see isPaintable)
     * this method should be a silent noop.
     *
     * @param gfx  Graphics object to paint into.
     * @param box  Rectangle within graphics object into which we should paint.
     */
    public void paintValue(java.awt.Graphics gfx, java.awt.Rectangle box) {
    }

    //----------------------------------------------------------------------

    /**
     * 获取属性初始的字符串，默认为三个英文问号
     * This method is intended for use when generating Java code to set
     * the value of the property.  It should return a fragment of Java code
     * that can be used to initialize a variable with the current property
     * value.
     * <p>
     * Example results are "2", "new Color(127,127,34)", "Color.orange", etc.
     *
     * @return A fragment of Java code representing an initializer for the
     *          current value.
     */
    public String getJavaInitializationString() {
        return "???";
    }

    //----------------------------------------------------------------------

    /**
     * 获取属性对应的字符串，默认调用属性对象的toString方法，通常情况下需要子类自行重写。
     * Gets the property value as a string suitable for presentation
     * to a human to edit.
     *
     * @return The property value as a string suitable for presentation
     *       to a human to edit.
     * <p>   Returns null if the value can't be expressed as a string.
     * <p>   If a non-null value is returned, then the PropertyEditor should
     *       be prepared to parse that string back in setAsText().
     */
    public String getAsText() {
        return (this.value != null)
                ? this.value.toString()
                : null;
    }

    /**
     * 用字符串去更新属性，如果属性本身就是字符串，直接设置即可，否则抛出异常。
     * 子类可重写此方法。
     * Sets the property value by parsing a given String.  May raise
     * java.lang.IllegalArgumentException if either the String is
     * badly formatted or if this kind of property can't be expressed
     * as text.
     *
     * @param text  The string to be parsed.
     */
    public void setAsText(String text) throws java.lang.IllegalArgumentException {
        if (value instanceof String) {
            setValue(text);
            return;
        }
        throw new java.lang.IllegalArgumentException(text);
    }

    //----------------------------------------------------------------------

    /**
     * 获取标签，即：可罗列的字符串值。
     * If the property value must be one of a set of known tagged values,
     * then this method should return an array of the tag values.  This can
     * be used to represent (for example) enum values.  If a PropertyEditor
     * supports tags, then it should support the use of setAsText with
     * a tag value as a way of setting the value.
     *
     * @return The tag values for this property.  May be null if this
     *   property cannot be represented as a tagged value.
     *
     */
    public String[] getTags() {
        return null;
    }

    //----------------------------------------------------------------------

    /**
     * A PropertyEditor may chose to make available a full custom Component
     * that edits its property value.  It is the responsibility of the
     * PropertyEditor to hook itself up to its editor Component itself and
     * to report property value changes by firing a PropertyChange event.
     * <P>
     * The higher-level code that calls getCustomEditor may either embed
     * the Component in some larger property sheet, or it may put it in
     * its own individual dialog, or ...
     *
     * @return A java.awt.Component that will allow a human to directly
     *      edit the current property value.  May be null if this is
     *      not supported.
     */

    public java.awt.Component getCustomEditor() {
        return null;
    }

    /**
     * Determines whether the propertyEditor can provide a custom editor.
     *
     * @return  True if the propertyEditor can provide a custom editor.
     */
    public boolean supportsCustomEditor() {
        return false;
    }

    //----------------------------------------------------------------------

    /**
     * 设置监听事件
     * Adds a listener for the value change.
     * When the property editor changes its value
     * it should fire a {@link PropertyChangeEvent}
     * on all registered {@link PropertyChangeListener}s,
     * specifying the {@code null} value for the property name.
     * If the source property is set,
     * it should be used as the source of the event.
     * <p>
     * The same listener object may be added more than once,
     * and will be called as many times as it is added.
     * If {@code listener} is {@code null},
     * no exception is thrown and no action is taken.
     *
     * @param listener  the {@link PropertyChangeListener} to add
     */
    public synchronized void addPropertyChangeListener(
                                PropertyChangeListener listener) {
        if (listeners == null) {
            listeners = new java.util.Vector<>();
        }
        listeners.addElement(listener);
    }

    /**
     * 移除监听事件
     * Removes a listener for the value change.
     * <p>
     * If the same listener was added more than once,
     * it will be notified one less time after being removed.
     * If {@code listener} is {@code null}, or was never added,
     * no exception is thrown and no action is taken.
     *
     * @param listener  the {@link PropertyChangeListener} to remove
     */
    public synchronized void removePropertyChangeListener(
                                PropertyChangeListener listener) {
        if (listeners == null) {
            return;
        }
        listeners.removeElement(listener);
    }

    /**
     * 触发监听事件
     * Report that we have been modified to any interested listeners.
     */
    public void firePropertyChange() {
        java.util.Vector<PropertyChangeListener> targets;
        synchronized (this) {
            if (listeners == null) {
                return;
            }
            targets = unsafeClone(listeners);
        }
        // Tell our listeners that "everything" has changed.
        PropertyChangeEvent evt = new PropertyChangeEvent(source, null, null, null);

        for (int i = 0; i < targets.size(); i++) {
            PropertyChangeListener target = targets.elementAt(i);
            target.propertyChange(evt);
        }
    }

    @SuppressWarnings("unchecked")
    private <T> java.util.Vector<T> unsafeClone(java.util.Vector<T> v) {
        return (java.util.Vector<T>)v.clone();
    }

    //----------------------------------------------------------------------

    private Object value;
    private Object source;
    private java.util.Vector<PropertyChangeListener> listeners;
}
```

## PropertyEditorRegistry

`PropertyEditorRegistry`是一个`PropertyEditor`注册器的接口，高度抽象了`PropertyEditor`的注册流程。

```java
public interface PropertyEditorRegistry {

	/**
	 * 注册自定义编辑器
	 * Register the given custom property editor for all properties of the given type.
	 * @param requiredType the type of the property
	 * @param propertyEditor the editor to register
	 */
	void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor);

	/**
	 * 注册自定义编辑器，同时指定属性路径
	 */
	void registerCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath, PropertyEditor propertyEditor);

	/**
	 * 根据类型和属性路径，查找属性编辑器
	 */
	@Nullable
	PropertyEditor findCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath);
}
```

## PropertyEditorRegistrySupport

```java
public class PropertyEditorRegistrySupport implements PropertyEditorRegistry {

    // 转换服务
	@Nullable
	private ConversionService conversionService;

    // 默认编辑器是否启用
	private boolean defaultEditorsActive = false;
    
    // 配置属性编辑器是否启用
	private boolean configValueEditorsActive = false;

    // 默认编辑器
	@Nullable
	private Map<Class<?>, PropertyEditor> defaultEditors;

    // 重写的默认编辑器
	@Nullable
	private Map<Class<?>, PropertyEditor> overriddenDefaultEditors;

    // 自定义属性编辑器
	@Nullable
	private Map<Class<?>, PropertyEditor> customEditors;

    // 自定义路径属性编辑器
	@Nullable
	private Map<String, CustomEditorHolder> customEditorsForPath;

    // 自定义属性编辑器缓存，仅仅当缓存过之后，才会存在这里面
	@Nullable
	private Map<Class<?>, PropertyEditor> customEditorCache;


	/**
	 * 设置转换服务
	 * Specify a Spring 3.0 ConversionService to use for converting
	 * property values, as an alternative to JavaBeans PropertyEditors.
	 */
	public void setConversionService(@Nullable ConversionService conversionService) {
		this.conversionService = conversionService;
	}

	/**
	 * 返回转换服务
	 * Return the associated ConversionService, if any.
	 */
	@Nullable
	public ConversionService getConversionService() {
		return this.conversionService;
	}


	//---------------------------------------------------------------------
	// Management of default editors
	//---------------------------------------------------------------------

	/**
	 * Activate the default editors for this registry instance,
	 * allowing for lazily registering default editors when needed.
	 */
	protected void registerDefaultEditors() {
		this.defaultEditorsActive = true;
	}

	/**
	 * Activate config value editors which are only intended for configuration purposes,
	 * such as {@link org.springframework.beans.propertyeditors.StringArrayPropertyEditor}.
	 * <p>Those editors are not registered by default simply because they are in
	 * general inappropriate for data binding purposes. Of course, you may register
	 * them individually in any case, through {@link #registerCustomEditor}.
	 */
	public void useConfigValueEditors() {
		this.configValueEditorsActive = true;
	}

	/**
	 * 重写的默认编辑器
	 * Override the default editor for the specified type with the given property editor.
	 * <p>Note that this is different from registering a custom editor in that the editor
	 * semantically still is a default editor. A ConversionService will override such a
	 * default editor, whereas custom editors usually override the ConversionService.
	 * @param requiredType the type of the property
	 * @param propertyEditor the editor to register
	 * @see #registerCustomEditor(Class, PropertyEditor)
	 */
	public void overrideDefaultEditor(Class<?> requiredType, PropertyEditor propertyEditor) {
		if (this.overriddenDefaultEditors == null) {
			this.overriddenDefaultEditors = new HashMap<>();
		}
		this.overriddenDefaultEditors.put(requiredType, propertyEditor);
	}

	/**
	 * 获取特定类型的默认编辑器
	 * 1. 当默认编辑器不启用时，返回null
	 * 2. 首先从重写的默认编辑器获取
	 * 3. 其次从默认编辑器获取
	 * Retrieve the default editor for the given property type, if any.
	 * <p>Lazily registers the default editors, if they are active.
	 * @param requiredType type of the property
	 * @return the default editor, or {@code null} if none found
	 * @see #registerDefaultEditors
	 */
	@Nullable
	public PropertyEditor getDefaultEditor(Class<?> requiredType) {
		if (!this.defaultEditorsActive) {
			return null;
		}
		if (this.overriddenDefaultEditors != null) {
			PropertyEditor editor = this.overriddenDefaultEditors.get(requiredType);
			if (editor != null) {
				return editor;
			}
		}
		if (this.defaultEditors == null) {
			createDefaultEditors();
		}
		return this.defaultEditors.get(requiredType);
	}

	/**
	 * 创建一系列的默认编辑器
	 * Actually register the default editors for this registry instance.
	 */
	private void createDefaultEditors() {
		this.defaultEditors = new HashMap<>(64);

		// Simple editors, without parameterization capabilities.
		// The JDK does not contain a default editor for any of these target types.
		this.defaultEditors.put(Charset.class, new CharsetEditor());
		this.defaultEditors.put(Class.class, new ClassEditor());
		this.defaultEditors.put(Class[].class, new ClassArrayEditor());
		this.defaultEditors.put(Currency.class, new CurrencyEditor());
		this.defaultEditors.put(File.class, new FileEditor());
		this.defaultEditors.put(InputStream.class, new InputStreamEditor());
		this.defaultEditors.put(InputSource.class, new InputSourceEditor());
		this.defaultEditors.put(Locale.class, new LocaleEditor());
		this.defaultEditors.put(Path.class, new PathEditor());
		this.defaultEditors.put(Pattern.class, new PatternEditor());
		this.defaultEditors.put(Properties.class, new PropertiesEditor());
		this.defaultEditors.put(Reader.class, new ReaderEditor());
		this.defaultEditors.put(Resource[].class, new ResourceArrayPropertyEditor());
		this.defaultEditors.put(TimeZone.class, new TimeZoneEditor());
		this.defaultEditors.put(URI.class, new URIEditor());
		this.defaultEditors.put(URL.class, new URLEditor());
		this.defaultEditors.put(UUID.class, new UUIDEditor());
		this.defaultEditors.put(ZoneId.class, new ZoneIdEditor());

		// Default instances of collection editors.
		// Can be overridden by registering custom instances of those as custom editors.
		this.defaultEditors.put(Collection.class, new CustomCollectionEditor(Collection.class));
		this.defaultEditors.put(Set.class, new CustomCollectionEditor(Set.class));
		this.defaultEditors.put(SortedSet.class, new CustomCollectionEditor(SortedSet.class));
		this.defaultEditors.put(List.class, new CustomCollectionEditor(List.class));
		this.defaultEditors.put(SortedMap.class, new CustomMapEditor(SortedMap.class));

		// Default editors for primitive arrays.
		this.defaultEditors.put(byte[].class, new ByteArrayPropertyEditor());
		this.defaultEditors.put(char[].class, new CharArrayPropertyEditor());

		// The JDK does not contain a default editor for char!
		this.defaultEditors.put(char.class, new CharacterEditor(false));
		this.defaultEditors.put(Character.class, new CharacterEditor(true));

		// Spring's CustomBooleanEditor accepts more flag values than the JDK's default editor.
		this.defaultEditors.put(boolean.class, new CustomBooleanEditor(false));
		this.defaultEditors.put(Boolean.class, new CustomBooleanEditor(true));

		// The JDK does not contain default editors for number wrapper types!
		// Override JDK primitive number editors with our own CustomNumberEditor.
		this.defaultEditors.put(byte.class, new CustomNumberEditor(Byte.class, false));
		this.defaultEditors.put(Byte.class, new CustomNumberEditor(Byte.class, true));
		this.defaultEditors.put(short.class, new CustomNumberEditor(Short.class, false));
		this.defaultEditors.put(Short.class, new CustomNumberEditor(Short.class, true));
		this.defaultEditors.put(int.class, new CustomNumberEditor(Integer.class, false));
		this.defaultEditors.put(Integer.class, new CustomNumberEditor(Integer.class, true));
		this.defaultEditors.put(long.class, new CustomNumberEditor(Long.class, false));
		this.defaultEditors.put(Long.class, new CustomNumberEditor(Long.class, true));
		this.defaultEditors.put(float.class, new CustomNumberEditor(Float.class, false));
		this.defaultEditors.put(Float.class, new CustomNumberEditor(Float.class, true));
		this.defaultEditors.put(double.class, new CustomNumberEditor(Double.class, false));
		this.defaultEditors.put(Double.class, new CustomNumberEditor(Double.class, true));
		this.defaultEditors.put(BigDecimal.class, new CustomNumberEditor(BigDecimal.class, true));
		this.defaultEditors.put(BigInteger.class, new CustomNumberEditor(BigInteger.class, true));

		// Only register config value editors if explicitly requested.
		if (this.configValueEditorsActive) {
			StringArrayPropertyEditor sae = new StringArrayPropertyEditor();
			this.defaultEditors.put(String[].class, sae);
			this.defaultEditors.put(short[].class, sae);
			this.defaultEditors.put(int[].class, sae);
			this.defaultEditors.put(long[].class, sae);
		}
	}

	/**
	 * 拷贝默认编辑器，包括参数等
	 * Copy the default editors registered in this instance to the given target registry.
	 * @param target the target registry to copy to
	 */
	protected void copyDefaultEditorsTo(PropertyEditorRegistrySupport target) {
		target.defaultEditorsActive = this.defaultEditorsActive;
		target.configValueEditorsActive = this.configValueEditorsActive;
		target.defaultEditors = this.defaultEditors;
		target.overriddenDefaultEditors = this.overriddenDefaultEditors;
	}


	//---------------------------------------------------------------------
	// Management of custom editors
	//---------------------------------------------------------------------

	@Override
	public void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor) {
		registerCustomEditor(requiredType, null, propertyEditor);
	}

    // 注册自定义编辑器
    // 1. 类型和属性路径不能全为null
    // 2. 属性路径不为null时，默认以属性路径的方式注册
    // 3. 当属性路径为null时，以类型的方式注册，并清空自定义缓存编辑器
	@Override
	public void registerCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath, PropertyEditor propertyEditor) {
		if (requiredType == null && propertyPath == null) {
			throw new IllegalArgumentException("Either requiredType or propertyPath is required");
		}
		if (propertyPath != null) {
			if (this.customEditorsForPath == null) {
				this.customEditorsForPath = new LinkedHashMap<>(16);
			}
			this.customEditorsForPath.put(propertyPath, new CustomEditorHolder(propertyEditor, requiredType));
		}
		else {
			if (this.customEditors == null) {
				this.customEditors = new LinkedHashMap<>(16);
			}
			this.customEditors.put(requiredType, propertyEditor);
			this.customEditorCache = null;
		}
	}

    /**
     * 查找自定义编辑器，同样的。
     * 1. 首先，以属性路径的方式进行查找
     * 2. 其次，其类型的方式进行查找
     * 3. 特别需要注意的是`addStrippedPropertyPaths`方法，其目的是剥离掉其内层的属性。
     *    比如：当nestedPath为`a[b[c[d]]]]`时，我们调用此方法，返回了`a]]]`
     */
	@Override
	@Nullable
	public PropertyEditor findCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath) {
		Class<?> requiredTypeToUse = requiredType;
		if (propertyPath != null) {
			if (this.customEditorsForPath != null) {
				// Check property-specific editor first.
				PropertyEditor editor = getCustomEditor(propertyPath, requiredType);
				if (editor == null) {
					List<String> strippedPaths = new ArrayList<>();
					addStrippedPropertyPaths(strippedPaths, "", propertyPath);
					for (Iterator<String> it = strippedPaths.iterator(); it.hasNext() && editor == null;) {
						String strippedPath = it.next();
						editor = getCustomEditor(strippedPath, requiredType);
					}
				}
				if (editor != null) {
					return editor;
				}
			}
			if (requiredType == null) {
				requiredTypeToUse = getPropertyType(propertyPath);
			}
		}
		// No property-specific editor -> check type-specific editor.
		return getCustomEditor(requiredTypeToUse);
	}

	/**
	 * 确定是否存在特定类型或属性路径的自定义编辑器
	 * Determine whether this registry contains a custom editor
	 * for the specified array/collection element.
	 * @param elementType the target type of the element
	 * (can be {@code null} if not known)
	 * @param propertyPath the property path (typically of the array/collection;
	 * can be {@code null} if not known)
	 * @return whether a matching custom editor has been found
	 */
	public boolean hasCustomEditorForElement(@Nullable Class<?> elementType, @Nullable String propertyPath) {
		if (propertyPath != null && this.customEditorsForPath != null) {
			for (Map.Entry<String, CustomEditorHolder> entry : this.customEditorsForPath.entrySet()) {
				if (PropertyAccessorUtils.matchesProperty(entry.getKey(), propertyPath) &&
						entry.getValue().getPropertyEditor(elementType) != null) {
					return true;
				}
			}
		}
		// No property-specific editor -> check type-specific editor.
		return (elementType != null && this.customEditors != null && this.customEditors.containsKey(elementType));
	}

	/**
	 * Determine the property type for the given property path.
	 * <p>Called by {@link #findCustomEditor} if no required type has been specified,
	 * to be able to find a type-specific editor even if just given a property path.
	 * <p>The default implementation always returns {@code null}.
	 * BeanWrapperImpl overrides this with the standard {@code getPropertyType}
	 * method as defined by the BeanWrapper interface.
	 * @param propertyPath the property path to determine the type for
	 * @return the type of the property, or {@code null} if not determinable
	 * @see BeanWrapper#getPropertyType(String)
	 */
	@Nullable
	protected Class<?> getPropertyType(String propertyPath) {
		return null;
	}

	/**
	 * 获取自定义属性编辑器
	 * Get custom editor that has been registered for the given property.
	 * @param propertyName the property path to look for
	 * @param requiredType the type to look for
	 * @return the custom editor, or {@code null} if none specific for this property
	 */
	@Nullable
	private PropertyEditor getCustomEditor(String propertyName, @Nullable Class<?> requiredType) {
		CustomEditorHolder holder =
				(this.customEditorsForPath != null ? this.customEditorsForPath.get(propertyName) : null);
		return (holder != null ? holder.getPropertyEditor(requiredType) : null);
	}

	/**
	 * 获取自定义属性编辑器，这儿为了加快检索速度，做了以下两件事情
	 * 1. 前置空校验
	 * 2. 后置缓存处理和查找
	 * Get custom editor for the given type. If no direct match found,
	 * try custom editor for superclass (which will in any case be able
	 * to render a value as String via {@code getAsText}).
	 * @param requiredType the type to look for
	 * @return the custom editor, or {@code null} if none found for this type
	 * @see java.beans.PropertyEditor#getAsText()
	 */
	@Nullable
	private PropertyEditor getCustomEditor(@Nullable Class<?> requiredType) {
		if (requiredType == null || this.customEditors == null) {
			return null;
		}
		// Check directly registered editor for type.
		PropertyEditor editor = this.customEditors.get(requiredType);
		if (editor == null) {
			// Check cached editor for type, registered for superclass or interface.
			if (this.customEditorCache != null) {
				editor = this.customEditorCache.get(requiredType);
			}
			if (editor == null) {
				// Find editor for superclass or interface.
				for (Iterator<Class<?>> it = this.customEditors.keySet().iterator(); it.hasNext() && editor == null;) {
					Class<?> key = it.next();
					if (key.isAssignableFrom(requiredType)) {
						editor = this.customEditors.get(key);
						// Cache editor for search type, to avoid the overhead
						// of repeated assignable-from checks.
						if (this.customEditorCache == null) {
							this.customEditorCache = new HashMap<>();
						}
						this.customEditorCache.put(requiredType, editor);
					}
				}
			}
		}
		return editor;
	}

	/**
	 * 推测属性对应的类型，这个方法会递归和遍历所有匹配的路径，效率相对较低
	 * Guess the property type of the specified property from the registered
	 * custom editors (provided that they were registered for a specific type).
	 * @param propertyName the name of the property
	 * @return the property type, or {@code null} if not determinable
	 */
	@Nullable
	protected Class<?> guessPropertyTypeFromEditors(String propertyName) {
		if (this.customEditorsForPath != null) {
			CustomEditorHolder editorHolder = this.customEditorsForPath.get(propertyName);
			if (editorHolder == null) {
				List<String> strippedPaths = new ArrayList<>();
				addStrippedPropertyPaths(strippedPaths, "", propertyName);
				for (Iterator<String> it = strippedPaths.iterator(); it.hasNext() && editorHolder == null;) {
					String strippedName = it.next();
					editorHolder = this.customEditorsForPath.get(strippedName);
				}
			}
			if (editorHolder != null) {
				return editorHolder.getRegisteredType();
			}
		}
		return null;
	}

	/**
	 * 拷贝当前对象，做的是浅拷贝的方式
	 * Copy the custom editors registered in this instance to the given target registry.
	 * @param target the target registry to copy to
	 * @param nestedProperty the nested property path of the target registry, if any.
	 * If this is non-null, only editors registered for a path below this nested property
	 * will be copied. If this is null, all editors will be copied.
	 */
	protected void copyCustomEditorsTo(PropertyEditorRegistry target, @Nullable String nestedProperty) {
		String actualPropertyName =
				(nestedProperty != null ? PropertyAccessorUtils.getPropertyName(nestedProperty) : null);
		if (this.customEditors != null) {
			this.customEditors.forEach(target::registerCustomEditor);
		}
		if (this.customEditorsForPath != null) {
			this.customEditorsForPath.forEach((editorPath, editorHolder) -> {
				if (nestedProperty != null) {
					int pos = PropertyAccessorUtils.getFirstNestedPropertySeparatorIndex(editorPath);
					if (pos != -1) {
						String editorNestedProperty = editorPath.substring(0, pos);
						String editorNestedPath = editorPath.substring(pos + 1);
						if (editorNestedProperty.equals(nestedProperty) || editorNestedProperty.equals(actualPropertyName)) {
							target.registerCustomEditor(
									editorHolder.getRegisteredType(), editorNestedPath, editorHolder.getPropertyEditor());
						}
					}
				}
				else {
					target.registerCustomEditor(
							editorHolder.getRegisteredType(), editorPath, editorHolder.getPropertyEditor());
				}
			});
		}
	}


	/**
	 * Add property paths with all variations of stripped keys and/or indexes.
	 * Invokes itself recursively with nested paths.
	 * @param strippedPaths the result list to add to
	 * @param nestedPath the current nested path
	 * @param propertyPath the property path to check for keys/indexes to strip
	 */
	private void addStrippedPropertyPaths(List<String> strippedPaths, String nestedPath, String propertyPath) {
		int startIndex = propertyPath.indexOf(PropertyAccessor.PROPERTY_KEY_PREFIX_CHAR);
		if (startIndex != -1) {
			int endIndex = propertyPath.indexOf(PropertyAccessor.PROPERTY_KEY_SUFFIX_CHAR);
			if (endIndex != -1) {
				String prefix = propertyPath.substring(0, startIndex);
				String key = propertyPath.substring(startIndex, endIndex + 1);
				String suffix = propertyPath.substring(endIndex + 1, propertyPath.length());
				// Strip the first key.
				strippedPaths.add(nestedPath + prefix + suffix);
				// Search for further keys to strip, with the first key stripped.
				addStrippedPropertyPaths(strippedPaths, nestedPath + prefix, suffix);
				// Search for further keys to strip, with the first key not stripped.
				addStrippedPropertyPaths(strippedPaths, nestedPath + prefix + key, suffix);
			}
		}
	}

	/**
	 * 自定义属性持有器，支持以类型继承的方式进行查找
	 * Holder for a registered custom editor with property name.
	 * Keeps the PropertyEditor itself plus the type it was registered for.
	 */
	private static final class CustomEditorHolder {

		private final PropertyEditor propertyEditor;

		@Nullable
		private final Class<?> registeredType;

		private CustomEditorHolder(PropertyEditor propertyEditor, @Nullable Class<?> registeredType) {
			this.propertyEditor = propertyEditor;
			this.registeredType = registeredType;
		}

		private PropertyEditor getPropertyEditor() {
			return this.propertyEditor;
		}

		@Nullable
		private Class<?> getRegisteredType() {
			return this.registeredType;
		}

		@Nullable
		private PropertyEditor getPropertyEditor(@Nullable Class<?> requiredType) {
			// Special case: If no required type specified, which usually only happens for
			// Collection elements, or required type is not assignable to registered type,
			// which usually only happens for generic properties of type Object -
			// then return PropertyEditor if not registered for Collection or array type.
			// (If not registered for Collection or array, it is assumed to be intended
			// for elements.)
			if (this.registeredType == null ||
					(requiredType != null &&
					(ClassUtils.isAssignable(this.registeredType, requiredType) ||
					ClassUtils.isAssignable(requiredType, this.registeredType))) ||
					(requiredType == null &&
					(!Collection.class.isAssignableFrom(this.registeredType) && !this.registeredType.isArray()))) {
				return this.propertyEditor;
			}
			else {
				return null;
			}
		}
	}

}
```

## PropertyAccessorUtils

其内部是一系列的工具方法，在配合bean属性解析时可考虑对比查看。
