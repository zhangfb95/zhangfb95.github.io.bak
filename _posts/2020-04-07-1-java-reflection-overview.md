---
layout: post
title:  Java反射（1）总览
date:   2020-04-07 00:00:01 +0800
categories: Java
tags: Java
---

* content
{:toc}

## 概念

Java的反射（reflection）机制是指在程序的运行状态中，可以构造任意一个类的对象，可以了解任意一个对象所属的类，可以了解任意一个类的成员变量和方法，可以调用任意一个对象的属性和方法。这种动态获取程序信息以及动态调用对象的功能称为Java语言的反射机制。反射被视为动态语言的关键。

## 功能

Java反射机制主要提供了以下功能：

1. 在运行时获取任意一个对象所属的类；
2. 在运行时构造任意一个类的对象；
3. 在运行时获取任意一个类所具有的成员变量和方法；
4. 在运行时调用任意一个对象的方法；
5. 生成动态代理。

有时候我们说某个语言具有很强的动态性，有时候我们会区分动态和静态的不同技术与做法。我们朗朗上口动态绑定（dynamic binding）、动态链接（dynamic linking）、动态加载（dynamic loading）等。

然而，“动态”一词其实没有绝对而普遍适用的严格定义，有时候甚至像面向对象当初被导入编程领域一样，一人一把号，各吹各的调。

一般而言，开发者社群说到动态语言，大致认同的一个定义是：“程序运行时，允许改变程序结构或变量类型，这种语言称为动态语言”。从这个观点看，Perl，Python，Ruby是动态语言，C++，Java，C#不是动态语言。

尽管在这样的定义与分类下Java不是动态语言，它却有着一个非常突出的动态相关机制：Reflection。这个单词的意思是“反射、映象、倒影”，用在Java身上指的是我们可以于运行时加载、探知、使用编译期间完全未知的classes。换句话说，Java程序可以加载一个运行时才得知名称的class，获悉其完整构造（但不包括methods定义），并生成其对象实体、或对其fields设值、或唤起其methods。这种“看透class”的能力（the ability of the program to examine itself）被称为introspection（内省、内观、反省）。Reflection和introspection是常被并提的两个术语。

Java如何能够做出上述的动态特性呢？这是一个深远话题，本文对此只简单介绍一些概念。整个篇幅最主要还是介绍Reflection APIs，也就是让读者知道如何探索class的结构、如何对某个“运行时才获知名称的class”生成一份实体、为其fields设值、调用其methods。本文将谈到`java.lang.Class`，以及`java.lang.reflect`中的`Method`、`Field`、`Constructor`等等classes。

## Class类

对于一个字节码文件.class，虽然表面上我们对该字节码文件一无所知，但该文件本身却记录了许多信息。Java在将.class字节码文件载入时，JVM将产生一个`java.lang.Class`对象代表该.class字节码文件，从该Class对象中可以获得类的许多基本信息，这就是反射机制。所以要想完成反射操作，就必须首先认识Class类。

反射机制所需的类主要有`java.lang`包中的`Class`类和`java.lang.reflet`包中的`Constructor`类、`Field`类、`Method`类和`Parameter`类。`Class`类是一个比较特殊的类，它是反射机制的基础，`Class`类的对象表示正在运行的Java程序中的类或接口，也就是任何一个类被加载时，即将类的.class文件（字节码文件）读入内存的同时，都自动为之创建一个java.lang.Class对象。Class类没有公共构造方法，其对象是JVM在加载类时通过调用类加载器中的defineClass()方法创建的，因此不能显式地创建一个Class对象。通过这个Class对象，才可以获得该对象的其他信息。下表列出了Class类的一些常用方法：

| 常用方法 | 功能说明 |
| --- | --- |
| public Package getPackage() | 获取Class对象所对应类的包 |
| public static Class<?> forName(String className) | 获取指定名称的类或接口的Class对象 |
| public String getName() | 获取类对应的“包.类名”形式的全名 |
| public String getSimpleName() | 获取类对应的“类名”形式的简单名称 |
| public native Class<? super T> getSuperclass() | 获取对应类的父类的Class对象 |
| public Class<?>[] getInterfaces() | 获取对应类所实现的所有**直接接口** |
| public Annotation[] getAnnotations() | 获取该程序元素上的所有注解 |
| public Annotation[] getDeclaredAnnotations() | 获取该程序元素上的**直接注解** |
| public Constructor<T> getConstructor(Class<?>... parameterTypes) | 获取对应类的指定参数列表的public构造方法 |
| public Constructor<?>[] getConstructors() | 获取对应类的所有public构造方法 |
| public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes) | 获取对应类的指定参数列表的所有构造方法，和访问权限无关 |
| public Constructor<?>[] getDeclaredConstructors() | 获取对应类的所有构造方法，和访问权限无关 |
| public Field getField(String name) | 获取对应类名称为name的public成员变量 |
| public Field[] getFields() | 获取对应类的所有public成员变量 |
| public Field getDeclaredField(String name) | 获取对应类名称为name的成员变量，和访问权限无关 |
| public Field[] getDeclaredFields() | 获取对应类的所有成员变量，和访问权限无关 |
| public Method getMethod(String name, Class<?>... parameterTypes) | 获取名称为name，且指定参数的public方法 |
| public Method[] getMethods() | 获取所有的public方法 |
| public Method getDeclaredMethod(String name, Class<?>... parameterTypes) | 获取名称为name，且指定参数的方法，和访问权限无关 |
| public Method[] getDeclaredMethods() | 获取所有的方法，和访问权限无关 |


> 说明：
> 1. 通过`getFields()`和`getMethods()`方法获得权限为public成员变量和成员方法时，还包括从父类继承得到的成员变量和成员方法；
> 2. 而通过`getDeclaredFields()`和`getDeclaredMethods()`方法只是获得在本类中定义的所有成员变量和成员方法。
> 3. 具体可参考JDK源码中的javadoc，如下所示：

```java
/**
 * Returns an array containing {@code Method} objects reflecting all the
 * public methods of the class or interface represented by this {@code
 * Class} object, including those declared by the class or interface and
 * those inherited from superclasses and superinterfaces.
 */
@CallerSensitive
public Method[] getMethods() throws SecurityException { }

/**
 * Returns an array containing {@code Method} objects reflecting all the
 * declared methods of the class or interface represented by this {@code
 * Class} object, including public, protected, default (package)
 * access, and private methods, but excluding inherited methods.
 */
@CallerSensitive
public Method[] getDeclaredMethods() throws SecurityException { }
```

每个类被加载之后，系统都会为该类生成一个对应的Class对象，通过Class对象就可以访问到JVM中该类的信息，一旦类被加载到JVM中，同一个类将不会被再次载入。被载入JVM的类都有一个唯一标识就是该类的全名，即包括包名和类名。在Java中，程序获得Class对象有如下3种方式：

1. 使用Class类的静态方法`forName(String className)`，其中参数className表示所需类的全名。如`Class cObj=Class.forName("java.lang.String");`。另外，`forName()`方法声明抛出ClassNotFoundException异常，因此调用该方法时必须捕获或抛出该异常。
2. 用类名调用该类的class属性来获得该类对应的Class对象，即`类名.class`。如`Class cObj=Cylinder.class;`将返回Cylinder类所对应的Class对象赋给cObj变量。
3. 用对象调用`getClass()`方法来获得该类对应的Class对象，即`对象.getClass()`。该方法是Object类中的一个方法，因此所有对象调用该方法都可以返回所属类对应的Class对象。如例8.8中的语句`Person per=new Person("张三"）;`，可以通过以下语句返回该类的Class对象：`Class cObj=per.getClass();`

通过类的class属性获得该类所对应的Class对象，会使代码更安全，程序性能更好。因此，大部分情况下建议使用第二种方式。但如果只获得一个字符串，例如获得String类对应的Class对象，则不能使用String.class方式，而是使用`Class.forName("java.lang.String")`。

注意：如果要想获得基本数据类型的Class对象，可以使用对应的打包类加上.TYPE。例如：`Integer.TYPE`可获得int的Class对象，但要获得`Integer.class`的Class对象，则必须使用`Integer.class`。在获得Class对象后，就可以使用表13.4中的方法来取得Class对象的基本信息。

## 反射包reflet中的常用类

反射机制中除了上面介绍的java.lang包中的Class类之外，还需要java.lang.reflet包中的Constructor类、Method类、Field类和Parameter类。Java 8以后在java.lang.reflect包中新增了一个Executable抽象类，该类对象代表可执行的类成员。Executable抽象类派生了Constructor和Method两个子类。

#### Executable

java.lang.reflect.Executable类提供了大量方法用来获取参数、修饰符或注解等信息，其常用方法如下表所示。

| 常用方法 | 功能说明 |
| --- | --- |
| public Parameter[] getParameters() | 返回所有形参 |
| public int getParameterCount() | 返回参数的个数 |
| public abstract Class<?>[] getParameterTypes() | 按照声明顺序以Class数组的形式返回各参数的类型 |
| public abstract int getModifiers() | 返回整数表示的修饰符，如：public、protected、private、final、static、abstract等关键字所对应的常量 |
| public boolean isVarArgs() | 是否包含数量可变的参数 |

getModifiers()方法返回的是以整数表示的修饰符。此时引入Modifier类，通过调用Modifier.toString(int mod)方法返回修饰符常量所应的字符串。

#### Constructor

java.lang.reflect.Constructor类是java.lang.reflect.Executable类的直接子类，用于表示类的构造方法。通过Class对象的getConstructors()方法可以获得当前运行时类的构造方法。java.lang.reflect.Constructor类的常用方法如下表所示。

| 常用方法 | 功能说明 |
| --- | --- |
| public String getName() | 获取构造方法的名称 |
| public T newInstance(Object ... initargs) | 通过指定参数列表，创建一个当前类对象。如果未传入参数，则表示采用默认无参的构造方法进行创建。 |
| public void setAccessible(boolean flag) throws SecurityException | 如果构造方法的访问权限为private或protected，默认不允许通过反射使用newInstance()方法创建对象。如果先执行该方法，并将入口参数设置为true，则允许创建。 |

#### Method

java.lang.reflect.Method类是java.lang.reflect.Executable类的直接子类，用于封装成员方法的信息，调用Class对象的getMethod()方法或getMethods()方法可以获得当前运行时类的指定方法或所有方法。下表是java.lang.reflect.Method类的常用方法。

| 常用方法 | 功能说明 |
| --- | --- |
| public String getName() | 获取方法的名称 |
| public Class<?> getReturnType() | 以Class对象的形式返回当前方法的返回值类型 |
| public Object invoke(Object obj, Object... args) | 通过指定参数列表执行指定对象obj中的该方法 |

#### Field

java.lang.reflect.Field类用于封装成员变量信息，调用Class对象的getField()方法或getFields()可以获得当前运行时类的指定成员变量或所有成员变量。java.lang.reflect.Field类的常用方法如下表所示。

| 常用方法 | 功能说明 |
| --- | --- |
| public String getName() | 获取字段的名称 |
| public Object get(Object obj) | 获取字段对应的值 |
| public void set(Object obj, Object value) | 设置字段的值 |
| public Class<?> getType() | 获取字段的类型 |

#### Parameter

java.lang.reflect.Parameter类是参数类，每个Parameter对象代表方法的一个参数。java.lang.reflect.Parameter类中提供了许多方法来获取参数信息，下表给出了java.lang.reflect.Parameter类的常用方法。

| 常用方法 | 功能说明 |
| --- | --- |
| public int getModifiers() | 获取参数的修饰符 |
| public String getName() | 获取参数的名称 |
| public Type getParameterizedType() | 获取带泛型的形参类型 |
| public Class<?> getType() | 获取形参类型 |
| public boolean isVarArgs() | 判断该参数是否为可变参数 |
| public boolean isNamePresent() | 判断.class文件中是否包含方法的形参名信息 |

> 说明：使用javac命令编译Java源文件时，默认生成.class字节码文件中不包含方法的形参名信息，因此调用getName（）方法不能得到参数的形参名，调用isNamePresent（）方法将返回false。如果希望javac命令编译Java源文件时保留形参信息，则需要为编译命令指定-parameters选项。

## 反射机制

#### 简介

反射这一概念最早由编程开发人员Smith在1982年提出，主要指应用程序访问、检测、修改自身状态与行为的能力。这一概念的提出立刻吸引了编程界的极大关注，各种研究工作随之展开，随之而来引发编程革命，出现了多种支持反射机制的面向对象语言。
在计算机科学领域，反射是指一类能够自我描述和自控制的应用。在Java编程语言中，反射是一种强有力的工具，是面向抽象编程一种实现方式，它能使代码语句更加灵活，极大提高代码的运行时装配能力。

#### 使用

```java
public class Person {

    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void info(String prof, int score) {
        System.out.println("我的专业：" + prof + "，我的成绩：" + score);
    }

    @Override
    public String toString() {
        return "姓名：" + this.name + "，年龄：" + age;
    }
}
```

```java
public class Main {

    public static void main(String[] args) throws Exception {
        Class<Person> pc = Person.class;

        Constructor<Person> constructor = pc.getConstructor(String.class, int.class);
        System.out.print("构造方法名：" + constructor.getName());
        Class<?>[] ptArray = constructor.getParameterTypes();
        for (int i = 0; i < ptArray.length; i++) {
            System.out.print("，参数：" + ptArray[i].getName());
        }

        System.out.print("\n\n");
        Field[] fields = pc.getDeclaredFields();
        for (Field field : fields) {
            int modifiers = field.getModifiers();
            System.out.print("成员变量修饰符：" + Modifier.toString(modifiers));
            Class<?> type = field.getType();
            System.out.print("，名称：" + field.getName());
            System.out.print("，类型：" + type.getName());
        }

        System.out.print("\n\n");
        Method[] methods = pc.getMethods();
        for (Method method : methods) {
            System.out.println("方法：" + method.getName());
            System.out.println("参数个数：" + method.getParameterCount());

            Parameter[] parameters = method.getParameters();
            for (int i = 0; i < parameters.length; i++) {
                Parameter parameter = parameters[i];
                //if (parameter.isNamePresent()) {
                System.out.println("----第" + (i + 1) + "个参数的信息");
                System.out.println("参数名：" + parameter.getName());
                System.out.println("参数类型：" + parameter.getType());
                System.out.println("泛型类型：" + parameter.getParameterizedType());
                System.out.println("-----------------");
                //}
            }
            System.out.println();
        }
    }
}
```

#### 意义

首先，反射机制极大的提高了程序的灵活性和扩展性，降低模块的耦合性，提高自身的适应能力。

其次，通过反射机制可以让程序创建和控制任何类的对象，无需提前硬编码目标类。

再次，使用反射机制能够在运行时构造一个类的对象、判断一个类所具有的成员变量和方法、调用一个对象的方法。

最后，反射机制是构建框架技术的基础所在，使用反射可以避免将代码写死在框架中。

正是反射有以上的特征，所以它能动态编译和创建对象，极大的激发了编程语言的灵活性，强化了多态的特性，进一步提升了面向对象编程的抽象能力，因而受到编程界的青睐。

## 特点

尽管反射机制带来了极大的灵活性及方便性，但反射也有缺点。反射机制的功能非常强大，但不能滥用。在能不使用反射完成时，尽量不要使用，原因有以下几点：

1、性能问题。
Java反射机制中包含了一些动态类型，所以Java虚拟机不能够对这些动态代码进行优化。因此，反射操作的效率要比正常操作效率低很多。我们应该避免在对性能要求很高的程序或经常被执行的代码中使用反射。而且，如何使用反射决定了性能的高低。如果它作为程序中较少运行的部分，性能将不会成为一个问题。

2、安全限制。
使用反射通常需要程序的运行没有安全方面的限制。如果一个程序对安全性提出要求，则最好不要使用反射。

3、程序健壮性。
反射允许代码执行一些通常不被允许的操作，所以使用反射有可能会导致意想不到的后果。反射代码破坏了Java程序结构的抽象性，所以当程序运行的平台发生变化的时候，由于抽象的逻辑结构不能被识别，代码产生的效果与之前会产生差异。

## 参考文档

+ https://baike.baidu.com/item/JAVA%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6/6015990
+ https://blog.csdn.net/losingcarryjie/article/details/78726264
+ https://www.jianshu.com/p/9be58ee20dee
+ https://blog.csdn.net/huangliniqng/article/details/88554510
+ https://baijiahao.baidu.com/s?id=1619604957177623053
+ https://blog.csdn.net/u014133119/article/details/80501677
+ https://www.cnblogs.com/adamjwh/p/9683705.html
+ https://www.cnblogs.com/myRichard/p/11742194.html
+ https://baijiahao.baidu.com/s?id=1645023991123759354&wfr=spider&for=pc
+ https://zhuanlan.zhihu.com/p/80519709
+ https://segmentfault.com/a/1190000015860183
+ https://baijiahao.baidu.com/s?id=1619748187138646880&wfr=spider&for=pc
+ https://www.sohu.com/a/225601602_414048
+ https://blog.51cto.com/4247649/2109128
+ https://zhidao.baidu.com/question/808071433707589172.html
+ https://www.zhihu.com/question/24304289?sort=created
+ http://www.imooc.com/article/1518
+ https://blog.csdn.net/ju_362204801/article/details/90578678