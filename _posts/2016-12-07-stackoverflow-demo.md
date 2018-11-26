---
layout: post
title:  "谨防stackOverFlowError陷阱"
date:   2016-12-07 10:20:37 +0200
uuid:   339105F6-879E-2846-B249-CD834D1D986A
categories: mix
---

---
layout: post
title:  谨防stackOverFlowError陷阱
date:   2016-12-07 10:20:37 +0800
categories: mix
tag: mix
---

* content
{:toc}

初学Java的时候，提到过一个异常`StackOverflowError`。我们知道此异常的意思，也大概知道什么时候会出现这个错误。
但是直到今天遇到一个场景，才知道这个异常的发生只需要一个很小的粗心！在此，以此博客记录下过程。

## 基础概念

`StackOverflowError`，又叫栈溢出错误。当一个应用递归得太深的时候，会抛出这个异常。

```java
// Thrown when a stack overflow occurs because an application
//  recurses too deeply
```

## 简单示例

示例代码如下

```java
public class StackOverFlowErrorTest {

    @Test
    public void test() {
        try {
            new Simple().call();
            Assert.fail();
        } catch (StackOverflowError e) {
            e.printStackTrace();
        }
    }

    private static class Simple {

        private void call() {
            callInner();
        }

        private void callInner() {
            call();
        }
    }
}
```

从代码中看出，`call()方法`调用了`callInner()方法`，而`callInner()方法`又反向调用了`call()`方法。
这样就会一直递归下去，直到栈空间满了抛出异常为止。而方法栈的大小可以由虚拟机参数`-Xss`来设置。默认为`1M`。

## 现实环境

我们做的是一个交易系统，这个系统使用`lombok`作为代码减写的工具，它可以减少我们对`toString`/`hashCode`/`equals`/`getter`/`setter`等的书写。
同时，我们还是用apache的开源类库`commons-beanutils`作为属性拷贝的工具。我们的代码大致如下：

```java
public class Test {

    @Data
    public static class Request<T> {
        private Map<String, Object> additionMap = new HashMap<>(8);
        private T content;
    }

    @Data
    public static class Trade {
        private Long id;
        private String uuid;
    }

    @Data
    public static class Payment {
        private Long paymentId;
    }

    /**
     * 创建订单并收银
     */
    private Request<Trade> createTradeAndPayment() {
        Request<Trade> tradeRequest = new Request<>();
        createPayment(tradeRequest);
        return tradeRequest;
    }

    /**
     * 收银操作
     *
     * @param tradeRequest 订单信息
     * @return 收银返回
     */
    private Request<Payment> createPayment(Request<Trade> tradeRequest) {
        Request<Payment> paymentRequest = new Request<>();
        paymentRequest.additionMap.put("trade", tradeRequest);
        try {
            BeanUtils.copyProperties(tradeRequest, paymentRequest);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return paymentRequest;
    }

    public static void main(String[] args) {
        System.out.println(new Test().createTradeAndPayment());
    }
}
```

上述代码中，我模拟了一些类，包括：请求、订单、收银。
同时模拟了一个操作————下单并收银，这个操作会先下单，在收银。可是在收银的时候，我们往附加map中添加了订单request对象。
这儿就造成了一个嵌套引用。`BeanUtils.copyProperties`进行属性拷贝的时候，对于`additionMap`只是拷贝了引用。
所以：

```
paymentRequest.additionMap -> A
tradeRequest.additionMap -> A (1)
A -> tradeRequest (2)
```
(1)和(2)已经形成了一个闭环了，而`toString`方法又会分别调用各自的引用，从而导致以下错误信息输出。

```java
Exception in thread "main" java.lang.StackOverflowError
	at java.lang.String.valueOf(String.java:2982)
	at java.lang.StringBuilder.append(StringBuilder.java:131)
	at com.keruyun.Test$Request.toString(Test.java:14)
	at java.lang.String.valueOf(String.java:2982)
	at java.lang.StringBuilder.append(StringBuilder.java:131)
	at java.util.AbstractMap.toString(AbstractMap.java:536)
	at java.lang.String.valueOf(String.java:2982)
	at java.lang.StringBuilder.append(StringBuilder.java:131)
	at com.keruyun.Test$Request.toString(Test.java:14)
```

## 特别注意

【注意】：不管是属性直接引用，还是容器引用，只要是嵌套引用，都需要特别小心和注意！！！防止出现这种异常的出现。