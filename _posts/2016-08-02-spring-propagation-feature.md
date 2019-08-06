---
layout: post
title:  Spring事务管理的Propagation特性
date:   2016-08-02 09:29:00 +0800
categories: SpringAOP
tag: Spring
---

* content
{:toc}

Spring事务管理中有一个很重要的特性“Propagation”，事务传播特性。
故名思意，它的主要目的就是配置当前需要调用的方法，与当前方法是否有事务之间的关系。特别注意的是，两个方法不能在同一个服务类中。
看上去有点儿抽象，不过，这也是为什么我想要写这篇博客的原因。看了后面的例子，大家应该就明白了。

## Propagation枚举值

|编号|枚举值|枚举说明|
|-----|-----|-----|
|1|REQUIRED（默认值）|在有事务状态下执行，如果当前没有事务，则创建新的事务|
|2|SUPPORTS|如果当前有事务，则在事务状态下执行；如果当前没有事务，则在无事务状态下执行|
|3|MANDATORY|必须在有事务状态下执行，如果当前没有事务，则抛出异常IllegalTransactionStateException|
|4|REQUIRES_NEW|创建新的事务并执行；如果当前已有事务，则将当前事务挂起|
|5|NOT_SUPPORTED|在无事务状态下执行；如果当前已有事务，则抛出异常IllegalTransactionStateException|
|6|NEVER|在无事务状态下执行；如果当前已有事务，则抛出异常IllegalTransactionStateException|
|7|NESTED|在有事务状态下执行；如果当前有事务，则在当前事务执行，且如果内层方法抛出异常，不影响外层事务；如果当前无事务，则不生效|

## 不能在同一个服务类中

因为在同一个类中，spring并不会知道它是一个切面方法，还是普通的类方法。

```java
class ServiceA {
    
    void doSomething() {
        methodA();
        methodB();
    }
    
    @Transactional // 这个注释在在本类调用时不起作用
    void methodA() {
    }
    
    @Transactional // 这个注释在在本类调用时不起作用
    void methodB() {
    }
}
```

不过在spring中，我们可以使用“自注入”的变通方式来实现切面拦截。这种方式在spring4.3.0以前版本可以这样实现：

```java
class ServiceA {
    @Autowired
    private ApplicationContext applicationContext;

    private ServiceA self;
    
    @PostConstruct
    private void init() {
        self = applicationContext.getBean(ServiceA.class);
    }
}
```

在spring4.3.0之后的版本，可以直接使用@Autowired来注入即可。

## REQUIRED与REQUIRED_NEW

上面描述的6种propagation属性配置中，比较难以理解，并且容易在transaction设计时出现问题的是REQUIRED和REQURED_NEW这两者的区别。
当程序在某些情况下抛出异常时，如果对于这两者不够了解，就可能很难发现而且解决问题。

下面我们给出三个场景进行分析：

#### 场景一

ServiceA.Java:

```java
public class ServiceA {

    @Transactional
    public void callB() {
        serviceB.doSomething();
    }
}
```

ServiceB.java

```java
public class ServiceB {

    @Transactional
    public void doSomething() {
        throw new RuntimeException("B throw exception");
    }
}
```

这种情况下，我们只需要在调用ServiceA.callB时捕获ServiceB中抛出的运行时异常，则transaction就会正常的rollback。


#### 场景二

在保持场景一中ServiceB不变，在ServiceA中调用ServiceB的doSomething时去捕获这个异常，如下：

```java
public class ServiceA {

    @Transactional
    public void callB() {
        try {
            serviceB.doSomething();
        } catch (RuntimeException e) {
            System.err.println(e.getMessage());
        }
    }
}
```

这个时候，我们再调用ServiceA的callB。程序会抛出org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only这样一个异常信息。原因是什么呢？

因为在ServiceA和ServiceB中的@Transactional propagation都采用的默认值：REQUIRE_ID。根据我们前面讲过的REQUIRED特性，当ServiceA调用ServiceB的时候，他们是处于同一个transaction中。

当ServiceB中抛出了一个异常以后，ServiceB会把当前的transaction标记为需要rollback。但是ServiceA中捕获了这个异常，并进行了处理，认为当前transaction应该正常commit。此时就出现了前后不一致，也就是因为这样，抛出了前面的UnexpectedRollbackException。

#### 场景三

在保持场景二中ServiceA不变，修改ServiceB中方法的propagation配置为REQUIRES_NEW，如下：

```java
public class ServiceB {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void doSomething() {
        throw new RuntimeException("B throw exception");
    }
}
```

此时，程序可以正常的退出了，也没有抛出UnexpectedRollbackException。原因是因为当ServiceA调用ServiceB时，serviceB的doSomething是在一个新的transaction中执行的。

所以，当doSomething抛出异常以后，仅仅是把新创建的transaction rollback了，而不会影响到ServiceA的transaction。ServiceA就可以正常的进行commit。

## 嵌套事务NESTED

NESTED是基于JDBC3.0（及以上）的保存点概念来实现的，保存点主要用于事务的部分回滚功能。其实现的伪代码如下：

```java
// ServiceA
class ServiceA {
    
    void doSomething() {
        Connection connection = null;
        try {
            connection = getConnection();
            connection.setAutoCommit(false);
            
            savePoint = connection.setSavePoint();
            try {
                serviceB.methodB();
            } catch(RuntimeException e) {
                connection.rollback(savePoint);
            } finally {
                // release resource.
            }
            
            methodA();
            connection.commit();
        } catch(RuntimeException e) {
            connection.rollback();
        }
    }
    
    void methodA() {
        System.out.println("methodA is invoked");
    }
}

class ServiceB {

    void methodB() {
        System.out.println("methodB is invoked");
    }
}
```

methodB调用失败的情况下，methodA仍然可能调用成功。
当methodB方法调用之前，调用setSavepoint方法，保存当前的状态到savepoint。
如果methodB方法调用失败，则恢复到之前保存的状态。
但是需要注意的是，这时的事务并没有进行提交，如果后续的代码(doSomeThingB()方法)调用失败，则回滚包括methodB方法的所有操作。

**【注】：嵌套事务的一个非常重要的特性就是内层事务依赖外层事务。当外层事务失败时，会回滚内层事务所做的动作。而内层事务操作失败并不会引起外层事务的回滚。**

## NESTED和REQUIRES_NEW的区别

1. 他们比较相像，看起来都像是嵌套事务。如果内层不存在一个活动事务时，都会开启一个新的事务。
1. 使用REQUIRES_NEW时，内层事务与外层事务就像两个独立的事务一样，一旦内层事务进行了提交，外层事务不能对其进行回滚，两个事务互不影响。两个事务不是真正的嵌套事务。
1. 使用NESTED时，外层事务的回滚可以引起内层事务的回滚，而内层事务的异常并不会导致外层事务的回滚，它是一个真正的嵌套事务。
1. REQUIRED应该是我们最常用的事务传播行为，它能满足我们绝大多数事务需求。

## 参考资料

1. http://www.iteye.com/topic/78674
1. http://www.iteye.com/topic/35907
1. http://blog.csdn.net/kiwi_coder/article/details/20214939