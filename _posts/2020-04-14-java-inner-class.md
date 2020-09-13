---
layout: post
title:  内部类和匿名内部类
date:   2020-04-14 09:21:19 +0800
categories: Java
tags: Java
---

* content
{:toc}

## 前言

内部类，定义在类或方法里面的类。通常情况下，我们将内部类总共拆分为以下几种类型：

+ 成员内部类
+ 局部内部类
+ 匿名内部类
+ 静态内部类

## 成员内部类

成员内部类是最普通的内部类，它定义在另一个类的内部，形如下面的形式：

```java
public class Outer {

    private String name;

    public Outer(String name) {
        this.name = name;
    }

    public class Inner {

        public void print() {
            System.out.println("this is an Inner class.");
        }
    }
}
```

这样看起来，类`Inner`像是类`Outer`的一个成员，`Outer`称为外部类。成员内部类可以无条件访问外部类的所有成员属性和成员方法（包括private成员和静态成员）。

```java
public class Outer {

    private String name;
    private static int code = 9527;

    public Outer(String name) {
        this.name = name;
    }

    public class Inner {

        public void print() {
            System.out.print("this is an Inner class, ");
            System.out.print("outer.name is [" + name + "], "
            System.out.print("code is [" + code + "]");
            System.out.println();
        }
    }
}
```

不过要注意的是，当成员内部类拥有和外部类同名的成员变量或者方法时，会发生隐藏现象，即默认情况下访问的是成员内部类的成员。如果要访问外部类的同名成员，需要以下面的形式进行访问：

```
外部类.this.成员变量
外部类.this.成员方法
```

虽然成员内部类可以无条件地访问外部类的成员，但是外部类想访问成员内部类的成员却不是这么随心所欲了。在外部类中如果要访问成员内部类的成员，必须先创建一个成员内部类的对象，再通过指向这个对象的引用来访问：

```java
public class Outer {

    private String name;
    private static int code = 9527;

    public Outer(String name) {
        this.name = name;
    }

    public Inner getInnerInstance() {
        return new Inner();
    }

    public class Inner {

        public void print() {
            System.out.print("this is an Inner class, ");
            System.out.print("outer.name is [" + name + "], "
            System.out.print("code is [" + code + "]");
            System.out.println();
        }
    }
}
```

成员内部类是依附外部类而存在的。也就是说，如果要创建成员内部类的对象，前提是必须存在一个外部类的对象。创建成员内部类对象的一般方式如下：

```java
public static void main(String[] args) {
    // 第一种方式
    Outer outer = new Outer("zhangsan");
    Inner inner1 = outer.new Inner();

    // 第二种方式
    Inner inner2 = outer.getInnerInstance();
}
```

内部类可以拥有`private`访问权限、`protected`访问权限、`public`访问权限及包访问权限。

比如上面的例子：

1. 如果成员内部类`Inner`用`private`修饰，则只能在外部类的内部访问；
2. 如果用`public`修饰，则任何地方都能访问；
3. 如果用`protected`修饰，则只能在同一个包下或者继承外部类的情况下访问；
4. 如果是默认访问权限，则只能在同一个包下访问。

这一点和外部类有一点不一样，外部类通常只能被`public`和包访问两种权限修饰。但是也有例外，因为内部类所在的类仍然可能是一个另一个类的内部类。

我个人是这么理解的，由于成员内部类看起来像是外部类的一个成员，所以可以像类的成员一样拥有多种权限修饰。

> 【注意】：成员内部类中不能出现static修饰符，也就是说不能使用static修饰成员变量或方法，编译期会报错。

## 局部内部类

局部内部类是定义在一个方法或者一个作用域里面的类，它和成员内部类的区别在于局部内部类的访问仅限于方法内或者该作用域内。

```java
public class Outer2 {

    class Person {
    }

    class Man {

        public Person getWoman() {
            class Woman extends Person {

                int age = 0;
            }
            return new Woman();
        }
    }
}
```

> 【注意】
> 1. 局部内部类就像是方法里面的一个局部变量一样，是不能使用 `public`、`protected`、`private`以及`static`修饰符的；
> 2. 局部内部类中的成员变量和方法，都不能使用static关键字进行修饰。

## 匿名内部类

顾名思义，匿名内部类就是没有名字的内部类，应该是平时我们编写代码时用得最多的，在编写事件监听的代码时使用匿名内部类不但方便，而且使代码更加容易维护。如下所示：

```java
scan_bt.setOnClickListener(new OnClickListener() {

    @Override
    public void onClick(View v) {
        // TODO Auto-generated method stub
    }
});
 
history_bt.setOnClickListener(new OnClickListener() {
     
    @Override
    public void onClick(View v) {
        // TODO Auto-generated method stub
    }
});
```

这段代码为两个按钮设置监听器，这里面就使用了匿名内部类。这段代码中的：

```java
scan_bt.setOnClickListener(new OnClickListener() {

    @Override
    public void onClick(View v) {
        // TODO Auto-generated method stub
    }
});
```

就是匿名内部类的使用。代码中需要给按钮设置监听器对象，使用匿名内部类能够在实现父类或者接口中的方法情况下同时产生一个相应的对象，但是前提是这个父类或者接口必须先存在才能这样使用。当然像下面这种写法也是可以的，跟上面使用匿名内部类达到效果相同。

```java
private void setListener() {
    scan_bt.setOnClickListener(new Listener1());
    history_bt.setOnClickListener(new Listener2());
}

class Listener1 implements View.OnClickListener {
    @Override
    public void onClick(View v) {
        // TODO Auto-generated method stub
    }
}
 
class Listener2 implements View.OnClickListener {
    @Override
    public void onClick(View v) {
        // TODO Auto-generated method stub
    }
}
```

这种写法虽然能达到一样的效果，但是既冗长又难以维护，所以一般使用匿名内部类的方法来编写事件监听代码。同样的，匿名内部类也是不能有访问修饰符和 `static`修饰符的。

匿名内部类是唯一一种没有构造器的类。正因为其没有构造器，所以匿名内部类的使用范围非常有限，大部分匿名内部类用于接口回调。匿名内部类在编译的时候由系统自动起名为类似`Outter$1.class`（如果有多个内部类，则$后面的数字依次累加）。一般来说，匿名内部类用于继承其他类或是实现接口，并不需要增加额外的方法，只是对继承方法的实现或是重写。

> 【注意】：
> 1. 匿名内部类内部可以使用静态成员变量，例如：`public static final String HELLO = "hello";`;
> 2. 匿名内部类不能使用static静态块和static静态方法。

## 静态内部类

静态内部类也是定义在另一个类里面的类，只不过在类的前面多了一个关键字`static`。静态内部类是不需要依赖于外部类的，这点和类的静态成员属性有点类似，并且它不能使用外部类的`非static`成员变量或者方法，这点很好理解，因为在没有外部类的对象的情况下，可以创建静态内部类的对象，如果允许访问外部类的`非static`成员就会产生矛盾，因为外部类的`非static`成员必须依附于具体的对象。

```java
public class Outter {

    public Outter() {

    }

    static class Inner {
        public Inner() {
        }
    }

    public static void main(String[] args)  {
        Outter.Inner inner = new Outter.Inner();
    }
}
```

## 内部类使用场景

为什么在Java中需要内部类？总结一下，主要有以下几点：

1. 每个内部类都能独立的继承一个接口的实现，所以无论外部类是否已经继承了某个接口的实现，对于内部类来说都没有影响。内部类使得多继承的解决方案变得完整。
2. 方便将存在一定逻辑关系的类组织在一起，又可以对外界隐藏。
3. 方便编写事件驱动程序。
4. 方便编写线程代码。

其实第一点正是对Java语言的极大补充，应该最为重要。但是楼主在实际工作中发现后面3点用得反而更多。

## 常见内部类相关的面试题

1. 根据注释填写(1)，(2)，(3)处的代码

```java
public class Test {

    public static void main(String[] args) {
        // 初始化Bean1
        (1)
        bean1.i++;
        // 初始化Bean2
        (2)
        bean2.j++;
        // 初始化Bean3
        (3)
        bean3.k++;
    }

    class Bean1 {
        public int i = 0;
    }

    static class Bean2 {
        public int j = 0;
    }
}

class Bean {
    class Bean3 {
        public int k = 0;
    }
}
```

根据前面可知，对于成员内部类，必须先产生外部类的实例化对象，才能产生内部类的实例化对象。而静态内部类不用产生外部类的实例化对象即可产生内部类的实例化对象。

创建静态内部类对象的一般形式为： 

```
外部类类名.内部类类名 xxx = new 外部类类名.内部类类名()
```

创建成员内部类对象的一般形式为：

```
外部类类名.内部类类名 xxx = 外部类对象名.new 内部类类名()
```

因此，（1），（2），（3）处的代码分别为：

```java
// （1）
Test test = new Test();    
Test.Bean1 bean1 = test.new Bean1();

// （2）
Test.Bean2 bean2 = new Test.Bean2();

// （3）
Bean bean = new Bean();     
Bean.Bean3 bean3 =  bean.new Bean3();
```

2. 下面这段代码的输出结果是什么？

```java
public class Test {
    public static void main(String[] args)  {
        Outter outter = new Outter();
        outter.new Inner().print();
    }
}
 
 
class Outter
{
    private int a = 1;
    class Inner {
        private int a = 2;
        public void print() {
            int a = 3;
            System.out.println("局部变量：" + a);
            System.out.println("内部类变量：" + this.a);
            System.out.println("外部类变量：" + Outter.this.a);
        }
    }
}
```

答案如下：

```
局部变量：3
内部类变量：2
外部类变量：1
```

最后再补充一点知识：关于成员内部类的继承问题。一般来说，内部类是很少用来作为继承用的。但是当用来继承的话，需要注意两点：

1. 成员内部类的引用方式必须为`Outter.Inner`
2. 构造器中必须有指向外部类对象的引用，并通过这个引用调用`super()`。这段代码摘自《Java编程思想》

```java
class Outer {
    class Inner{
    }
}

class InheritInner extends Outer.Inner {
      
    // InheritInner() 是不能通过编译的，一定要加上形参
    InheritInner(Outer outer) {
        outer.super(); // 必须有这句调用
    }
  
    public static void main(String[] args) {
        Outer outer = new Outer();
        InheritInner obj = new InheritInner(outer);
    }
}
```

## 参考链接

+ https://www.runoob.com/w3cnote/java-inner-class-intro.html
+ https://www.cnblogs.com/dolphin0520/p/3811445.html
+ https://www.cnblogs.com/geeksongs/p/9836154.html
+ https://blog.csdn.net/guyuealian/article/details/51981163
+ https://www.cnblogs.com/livterjava/p/4716693.html
+ https://www.cnblogs.com/outmanxiaozhou/p/11208019.html