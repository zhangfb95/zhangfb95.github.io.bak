---
layout: post
title:  Java反射（3）Modifiers
date:   2020-04-09 22:50:27 +0800
categories: Java
tags: Java
---

* content
{:toc}

## 前言

在`Class`、`Field`、`Method`、`Constructor`和`Parameter`中，我们都看到一个无参方法`public int getModifiers()`，它返回了一个int类型的数字。该方法用于获取元素的修饰符，包括：private、protected、public、static、final等等。

那么，为什么一个int类型就可以包含这么多的修饰符信息呢？主要实现手段为：将int类型看成多位的二进制数，让特定位和修饰符开关进行按位与操作，从而判断其是否包含某个特定的修饰符。这样既省了内存，又提高了判断效率。

要想更好地判断修饰符，我们可以借鉴于`java.lang.reflect.Modifier`工具类，从而简化我们的代码。

## 判断是否包含指定修饰符

这些方法都非常简单，大致如下：

```java
/**
 * Return {@code true} if the integer argument includes the
 * {@code public} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code public} modifier; {@code false} otherwise.
 */
public static boolean isPublic(int mod) {
    return (mod & PUBLIC) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code private} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code private} modifier; {@code false} otherwise.
 */
public static boolean isPrivate(int mod) {
    return (mod & PRIVATE) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code protected} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code protected} modifier; {@code false} otherwise.
 */
public static boolean isProtected(int mod) {
    return (mod & PROTECTED) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code static} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code static} modifier; {@code false} otherwise.
 */
public static boolean isStatic(int mod) {
    return (mod & STATIC) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code final} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code final} modifier; {@code false} otherwise.
 */
public static boolean isFinal(int mod) {
    return (mod & FINAL) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code synchronized} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code synchronized} modifier; {@code false} otherwise.
 */
public static boolean isSynchronized(int mod) {
    return (mod & SYNCHRONIZED) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code volatile} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code volatile} modifier; {@code false} otherwise.
 */
public static boolean isVolatile(int mod) {
    return (mod & VOLATILE) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code transient} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code transient} modifier; {@code false} otherwise.
 */
public static boolean isTransient(int mod) {
    return (mod & TRANSIENT) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code native} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code native} modifier; {@code false} otherwise.
 */
public static boolean isNative(int mod) {
    return (mod & NATIVE) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code interface} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code interface} modifier; {@code false} otherwise.
 */
public static boolean isInterface(int mod) {
    return (mod & INTERFACE) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code abstract} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code abstract} modifier; {@code false} otherwise.
 */
public static boolean isAbstract(int mod) {
    return (mod & ABSTRACT) != 0;
}

/**
 * Return {@code true} if the integer argument includes the
 * {@code strictfp} modifier, {@code false} otherwise.
 *
 * @param   mod a set of modifiers
 * @return {@code true} if {@code mod} includes the
 * {@code strictfp} modifier; {@code false} otherwise.
 */
public static boolean isStrict(int mod) {
    return (mod & STRICT) != 0;
}
```

## 特定元素的可用修饰符列表

这些方法都非常简单，大致如下：

```java
/**
 * Return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a class.
 * @return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a class.
 *
 * @jls 8.1.1 Class Modifiers
 * @since 1.7
 */
public static int classModifiers() {
    return CLASS_MODIFIERS;
}

/**
 * Return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to an interface.
 * @return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to an interface.
 *
 * @jls 9.1.1 Interface Modifiers
 * @since 1.7
 */
public static int interfaceModifiers() {
    return INTERFACE_MODIFIERS;
}

/**
 * Return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a constructor.
 * @return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a constructor.
 *
 * @jls 8.8.3 Constructor Modifiers
 * @since 1.7
 */
public static int constructorModifiers() {
    return CONSTRUCTOR_MODIFIERS;
}

/**
 * Return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a method.
 * @return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a method.
 *
 * @jls 8.4.3 Method Modifiers
 * @since 1.7
 */
public static int methodModifiers() {
    return METHOD_MODIFIERS;
}

/**
 * Return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a field.
 * @return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a field.
 *
 * @jls 8.3.1 Field Modifiers
 * @since 1.7
 */
public static int fieldModifiers() {
    return FIELD_MODIFIERS;
}

/**
 * Return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a parameter.
 * @return an {@code int} value OR-ing together the source language
 * modifiers that can be applied to a parameter.
 *
 * @jls 8.4.1 Formal Parameters
 * @since 1.8
 */
public static int parameterModifiers() {
    return PARAMETER_MODIFIERS;
}
```