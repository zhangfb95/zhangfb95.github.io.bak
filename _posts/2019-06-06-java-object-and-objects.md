---
layout: post
title:  Java中的Object和Objects
date:   2019-06-06 23:47:00 +0800
categories: java
tag: java
---

* content
{:toc}

## 前言

Java中的所有类最终都继承于Object。那么，Object有何神妙的地方，它都有哪些方法或属性呢？

另外，有没有工具类可以简化我们使用频度非常高的一些操作呢？比如：判断null、比较操作、字符串操作等等。

## Java对象的鼻祖 - Object

### 1. getClass()

`public final native Class<?> getClass();`。获取对象的运行时class对象，class对象就是描述对象所属类的对象。这个方法通常是和Java反射机制搭配使用的。

### 2. hashCode()

`public native int hashCode();`。获取对象的散列值，散列值主要用在散列表中。在Java中，最常用的散列表实现为`java.util.HashMap`。Object中该方法默认返回的是对象的堆内存地址。

### 3. equals()

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

该方法用于比较两个对象，如果这两个对象引用指向的是同一个对象，那么返回true，否则返回false。

### 4. clone()

`protected native Object clone() throws CloneNotSupportedException;`。

这个方法用于克隆对象。被克隆的对象必须实现`java.lang.Cloneable`接口，否则会抛出`CloneNotSupportedException`异常。

默认的clone方法是`浅拷贝模式`。所谓浅拷贝，指的是对象内属性引用的对象只会拷贝引用地址，而不会将引用的对象重新分配内存。深拷贝则是会连引用的对象也重新创建。

### 5. toString()

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

这个方法的使用频率非常高，用于返回一个可代表对象的字符串。

默认返回格式如下：`对象的class名称` + `@` + `hashCode的十六进制字符串`

### 6. finalize()

这个方法用于在GC的时候再次被调用，如果我们实现了这个方法，对象可能在这个方法中再次`复活`，从而避免被GC回收。

通常情况下，我们不需要自己实现这个方法。

有的同学可能会觉得，可以使用这个方法来释放一些资源，比如关闭数据库连接、回收对象池中的对象、关闭IO流等等。而这往往不是一个好的选择，因为GC并不会保证这个方法一定会被调用，也不保证调用成功。

### 7. wait() / notify() / notifyAll()

这几个方法主要用于多线程间的通信，而多线程通信是一个相对比较复杂的话题，此文不再过多介绍。

## Objects工具类 - 简化我们的操作

在JDK7版本的时候，Java引入了`java.util.Objects`工具类，用于封装一些平时使用频度很高或容易出错的操作，这些操作形成了Objects的各个方法，下来我们来看看这些方法。

### 1. Objects()构造方法

```java
private Objects() {
    throw new AssertionError("No java.util.Objects instances for you!");
}
```

这个方法没有特别的地方，但有两点需要特别注意：

1. 该方法为私有方法
2. 其实现为抛出一个异常（或错误）

其主要目的是为了避免对象的构造，即使通过反射机制也做不到！

### 2. equals()

有别于`Object.equals()`，这个方法可以避免空指针异常。

```java
public static boolean equals(Object a, Object b) {
    return (a == b) || (a != null && a.equals(b));
}
```

### 3. deepEquals()

`Object.equals()`用于比较两个对象的引用是否相同，而`deepEquals()`却扩展成了可以支持数组。

```java
 public static boolean deepEquals(Object a, Object b) {
    if (a == b)
        return true;
    else if (a == null || b == null)
        return false;
    else
        return Arrays.deepEquals0(a, b);
}
```

### 4. hashCode(Object o)

和`Object.hashCode()`类似，只是在对象为null时返回的散列值为0而已。

```java
public static int hashCode(Object o) {
    return o != null ? o.hashCode() : 0;
}
```

### 5. hash()

生成对象的散列值，包括数组。

```java
public static int hash(Object... values) {
    return Arrays.hashCode(values);
}
```

### 6. toString()

归根结底，其内部最终调用了对象的`toString()`方法。只是额外多了空指针判断而已。

```java
public static String toString(Object o) {
    return String.valueOf(o);
}
public static String toString(Object o, String nullDefault) {
    return (o != null) ? o.toString() : nullDefault;
}
```

### 7. compare()

用于比较两个对象。

```java
public static <T> int compare(T a, T b, Comparator<? super T> c) {
    return (a == b) ? 0 :  c.compare(a, b);
}
```

### 8. requireNonNull()

在对象为空指针时，抛出特定message的空指针异常。

```java
public static <T> T requireNonNull(T obj) {
    if (obj == null)
        throw new NullPointerException();
    return obj;
}
public static <T> T requireNonNull(T obj, String message) {
    if (obj == null)
        throw new NullPointerException(message);
    return obj;
}
public static <T> T requireNonNull(T obj, Supplier<String> messageSupplier) {
    if (obj == null)
        throw new NullPointerException(messageSupplier.get());
    return obj;
}
```

### 9. isNull() 和 nonNull()

这两个方法用于判断对象为null和对象不为null。通常情况下，我们不会直接使用这两个方法，而是使用比较操作符`==`和`!=`。这两个方法主要用在jdk8开始支持的流计算里面。

```java
public static boolean isNull(Object obj) {
    return obj == null;
}
public static boolean nonNull(Object obj) {
    return obj != null;
}
```

## 总结

我们全面分析了Java对象的根类 - Object。同时，也分析了我们使用频率最高的一些操作封装成的工具类 - Objects。

通过这篇文章，我们对这两个类有了比较深入的理解。