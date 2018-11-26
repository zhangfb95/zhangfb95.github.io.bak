---
layout: post
title:  lombok使用备忘
date:   2016-10-11 10:07:32 +0800
categories: java
tag: java
---

* content
{:toc}


lombok是为了简化java中一些必不可少的，但看起来却很臃肿的代码，尤其是对于POJO，以及日志类的声明。

## 如何使用

lombok对外提供了一个实现JSR269的类库jar包。如果我们要使用lombok，只需要引入此jar包即可。maven引入如下：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.10</version>
</dependency>
```

## 原理剖析

自JDK6开始，javac就支持JSR269规范（Pluggable Annotation Processing API，插件式注解处理API）,只要实现了此规范，就能再javac运行时得到调用。
例如有一个程序A实现了JSR269，那么javac编译的流程就会变成下面这样：

<img src="/img/2016-10-11-lombok-usage-img/principle.jpg">

1. javac分析源代码，生成一颗抽象语法树（AST）
1. 运行过程中，调用程序A
1. 程序A完成自己的逻辑，如：添加getter/setter/toString等，并修改步骤1的AST
1. javac使用修改后的AST生成字节码文件

所以，本质上lombok完成的就是程序A的工作。

## 是与非

是：lombok带来了代码的简写，以前在POJO或者日志操作方法需要写很多冗余重复的地方，现在已经简化成极其简单的几句annotation。

非：使用lombok虽然能简化代码的书写，但是对于不属于lombok的人来说，阅读代码会成为其障碍。所以有一些比较隐晦或者冷门的lombok用法，还是建议不要使用。

## val

`val`可以用作临时变量的声明定义，用以代替真实的类名称。val声明的变量类型，会从上下文环境中推断出来。

```java
void useLombok() {
    val example = new HashMap<String, Object>;
}

// 等同于下面

void useNoLombok() {
    Map<String, Object> example = new HashMap<>;
}
```

## @NotNull

`@NotNull`用在一个构造方法或普通方法的参数声明，用以校验参数为空。如果参数为空，则抛出NullPointerException。
如果是普通方法，非空校验的代码将插入在方法的顶部；如果是构造方法，则会插入到紧随`this()`或`super()`调用之后。

```java
void useLombok(@NotNull User user) {
    // code is here
}

// 等同于下面

void useNoLombok(User user) {
    if (user == null) {
        throw new NullPointerException("user");
    }
    // code here
}
```

## @Cleanup

这个annotation用于对资源进行清理动作，例如：数据库连接的释放、文件句柄的关闭。
它默认的清理方法为无参的close()，如果清理方法名字不是这个，则需要手动声明清理方法的名称，如：`@Cleanup("clearup");`。

```java
void useLombok() {
    @Cleanup InputStream is = new FileInputStream("aa");
}

// 等同于下面

void useNoLombok() {
    InputStream is = null;
    try {
        is = new FileInputStream("aa");
    } finally {
        if (is != null) {
            is.close();
        }
    }
}
```

## @Getter / @Setter

这两个注解可以作用于非静态字段上面，lombok会自动生成getter和setter方法，不过生成的方法名称遵循POJO规范。

同时这两个注解可以作用于类上面，lombok会这个类的所有非静态字段都添加gettter和setter方法。
如果某一个字段不想使用其自动生成的方法，可以使用@Setter或@Getter的属性AccessLevel来设置。

示例代码如下：

```java
public class UseLombok {
    @Getter @Setter private String name;
    @Setter(AccessLevel.PROTECTED) private Integer age;
}

public class UseNoLombok {
    private String name;
    private Integer age;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Integer getAge() { return age; }
    public void setAge(Integer age) { this.age = age; }
}

@Setter @Getter
public class UseLombokCls {
    private String name;
    private Integer age;
    @Setter(AccessLevel.NONE)
    private Integer sex; // 这个属性不会生成setter方法
}
```

## @EqualsAndHashCode

用以自动生成`equals`和`hashCode`方法。默认会使用所有非静态、非transient的字段。如果要刻意排除一些字段不纳入方法的生成，可以使用`exclude`参数。
另外它的另一个参数`callSuper`用于表示是否继承父类的`equals`和`hashCode`方法，默认为false。

```java
@EqualsAndHashCode
// @EqualsAndHashCode(exclude = "age") 这个参数可选
// @EqualsAndHashCode(callSuper = true) 这个参数可选
public class UseLombok {
    private String name;
    private Integer age;
    private Integer sex;
}
```

## @ToString

用以生成`toString()`方法，用法和`@EqualsAndHashCode`类似，它只输出非静态的属性。它也有`exclude`和`callSuper`参数，额外多了`@includeFieldNames`。
`@includeFieldNames`参数用以是否显示属性名称，默认为true。

```java
@ToString(exclude = "age")
public class UseLombok {
    private String name;
    private Integer age;
    private Integer sex;
    private int no = 20;
}

// 等同于下面

public class UseLombok {
    private String name;
    private Integer age;
    private Integer sex;
    private int no = 20;
    public String toString() {
        return "UseLombok(name=" + this.name + ", sex=" + this.sex + ", no=" + this.no + ")";
    }
}
```

## @Constructor系列

构造方法注解一共有3个，@NoArgsConstructor, @RequiredArgsConstructor, @AllArgsConstructor。

1. @NoArgsConstructor 无参构造，会生成一个默认构造方法
1. @RequiredArgsConstructor 特殊处理的构造方法，它会对标注了@NotNull的字段进行构造方法生成，且生成的构造方法的参数也是@NotNull的。如果调用此方法时传入空指针，则会抛出空指针异常
1. @AllArgsConstructor 全参构造，会生成一个带所有非静态字段参数构造方法

他们也有一些公用的参数，包括：`staticName`、`access`、`suppressConstructorProperties`。

1. staticName 它会将生成的构造方法设置成`private`，并且会增加一个静态工厂方法，这个方法的参数和构造方法参数一样，且返回当前类对象。
1. access 可设置构造方法的访问级别，`AccessLevel`
1. suppressConstructorProperties 在JDK1.6之后，增加了一个注解`java.beans.ConstructorProperties`，用以在构造方法上加入构造的参数名称。
如果不想自动生成这个注解，可以将其设置为true，默认为false。

```java
@NoArgsConstructor
@RequiredArgsConstructor(staticName = "of")
@AllArgsConstructor
public class UseLombok {
    @NonNull
    private String name;
    @NonNull
    private Integer age;
    private Integer sex;
    private final int no = 20;
}

// 等同于下面的代码

public class UseLombok {
    @NonNull
    private String name;
    @NonNull
    private Integer age;
    private Integer sex;
    private final int no = 20;

    public UseLombok() {
    }

    private UseLombok(@NonNull String name, @NonNull Integer age) {
        if(name == null) {
            throw new NullPointerException("name");
        } else if(age == null) {
            throw new NullPointerException("age");
        } else {
            this.name = name;
            this.age = age;
        }
    }

    public static UseLombok of(@NonNull String name, @NonNull Integer age) {
        return new UseLombok(name, age);
    }

    @ConstructorProperties({"name", "age", "sex"})
    public UseLombok(@NonNull String name, @NonNull Integer age, Integer sex) {
        if(name == null) {
            throw new NullPointerException("name");
        } else if(age == null) {
            throw new NullPointerException("age");
        } else {
            this.name = name;
            this.age = age;
            this.sex = sex;
        }
    }
}
```

## @Data / @Value

`@Data`是lombok最常用的注释，主要用于一个POJO繁琐的getter/setter/toString/equals/hashCode/构造方法的生成。
它相当于集成了`@ToString`, `@EqualsAndHashCode`, `@Getter / @Setter` and `@RequiredArgsConstructor`。
同时它只有唯一属性`staticConstructor`，用以表示使用静态工厂构造方法。

```java
@Data
public class UseLombok {

    @NonNull
    private String name;
    @NonNull
    private Integer age;
    private Integer sex;
    private final int no = 20;
}
```

`@Value`和`@Data`类似，只是它表示【一个值】。值的意思就是不可变。它会将所有属性设置成`private final`。不会生成`setter`方法。
实际上，它是对一些特性的集成，包括：
1. final 字段不可变
1. @ToString
1. @EqualsAndHashCode
1. @AllArgsConstructor
1. @FieldDefaults(makeFinal = true, level = AccessLevel.PRIVATE) @Getter

```java
@Value
public class UseLombok {
    @NonNull
    private String name;
    @NonNull
    private Integer age;
    private Integer sex;
    private final int no = 20;
}

// 等同于下面

public final class UseLombok {
    @NonNull
    private final String name;
    @NonNull
    private final Integer age;
    private final Integer sex;
    private final int no = 20;

    @ConstructorProperties({"name", "age", "sex"})
    public UseLombok(@NonNull String name, @NonNull Integer age, Integer sex) {
        if(name == null) {
            throw new NullPointerException("name");
        } else if(age == null) {
            throw new NullPointerException("age");
        } else {
            this.name = name;
            this.age = age;
            this.sex = sex;
        }
    }

    @NonNull
    public String getName() { return this.name; }
    @NonNull
    public Integer getAge() { return this.age; }
    public Integer getSex() { return this.sex; }
    public int getNo() { this.getClass(); return 20; }

    // ignored
    public boolean equals(Object o) { return false; }
    // ignored
    public int hashCode() { return -1; }
    // ignored
    public String toString() { return ""; }
}
```

## @Builder

在新版的lombok中产生的这个注释，用以生成一个建造者，它实际上是对建造者模式的支持，且对`链式编程`支持得非常好。
我们可以使用以下的代码来简单书写我们的赋值操作。可是如果我们要实现这样的代码，需要书写很多额外多余的代码。这一点，lombok很好地为我们想到了。

```java
Person p = Person.builder().name("Mr.zhangfb").city("Chengdu").job("Poetc").job("Poetc2").build();
```

其实`@Builder`会为我们做以下几件事：

1. 生成一个叫做`XxxxBuilder`的静态内部类
1. 在Builder内部，生成私有的非静态、非final字段的参数，和外部类一样
1. 在Builder内部，一个私有的无参构造器
1. 在Builder内部，类`setter`方法生成，方法名同参数名，返回Builder本身
1. 在Builder内部，生成`build`方法，将每个参数赋值给外部类，且返回外部类对象
1. 在Builder内部，`toString`方法的生成
1. 外部类中，有一个静态的`builder()`方法，创建一个Builder的类实例。

同时，它还支持在属性上添加`@Singular`，用以表示此字段为一个集合。可以“逐个添加”。

```java
@Builder
public class BuilderExample {
    private String name;
    private int age;
    @Singular private Set<String> occupations;
}

// 等同于下面

public class BuilderExample {
    private String name;
    private int age;
    private Set<String> occupations;

    BuilderExample(String name, int age, Set<String> occupations) {
        this.name = name;
        this.age = age;
        this.occupations = occupations;
    }

    public static BuilderExampleBuilder builder() {
        return new BuilderExampleBuilder();
    }

    public static class BuilderExampleBuilder {
        private String name;
        private int age;
        private java.util.ArrayList<String> occupations;

        BuilderExampleBuilder() {
        }

        public BuilderExampleBuilder name(String name) {
            this.name = name;
            return this;
        }

        public BuilderExampleBuilder age(int age) {
            this.age = age;
            return this;
        }

        public BuilderExampleBuilder occupation(String occupation) {
            if (this.occupations == null) {
                this.occupations = new java.util.ArrayList<String>();
            }

            this.occupations.add(occupation);
            return this;
        }

        public BuilderExampleBuilder occupations(Collection<? extends String> occupations) {
            if (this.occupations == null) {
                this.occupations = new java.util.ArrayList<String>();
            }

            this.occupations.addAll(occupations);
            return this;
        }

        public BuilderExampleBuilder clearOccupations() {
            if (this.occupations != null) {
                this.occupations.clear();
            }

            return this;
        }

        public BuilderExample build() {
            // complicated switch statement to produce a compact properly sized immutable set omitted.
            // go to https://projectlombok.org/features/Singular-snippet.html to see it.
            // Set<String> occupations =...;
            return new BuilderExample(name, age, occupations);
        }

        @java.lang.Override
        public String toString() {
            return "BuilderExample.BuilderExampleBuilder(name = " + this.name + ", age = " + this.age + ", occupations = " + this.occupations + ")";
        }
    }
}
```

## @Log系列

写代码必不可免地会产生bug，而bug的追踪需要一系列操作的痕迹，日志是重要的痕迹保留手段。@Log根据日志类型的不同，会有不同的扩展。主要包括如下：

1. @CommonsLog 创建 `private static final org.apache.commons.logging.Log log = org.apache.commons.logging.LogFactory.getLog(LogExample.class);`
1. @JBossLog 创建 `private static final org.jboss.logging.Logger log = org.jboss.logging.Logger.getLogger(LogExample.class);`
1. @Log 创建 `private static final java.util.logging.Logger log = java.util.logging.Logger.getLogger(LogExample.class.getName());`
1. @Log4j 创建 `private static final org.apache.log4j.Logger log = org.apache.log4j.Logger.getLogger(LogExample.class);`
1. @Log4j2 创建 `private static final org.apache.logging.log4j.Logger log = org.apache.logging.log4j.LogManager.getLogger(LogExample.class);`
1. @Slf4j 创建 `private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(LogExample.class);`
1. @XSlf4j 创建 `private static final org.slf4j.ext.XLogger log = org.slf4j.ext.XLoggerFactory.getXLogger(LogExample.class);`

```java
@Slf4j
public class UseLombok {
}
```

而如果主题名称需要变更，可以使用参数`topic`来修改。

```java
@Slf4j(topic = "demoTopic")
public class UseLombok {
}
```

## @SneakyThrows

这个注释用以将检查性异常包装成运行时异常，以便减少强制性捕获。默认会将`Throwable`异常进行转换

```java
public class SneakyThrowsExample implements Runnable {
    @SneakyThrows(UnsupportedEncodingException.class)
    public String utf8ToString(byte[] bytes) {
        return new String(bytes, "UTF-8");
    }

    @SneakyThrows
    public void run() {
        throw new Throwable();
    }
}

// 等同于下面

public class SneakyThrowsExample implements Runnable {
    public String utf8ToString(byte[] bytes) {
        try {
            return new String(bytes, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            throw Lombok.sneakyThrow(e);
        }
    }

    public void run() {
        try {
            throw new Throwable();
        } catch (Throwable t) {
            throw Lombok.sneakyThrow(t);
        }
    }
}
```

## @Synchronized

用于对方法体进行加锁，生成`synchronized`关键字，用以减少代码缩进。它遵循以下几点规则：

1. 静态方法使用`@Synchronized`，创建一个静态锁对象`Object $LOCK`，并使用它
1. 非静态方法使用`@Synchronized`，创建一个非静态的锁对象`Object $lock`，并使用它
1. 非静态方法带参数的`@Synchronized("readLock")`，需要自行手动创建锁对象，此方法会使用它

```java
public class SynchronizedExample {
    private final Object readLock = new Object();

    @Synchronized
    public static void hello() {
        System.out.println("world");
    }

    @Synchronized
    public int answerToLife() {
        return 42;
    }

    @Synchronized("readLock")
    public void foo() {
        System.out.println("bar");
    }
}

// 等同于下面

public class SynchronizedExample {
    private static final Object $LOCK = new Object[0];
    private final Object $lock = new Object[0];
    private final Object readLock = new Object();

    public static void hello() {
        synchronized($LOCK) {
            System.out.println("world");
        }
    }

    public int answerToLife() {
        synchronized($lock) {
            return 42;
        }
    }

    public void foo() {
        synchronized(readLock) {
            System.out.println("bar");
        }
    }
}
```

## 自定义配置的使用

有时，我们并不想使用lombok内置的一些设置，我们可以通过使用它提供的扩展机制来进行修改。
比如`@Log`会自动生成一个`log`对象，而又的代码检查工具（如sonar）会检查静态常量对象必须为大写。这时我们就需要通过此种方式来变更。
使用方式如下：`java -jar lombok.jar config -g --verbose`

另外，我们可以在编译的目录下创建一个配置文件`lombok.config`，也可以达到相同的效果。

```
lombok.log.fieldName = foobar
```

## 参考文档

+ [lombok-features](https://projectlombok.org/features/)
+ [https://my.oschina.net/darkness/blog/510808](https://my.oschina.net/darkness/blog/510808)
+ [http://www.blogjava.net/fancydeepin/archive/2012/07/12/lombok.html](http://www.blogjava.net/fancydeepin/archive/2012/07/12/lombok.html)
