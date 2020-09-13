---
layout: post
title:  Java异常（1）异常机制
date:   2020-04-27 23:25:59 +0800
categories: Java
tags: Java
---

* content
{:toc}

## 简介

#### 什么是异常

所谓异常，就是程序在运行过程中，由于外部问题导致程序不正常的事件，发生的异常会中断程序的运行。在Java中，异常本身是一个对象，产生异常就是产生了一个异常对象。

#### 不使用异常的问题

假如我们要计算两个整数相除的结果，在不使用异常时，程序代码大概是这样的：

```java
public void divide() {
    System.out.println("请输入一个被除数：");
    Scanner scanner = new Scanner(System.in);
    if (scanner.hasNextInt()) {
        int first = scanner.nextInt();
        System.out.println("请输入一个除数：");
        if (scanner.hasNextInt()) {
            int second = scanner.nextInt();
            if (second == 0) {
                System.out.println("除数不能为0！");
            } else {
                int dividedNumber = first / second;
                System.out.println("dividedNumber = " + dividedNumber);
            }
        } else {
            System.out.println("除数输入不合法！");
        }
    } else {
        // 在控制台有可能输入字符串
        System.out.println("被除数输入不合法！");
    }
}
```

根据上面的例子，我们可以看出，一个极其简单的业务需求，代码也会写得很长。因为要考虑的问题很多，所以代码中会出现大量的条件判断，这就导致写代码和读代码都很累。一旦出现问题，程序就会中断，不会再执行后面的代码。因此，Java编程语言使用异常处理机制为程序提供异常处理的能力，这种能力可以简化我们的代码，同时也便于别人更好地理解和阅读我们的代码。

#### Java异常主要流程

在Java中，异常处理的过程如下：

```
预先设置好处理异常的方法
↓
程序运行
↓
异常
↓
处理异常
↓
处理完毕，程序继续运行
```

## 异常分类

在Java中，所有异常都有一个共同的父类`Throwable`（英文翻译为可抛出）。`Throwable`表示代码中可用异常传播机制传递的任何问题的共性。

`Throwable`有两个重要的子类，即`Exception`（异常）和`Error`（错误）。两者都是Java异常处理的重要子类，各自包含大量的子类。

#### 分类图

```
Throwable
    - Error
        - VirtualMachineError
        - LinkageError
            - NoClassDefFoundError
            - ...
        - ...
    - Exception
        - IOException
        - SQLException
        - ...
        - RuntimeException
            - NullPointerException
            - ...
```

#### Error

Error，是程序无法处理的错误，表示应用程序运行中出现了比较严重的问题。大多数错误与代码编写者执行的操作无关，而表示代码在运行时，JVM（Java虚拟机）出现的问题。例如，Java虚拟机运行错误（VirtualMachineError），当JVM不再有继续执行操作所需的内存资源时，将出现OutOfMemoryError。这些异常发生时，Java虚拟机（JVM）一般会选择终止线程。

这些错误表示故障发生于虚拟机自身、或者发生在虚拟机试图执行应用时，如Java虚拟机运行错误（VirtualMachineError）、类定义错误（NoClassDefFoundError）等。这些错误是不可查的，因为它们在应用程序的控制和处理能力之外，而且绝大多数是程序运行时不允许出现的状况。对于设计合理的应用程序来说，即使确实发生了错误，本质上也不应该试图去处理它所引起的异常状况。在Java中，错误通过Error的子类描述。

#### Exception

Exception，是程序本身可以处理的异常。Exception类有一个重要的子类，即：RuntimeException。RuntimeException类及其子类表示`JVM常用操作`引发的错误。例如，如果试图使用空值对象引用、除数为零或数组越界，则分别会引发运行时异常（NullPointerException、ArithmeticException和ArrayIndexOutOfBoundException）。

> 注意：异常和错误的区别在于，异常能被程序本身处理，错误则是无法处理的。通常，Java的异常（包括Exception和Error）分为可查的异常（checked exceptions）和不可查的异常（unchecked exceptions）。

1. 可查异常（编译器要求必须处理的异常）：正确的程序在运行中，很容易出现的、可以理解的异常状况。可查异常虽然是异常状况，但在一定程度上它的发生是可以预测的，而且一旦发生这种异常状况，就必须采取某种方式进行处理。除了RuntimeException及其子类以外，其他的Exception类及其子类都属于可查异常。这种异常的特点是Java编译器会检查它。也就是说，当程序中可能出现这类异常时，要么用`try-catch语句`捕获它，要么用`throws子句`声明抛出它，否则编译不会通过。
2. 不可查异常（编译器不要求强制处理的异常）：包括运行时异常（RuntimeException及其子类）和错误（Error及其子类）。

Exception这种异常又分为两个大类，运行时异常和非运行时异常（编译期异常）。程序中应当尽可能去处理这些异常。

1. 运行时异常：都是RuntimeException类及其子类异常，如NullPointerException（空指针异常）、IndexOutOfBoundsException（数组下标越界异常）等。这些异常是不可查异常，程序中可以选择捕获处理，也可以不处理。这些异常一般是由程序逻辑错误引起的，程序应该从逻辑角度尽可能避免这类异常的发生。运行时异常的特点是Java编译器不会检查它。也就是说，当程序中可能出现这类异常时，即使没有用`try-catch语句`捕获它，也没有用`throws子句`声明抛出它，也能编译通过。
2. 非运行时异常 （编译异常）：是除了RuntimeException类及其子类之外的异常，类型上都属于Exception类及其子类。从程序语法角度来说，是必须要进行处理的异常。如果不处理，程序就不能编译通过。如IOException、SQLException等，以及用户自定义的Exception异常。不过，一般情况下不会自定义非运行时异常。

## Java异常机制

上面的描述应该将Java异常解释清楚了，接下来我们看看在Java中是如何处理异常的。

#### try-catch

语法如下所示：

```java
try {
    // 有可能出现异常的代码段1
    // 有可能出现异常的代码段2
} catch (异常类型1 e) {
    // 处理异常的代码段3
} catch (异常类型2 e) {
    // 处理异常的代码段4
}
```

demo例子如下所示：

```java
void divide(int first, int second) {
    try {
        System.out.println("开始执行！");
        int third = first / second;
        System.out.println("正常执行完毕！");
    } catch (Exception e) {
        System.out.println("这儿处理异常！");
    }
    System.out.println("程序执行完毕！");
}
```

`try-catch语句`执行过程中总共可能会出现3种可能。

第一种、没有遇到异常，即正常执行。

```java
-3->|public void method() {
    |   try {
-1->|        // 代码段1（此处不会产生异常）
    |    } catch (异常类型 e) {
    |        // 对异常进行处理的代码段2
    |    }
-2->|    // 代码段3
    |}
```

第二种、匹配到异常。当`try{}`中的代码遇到异常时，会与`catch()`中括号里的异常进行比对，如果遇到的异常属于`catch`的异常（类型相同或父类），就会执行`catch块`中的代码，然后执行`try-catch块`后面的代码

```java
-5->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 产生异常的代码段2
    |        // 代码段3
    |    } catch (异常类型 e) {
-3->|        // 对异常进行处理的代码段4
    |    }
-4->|    // 代码段5
    |}
```

第三种、异常匹配不成功。

```java
-3->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 产生异常的代码段2
    |        // 代码段3
    |    } catch (异常类型 e) {
    |        // 对异常进行处理的代码段4
    |    }
    |    // 代码段5
    |}
```

#### try-catch-finally

`try{}代码块`用于执行可能存在异常的代码，`catch{}代码块`用于捕获并处理异常。而`finally{}代码块`用于回收资源（关闭文件、关闭数据库、关闭管道等）。

当`try{}代码块`得到执行的情况下，`finally{}代码块`通常情况下必然会得以执行，不管是否出现异常。`finally{}代码块`不执行的情况（一种情况就是`System.exit(0)`，即JVM正常退出。）比较特殊，在本文最后，我们将阐述其原因。

`try-catch-finally`执行总共可能出现以下6种情况。

1. catch块没有return语句，且没有遇到异常

```java
-6->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 代码段2
-3->|        // 代码段3
    |    } catch (异常类型 e) {
    |        // 对异常进行处理的代码段4
    |    } finally {
-4->|        // 代码段5
    |    }
-5->|    // 代码段6
    |}
```

2. catch块没有return语句，且遇到异常并匹配到异常

```java
-6->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 产生异常的代码段2
    |        // 代码段3
    |    } catch (异常类型 e) {
-3->|        // 对异常进行处理的代码段4
    |    } finally {
-4->|        // 代码段5
    |    }
-5->|    // 代码段6
    |}
```

3. catch块没有return语句，且遇到异常却没有匹配到异常

```java
-4->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 产生异常的代码段2
    |        // 代码段3
    |    } catch (异常类型 e) {
    |        // 对异常进行处理的代码段4
    |    } finally {
-3->|        // 代码段5
    |    }
    |    // 代码段6
    |}
```

4. catch块有return语句（或者重新抛出异常），且没有遇到异常

```java
-5->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 代码段2
-3->|        // 代码段3
    |    } catch (异常类型 e) {
    |        // 对异常进行处理的代码段4
    |        return;
    |    } finally {
-4->|        // 代码段5
    |    }
    |    // 代码段6
    |}
```

5. catch块有return语句（或者重新抛出异常），且遇到异常并匹配到异常

```java
-6->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 产生异常的代码段2
    |        // 代码段3
    |    } catch (异常类型 e) {
-3->|        // 对异常进行处理的代码段4
-5->|        return;
    |    } finally {
-4->|        // 代码段5
    |    }
    |    // 代码段6
    |}
```

6. catch块有return语句（或者重新抛出异常），且遇到异常却没有匹配到异常

```java
-4->|public void method() {
    |    try {
-1->|        // 代码段1
-2->|        // 产生异常的代码段2
    |        // 代码段3
    |    } catch (异常类型 e) {
    |        // 对异常进行处理的代码段4
    |        return;
    |    } finally {
-3->|        // 代码段5
    |    }
    |    // 代码段6
    |}
```

#### finally全解析

通过上一小节的罗列，我们知道finally修饰的语句块必然会执行。但是，的确是这样的吗？答案是否定的，我们先来看一个简单的例子。

```java
// 清单1
public class Main {
    public static void main(String[] args) {
        System.out.println("return value of test(): " + test());
    }

    private static int test() {
        int i = 1;
        /*if (i == 1)
            return 0;*/
        System.out.println("the previous statement of try block");
        i = i / 0;

        try {
            System.out.println("try block");
            return i;
        } finally {
            System.out.println("finally block");
        }
    }
}
```

清单1执行结果如下：

```
the previous statement of try block
Exception in thread "main" java.lang.ArithmeticException: / by zero
	at com.juconcurrent.Main.test(Main.java:18)
	at com.juconcurrent.Main.main(Main.java:10)
```

即使我们将清单1里面的注释代码放开，finally代码块也不会得到执行。

```
return value of test(): 0
```

以上两种情况，`finally{}语句块`都没有执行，说明什么问题呢？只有与 `finally{}语句块`相对应的`try{}语句块`得到执行的情况下，`finally{} 语句块`才会执行。以上两种情况，都是在`try{}语句块`之前返回（return）或者抛出异常，所以`finally{}语句块`没有执行。

那么，即使与`finally{}语句块`相对应的`try{}语句块`得到执行的情况下，`finally{}语句块`一定会执行吗？其实不然，我们看看下面这个例子。 

```java
// 清单2
public class Main {
    public static void main(String[] args) {
        System.out.println("return value of test(): " + test());
    }

    public static int test() {
        int i = 1;

        try {
            System.out.println("try block");
            System.exit(0);
            return i;
        } finally {
            System.out.println("finally block");
        }
    }
}
```

清单2执行结果如下：

```
try block
```

`finally{}语句块`仍然没有执行，为什么呢？因为我们在`try{}语句块`中执行了`System.exit(0);`语句，终止了Java虚拟机的运行。你有可能会说，在一般的Java应用中，基本上是不会调用这个`System.exit(0);`方法的。确实如此，如果我们不调用`System.exit(0);`这个方法，那么`finally{}语句块`就一定会执行吗？

答案是：不一定。当一个线程在执行`try{}语句块`或者`catch{}语句块`时被中断（interrupted）或者被终止（killed），与其相对应的`finally{}语句块`可能不会被执行。还有更极端的情况，就是当线程运行`try{}语句块`或者`catch{}语句块`时，突然死机或者断电，`finally{}语句块`肯定也不会执行的。可能有人认为死机、断电这些理由有些强词夺理，没有关系，我们只是为了说明这个问题。

为了更清晰地说明`finally{}语句块`的作用，我们参考[《The Java™ Tutorials》](https://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html)，以下为摘录内容：

> The finally block always executes when the try block exits. This ensures that the finally block is executed even if an unexpected exception occurs. But finally is useful for more than just exception handling — it allows the programmer to avoid having cleanup code accidentally bypassed by a return, continue, or break. Putting cleanup code in a finally block is always a good practice, even when no exceptions are anticipated.<br/>
> **Note:** If the JVM exits while the try or catch code is being executed, then the finally block may not execute. Likewise, if the thread executing the try or catch code is interrupted or killed, the finally block may not execute even though the application as a whole continues. 

当我们完全看懂了上面两段英文描述之后，我们就能明白`finally{}语句块`什么时候不会执行了。我们再将其翻译成中文，以便我们能更清晰地理解。

> finally块总是在try块退出时执行。这能确保finally块总是执行，即使一个非预期的异常发生。但是，finally除了异常发生时有用外，额外还允许开发者在使用`return`、`continue`或者`break`时，意外绕过资源释放过程。将资源释放的代码段放置在finally块中，永远是一个好的建议，即使没有异常出现。<br/>
> 【注意】：当try或catch块正在执行时，如果JVM虚拟机退出了，那么finally块可能不会执行。同样的，如果正在执行try或者catch块的线程被中断了或者被终止了，finally块也可能没有来得及执行，即使进程还在运行。

在排除了`finally{}语句块`不执行的情况后，`finally{}语句块`必然要保证得到执行。那么，我们提出以下进一步的疑问：

1. `try{}语句块`、`catch{}语句块`和`finally{}语句块`，他们的执行顺序是怎样的？
2. 如果`try{}语句块`或者`catch{}语句块`中有return语句，`finally{}语句块`究竟是在return前执行，还是return后执行呢？

接下来，我们看看[《The Java™ Programming Language, Fourth Edition》](http://www.acs.ase.ro/Media/Default/documents/java/ClaudiuVinte/books/ArnoldGoslingHolmes06.pdf#%5B%7B%22num%22%3A6033%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C0%2C281%2C0%5D)中关于try、catch和finally的描述：

> You catch exceptions by enclosing code in try blocks. The basic syntax for a TRy block is:

```java
try {
    statements
} catch (exception_type1 identifier1) {
    statements
} catch (exception_type2 identifier2) {
    statements...
} finally {
    statements
}
```

> where either at least one catch clause, or the finally clause, must be present. The body of the try statement is executed until either an exception is thrown or the body finishes successfully. If an exception is thrown, each catch clause is examined in turn, from first to last, to see whether the type of the exception object is assignable to the type declared in the catch. When an assignable catch clause is found, its block is executed with its identifier set to reference the exception object. No other catch clause will be executed. Any number of catch clauses, including zero, can be associated with a particular Try as long as each clause catches a different type of exception. If no appropriate catch is found, the exception percolates out of thetry statement into any outer try that might have a catch clause to handle it.

> If a finally clause is present with a try, its code is executed after all other processing in the try is complete. This happens no matter how completion was achieved, whether normally, through an exception, or through a control flow statement such as return or break.

上面这段文字的大体意思是说，不管`try{}语句块`是正常结束，还是异常结束，`finally{}语句块`总是保证要执行的。如果`try{}语句块`正常结束，那么当`try{}语句块`中的语句都执行完之后，再执行`finally{}语句块`。如果 `try{}语句块`中有控制转移语句（return、break或continue）的话，那么`finally{}语句块`是在控制转移语句之前执行，还是之后执行呢？根据上面的描述，我们很难看出结果。不过，在后面的讲解中，我们会分析这个问题。如果`try{}语句块`异常结束，应该先去相应的`catch{}语句块`做异常处理，然后执行`finally{}语句块`。同样的问题，如果`catch{}语句块`中包含控制转移语句呢？ `finally{}语句块`是在这些控制转移语句之前，还是之后执行呢？我们也会在后续讨论中提到。

其实，关于try、catch和finally的执行流程，是非常复杂的，如果各位感兴趣，可以深入去看《The Java™ Programming Language, Fourth Edition》中对于`try, catch, and finally`的描述。限于篇幅的原因，本文不做摘录。

我们再来看一个例子。

```java
// 清单3
public class Main {
    public static void main(String[] args) {
        try {
            System.out.println("try block");
            return;
        } finally {
            System.out.println("finally block");
        }
    }
}
```

执行结果如下。说明`finally{}语句块`是在return语句之前执行的。

```java
try block
finally block
```

接下来，我们再看一个更加深入的例子。

```java
// 清单4
public class Main {
    public static void main(String[] args) {
        System.out.println("return value of test() : " + test());
    }

    public static int test() {
        int i = 1;

        try {
            System.out.println("try block");
            i = 1 / 0;
            return 1;
        } catch (Exception e) {
            System.out.println("exception block");
            return 2;
        } finally {
            System.out.println("finally block");
        }
    }
}
```

执行结果如下。同样的，说明了`finally{}语句块`在`catch{}语句块`中的return语句之前执行的。

```
try block
exception block
finally block
return value of test() : 2
```

由此，我们可以大致得出一个结论：

> 其实，`finally{}语句块`是在`try{}`或者`catch{}`中的`return语句`之前执行的。<br/>
> 通俗地讲，`finally{}语句块`应该是在控制转移语句之前执行，控制转移语句除了`return`外，还有`break`和`continue`。另外，`throw语句`也属于控制转移语句。虽然`return`、`throw`、`break`和`continue`都是控制转移语句，但是它们之间是有区别的。其中，`return`和`throw`把程序的控制权转交给它们的调用者（invoker），而`break`和`continue`的控制权是在当前方法内进行转移。

在《The Java™ Programming Language, Fourth Edition》中，有关于此的一段说明描述：

> A finally clause can also be used to clean up for break, continue, and return, which is one reason you will sometimes see a try clause with no catch clauses. When any control transfer statement is executed, all relevant finally clauses are executed. There is no way to leave a try block without executing its finally clause.

那么，`finally{}语句块`的说明是否言尽于此了呢？其实不然，我们接着看两个例子：

```java
// 清单5
public class Main {
    public static void main(String[] args) {
        System.out.println("return value of getValue(): " + getValue());
    }

    public static int getValue() {
        try {
            return 0;
        } finally {
            return 1;
        }
    }
}
```

清单5执行结果如下：

```
return value of getValue(): 1
```

```java
// 清单6
public class Main {
    public static void main(String[] args) {
        System.out.println("return value of getValue(): " + getValue());
    }

    public static int getValue() {
        int i = 1;
        try {
            return i;
        } finally {
            i++;
        }
    }
}
```

清单6执行结果如下：

```
return value of getValue(): 1
```

我们分析一下清单5，最后执行的是`finally{}语句块`，所以返回结果为1，是正常的。而清单6，在`finally{}语句块`执行完成之后，变量i应该变成了2，再通过`try{}语句块`的return语句，预期的返回结果应该为2才对，但是实际的返回结果却是1。这就很让人懵圈了，这是为什么呢？

其实，这儿就涉及到Java虚拟机如何编译`finally`的问题了。在《The Java™ Virtual Machine Specification》的7.13小节有描述如何编译finally了。

实际上，Java虚拟机会把`finally{}语句块`作为`subroutine`（对于这个`subroutine`不知该如何翻译为好，干脆就不翻译了，免得产生歧义和误解。）直接插入到`try{}语句块`或者`catch{}语句块`的控制转移语句之前。但是，还有另外一个不可忽视的因素，那就是在执行`subroutine`（也就是`finally{}语句块`）之前，`try{}`或者`catch{}语句块`会保留其返回值到本地变量表（Local Variable Table）中。当`subroutine`执行完毕之后，再恢复保留的返回值到操作数栈中，然后通过`return`或者`throw语句`将其返回给该方法的调用者（invoker）。请注意，前面我们曾经提到过`return`、`throw`和`break`、`continue`的区别，对于这条规则（保留返回值），只适用于`return`和`throw语句`，不适用于`break`和`continue语句`，因为它们根本就没有返回值。

我们将清单6的代码编译之后的字节码文件分析一下。

```java
Compiled from "Main.java"
public class com.juconcurrent.Main {
  public com.juconcurrent.Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: new           #3                  // class java/lang/StringBuilder
       6: dup
       7: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      10: ldc           #5                  // String return value of getValue():
      12: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      15: invokestatic  #7                  // Method getValue:()I
      18: invokevirtual #8                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      21: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      24: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: return

  public static int getValue();
    Code:
       0: iconst_1
       1: istore_0
       2: iload_0
       3: istore_1                                  ##! 将原始值设置到本地变量表
       4: iinc          0, 1                        ##! 变量+1
       7: iload_1                                   ##! 将本地变量表的值取回来
       8: ireturn                                   ##! 将取回来的变量返回回去
       9: astore_2
      10: iinc          0, 1
      13: aload_2
      14: athrow
    Exception table:
       from    to  target type
           2     4     9   any
}
```

至于这些字节码命令分别是什么意思，如果读者感兴趣，可自行百度。

#### 声明异常

当我们在定义方法的时候，事先知道方法在调用时会出现异常，但不知道该如何处理。这时，可以在该方法上声明异常，表示该方法在调用过程中可能会出现异常，请调用者自行处理。

在java中，使用`throws`声明异常。一个方法可以声明多个异常，用英文逗号（,）分割，写法如下：

```java
public void test2() throws IOException, RuntimeException {
    // 有异常抛出的代码，但这儿没有处理。
}
```

> 注意：声明异常和方法重载没有任何关系。

> 注意：声明异常与方法重写有关系，如下：
> 1. 如果父类方法声明了异常（检查时或运行时），子类方法可以完全遵循父类异常，也可以不声明异常；
> 2. 如果父类方法没有声明异常，子类可以不声明异常，也可以声明RuntimeException，但不能声明Exception；
> 3. 如果父类声明了运行时异常，子类可以完全遵循父类异常，也可以不声明异常。

#### 抛出异常

当系统异常满足不了开发的需求时，开发者可以根据需要自行抛出异常。

`throw`用于手动抛出异常。

如果异常一直都没有处理（即：没有用`try-catch`等语句），那么将会重新抛给调用者，一直抛到`main()`函数处，如果在`main()`函数中也没有处理，而是继续在`main()`函数后抛出异常，这时候会抛给JVM处理，进程结束。例如：

```java
public class Test01 {

    public static void test1() throws IOException, RuntimeException {
        // 有异常抛出得代码，在此处没有处理，例如：throw new Exception("异常信息");
    }

    public static void test2() throws IOException, RuntimeException {
        test1(); // 调用有抛出异常的方法，但此处并没有处理
    }

    public static void main(String[] args) throws IOException, RuntimeException {
        test2(); // main()调用有抛出异常的方法，但此处并没有处理
    }
}
```

> 注意: 开发者根据需求，可以选择抛出检查时异常和运行时异常。

#### 自定义异常

当JDK中的异常类型不能满足程序的需要时，我们可以自定义异常类。自定义异常的步骤如下所示：

1. 确定异常的类型，是继承Excepion，还是RuntimeException；
2. 编写自定义异常类，并实现构造方法；
3. 在方法需要的地方手动声明和抛出异常。

```java
public class MyException extends Exception {

    public MyException() {
        super();
    }

    public MyException(String message) {
        super(message);
    }

    // 自定义异常中的方法，以满足自己的需求
    public void showInfo() {
        System.out.println("@Line:" + super.getMessage());
    }
}
```

> 注意：通常情况下，我们的自定义异常是运行时异常，即：直接或间接继承于`RuntimeException`。

## 常见异常

#### 检查型异常

+ IOException -- IO操作相关的异常
    + FileNotFoundException -- 文件不存在
    + SocketException -- socket异常，主要应用在网络IO
    + InterruptedIOException -- 中断的IO异常
+ SQLException -- SQL异常
+ InterruptedException -- 中断的异常，主要运用在线程
+ ReflectiveOperationException -- 反射操作异常
    + ClassNotFoundException -- 类没有找到
    + IllegalAccessException -- 非法访问
    + NoSuchFieldException -- 字段不存在
    + NoSuchMethodException -- 方法不存在

#### 运行时异常

+ NullPointerException -- 空指针异常
+ IndexOutOfBoundsException -- 数据越界
+ ArithmeticException -- 除数为0的情况
+ ClassCastException -- 类型强转异常
+ IllegalStateException // 非法状态异常
+ UnsupportedOperationException // 不支持的操作
+ IllegalArgumentException // 非法参数异常
+ ArrayStoreException // 操作数组时类型不一致

## 异常使用的技巧

#### 1. 异常处理不能代替简单的测试

捕获异常对资源有一定的要求。不要通过捕获异常来做判断。例如，在调用`stack.pop()`方法前先调用`stack.empty()`判断一下，而不是直接调用再捕获异常，最后对异常进行处理。即通过捕获异常的方式来说明stack是空的，这种方法不可取。

#### 2. 不要过分细化异常

不要每句语句都加一个异常捕获。

#### 3. 利用继承，让异常具有层次结构

1. 不要只抛出RuntimeException异常，应该寻找更加适当的子类或者创建自己的异常类。
2. 不要只捕获Throwable异常。

#### 4. 不要压制异常

参考《Java核心技术卷一 P492》

#### 5. 仔细思考，是返回一个错误标记，还是抛出异常？

例如，当栈为空时，`stack.pop()`是返回一个null，还是抛出一个异常？个人觉得，在出错的地方抛出一个`EmptyStackException`异常，要比在后面抛出一个`NullPointerExcepion`异常要更好，这样更能见名知意。

#### 6. 不要羞于传递异常

不能处理的异常要抛出到上层，方便上层进行更合理地处理。

## 总结

本文简单地对异常的使用方式和工作原理进行了说明阐述，也提到了使用异常需要注意的几点意见，算是对使用者的一个比较全面的讲解吧。

## 参考文档

+ https://www.sohu.com/a/338323114_99923293
+ https://www.jianshu.com/p/7a294669ee2b
+ https://cloud.tencent.com/developer/article/1398165
+ https://cloud.tencent.com/developer/article/1442626
+ https://www.jianshu.com/p/49d2c3975c56
+ https://www.cnblogs.com/fzz9/p/8980266.html
+ https://www.bbsmax.com/A/qVde2RNAzP/
+ https://www.jianshu.com/p/359b150e4cb2
+ https://blog.csdn.net/u010533180/article/details/53018450