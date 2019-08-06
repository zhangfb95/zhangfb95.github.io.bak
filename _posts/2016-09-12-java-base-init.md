---
layout: post
title:  Java基础（01）初始化动作
date:   2016-09-12 08:09:34 +0800
categories: Java基本技能
tag: java
---

* content
{:toc}


在C/C++中，分配内存和释放内存需要用户手动进行操作，一旦用户忘记或忽略了，极容易导致一些“不安全”的事故。
Java借鉴了C++的构造器概念，并基于此添加了补充特性，以此来保证正确的初始化和内存管理。

## 简单的构造器

Java的构造器也是一个方法，方法名和类名相同。

```java
class Simple {
    Simple() {

    }
}
```

## Java的初始化顺序

在Java中，有以下几个地方可以对类变量进行初始化。

1. 静态成员变量初始化
1. 静态代码块
1. 成员变量初始化
1. 构造方法
1. 代码块

其中静态相关的初始化，只会涉及到静态成员变量。Java对这些变量的初始化有一定的先后顺序。其基本规则如下：

1. 静态初始化 > 非静态初始化
1. 父类初始化 > 子类初始化
1. 成员变量初始化 > 构造方法初始化
1. 代码块初始化 > 构造方法初始化
1. 代码块和成员变量，谁先谁后，取决于他们放置在类中的位置

【说明】：其中最后一点需要特别注意，块可以放置于类方法和变量之外的任何位置。例如以下的标注一、标注二、标注三。

```java
public class Base {

    // 【标注一】
    {
        System.out.println("base code block 1");
    }

    private int baseParam = initBaseParam();

    // 【标注二】
    {
        System.out.println("base code block 2");
    }

    public Base() {
        System.out.println("base construct");
    }

    // 【标注三】
    {
        System.out.println("base code block 3");
    }

    private int initBaseParam() {
        System.out.println("base param");
        return 33;
    }
}
```

执行结果如下：

```
base code block 1
base param
base code block 2
base code block 3
base construct
```