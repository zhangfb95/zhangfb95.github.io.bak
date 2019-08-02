---
layout: post
title:  Java中的访问修饰符
date:   2019-06-12 21:33:00 +0800
categories: java
tag: java
---

* content
{:toc}

Java中的访问修饰符主要有以下4种：

1. private
2. protected
3. default - 什么都不写
4. public

这些修饰符可作用于属性、方法和类型，其中的类型主要包括class、interface、enum等。这些修饰符的访问权限范围又是怎样的呢？我们以一个表格来说明一下。

| 修饰符      | 当前java文件 | 同一个包 | 子类 | 其他的包 | 说明 |
| ---             | --- | --- | --- | --- | --- |
| public       | √ | √ | √ | √ |   我的东西任何人都可以分享。 |
| protected  | √ | √ | √ | × | 我的东西可以和跟我一块住的那个人分享，<br>另外也可以跟不在家的儿子分享。 |
| default      | √ | √ | × | × | 我的东西可以和跟我一块住的那个人分享。 |
| private      | √ | × | × | × | 我什么都不跟别人分享，只有自己知道。 |

这儿需要特别说明三点：

1. 这4种修饰符都可作用于`当前java文件`，而非网上说的`当前类`。
2. 在Java中，`外部类`只能使用public和default来进行修饰，而不能使用protected和private。所谓外部类，指的是java文件中最外层的根类。
3. 这4种访问修饰符都只在编译期进行范围权限检查。而在运行期，我们可以通过反射机制，对任意位置的任意属性和方法进行访问。例如：可以对另外一个包下面的私有属性和私有方法进行访问。

说明1的例子：

```java
/**
 * OuterClass为外部类类，而InnerClassB和InnerClassC为内部类
 * private修饰的内部类，可在当前java文件所访问，而非仅仅局限在当前类
 */
public class OuterClass {
    private static class InnerClassB {
        private int fieldB;
        private void methodB() {
        }
    }

    private static class InnerClassC {
        InnerClassB b = new InnerClassB();
        private void methodC() {
            b.methodB();
            b.fieldB = 3;
        }
    }
}
```

说明2的例子：

![private不能修饰外部类](https://upload-images.jianshu.io/upload_images/845143-cda3de9f0348d61c.png?jianshufrom=true)

![protected不能修饰外部类](https://upload-images.jianshu.io/upload_images/845143-ccaa8b7a447340c9.png?jianshufrom=true)
