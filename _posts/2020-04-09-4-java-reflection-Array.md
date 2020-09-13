---
layout: post
title:  Java反射（4）Array
date:   2020-04-09 22:51:27 +0800
categories: Java
tags: Java
---

* content
{:toc}

## 为什么需要用到Array

我们知道，所有的类最终都是继承于Object。我们可以将子类赋值给父类，但是却不能将父类赋值给子类。同样的，对于数组来说，我们可以将子类数组赋值给父类数组，却不能将父类数组赋值给子类，这样会抛出异常`java.lang.ClassCastException`。

下面代码就没有任何问题，可以编译，也可以允许。

```java
String[] strings1 = new String[5];
Object[] objects1 = strings1;
```

而下面的代码却会出现强转异常。

```java
Object[] objects2 = new Object[5];
String[] strings2 = (String[])objects2;
```

将会在运行时出现以下错误。

```
java.lang.ClassCastException: [Ljava.lang.Object; cannot be cast to [Ljava.lang.String;
```

## Array

该类的完整类路径为`java.lang.reflect.Array`，其所有方法皆为静态方法。其常用方法如下：

```java
// 以特定类型，构建指定长度的一维数组
public static Object newInstance(Class<?> componentType, int length);

// 以特定类型，构建指定维度、指定长度的多维数组
public static Object newInstance(Class<?> componentType, int... dimensions);

// 获取数组的长度
public static native int getLength(Object array);

// 获取数组指定下标的值
public static native Object get(Object array, int index);

// 获取数组指定下标的值（主要为基本数组类型）
public static native boolean getBoolean(Object array, int index);
public static native byte getByte(Object array, int index);
public static native char getChar(Object array, int index);
// ...

// 在数组的指定下标位置设置值
public static native void set(Object array, int index, Object value)

// 在数组的指定下标位置设置值（主要用于设置基本类型数组的值）
public static native void setBoolean(Object array, int index, boolean z);
public static native void setByte(Object array, int index, byte b);
public static native void setChar(Object array, int index, char c);
// ...
```

## Arrays

Array用于通过反射操作数组，而Arrays则是通过一系列的方法来操作数组。其完成类路径为`java.util.Arrays`。例如：

+ 串行排序
+ 并行排序
+ 二分查找
+ 填充数值
+ 数组拷贝
+ 转换list
+ 流式计算
+ 深拷贝和深hash

这些操作更多的是以静态方法的方式提供。而这些方法中，和Array关联比较大的在于数组拷贝。我们来看看：

```java
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}

public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```

最终调用了下面的方法，这个方法做了几件事情：

1. 判断拷贝后的类型和拷贝前的类型是否都是Object数组？
  1. 如果是，则创建指定长度的Object数组
  2. 如果不是，则通过Array的创建数组静态方法，创建指定类型、指定长度的数组
2. 将原始数组的元素依次拷贝到新数组

## 例子

```java
public static void main(String[] args) {
    Object array = Array.newInstance(int.class, 5);
    Array.set(array, 0, 99);
    Array.set(array, 1, 98);
    Array.set(array, 2, 97);
    Array.set(array, 3, 96);
    Array.set(array, 4, 95);

    System.out.println(Array.get(array, 0));
    System.out.println(Array.get(array, 1));
    System.out.println(Array.get(array, 2));
    System.out.println(Array.get(array, 3));
    System.out.println(Array.get(array, 4));
}
```

输出结果如下：

```
99
98
97
96
95
```

## 参考文档

+ https://blog.csdn.net/vonzhoufz/article/details/39377647
+ https://www.jianshu.com/p/754ed930adce
+ https://www.cnblogs.com/ixenos/p/5690251.html