---
layout: post
title:  深入理解Tomcat（六）Digester组件
date:   2018-11-19 23:52:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

Tomcat中的xml解析，是使用的apache开源组件`digester`。在tomcat源码中，`Digester类`所在位置为`org.apache.tomcat.util.digester.Digester`，它把开源组件`digester`的源代码拷贝了过来。

本文，我们主要是想了解一下`digester`的用法。以便在阅读tomcat源码的时候，看到xml解析相关代码的时候不会一脸茫然和懵逼。

`digester`有两种使用方式：

1. 一种为tomat内嵌的`org.apache.tomcat.util.digester.Digester`；
2. 另一种为`digester maven依赖`。

本文采用第二种--maven依赖的方式。

## digester实现原理

`digester`最初是作为struct的一个工具模块，来完成xml解析的功能。但是很快地，有人发现并觉得digester不应该仅仅局限在struct，而应该变得更通用。于是经过apache的孵化，最终加入到了apache commons类库家族中，并形成了一个xml另类解析的工具类库。

`digester`底层是基于`SAX`+`事件驱动`+`栈`的方式来搭建实现的。那么在digester中，这三种元素分别起到什么作用呢？

1. SAX，用于解析xml
2. 事件驱动，在SAX解析的过程中加入事件来支持我们的对象映射
3. 栈，当解析xml元素的开始和结束的时候，需要通过xml元素映射的类对象的入栈和出栈来完成事件的调用

通过一些实实在在的场景和例子，我们发现一个元素的作用无非是在其解析前后加入一些扩展逻辑！例如：

1. 开始解析某个节点的时候，是否需要创建一个类
2. 开始解析某个节点的时候，是否需要入栈操作
2. 结束解析某个节点的时候，是否需要执行某个方法
3. 结束解析某个节点的时候，是否需要出栈操作

## 如何引入依赖包

以maven为例，使用下面的dependency。

```xml
<dependency>
    <groupId>commons-digester</groupId>
    <artifactId>commons-digester</artifactId>
    <version>2.1</version>
</dependency>
```

## 如何使用

假如我们需要解析的xml为下面的格式。

```xml
<?xml version='1.0' encoding='utf-8'?>
<School name="Jen">
    <Grade name="1">
        <Class name="1" number="31"/>
        <Class name="2" number="32"/>
    </Grade>
    <Grade name="2">
        <Class name="1" number="41"/>
        <Class name="2" number="42"/>
        <Class name="3" number="37"/>
    </Grade>
</School>
```

同时，我们假设下面的约定成立：

1. 一个学校有名字属性，下面有多个年级
2. 每个年级有名字属性，下面有多个班
3. 每个班有名字和学生人数两个属性

根据上面的规则，我们需要创建关联的3个类，`School`、`Grade`和`Class`。
School有一个方法`addGrade`用于往学校对象中添加年级。

```java
package com.juconcurrent.learn.apache.digester;

public class School {
    private String name;
    private Grade grades[] = new Grade[0];
    private final Object servicesLock = new Object();

    public void addGrade(Grade g) {
        synchronized (servicesLock) {
            Grade results[] = new Grade[grades.length + 1];
            System.arraycopy(grades, 0, results, 0, grades.length);
            results[grades.length] = g;
            grades = results;
        }
    }

    public Grade[] getGrades() {
        return grades;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

同样的，年级有一个`addClass`方法，用于往`Grade对象`添加`Class`班对象。

```java
package com.juconcurrent.learn.apache.digester;

public class Grade {
    private String name;
    private Class classes[] = new Class[0];
    private final Object servicesLock = new Object();

    public void addClass(Class c) {
        synchronized (servicesLock) {
            Class results[] = new Class[classes.length + 1];
            System.arraycopy(classes, 0, results, 0, classes.length);
            results[classes.length] = c;
            classes = results;
        }
    }

    public Class[] getClasses() {
        return classes;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

Class就比较简单了，只是一个简单的POJO对象。

```java
package com.juconcurrent.learn.apache.digester;

public class Class {
    private String name;
    private int number;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getNumber() {
        return number;
    }

    public void setNumber(int number) {
        this.number = number;
    }
}
```

好了，我们已经定义好了我们所需要创建对象的类。那么如何使用digester来创建我们所需的数据呢？我们先给出例子，然后再来详细分析其中的关键方法。

```java
package com.juconcurrent.learn.apache.digester;

import org.apache.commons.digester.Digester;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;

public class DigesterTest {
    // 属性和get/set方法，假设我们解析出来的School对象放在这儿
    private School school;
    public School getSchool() {
        return school;
    }
    public void setSchool(School s) {
        this.school = s;
    }

    private void digester() throws IOException, SAXException {
        // 读取根据文件的路径，创建InputSource对象，digester解析的时候需要用到
        File file = new File("/Users/pro/ws/learn/learn-javaagent/src/main/resources/School.xml");
        InputStream inputStream = new FileInputStream(file);
        InputSource inputSource = new InputSource(file.toURI().toURL().toString());
        inputSource.setByteStream(inputStream);

        // 创建Digester对象
        Digester digester = new Digester();
        // 是否需要用DTD验证XML文档的合法性
        digester.setValidating(true);
        // 将当前对象放到对象堆的最顶层，这也是这个类为什么要有school属性的原因！
        digester.push(this);

        /*
         * 下面开始为Digester创建匹配规则
         * Digester中的School、School/Grade、School/Grade/Class，分别对应School.xml的School、Grade、Class节点
         */

        // 为School创建规则

        /*
         * Digester.addObjectCreate(String pattern, String className, String attributeName)
         * pattern, 匹配的节点
         * className, 该节点对应的默认实体类
         * attributeName, 如果该节点有className属性, 用className的值替换默认实体类
         *
         * Digester匹配到School节点
         *
         * 1. 如果School节点没有className属性，将创建com.juconcurrent.learn.apache.digester.School对象；
         * 2. 如果School节点有className属性，将创建指定的(className属性的值)对象
         */
        digester.addObjectCreate("School", School.class.getName(), "className");
        // 将指定节点的属性映射到对象，即将School节点的name的属性映射到School.java
        digester.addSetProperties("School");

        /*
         * Digester.addSetNext(String pattern, String methodName, String paramType)
         * pattern, 匹配的节点
         * methodName, 调用父节点的方法
         * paramType, 父节点的方法接收的参数类型
         * Digester匹配到School节点，将调用DigesterTest(School的父节点)的setSchool方法，参数为School对象
         */
        digester.addSetNext("School", "setSchool", School.class.getName());

        // 为School/Grade创建规则
        digester.addObjectCreate("School/Grade", Grade.class.getName(), "className");
        digester.addSetProperties("School/Grade");

        // Grade的父节点为School
        digester.addSetNext("School/Grade", "addGrade", Grade.class.getName());

        // 为School/Grade/Class创建规则
        digester.addObjectCreate("School/Grade/Class", Class.class.getName(), "className");
        digester.addSetProperties("School/Grade/Class");
        digester.addSetNext("School/Grade/Class", "addClass", Class.class.getName());
        // 解析输入源
        digester.parse(inputSource);
    }

    // 只是将School对象进行控制台输出
    private void print(School s) {
        if (s != null) {
            System.out.println(s.getName() + "有" + s.getGrades().length + "个年级");
            for (int i = 0; i < s.getGrades().length; i++) {
                if (s.getGrades()[i] != null) {
                    Grade g = s.getGrades()[i];
                    System.out.println(g.getName() + "年级 有 " + g.getClasses().length + "个班：");
                    for (int j = 0; j < g.getClasses().length; j++) {
                        if (g.getClasses()[j] != null) {
                            Class c = g.getClasses()[j];
                            System.out.println(c.getName() + "班有" + c.getNumber() + "人");
                        }
                    }
                }
            }
        }
    }

    // 入口main()方法
    public static void main(String[] args) throws IOException, SAXException {
        DigesterTest digesterTest = new DigesterTest();
        digesterTest.digester();
        digesterTest.print(digesterTest.school);
    }
}
```

这儿我们需要着重说明一下digester里面的几个方法，大体上我们可以将其方法分为两类：操作类和规则类。

1. 操作类
    + public void setValidating(boolean validating) // `是否根据DTD校验XML`
    + public void push(Object object) // `将对象压入栈`
    + public Object peek() // `获取栈顶对象`
    + public Object pop() // `弹出栈顶对象`
    + public Object parse(InputSource input) // `解析输入源`
2. 规则类
    + public void addObjectCreate(String pattern, String className, String attributeName) // `增加对象创建规则，当匹配到pattern模式时，如果指定了attributeName，则根据attributeName创建类对象；否则根据className创建类对象`
    + public void addSetProperties(String pattern) // `增加属性设置规则，当匹配到pattern模式时，就填充其属性`
    + public void addSetNext(String pattern, String methodName, String paramType) // `增加设置下一个规则，当匹配到pattern模式时，调用父节点的methodName方法，paramType为方法传入参数的类型`
    + public void addRule(String pattern, Rule rule) // `当匹配到pattern模式时，增加一个自定义规则`
    + public void addRuleSet(RuleSet ruleSet) // `增加规则集，一个规则集指的是对一个节点及下面的所有后续节点（子节点、子节点的子节点...）的解析`

## Tomcat中的规则解析例子

上面我们写了一个非常简单的例子，相信通过这样的例子我们可以很快地入门了。那么tomcat里面又是怎样写的呢？我们看看`org.apache.catalina.startup.Catalina.createStartDigester`这个方法，这个方法用于定义对server.xml的解析。该方法比较长，但是我并不打算对这个方法进行阉割和压缩，而是原封不动地拷贝到这儿，以便大家对此有一个比较完整的认识。

```java
protected Digester createStartDigester() {
    long t1=System.currentTimeMillis();
    // Initialize the digester
    Digester digester = new Digester();
    digester.setValidating(false);
    digester.setRulesValidation(true);

    // 这儿设置无效的属性，fake是赝品的意思，也就是在检查到这些属性直接认为是无效的
    Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
    List<String> objectAttrs = new ArrayList<>();
    objectAttrs.add("className");
    fakeAttributes.put(Object.class, objectAttrs);
    // Ignore attribute added by Eclipse for its internal tracking
    List<String> contextAttrs = new ArrayList<>();
    contextAttrs.add("source");
    fakeAttributes.put(StandardContext.class, contextAttrs);
    digester.setFakeAttributes(fakeAttributes);

    // 设置是否使用线程上下文类加载器
    digester.setUseContextClassLoader(true);

    // Configure the actions we will be using
    digester.addObjectCreate("Server",
                             "org.apache.catalina.core.StandardServer",
                             "className");
    digester.addSetProperties("Server");
    digester.addSetNext("Server",
                        "setServer",
                        "org.apache.catalina.Server");

    digester.addObjectCreate("Server/GlobalNamingResources",
                             "org.apache.catalina.deploy.NamingResourcesImpl");
    digester.addSetProperties("Server/GlobalNamingResources");
    digester.addSetNext("Server/GlobalNamingResources",
                        "setGlobalNamingResources",
                        "org.apache.catalina.deploy.NamingResourcesImpl");

    digester.addObjectCreate("Server/Listener",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Listener");
    digester.addSetNext("Server/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    digester.addObjectCreate("Server/Service",
                             "org.apache.catalina.core.StandardService",
                             "className");
    digester.addSetProperties("Server/Service");
    digester.addSetNext("Server/Service",
                        "addService",
                        "org.apache.catalina.Service");

    digester.addObjectCreate("Server/Service/Listener",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Service/Listener");
    digester.addSetNext("Server/Service/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    //Executor
    digester.addObjectCreate("Server/Service/Executor",
                     "org.apache.catalina.core.StandardThreadExecutor",
                     "className");
    digester.addSetProperties("Server/Service/Executor");

    digester.addSetNext("Server/Service/Executor",
                        "addExecutor",
                        "org.apache.catalina.Executor");


    digester.addRule("Server/Service/Connector",
                     new ConnectorCreateRule());
    digester.addRule("Server/Service/Connector",
                     new SetAllPropertiesRule(new String[]{"executor", "sslImplementationName"}));
    digester.addSetNext("Server/Service/Connector",
                        "addConnector",
                        "org.apache.catalina.connector.Connector");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig",
                             "org.apache.tomcat.util.net.SSLHostConfig");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig",
            "addSslHostConfig",
            "org.apache.tomcat.util.net.SSLHostConfig");

    digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                     new CertificateCreateRule());
    digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                     new SetAllPropertiesRule(new String[]{"type"}));
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/Certificate",
                        "addCertificate",
                        "org.apache.tomcat.util.net.SSLHostConfigCertificate");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig/OpenSSLConf",
                             "org.apache.tomcat.util.net.openssl.OpenSSLConf");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig/OpenSSLConf");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/OpenSSLConf",
                        "setOpenSslConf",
                        "org.apache.tomcat.util.net.openssl.OpenSSLConf");

    digester.addObjectCreate("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd",
                             "org.apache.tomcat.util.net.openssl.OpenSSLConfCmd");
    digester.addSetProperties("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd");
    digester.addSetNext("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd",
                        "addCmd",
                        "org.apache.tomcat.util.net.openssl.OpenSSLConfCmd");

    digester.addObjectCreate("Server/Service/Connector/Listener",
                             null, // MUST be specified in the element
                             "className");
    digester.addSetProperties("Server/Service/Connector/Listener");
    digester.addSetNext("Server/Service/Connector/Listener",
                        "addLifecycleListener",
                        "org.apache.catalina.LifecycleListener");

    digester.addObjectCreate("Server/Service/Connector/UpgradeProtocol",
                              null, // MUST be specified in the element
                              "className");
    digester.addSetProperties("Server/Service/Connector/UpgradeProtocol");
    digester.addSetNext("Server/Service/Connector/UpgradeProtocol",
                        "addUpgradeProtocol",
                        "org.apache.coyote.UpgradeProtocol");

    // Add RuleSets for nested elements
    digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
    digester.addRuleSet(new EngineRuleSet("Server/Service/"));
    digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
    digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
    addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
    digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));

    // When the 'engine' is found, set the parentClassLoader.
    digester.addRule("Server/Service/Engine",
                     new SetParentClassLoaderRule(parentClassLoader));
    addClusterRuleSet(digester, "Server/Service/Engine/Cluster/");

    // 根据t1和t2，算出整个server.xml的Digester创建花费的时间
    long t2=System.currentTimeMillis();
    if (log.isDebugEnabled()) {
        log.debug("Digester for server.xml created " + ( t2-t1 ));
    }
    return (digester);
}
```

这儿我们看到，在tomcat中明显地用到了前面例子中说明的几个规则，我们再简单罗列一下：
1. addObjectCreate，对象创建规则
2. addSetProperties，属性设置规则
3. addSetNext，设置下一个规则
4. addRule，自定义规则
5. digester.addRuleSet，自定义规则集

## 总结

本文我们对digester做了一个使用说明。

我们首先简单地说明了一下digester是什么，内部基于什么原理来实现的。然后通过一个School、Grade和Class这样的生活中的例子来说明digester的用法。最后通过查看tomcat中关于digester的例子代码，加深了我们对于digester的理解。

相信通过这篇文章，让我们在阅读tomcat源码的过程中不再对xml解析产生疑惑！

## 参考链接

1. https://blog.csdn.net/qq_24451605/article/details/51289519
2. https://blog.csdn.net/flyliuweisky547/article/details/23872231
3. https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/tomcat/util/digester/package-summary.html