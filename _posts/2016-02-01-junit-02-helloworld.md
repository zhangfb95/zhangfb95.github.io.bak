---
layout: post
title:  单元测试（2）helloworld
date:   2016-02-01 10:47:00 +0800
categories: Junit
tag: Junit
---

* content
{:toc}

# hello world

## 简介

编程入门的第一个例子，通常都是hello world。但是为了更完整地展示单元测试的优缺点，我们增加了额外的功能代码。

## 操作

首先，我们使用以下代码代替main方法。

```java
public class JunitTest {

    @Test
    public void test() throws Exception {
        System.out.println("test() invoked");
    }
}
```

其次，我们在JunitTest中增加多个单元测试，同时在每个单元测试执行前后加上额外的逻辑。

```java
public class JunitTest {

    @Before
    public void before() throws Exception {
        System.out.println("before() invoked");
    }

    @After
    public void after() throws Exception {
        System.out.println("after() invoked");
    }

    @Test
    public void test() throws Exception {
        System.out.println("test() invoked");
    }

    @Test
    public void test2() throws Exception {
        System.out.println("test2() invoked");
    }
}
```

在某些场景，我们可能需要每个单元测试访问相同的资源，比如测试类初始化、共享对象池等等。这时，我们可以使用@BeforeClass和@AfterClass来处理。**这儿需要注意一点，注解的方法必须是public static void的，且放在所有方法的最前面**

```java
public class JunitTest {
    
    @BeforeClass
    public static void beforeClass() throws Exception {
        System.out.println("beforeClass() invoked");
    }

    @AfterClass
    public static void afterClass() throws Exception {
        System.out.println("afterClass() invoked");
    }

    @Before
    public void before() throws Exception {
        System.out.println("before() invoked");
    }

    @After
    public void after() throws Exception {
        System.out.println("after() invoked");
    }

    @Test
    public void test() throws Exception {
        System.out.println("test() invoked");
    }

    @Test
    public void test2() throws Exception {
        System.out.println("test2() invoked");
    }
}
```

在某些场景，我们可能会忽略某些单元测试，这时使用可以使用@Ignore，在4.x中，我们也可以使用“假设”来实现这种方式。

```java
@Test
@Ignore("ignored function have not implemented")
public void testIgnored() {
    Assert.fail("ignored can't be invoked");
}
```
```java
@Test
public void testOneEqualsOne() {
    assumeThat('1', is('1'));
    System.out.println("1 == 1");
}

@Test
public void testOneNotEqualsTwo() {
    assumeThat('1', is('2'));
    System.out.println("1 == 2");
}
```

## 优缺点
凡事有利有弊，重在权衡。

我认为的优点：

 - 代替main方法，在一个类中main只能有一个，而单元测试方法却可以有多个；
 - 提高开发人员对代码的认知度，集成测试框架引入@Test、@Before、@After、@BeforeClass、@AfterClass、Ignored等特性和机制，让我们更全面、更快地知道代码的不足，也从功能上完整地分析自己的代码优势、不足和bug。
 - 减轻QA的压力，单元测试能提早暴露问题，在开发阶段就发现问题并修复，而不用延迟到测试阶段，避免不必要的bug外放；
 - 方便引入XP和TDD。

我认为的缺点：

 - 增加了开发阶段额外的人力投入。