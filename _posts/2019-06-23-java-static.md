---
layout: post
title:  Java中的static关键字
date:   2019-06-23 00:02:00 +0800
categories: java
tag: java
---

* content
{:toc}

## 前言

相对于C++而言，Java中static功能的语义和用法有了很大的不同。在本文中，我们不讨论static在C++和Java中的异同，也不讨论更深层次下的JVM实现。而是站在使用者角度，来对static进行一个全面、简单的讲解。

## static的核心作用

在《Java编程思想》第四版P86页中，有一个地方对static进行了简短的描述。

> static方法就是没有this的方法。在static方法内部不能调用非静态方法，反过来倒是可以的。而且可以在没有创建任何对象的前提下，仅仅通过类本身来调用static方法。这实际上正是static方法的主要用途。

这段话简单明了地说明了static的用途，也就是  --  `在没有创建对象的情况下调用方法或变量`。

static修饰的方法或变量，在可访问的范围内，不必创建对象就可进行访问。

static虽然避免了对象的创建，但在实际中，并不会大量地使用，原因在于其不是面向对象的。当出现大量使用static的情况时，就需要慎重反思一下，是否设计出现了问题。尽管如此，static也有其应用场景，例如：单例、多例、工具类和全局缓存等等。

## static的四种用途

static有以下5种用途，`修饰变量`和`修饰方法`通常会联合起来使用，从而达到单例和缓存的效果。

1. 修饰变量
2. 修饰方法
3. 修饰代码块
4. 静态导入包
5. 静态内部类

#### 1. 修饰变量

static变量和非static变量的区别在于，static变量全局只有一个副本。而非static变量，也叫成员变量，每创建一个对象就会产生一个副本。static变量在使用时应尽量带上类名，尽量避免通过对象来访问static变量。

```java
public class Main {
    private static int number = 33;
    public static void main(String[] args) {
        // 不正确的方式
        // 但通过这种方式，我们能看到共享副本的效果
        Main main1 = new Main();
        main1.number = 3;
        System.out.println(main1.number);
        Main main2 = new Main();
        main2.number = 5;
        System.out.println(main1.number);
        // 正确的方式
        System.out.println(Main.number);
    }
}
```

以上代码输出结果如下：

```java
3
5
5
```

#### 2. 修饰方法

static修饰的方法被称作静态方法，静态方法不能访问非静态方法，也不能访问非静态属性。非静态方法可以访问静态方法和静态属性。当违背此条规则时，编译器将报错。

```java
public class Main {
    private static int number = 33;
    private int number1 = 3;
    
    public static void main(String[] args) {
        System.out.println(number1);
        System.out.println(add(1,2));
    }

    private int add(int a, int b) {
        return a + b;
    }
}
```

编译的时候报如下错误

![编译出错](https://upload-images.jianshu.io/upload_images/845143-0d0891b913e7a32e.png?jianshufrom=1)

#### 3. 修饰代码块

static可修饰代码块，在类加载的时候就会被调用。同时，一个类里面还可以有多个static代码块，其执行的顺序和书写的顺序一致。

类加载会优先于类的构造方法调用，且由于受类加载机制的影响（双亲委派机制），父类的static总是优先于子类的static调用。切记，这一点非常重要！

接下来，我们通过一个例子来看看是否如此。

```java
// 父类
public class Parent {
    static {
        System.out.println("parent static first");
    }
    public Parent() {
        System.out.println("parent construct");
    }
    static {
        System.out.println("parent static second");
    }
}

// 子类
public class Sub extends Parent {
    static {
        System.out.println("sub static first");
    }
    public Sub() {
        System.out.println("sub construct");
    }
    static {
        System.out.println("sub static second");
    }
}

// 调用者
public class Client {
    public static void main(String[] args) {
        new Sub();
    }
}
```

输出结果如下，从而验证了我们的想法。

![父子类static](https://upload-images.jianshu.io/upload_images/845143-ac98f3eea3d8a0cb.png?jianshufrom=1)

#### 4. 静态导入包

静态导入包机制，实际项目中可能接触或留意得比较少。由于其使用简单，同时功能强大。使用恰当的话，可有效地减少代码量，从而提高代码的可读性。

```java
package com.juconcurrent.demo;

public class Helper {

    public static int MODE_COUNT = 9527;

    public static int add(int a, int b) {
        return a + b;
    }
}
```

```java
import static com.juconcurrent.demo.Helper.MODE_COUNT;
import static com.juconcurrent.demo.Helper.add;

public class Main {

    public static void main(String[] args) {
        System.out.println(add(1, 3));
        System.out.println(MODE_COUNT);
    }
}
```

我们很直观地看出了其用法。没错，就是通过`import static {package.clazz.static_param}`和`import static {package.clazz.static_method}`的方式，让静态属性和静态方法可以在使用的时候省略掉所属的类名！

这种方式带来的好处是类名省掉了，但是坏处却也正因为此。在属性名和方法名冲突时，将带来不可预知的编译错误。所以，凡事有利有弊，需要根据业务场景酌情考虑。

#### 5. 静态内部类

static修饰的内部类，将不会持有外部类的对象，使之更容易和外部类进行隔离，降低依赖。这种方式特别适合一种场景 - `希望将逻辑紧密的对象写在一个文件，降低阅读和修改成本，同时又使其相互独立`。例如：在设计接口返回类时，返回包含一个泛型List。

```java
public class UserResponse {
    private List<Book> books; // 阅读书籍列表

    public static class Book {
        private String name; // 姓名
        private String viewCount; // 浏览数
        private String followCount; // 喜欢数
    }
}
```

相对于static静态内部类，非static修饰的内部类，在创建类之前必须先创建外部类。我们来看一个例子：

```java
public class Main {
    public static void main(String[] args) {
        // 正确的方式
        new Inner();
        System.out.println(new Main().new Inner2());

        // 错误的方式，将产生编译错误 - Error:(11, 9) java: 无法从静态上下文中引用非静态 变量 this
        new Inner2();

        // 错误的方式，将产生编译错误 - Error:(14, 9) java: 限定的新静态类
        new Main().new Inner();
    }

    static class Inner {
    }

    class Inner2 {
    }
}
```

## 总结

static在java中比较重要，主要有四种用途。而static最常用的场景在于单例和工具类的实现。

## 参考链接

+ https://www.cnblogs.com/dotgua/p/6354151.html?utm_source=itdadao&utm_medium=referral
+ https://www.cnblogs.com/dolphin0520/p/3799052.html
