---
layout: post
title:  Java注解(annotation)机制
date:   2018-12-04 16:16:00 +0800
categories: 深入理解Java核心
tag: java
---

* content
{:toc}

## 前言

`jdk1.5`引入了`注解机制`（`Annotation`），用于对java里面的元素（如：Class、Method、Field等等）进行标记。同时，java的反射类库也加入了对`Annotation`的支持，因此我们可以利用反射来对特殊的`Annotation`进行特殊的处理，增强代码的语义。

本文主要是对`Annotation的语法`和`Annotation的用法`进行分析阐述。然后对一些java自带的、常用的`Annotation`进行说明，加深读者的理解。

## 整体结构

借用网上的一张图，来说明整体结构。

![Annotation整体结构图](https://upload-images.jianshu.io/upload_images/845143-bc038352732d72d2.png?jianshufrom=true)

通过这张图我们看到下面的信息

1. 一个`Annotation`是一个接口
2. 一个`Annotation`和一个`RetentionPolicy`关联
3. 一个`Annotation`和多个`ElementType`关联
4. `Annotation`可以有很多实现，包括java自带的`@Override`、`@Deprecated`和`@Inherited`等等，用户也可以自己定义

为了更好地理解`Annotation`，我们将`Annotation整体结构`拆分为左右两个部分来讲解。

## Annotation的组成

先来看看整体结构的左边部分

![整体结构左边部分](https://upload-images.jianshu.io/upload_images/845143-0b5b5732eb05310b.png?jianshufrom=true)


`Annotation`关联了3个重要的类，分别为`Annotation`、`ElementType`和`RetentionPolicy`，这3个类所在的包为`java.lang.annotation`，下面我们来看看java里面的定义。

### 1. Annotation接口

```java
public interface Annotation {
    boolean equals(Object obj);
    int hashCode();
    String toString();
    Class<? extends Annotation> annotationType();
}
```

### 2. ElementType枚举

```java
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE, // 类、接口（包括annotation类型）或者枚举声明

    /** Field declaration (includes enum constants) */
    FIELD, // 字段声明（包括枚举常量）

    /** Method declaration */
    METHOD, // 方法声明

    /** Formal parameter declaration */
    PARAMETER, // 参数声明

    /** Constructor declaration */
    CONSTRUCTOR, // 构造器声明

    /** Local variable declaration */
    LOCAL_VARIABLE, // 本地变量（也叫临时变量）声明

    /** Annotation type declaration */
    ANNOTATION_TYPE, // annotation类型声明

    /** Package declaration */
    PACKAGE, // 包声明

    /**
     * Type parameter declaration
     *
     * @since 1.8
     */
    TYPE_PARAMETER, // 类型参数声明（泛型的类型参数）

    /**
     * Use of a type
     *
     * @since 1.8
     */
    TYPE_USE
}
```

### 3. RetentionPolicy枚举

```java
public enum RetentionPolicy {
    /**
     * Annotations are to be discarded by the compiler.
     */
    SOURCE, // 仅用于编译期间，编译完成之后，该annotation就无用了

    /**
     * Annotations are to be recorded in the class file by the compiler
     * but need not be retained by the VM at run time.  This is the default
     * behavior.
     */
    CLASS, // 编译器将该annotation编译进class里面，但vm运行期间不会保留，默认行为

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *
     * @see java.lang.reflect.AnnotatedElement
     */
    RUNTIME // 编译器将该annotation编译近class，同时vm运行期间也可以对此annotation使用反射读取
}
```

接下来我们对这3个类做一下总结

1. `Annotation`是一个接口，一个`Annotation`包含`一个RetentionPolicy`和`多个ElementType`
2. `ElementType`是类型属性，一种类型可以简单地看成一种用途，每个`Annotation`可以有多种用途
3. `RetentionPolicy`是策略属性，共有3种策略，每个`Annotation`可选择一种策略，默认为`CLASS`策略
    + `SOURCE`，仅用于编译期间，编译完成之后，该`Annotation`就无用了。`@Override`就属于这种策略，仅仅在编译期检查子类是否可覆盖父类
    + `CLASS`，编译期将该`Annotation`编译进class里面，但vm运行期间不会保留，默认行为
    + `RUNTIME`，编译期将该`Annotation`编译进class，同时vm运行期间也可以对此`Annotation`使用反射读取

### 通用定义

我们来看看`Annotation`的通用定义，先来写一个`自定义Annotation`

```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value();
}
```

+ `@interface` -- 该修饰符是在关键字`interface`前加了一个`@`表示这是一个`Annotation`
+ `@Documented` -- 表示该`Annotation`可以在`javadoc`中出现，默认不会在`javadoc`出现
+ `@Target(ElementType.TYPE)` -- 表示该`Annotation`可以出现在类型声明上面，比如类、接口、枚举等等
+ `@Retention(RetentionPolicy.RUNTIME)` -- 表示该`Annotation`可以被编译进class，同时在运行期vm也可以通过反射读取到它
+ `Annotation`可以声明成员变量，需要注意以下几点
    1. 声明的方式为无参方法，且没有实现体，如：`{type} methodName();`
    2. 方法的返回类型为成员变量类型
    3. 方法名为成员变量名字
    4. 成员变量类型只能是`基本类型`、`String`和`自定义枚举`，其他的Interface、Class等都不能当成成员变量
    5. 成员变量可以使用`default`来设置默认值，使用时可以不传入值，如：`{type} methodName() default {defaultValue};`

## 常用Annotation

我们再来看整体结构的右边部分，我们看到很多我们经常接触到的`Annotation`。我们会逐一来分析一下。

![整体结构的右边部分](https://upload-images.jianshu.io/upload_images/845143-4189f6b5f5cae892.png?jianshufrom=true)


> 【元注解】：网上有`元注解`一说，其实`注解上面的注解叫做元注解`。当我们通过`@Target(ElementType.ANNOTATION_TYPE)`来修饰一个`Annotation`的时候，就表示该`Annotation`是一个元注解。

在jdk里面，有一些`Annotation`是经常用到的，为了加深我们对`Annotation`的理解，我们需要对这些`Annotation`进行分析。

+ `@Documented`，所标注内容，可以出现在`javadoc`中。
+ `@Inherited`，只能被用来标注`Annotation类型`，它所标注的`Annotation`具有继承性。
+ `@Retention`，只能被用来标注`Annotation类型`，而且它被用来指定`Annotation`的`RetentionPolicy`属性。
+ `@Target`，只能被用来标注`Annotation类型`，而且它被用来指定`Annotation`的`ElementType`属性。
+ `@Deprecated`，所标注内容，不再被建议使用。
+ `@Override`，只能标注方法，表示该方法覆盖父类中的方法。
+ `@SuppressWarnings`，所标注内容产生的警告，编译器会对这些警告保持静默。

### 1. @Inherited

我们写一个例子就能很容易看出`@Inherited`的作用了。

```java
public class InheritedTest {

    @Documented
    @Inherited
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME) @interface MyAnnotation {
    }

    @MyAnnotation class Father {
    }

    class Child extends Father {
    }

    public static void main(String[] args) {
        MyAnnotation[] annotations = Child.class.getAnnotationsByType(MyAnnotation.class);
        System.out.println("annotations length: " + annotations.length);
        for (MyAnnotation annotation : annotations) {
            System.out.println(annotation);
        }
    }
}
```

当`Annotation`用`@Inherited`修饰时，运行结果如下：

![使用@Inherited的效果](https://upload-images.jianshu.io/upload_images/845143-417b4b015003564c.png?jianshufrom=true)

当`Annotation`不用`@Inherited`修饰时，运行结果如下：

![不使用@Inherited的效果](https://upload-images.jianshu.io/upload_images/845143-31b1c4b700c2771f.png?jianshufrom=true)

### 2. @Retention

1. `RetentionPolicy.SOURCE`类型的class结构和运行效果

![class结构图](https://upload-images.jianshu.io/upload_images/845143-b0d6503a772dc4d0.png?jianshufrom=true)

![运行效果图](https://upload-images.jianshu.io/upload_images/845143-04358662845d6143.png?jianshufrom=true)

2. `RetentionPolicy.CLASS`类型的class结构和运行效果
![class结构图](https://upload-images.jianshu.io/upload_images/845143-9a6c7adb5efa2e87.png?jianshufrom=true)

![运行效果图](https://upload-images.jianshu.io/upload_images/845143-7116f3ec55dccfa3.png?jianshufrom=true)

3. `RetentionPolicy.RUNTIME`类型的class结构和运行效果
![class结构图](https://upload-images.jianshu.io/upload_images/845143-21d8563a3724bad5.png?jianshufrom=true)

![运行效果图](https://upload-images.jianshu.io/upload_images/845143-69623165c84b9aa3.png?jianshufrom=true)

### 3. @Target

`ElementType.TYPE`修饰的`Annotation`不能作用于字段和方法，在IntellijIdea里面直接就有错误提示，同时编译的时候也会出错。

![IDEA警告](https://upload-images.jianshu.io/upload_images/845143-0ccb466274d3610c.png?jianshufrom=true)

![编译错误](https://upload-images.jianshu.io/upload_images/845143-d574b0c5305d352b.png?jianshufrom=true)

### 4. @Deprecated

使用`@Deprecated`修饰的类或方法，编译不会出错，运行也不会出错，但是会给出警告。

![IDEA警告](https://upload-images.jianshu.io/upload_images/845143-ceebaf237be4d40a.png?jianshufrom=true)

### 5. @Override

![IDEA警告](https://upload-images.jianshu.io/upload_images/845143-70f41d25f419fa98.png?jianshufrom=true)

![编译出错](https://upload-images.jianshu.io/upload_images/845143-d8733d39d8e64c0c.png?jianshufrom=true)

`Child.sayHello2()`虽然会有IDEA警告，但是不会编译出错；`Child.sayHello3()`因为使用了`@Override`修饰，但是在父类里面并没有`sayHello3()`这个方法，所以会编译出错；`Child.sayHello1()`属于正常使用。

### 6. @SuppressWarnings

`@SuppressWarnings`可用于消除警告，可以消除哪些情况下的警告呢，我们举一个例子。

![例子图](https://upload-images.jianshu.io/upload_images/845143-0e68652861137cfc.png?jianshufrom=true)

在这个例子里面

1. 我们使用了已经废弃的`java.util.Date`构造方法`public Date(int year, int month, int date)`，所以在编译的时候会出现编译警告。
2. 因为泛型擦除的原因，往指定泛型插入非泛型类型时会出现警告。
3. 通过使用`@SuppressWarnings`，我们可以消除编译期警告

## Annotation的作用

我们总结一下`Annotation的作用`，大概有下面这几种

1. 编译检查，jdk自带的`@SuppressWarnings`、`@Deprecated`和`@Override`都有编译检查的功能
2. 在反射中使用Annotation，这在很多的框架代码里面大量出现，比如Spring、Mybatis、Hibernate等等
3. 根据`Annotation`生成帮助文档，通过使用`@Documented`，我们可以将`Annotation`也生成到javadoc里面
4. 能够帮忙查看查看代码，比如通过使用`@Override`和`@Deprecated`，我们很容易知道我们的集成层级、废弃状态等等

## 拾遗1 - @SuppressWarnings可以消除哪些警告

不同的编译器，可以消除的警告会有所不同，如果我们使用的是javac编译器，那么我们可以通过命令`javac -X`来列出所有支持的可以消除的警告。常用的有下面这些

+ `deprecation` - to suppress warnings relative to deprecation（抑制过期方法警告）
+ `rawtypes` - to suppress warnings relative to un-specific types when using +  generics on class params（使用generics时忽略没有指定相应的类型）
+ `unchecked` - to suppress warnings relative to unchecked operations（抑制没有进行类型检查操作的警告）
+ `unused` - to suppress warnings relative to unused code （抑制没被使用过的代码的警告）

## 拾遗2 - Annotation中的value和default

在`Annotation`的成员变量里面，`value`很多时候可以简化我们对`Annotation`的使用。

1. 当我们使用`Annotation`的时候，如果只有一个`value`需要传入的时候，我们省略掉`value`，即：`Element`。
2. 当`value`声明为数组，使用时需要用大括号括起来+逗号分隔，如`{"FirstElement", "SecondElement"}`
3. 当`value`声明为数组，使用时只有一个元素时，可当成单元素使用，即：`"Element"`

```java
public class ValueTest {

    @Documented
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME) @interface OneElement {
        String value();
    }

    @Documented
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME) @interface MultiElement {
        String[] value();
    }

    @Documented
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME) @interface MultiButOnlyInputOneElement {
        String[] value();
    }

    @OneElement("abc")
    @MultiElement({"abc", "cba"})
    @MultiButOnlyInputOneElement("abc")
    class User {
    }
}
```

同时，在定义Annotation成员变量的时候，我们可以使用`default`来设置默认值，使用时如果该成员变量的值和默认值相同，则可省略此成员变量的值传入。

```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME) @interface MyAnnotation {
    String value() default "is null";
}
```

## 总结

本文，我们首先给出了`Annotation的整体结构图`，然后分析了`Annotation`的语法和用法，最后我们给出了一些例子来说明了jdk自带的`Annotation`的用法，并总结了`Annotation`的使用场景。同时，我们通过对一些技巧性使用的补充，加深了我们对`Annotation`的印象。

## 参考链接

+ https://juejin.im/post/5bd084dee51d455d3d219048
+ https://www.cnblogs.com/be-forward-to-help-others/p/6846821.html
+ https://www.cnblogs.com/skywang12345/p/3344137.html
+ https://blog.csdn.net/moxiaoya1314/article/details/80763201