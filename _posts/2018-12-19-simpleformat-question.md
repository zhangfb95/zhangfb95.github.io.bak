---
layout: post
title:  SimpleDateFormat安全性问题
date:   2018-12-19 15:46:00 +0800
categories: Java基本技能
tag: 工具
---

* content
{:toc}

## demo演示

我们都知道DateFormat和SimpleDateFormat都是非线程安全的类，在多线程环境下并发访问会出现问题。现以一个demo引出。

```java
public class Test {
    private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static  String formatDate(Date date)throws ParseException {
        return sdf.format(date);
    }

    public static Date parse(String strDate) throws ParseException{

        return sdf.parse(strDate);
    }

    public static void main(String[] args) throws Exception {
        Thread[] threads = new Thread[3];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread() {
                @Override public void run() {
                    try {
                        parse("2018-09-13 16:03:03");
                    } catch (ParseException e) {
                        e.printStackTrace();
                    }
                }
            };
        }
        for (Thread thread : threads) {
            thread.start();
        }
        Thread.sleep(30000L);
    }
}
```

我们可以看到有异常抛出，从而证明了我们的猜想。

```
Exception in thread "Thread-1" Exception in thread "Thread-2" java.lang.NumberFormatException: multiple points
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at com.keruyun.core.trade.biz.atomic.trade.action.Test.parse(Test.java:20)
	at com.keruyun.core.trade.biz.atomic.trade.action.Test$1.run(Test.java:29)
java.lang.NumberFormatException: multiple points
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at com.keruyun.core.trade.biz.atomic.trade.action.Test.parse(Test.java:20)
	at com.keruyun.core.trade.biz.atomic.trade.action.Test$1.run(Test.java:29)
```

## 为什么它是不是线程安全的呢？

我们看一下`format()`方法，关键的方法调用`calendar.setTime(date);`，calendar在并发地写数据，而`subFormat`方法又在读calendar的数据。从而出现线程安全的问题。

```java
public StringBuffer format(Date date, StringBuffer toAppendTo,
                               FieldPosition pos)
{
	pos.beginIndex = pos.endIndex = 0;
	return format(date, toAppendTo, pos.getFieldDelegate());
}

// Called from Format after creating a FieldDelegate
private StringBuffer format(Date date, StringBuffer toAppendTo,
							FieldDelegate delegate) {
	// Convert input date to time field list
	calendar.setTime(date);

	boolean useDateFormatSymbols = useDateFormatSymbols();

	for (int i = 0; i < compiledPattern.length; ) {
		int tag = compiledPattern[i] >>> 8;
		int count = compiledPattern[i++] & 0xff;
		if (count == 255) {
			count = compiledPattern[i++] << 16;
			count |= compiledPattern[i++];
		}

		switch (tag) {
			case TAG_QUOTE_ASCII_CHAR:
				toAppendTo.append((char)count);
				break;

			case TAG_QUOTE_CHARS:
				toAppendTo.append(compiledPattern, i, count);
				i += count;
				break;

			default:
				subFormat(tag, count, delegate, toAppendTo, useDateFormatSymbols);
				break;
		}
	}
	return toAppendTo;
}
```

## 如何解决？

1. 使用局部变量代替成员变量，我们知道局部变量是线程私有的，所以不会被多线程访问，故而也不会出现多线程问题
2. 由于`1`可能出现每个线程的每次调用都new一个对象的情况，为了改善此问题，可以使用ThreadLocal对一个线程的变量进行缓存
3. 因为多线程可以访问同一个变量会出问题，我们可以使用锁来将变量的访问变为串行
4. 废弃jdk自带的SimpleDateFormat，改为使用第三方安全的工具类，例如：Apache commons里的FastDateFormat或者Joda-Time。推荐使用Joda-Time。

## 参考link

https://blog.csdn.net/zxh87/article/details/19414885
