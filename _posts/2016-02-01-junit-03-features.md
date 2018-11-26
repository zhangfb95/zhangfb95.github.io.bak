---
layout: post
title:  单元测试（3）junit特性
date:   2016-02-01 10:48:00 +0800
categories: Junit
tag: Junit
---

* content
{:toc}

# junit特性

> 项目多元化，导致最基本的功能有时难以应付。所以，junit自4.x发布以来，每次新版本的推出都引入很多测试理念和机制。在实际项目之中，不一定非要用到所有的这些特性，选择适合的才是最好的。

## 基本功能回顾

 - @Test，最小测试方法注释
 - @BeforeClass，测试类执行前操作，用于初始化，必须是public static void，且放在类的开始处
 - @AfterClass，测试类执行后操作，用于清理、释放，必须是public static void，且放在类的开始处
 - @Before，@Test方法执行前逻辑
 - @After，@Test方法执行后逻辑
 - @Ignored，忽略@Test方法执行

## 断言机制及断言增强
断言是单元测试最基本，最核心，最重要的概念。我们可以将断言理解为**一个语句是否成立**。junit最初的断言，做了最基本的功能——两个值是相同、结果是否为空指针、断言语句是否为True（或者False），且增加失败时的信息输出。

通用格式如下（谓主宾格式）：

```java
Assert.assertEquals([optional]message, expected, actual);
```

完整实例如下：

```java
@Test
public void testAssert() {
    Assert.assertEquals(1L, 1L);
    Assert.assertNotEquals(1L, 2L);

    Assert.assertNotNull(new Object());
    Assert.assertNull(null);

    Assert.assertTrue(true);
    Assert.assertFalse(false);

    Assert.assertNotSame("the two objects not same", new Object(), new Object());

    final Object sameObj = new Object();
    Assert.assertSame("sameObj is not same of sameObj", sameObj, sameObj);

    final int[] expecteds = {1, 2, 3};
    final int[] actuals = {1, 2, 3};
    Assert.assertArrayEquals(expecteds, actuals);
}
```

这种断言机制引入之后，极大地增强了代码的可读性和完整性。不过，事情总是朝着好的方向发展。junit社区有人反馈，认为这种反人类语言的格式不是很好用，junit作者也善意地采纳其意见，并决定在后期版本中加入更易懂的方式（主谓宾格式），以便更亲近人类，语义也更直观。所以，引入了**assertThat**，它使用Matcher（匹配器）来完成它的职责，Matcher本质是使用链式编程的方式实现引用代入。核心的Matcher都放在org.hamcrest.CoreMatchers下面的。

通用格式如下：

```java
assertThat(actual, Matcher<? super T> matcher);
```

完整实例如下：

```java
@Test
public void testAssertThat() {
    final int id = 3;
    Assert.assertThat(id, is(3));
    Assert.assertThat(id, is(not(4)));

    final boolean trueValue = true;
    Assert.assertThat(trueValue, is(true));

    final boolean falseValue = false;
    Assert.assertThat(falseValue, is(false));

    final Object nullObject = null;
    Assert.assertThat(nullObject, nullValue());

    final String helloWord = "Hello xxx World";
    Assert.assertThat(helloWord, both(startsWith("Hello")).and(endsWith("World")));
}
```

## 自定义Matcher

有时，junit自带的Matcher并不能完成我们想要完成的匹配，这时我们就需要**自定义Matcher**，以此来用于特定语境下的特定处理。org.hamcrest.Matcher是一个接口，它的注释上明确写明，不能直接继承它，需要继承org.hamcrest.BaseMatcher。引用其注释如下：

> Matcher implementations <span style="color:red;">should NOT directly implement this interface</span>. Instead, <span style="color:red;">extend the BaseMatcher abstract class</span>, which will ensure that the Matcher API can grow to support new features and remain compatible with all Matcher implementations.

举例来说，我们想要实现这样的Matcher，用于判断User对象的username和password都是admin。这种应用场景虽然比较BT，但是也是有可能的，这儿我们实现自己的Matcher，代码如下：

```java
class User {
    private String username;
    private String password;
    // omited...
}
```

```java
/**
 * <p>
 * Matcher implementations should <b>NOT directly implement this interface</b>.
 * Instead, <b>extend</b> the {@link BaseMatcher} abstract class,
 * which will ensure that the Matcher API can grow to support
 * new features and remain compatible with all Matcher implementations.
 * <p/>
 * @see org.hamcrest.Matcher
 */
class IsAdminMatcher extends BaseMatcher<User> {

    @Override
    public boolean matches(Object item) {
        if (item == null) {
            return false;
        }

        User user = (User) item;
        return "admin".equals(user.getUsername()) && "admin".equals(user.getPassword());
    }

    /**
     * real description about the actual value
     *
     * @param description the simple description obj
     */
    @Override
    public void describeTo(Description description) {
        description.appendText("Administrator with 'admin' as username and password");
    }

    /**
     * while meeting assert fail, it will be printed out
     *
     * @param item actual value
     * @param description the simple description obj
     */
    @Override
    public void describeMismatch(Object item, Description description) {
        if (item == null) {
            description.appendText("was null");
        } else {
            User user = (User) item;
            description.appendText("was a common user (")
                    .appendText("username: ").appendText(user.getUsername()).appendText(", ")
                    .appendText("password: ").appendText(user.getPassword()).appendText(")");
        }
    }
}
```

```java
User user = new User("admin", "admin");
Assert.assertThat(user, new IsAdminMatcher());
```

## test方法执行顺序

在测试类中，如果我们要指定方法的执行顺序，可以使用注解FixMethodOrder。这样，test case不会乱序执行。样例代码如下：

```java
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class MethodOrderTest {

    @Test
    public void testA() {
        System.out.println("first");
    }

    @Test
    public void testB() {
        System.out.println("second");
    }

    @Test
    public void testC() {
        System.out.println("third");
    }
}
```

MethodSorter是一个枚举，它有以下枚举项，默认为DEFAULT。

```java
public enum MethodSorters {
    /**
     * 按照字母升序执行
     */
    NAME_ASCENDING(MethodSorter.NAME_ASCENDING),

    /**
     * 按照JVM中方法加载顺序执行
     */
    JVM(null),

    /**
     * 默认顺序，由方法名hashcode值来决定，如果hash值大小一致，则按名字的字典顺序确定。
     * 由于hashcode的生成和操作系统相关(以native修饰），所以对于不同操作系统，可能会出现不一样的执行顺序。
     * 在某一操作系统上，多次执行的顺序不变。
     */
    DEFAULT(MethodSorter.DEFAULT);
}
```

## suite，聚合test cases

suite，顾名思义，就是套件的意思。在junit中，它主要用于将一堆test cases聚合起来，形成一个套件。有两种suite使用方式，一种为硬编码方式，一种为注解方式。

> 【注】：可以将TestSuite看成一种特殊的Test。从代码层面我们也可以看出，TestSuite继承了Test。

首先，假设我们有以下的test cases：

```java
public class DemoTest {

    public static class TestSuite2 {
        public static junit.framework.Test suite() {
            TestSuite suite = new TestSuite("Test for package2");

            suite.addTest(new JUnit4TestAdapter(Test4.class));
            suite.addTest(new JUnit4TestAdapter(Test5.class));
            return suite;
        }
    }

    public static class Test1 {

        @Test
        public void test1() {
            System.out.println("test1 invoked");
        }
    }

    public static class Test2 {

        @Test
        public void test2() {
            System.out.println("test2 invoked");
        }
    }

    public static class Test3 {

        @Test
        public void test3() {
            System.out.println("test3 invoked");
        }
    }

    public static class Test4 {

        @Test
        public void test4() {
            System.out.println("test4 invoked");
        }
    }

    public static class Test5 {

        @Test
        public void test5() {
            System.out.println("test5 invoked");
        }
    }
}
```

其中TestSuite2可以看成是Test4和Test5的组合，现在我们想将Test1、Test2、Test3和TestSuite2组合起来一起执行，应该怎么办呢？这时候就是suite上场的时候了。

1、硬编码方式

```java
public class Suite1Test {

    public static Test suite() {
        TestSuite suite = new TestSuite("Test for package1");

        suite.addTest(new JUnit4TestAdapter(DemoTest.Test1.class));
        suite.addTest(new JUnit4TestAdapter(DemoTest.Test2.class));
        suite.addTest(new JUnit4TestAdapter(DemoTest.Test3.class));

        suite.addTest(DemoTest.TestSuite2.suite());
        return suite;
    }
}
```

2、注解方式

```java
@RunWith(Suite.class)
@Suite.SuiteClasses({
        DemoTest.Test1.class,
        DemoTest.Test2.class,
        DemoTest.Test3.class,
        DemoTest.TestSuite2.class
})
public class Suite2Test {
}
```

综合分析，注解方式代码更少、更简洁，在实际项目中，我们更偏向使用注解方式。另外需要注意的是，TestSuite应该有一个**public static junit.framework.Test suite()**方法。

## 参数化测试

它可以看做是suite的一种特例，目的是为了对test case进行多次执行，已达到较为全面的覆盖。参数化测试需要以一种特殊的Runner执行，**@RunWith(Parameterized.class)**。下面我们以斐波那契来举例。

> 关于什么是斐波那契序列，请移步<a href="http://baike.baidu.com/link?url=NKEK8_98-RL4eXL1AwokjkWsas2rADmI54EZcvdALURI6QJ8o8IUU5aLN9bx5cuut5l0DsbuJOXSHpv0T6w-L_" target="_blank">百度百科</a>。

```java
@RunWith(Parameterized.class)
public class FibonacciTest {

    /**
     * In order to easily identify the individual test cases in a Parameterized test,
     * you may provide a name using the @Parameters annotation.
     * This name is allowed to contain placeholders that are replaced at runtime:<br>
     * {index}: the current parameter index<br>
     * {0}, {1}, …: the first, second, and so on, parameter value. NOTE: single quotes ' should be escaped as two single quotes ''.<br>
     * <p/>
     * In the example given above, the Parameterized runner creates names like [1: fib(3)=2].
     * If you don't specify a name, the current parameter index will be used by default.
     *
     * @return
     */
    //@Parameterized.Parameters
    @Parameterized.Parameters(name = "{index}: fib({0})={1}")
    public static Collection data() {
        return Arrays.asList(new Object[][]{{0, 0}, {1, 1}, {2, 1}, {3, 2}, {4, 3}, {5, 5}, {6, 8}});
    }

    private int input;
    private int expected;

    public FibonacciTest(int input, int expected) {
        this.input = input;
        this.expected = expected;
    }

    @Test
    public void test() {
        Assert.assertEquals(expected, Fibonacci.compute(input));
    }
}
```

这儿我们可以给参数取一个名字，一般情况下使用默认的就可以了。

个人觉得，参数化测试带来了一些弊端——如果有多个test case需要进行参数化，需要增加至多个测试类。粒度为类，而不是方法。后面的特性中，我们会介入解决这种问题。

## 自定义Rule

自定义规则的意图是为了丰富test case，增加其灵活性。我们可以简单地罗列一些场景：

 - 循环执行N次，N为一个变量；
 - 满足条件时，执行**M次**，不满足条件时，执行**N次**；
 - 循环执行，直到条件满足时跳出循环。

这些场景中，如果我们把test case看成一个单元，规则则是在filter这个单元之后，对其进行判断、循环和额外逻辑处理的Runner。极大地满足了某些应用场景，且增加了单元测试的灵活性。

下面我们以一个例子来演示：

1、定义规则注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({java.lang.annotation.ElementType.METHOD})
public @interface Repeat {
    int times();
}
```

2、定义规则

```java
class RepeatRule implements TestRule {

    private static class RepeatStatement extends Statement {

        private final int times;
        private final Statement statement;

        private RepeatStatement(int times, Statement statement) {
            this.times = times;
            this.statement = statement;
        }

        @Override
        public void evaluate() throws Throwable {
            for (int i = 0; i < times; i++) {
                statement.evaluate();
            }
        }
    }

    @Override
    public Statement apply(Statement statement, Description description) {
        Statement result = statement;
        Repeat repeat = description.getAnnotation(Repeat.class);
        if (repeat != null) {
            int times = repeat.times();
            result = new RepeatStatement(times, statement);
        }
        return result;
    }
}
```

3、使用规则

```java
public class TestRuleTest {

    @Rule
    public RepeatRule repeatRule = new RepeatRule();

    @Test
    @Repeat(times = 100)
    public void testCalculateRangeValue() {
        long center = 0;
        long radius = 10;
        RandomRangeValueCalculator calculator = new RandomRangeValueCalculatorImpl();

        long actual = calculator.calculateRangeValue(center, radius);
        System.out.println(actual);

        Assert.assertTrue(center + radius >= actual);
        Assert.assertTrue(center - radius <= actual);
    }
}
```

## 分组测试-Categories

分组测试，其实也属于一种特殊的suite，用于对test case进行分组。并使用**@RunWith(Categories.class)**来执行和筛除分组的test case。下面以一个例子来清晰表明如何进行分组测试。

1、定义分组

```java
// 声明两个什么都没有的接口
public interface FastTests { }
public interface SlowTests { }
```

2、书写单元测试

```java
public class ATest {

    @Test
    public void a() {
        System.out.println("A a()");
    }

    @Category(SlowTests.class)
    @Test
    public void b() {
        System.out.println("A b()");
    }
}

@Category({SlowTests.class, FastTests.class})
public class BTest {

    @Test
    public void c() {
        System.out.println("B c()");
    }
}
```

3、书写分组suite

```java
@RunWith(Categories.class) // 这个地方与一般的套件测试有所不同
@Categories.IncludeCategory(SlowTests.class)
@Suite.SuiteClasses({
        ATest.class,
        BTest.class
}) // Note that Categories is a kind of Suite
public class SlowTestSuite1 {
}
```

```java
@RunWith(Categories.class)
@Categories.IncludeCategory(SlowTests.class)
@Categories.ExcludeCategory(FastTests.class)
@Suite.SuiteClasses({
        ATest.class,
        BTest.class
})
public class SlowTestSuite2 {
}
```

## 假设机制-assume

假设机制是用于在条件满足时执行test case，条件不满足时忽略test case的特殊机制。它使用**assumeThat**来进行判断。

```java
public class AssumeTest {

    @Test
    public void testOneEqualsOne() {
        // sample for actual
        // assumeThat(File.separatorChar, is('/'));
        // System.out.println("is executed");
        assumeThat('1', is('1'));
        System.out.println("1 == 1");
    }

    @Test
    public void testOneNotEqualsTwo() {
        assumeThat('1', is('2'));
        System.out.println("1 == 2");
    }
}
```

## 理论机制-Theories

Theories，英文意思为理论、推断。它是一种特殊的Runner，提供了除Parameterized之外的另外一个更为强大的参数化测试解决方案。Theories不是使用带参的构造方法，而是使用受参的测试方法。test case的修饰注解也从@Test变化为@Theory，参数的提供也变化为@DataPoint或者@Datapoints，他们两的不同之处在于前者代表一个数据，后者代表一组数据。Theories会尝试所有类型匹配的参数作为测试方法的入参。我们举一个简单的例子。

```java
@RunWith(Theories.class)
public class UserTest {

    @DataPoint
    public static String GOOD_USERNAME = "optimus";
    @DataPoint
    public static String USERNAME_WITH_SLASH = "optimus/prime";
    @DataPoints
    public static String[] usernames = {"optimus", "optimus/prime"};

    @Theory
    public void filenameIncludesUsername(String username) {
        assumeThat(username, not(containsString("/")));
        assertThat(username, is(GOOD_USERNAME));
        assertThat(new User(username).configFileName(), containsString(username));
    }

    static class User {

        private String username;

        public User(String username) {
            this.username = username;
        }

        public String getUsername() {
            return username;
        }

        public void setUsername(String username) {
            this.username = username;
        }

        public String configFileName() {
            return username + "/sb";
        }
    }
}
```

因为有**assumeThat(username, not(containsString("/")));**，所以只有不带“/”的参数才会被代入。



Theories还支持了自定义数据提供方式，需要继承Junit的ParameterSupplier。

1，定义参数注解

```java
@Retention(RetentionPolicy.RUNTIME)
@ParametersSuppliedBy(BetweenSupplier.class)
@interface Between {
    int first();

    int last();
}
```

2，提供参数支持类，继承ParameterSupplier类

```java
public class BetweenSupplier extends ParameterSupplier {

    @Override
    public List<PotentialAssignment> getValueSources(ParameterSignature sig) {
        Between annotation = sig.getAnnotation(Between.class);

        List<PotentialAssignment> list = new ArrayList<>();
        for (int i = annotation.first(); i <= annotation.last(); i++) {
            list.add(PotentialAssignment.forValue("value", i));
        }
        return list;
    }
}
```

3、test case使用参数注解

```java
@RunWith(Theories.class)
public class DollarTest {

    @Theory
    public void multiplyIsInverseOfDivideWithInlineDataPoints(@Between(first = -100, last = 100) int amount,
                                                              @Between(first = -100, last = 100) int m) {
        assumeThat(m, not(0));
        System.out.println(amount + ":" + m);
        assertThat(new Dollar(amount).times(m).divideBy(m).getAmount(), is(amount));
    }
}
```

Junit自带了TestedOn注解，用于输入一个int数组，样例代码如下：

```java
@RunWith(Theories.class)
public class DollarTest {

    @Theory
    public final void test(@TestedOn(ints = {0, 1, 2}) int i) {
        assertTrue(i >= 0);
    }

    @Theory
    public void multiplyIsInverseOfDivide(@TestedOn(ints = {0, 5, 10}) int amount,
                                          @TestedOn(ints = {0, 1, 2}) int m) {
        assumeThat(m, not(0));
        assertThat(new Dollar(amount).times(m).divideBy(m).getAmount(), is(amount));
    }
}
```

## 多线程下的单元测试

在多线程下，单元测试很难保证线程安全。junit并没有直接提供多线程环境下的测试机制，但是指明了使用某些第三方类库可以达到这样的目的。concurrentunit就是其中的一种，在gradle下面，我们可以使用**compile 'net.jodah:concurrentunit:${version}'**引入依赖。下面我们提供一个非常简单的多线程示例，Waiter提供了类似CountDownLatch机制，关于什么是CountDownLatch，请自行百度。

```java
@Test
public void shouldSupportMultipleThreads() throws Throwable {
    final Waiter waiter = new Waiter();

    for (int i = 0; i < 5; i++) {
        new Thread(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    System.out.println(e.getLocalizedMessage() + e);
                }
                waiter.assertTrue(true);
                waiter.resume();
            }
        }).start();
    }

    waiter.await();
}
```

## 参考文献

junit百度百科

 - http://baike.baidu.com/link?url=0znEazMbQ3Mb3utV6nVPl1fcUdQMCZM5di6s2Xzx4qUtHcjHFGqNM-64Hy2tYoOqn0k7wQZg8oeq27IzWD1WAK

junit断言和Matcher

 - https://github.com/junit-team/junit/wiki/Assertions
 - http://www.cxyclub.cn/n/47421/
 - http://blog.sina.com.cn/s/blog_700aa8830101jpcj.html

执行顺序

 - http://www.cnblogs.com/nexiyi/p/junit_test_in_order.html
 - http://blog.csdn.net/renfufei/article/details/36421087
 - http://jackyrong.iteye.com/blog/2025609

suite

 - https://github.com/junit-team/junit/wiki/Aggregating-tests-in-suites
 - http://stackoverflow.com/questions/190224/junit-and-junit-framework-testsuite-no-runnable-methods
 - http://blog.csdn.net/listeningsea/article/details/7667103

参数化测试

 - https://github.com/junit-team/junit/wiki/Parameterized-tests
 - http://www.cnblogs.com/mengdd/archive/2013/04/13/3019336.html

自定义Rule

 - https://github.com/junit-team/junit/wiki/Rules
 - http://blog.csdn.net/yqj2065/article/details/39945617
 - http://fansofjava.iteye.com/blog/504936

分组测试

 - http://fansofjava.iteye.com/blog/556839
 - https://github.com/junit-team/junit/wiki/Categories

假设机制

 - https://github.com/junit-team/junit/wiki/Assumptions-with-assume

理论机制

 - https://github.com/junit-team/junit/wiki/Theories
 - http://www.importnew.com/9501.html
 - http://www.ibm.com/developerworks/cn/java/j-lo-junit44/

多线程下的junit

 - https://github.com/junit-team/junit/wiki/Multithreaded-code-and-concurrency