---
layout: post
title:  Java基础（02）异常处理
date:   2016-10-09 11:14:34 +0800
categories: Java基本技能
tag: java
---

* content
{:toc}

异常发现的理想时机是在编译阶段，但是并非所有的异常都能在此阶段发现，更多的是在运行期间。错误恢复机制是代码健壮性的有力保障，一个系统的健壮往往依赖于很多构件，构件的健壮关系着系统。Java使用异常处理来保证系统的健壮，同时异常处理也是唯一正式的错误报告机制。

## 异常是什么

C语言及其早期其他语言并没有异常机制，而是使用约定代替，并非语言本身的一部分。例如：通过方法返回一个特定值来标识某种错误。这往往会导致大量的关于特定值的注释。随着系统越来越大，这些特定值会越来越多，且是不可重复的。到最后，基本很难对这些错误进行处理，即使强硬地处理，也会导致“垃圾”代码越来越多。所以，很多C程序员会抱怨，C构建大型、健壮和可维护的程序，有一定难度。

基于此，C++对C进行兼容的同时，也加入了异常机制，而Java在此基础之上又进一步进行填充和完善。

“异常”（exception），英文翻译为意外的情况。当问题出现，我们或许不知道应该如何处理，但是也不应该忽略；这时，我们需要看看是否有人在别的地方，能够处理此问题。在当前环境中没有足够的信息来解决这个问题，需要提交到更高的环境。

异常处理还有一个好处，即能降低错误处理代码的复杂度。如果不使用异常，那么在许多地方都处理特定的错误。同时，它解耦了“正常执行过程中做什么事”和“出了问题怎么办”。

## 异常入门

异常是为了阻止当前方法或作用域继续执行。

抛出异常很简单，只需要使用关键字`throw`，后面跟一个异常对象即可。

```java
if (exp == null) {
    throw new NullPointerException();
}
```

异常对象同普通对象一样，都是在堆上分配内存，所以它也可以传入参数。基本的异常包括默认构造方法和接受字符串参数的构造方法。我们可以看出throw产生的结果类似于“返回”。可以简单看出是一种不同的返回机制，但是它又有一定的不同，它可以“返回多层”。异常可以跨越方法调用栈的多个层次。所有异常都继承于Throwable，它是异常类型的基类。

## 异常捕获

异常捕获是针对`监控区域`来说的。所谓监控区域，指的是可能产生异常的代码，紧跟着处理这些异常代码。

#### try

try作用于监控区域。

```java
try {
    // code that might create exceptions
}
```

#### 异常处理程序

当try试着捕获异常之后，异常需要得到一定的处理，这个处理的机制叫做“异常处理程序”。

```java
try {
    // code that migth create exceptions
} catch (FirstException e) {
    // handle and deal it
} catch (SecondException e) {
    // handle and deal it
}
```

## 自定义异常

java内置了一些异常类型，同时也提供异常的扩展。我们可以通过继承于`Exception`或`RuntimeException`来达到扩展的目的。

```java
class SimpleException extend Exception {
}
```

【注意】：因为异常类其实也是类的一种，所以继承的时候避免不了构造方法的局限——它只能继承默认的构造方法。如果要加入额外的构造方法，需要自行声明。

异常最重要的是类名，它表示这个异常类的类型，除非异常类本身需要掺带更多的参数，例如：错误码、错误信息、错误队列等等。

## 异常声明

异常除了可以try之外，在方法层级还可以进行声明。在调用这些声明了异常的方法时，如果异常类型为检查异常，则需要进行处理，或者在调用方法也进行声明。异常声明使用关键字`throws`。

```java
void function1() throws FirstException, SecondException {
    // code here
}
```

如果不声明的话，就说明这个方法不会抛出异常（除了RuntimeException）。

```java
void function2() {
    // code here
}
```

## 异常捕获

`Exception`是异常类的基类，所以在使用`catch`关键字进行捕获的时候，可以捕获这个基类。这样就会把这个Exception以及从它继承的子类都捕获到。
`Exception`本身并不包含很多实用的方法，它更多的是一些构造方法的声明。而我们要使用方法，需要从它上一级的基类`Throwable`来看。

1. String getMessage() 获取message信息
1. String getLocalizedMessage() 获取详细信息，或者叫做本地语言标识的详细信息，它往往比getMessage()信息更多
1. String toString() 获取异常的描述，钥匙有详细信息的话，也会包含在内，它比getLocalizedMessage()包含的信息更多
1. printStackTrace() 以标准错误流打印调用栈轨迹
1. void printStackTrace(PrintStream s) 以指定打印流打印
1. void printStackTrace(PrintStreamOrWriter s) 以指定打印流或writer打印
1. void printStackTrace(PrintWriter s) 以writer打印
1. Throwable fillInStackTrace() 用于在Throwable对象内部记录栈帧的当前状态
1. StackTraceElement[] getStackTrace() 获取栈帧数组

#### 栈轨迹

栈轨迹可以通过`getStackTrace()`和`printStackTrace()`来操作。所谓栈轨迹，指的是从出现异常到异常处理的方法栈顺序，最靠近处理程序的栈顺序最先。

#### 异常重抛

异常捕获之后，或许只是记录一下日志，还是需要上层进行处理，这时不用new一个异常对象，直接抛出即可。

```java
catch(FirstException e) {
    System.out.println("exception was thrown");
    throw e;
}
```

异常重抛带来了一定的问题，我不能看到重新抛出异常的位置。`printStackTrace()`将只打印原异常的调用栈信息。如果要想更新抛出点这个信息，可以手动调用`fillInStackTrace()`，这个方法返回Throwable对象，我们再将Throwable对象重新抛出即可。

```java
public class ThrowableTest {

    @Test
    public void test() {
        try {
            g();
        } catch (Exception e) {
            e.printStackTrace();
        }

        try {
            gg();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    void f() throws Exception {
        throw new Exception("first");
    }

    void g() throws Exception {
        try {
            f();
        } catch (Exception e) {
            throw e;
        }
    }

    void ff() throws Exception {
        throw new Exception("second");
    }

    void gg() throws Exception {
        try {
            ff();
        } catch (Exception e) {
            throw (Exception) e.fillInStackTrace();
        }
    }
}
```

输出结果如下：

```
java.lang.Exception: first
    at com.learn.spring.throwable.ThrowableTest.f(ThrowableTest.java:26)
    at com.learn.spring.throwable.ThrowableTest.g(ThrowableTest.java:31)
    at com.learn.spring.throwable.ThrowableTest.test(ThrowableTest.java:13)
java.lang.Exception: second
    at com.learn.spring.throwable.ThrowableTest.gg(ThrowableTest.java:45)
    at com.learn.spring.throwable.ThrowableTest.test(ThrowableTest.java:19)
```

这也证实了，fillInStackTrace()就变成了异常的起点了，其实是和`throw new`一样的效果。

#### 异常链

有时我们想要在捕获一个异常之后，抛出另外一个异常，且保留原异常的信息，这种要求被称做`异常链`。异常链从JDK1.4之后就开始支持了。Throwable的构造方法有能接受cause对象（Throwable类型）作为参数，这个cause对象其实就是原始异常，这样就能把原始异常传递给新异常了。即使捕获异常之后，再行抛出新异常，也能通过异常链找到异常最初发生的位置。

我们发现在Throwable的内置异常子类中，只有极少的异常类带有cause参数。如果要把其他类型的异常也链接起来，应该使用initCause()方法，而非构造方法。

```java
class MyDefException extends Exception {
}

class Main {
    void f(int value) {
        if (value < 0) {
            MyDefException e = new MyDefException();
            e.initCause(new NullPointerException());
        }
    }
    public static void main(String[] args) {
        new Main().f(-5);
    }
}
```

## Java异常分类

异常的继承层次中，我们比较关心的有4个。`Throwable`、`Error`、`Exception`和`RuntimeException`。

```
Throwable -> Error/Exception
Exception -> RuntimeException
```

其中Error是我们不必关心的。Throwable作为异常基类，它的大多数方法是我们实际需要用到的。Exception是检查性异常的基类，需要我们在程序中强制处理或重新抛出。RuntimeException是运行时异常，我们不必再程序中进行显式捕获或声明，只需要在同一的地方进行异常处理即可。

## finally关键字

finally是在异常处理程序中使用的，它的作用是不管是否抛出异常，最后都需要执行`finally`块中的代码。

```java
try {
    // code here
} catch(Exception e) {
    // handle the exception
} finally {
    // do it no matter the exception throw
}
```

finally的作用主要用于进行资源的关闭，比如：文件句柄、数据库连接池等。

由于异常和异常处理都可以使用关键字`return`进行正常返回。那么这时finally块中的代码是否也会执行呢？答案是“yes”。至于测试代码，需要大家自行填充和脑补。

#### finally和异常丢失

首先看一种现象。
```java
public class ThrowableTest {

    @Test
    public void test() {
        try {
            deal();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    void deal() throws Exception {
        try {
            demo1();
        } finally {
            throw new DisposeException();
        }
    }

    void demo1() throws VeryImportantException {
        throw new VeryImportantException();
    }

    static class VeryImportantException extends Exception {}

    static class DisposeException extends Exception {}
}
```

打印结果如下：

```
com.learn.spring.throwable.ThrowableTest$DisposeException
    at com.learn.spring.throwable.ThrowableTest.deal(ThrowableTest.java:23)
    at com.learn.spring.throwable.ThrowableTest.test(ThrowableTest.java:13)
```

我们看出异常`VeryImportantException`居然神奇地消失了！它已经被finally中重新抛出的异常所取代，这个问题非常的严重，针对这种情况，目前没有比较好的解决办法，但是使用bug发现工具倒是能很容易地发现。

第二个现象源于finally中使用return子句。

```java
public class ThrowableTest {

    @Test
    public void test() {
        try {
            deal();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    void deal() throws Exception {
        try {
            demo1();
        } finally {
            return;
        }
    }

    void demo1() throws VeryImportantException {
        throw new VeryImportantException();
    }

    static class VeryImportantException extends Exception {}
}
```

这种情况更为恶劣，它完全不抛出异常了！

## 继承体系中的异常抛出

在继承关系中，基类的方法如果抛出检查性异常A，那么子类方法抛出的异常也必须是异常A或异常A的子类，这一点尤为重要！如果子类抛出的检查性异常并不是在基类中声明的，那么就会出现不可控的情况。

```java
class AException extends Exception {}
class BException extends Exception {}
class CException extends AEXception {}

interface Service {
    void f() throws AException;
}

class ServiceImpl implements Service {
    void f() throws AException {} // right
    void f() throws BException {} // wrong
    void f() throws CException {} // right
}
```

## 异常匹配顺序

异常抛出之后，异常处理程序会按照代码的书写顺序找出“最近”的处理程序。找到匹配的处理程序之后，就认为得到了处理，然后就不再继续查找了。这个限制在高版本的jdk中已经有编译期检查了。

```java
@Test
public void test() {
    try {
        deal();
    } catch (Exception e) {
        e.printStackTrace();
    } catch (RuntimeException e) {
    }
}
```

这段代码在jdk1.5是完全没有问题的，但是在jdk1.7及之后的版本就会出现编译时错误。

## 总结

异常处理的一个重要准则是“只有在你知道如何处理的情况下才捕获异常”。异常处理的目标就是讲错误处理的代码和错误发生的地点解耦，从而使用户更专注于自己要完成的事情，至于异常如何处理，则在另一段代码中完成。

异常是java程序不可或缺的部分，如果不了解它们，任务的完成会比较low，或者不能完成。而我们在使用异常的时候，需要更多地将它看成是一种工具，使得运行时报出的错误并不是致命的。






