---
layout: post
title:  Java反射（2）getDeclaredXxxx()和getXxxx()的比较
date:   2020-04-07 22:10:04 +0800
categories: Java
tags: Java
---

* content
{:toc}

## 简介

通过查看Class的接口声明，我们发现很多类似`getDeclaredXxxx()`的方法，这些方法和`getXxxx()`方法看上去很相似，都可以用以获取构造方法、普通方法、注解、字段等。那么，他们之间有什么联系和区别呢？通过本文，我们将主要从继承和访问范围两个方面，对其进行分析。

## Annotation相关

```java
// getDeclared相关方法
public <A extends Annotation> A getDeclaredAnnotation(Class<A> annotationClass);
public Annotation[] getDeclaredAnnotations()'
public <A extends Annotation> A[] getDeclaredAnnotationsByType(Class<A> annotationClass);

// get相关方法
public <A extends Annotation> A getAnnotation(Class<A> annotationClass);
public Annotation[] getAnnotations();
public <A extends Annotation> A[] getAnnotationsByType(Class<A> annotationClass);
```

`getDeclaredAnnotation()`返回直接存在于此元素上的所有注解。请注意，该方法将忽略继承的注解！如果没有注解直接存在于此元素上，则返回长度为零的一个数组。该方法的调用者可以随意修改返回的数组，这不会对其他调用者返回的数组产生任何影响。

`getAnnotation()`返回此元素上存在的所有注解。如果此元素没有注解，则返回长度为零的数组。该方法的调用者可以随意修改返回的数组，这不会对其他调用者返回的数组产生任何影响。

`getDeclaredAnnotations()`得到的是当前元素所有的注释，不包括继承的。而`getAnnotations()`得到的是包括继承的所有注解。

> 注意：要想让注解被获取到，需要在注解上声明`@Retention(value = RetentionPolicy.RUNTIME)`注解，这样才能在运行期由VM获取到。

> 同时，注解的继承特性，是通过在注解上声明`@Inherited`的方式来实现的，要想通过`getAnnotation()`获取注解，`@Inherited`也是必须要添加的。

## Constructor相关

```java
// getDeclared相关方法
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes);
public Constructor<?>[] getDeclaredConstructors();

// get相关方法
public Constructor<T> getConstructor(Class<?>... parameterTypes)
public Constructor<?>[] getConstructors();
```

`getDeclaredConstructors()`方法会返回全部构造器，包含private、protected和public类型的。其实它会返回所有的构造器，和具体的修饰符无关。

`getConstructors()`方法仅仅返回public类型的构造器。它返回的构造器是`getDeclaredConstructors()`返回的构造器的子集。

## Method相关

```java
public Method getDeclaredMethod(String name, Class<?>... parameterTypes);
public Method[] getDeclaredMethods();

public Method getMethod(String name, Class<?>... parameterTypes);
public Method[] getMethods();
```

`getDeclaredMethods()`获取当前类所有声明的方法，包括private、protected和public修饰的方法。需要注意的是，这些方法一定是在当前类中声明的。从父类中继承的不算。实现接口的方法由于在当前类有声明，所以包括在内。

`getMethods()`获取当前类和父类的所有的public方法。这里的父类，指的是继承层次中的所有父类。例如：A继承B，B继承C，那么B和C都属于A的父类。

> 注意：由于Java语言的面向对象特性，所以JAVA语言一定会遵循多态特性。因此只要子类重写了父类的方法，我们就没办法在使用子类的时候直接调用父类的方法。即使使用反射也办不到。当然，如果我们在子类的某个方法里面通过调用`super.xxx()`的方式来间接调用，还是能够办得到的。

我们来看一个例子：

```java
public class Main {

    public static void main(String[] args) throws Exception {
        Method method = Child.class.getDeclaredMethod("hello");
        method.invoke(new Child());
    }

    public static class Parent {

        public void hello() {
            System.out.println("Parent hello() called");
        }
    }

    public static class Child extends Parent {

        @Override
        public void hello() {
            System.out.println("Child hello() called");
        }
    }
}
```

输出结构如下：

```
Child hello() called
```

## Classes相关

```java
public Class<?>[] getDeclaredClasses();

public Class<?>[] getClasses();
```

`getDeclaredClasses()`得到该类所有的内部类，不包括父类的，会忽略修饰符。

`getClasses()`得到该类及其父类所有的public内部类。

## Field相关

```java
public Field getDeclaredField(String name);
public Field[] getDeclaredFields();

public Field getField(String name);
public Field[] getFields();
```

`getDeclaredField()`只能获取自己声明的各种字段，包括private、protected和public。实际上，获取到的字段和修饰符无关。

`getFields()`只能获取public的字段，包括父类的；

和获取普通方法的反射API不同，我们可以通过在父类Field的方式给子类对象赋值，从而给父类私有字段设置值。这种方法，在不使用反射的情况下是没办法实现的。我们来看一个例子：

```java
public class Main {

    public static void main(String[] args) throws Exception {
        Field field = Parent.class.getDeclaredField("age");
        field.setAccessible(true);
        Child child = new Child();
        field.set(child, 33);
        System.out.println("child: " + child);
    }

    public static class Parent {

        private int age;

        @Override
        public String toString() {
            return "age=" + age;
        }
    }

    public static class Child extends Parent {

        @Override
        public String toString() {
            return super.toString();
        }
    }
}
```

输出结构如下所示：

```
child: age=33
```

## 总结

总的来说，`getDeclaredXxxx()`方法返回的是不包括继承的，和具体修饰符无关。`getXxxx()`包括继承的，但是只包括public修饰的。

至此，我们已经完成了所有方法的比较，并进行了比较详细的说明。