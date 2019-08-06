---
layout: post
title:  Java基本数据类型及其包装类型
date:   2019-06-05 00:03:00 +0800
categories: Java基本技能
tag: java
---

* content
{:toc}

## 前言

Java基础有很多方面，基本数据类型及其包装类型算是其一，我们必须得掌握。

## 一、基本数据类型

在Java中，基本数据类型可分为以下几类：

+ 数值型
  + 整数类型
    + byte
    + short
    + int
    + long
  + 浮点类型
    + float
    + double
+ 字符型
  + char
+ 布尔型
  + boolean

而整数类型又可使用三种进制来表示。

1. 十进制 （默认）
2. 八进制 （以0开头）
3. 十六进制 （以0X或0x开头）

整数类型占用的内存及范围如下所示。默认的整数类型为`int`，如果要使用`long`，需要在数值末尾加上小写的l或大写的L，推荐使用大写的L，避免和字母i及其大写字母I混淆。

| 整数类型 | 内存 | 取值范围 |
| --- | --- | --- |
| byte | 8 bit | -128 - 127 |
| short | 16 bit | -32768 - 32767 |
| int | 32 bit | -2^32 - 2^32-1 |
| long | 64 bit | -2^32 - 2^32-1 |

对于浮点类型，默认是`double`类型，如果要使用`float`，需在数值末尾加上小写的f或大写的F，推荐使用大写的F。

boolean类型只有1位，false用0表示，true用1表示。

char类型用于表示unicode表中的所有字符，unicode表几乎涵盖了所有国家所有语言的所有字符。其范围为0 - 65535。char为字符串的组成单元，有着举足轻重的作用。

char本质上还是一个数值，表示unicode表中的位置，例如下面的两个语句其实是等价的。

```java
char ch = 'a';
char ch = 97;
```

另外，在数值未越界的情况下，char和int可以自动转换。

```java
char word = 'd';
int p = 23045;
System.out.println("word is " + (int)word);
System.out.println("p is " + (char)p);
```

Java中有一些转意字符，有特殊的含义，例如：

1. \\' -- 单引号
2. \\\ -- 反斜杠
3. \t -- 跳格
4. \r -- 回车
5. \n -- 换行
6. \b -- 退格
7. \f -- 换页

## 二、包装类型

上面我们提到了8种基本数据类型，而每种数据类型又对应一种包装类型，如下所示：

| 基本数据类型 | 包装类型 |
| --- | --- |
| boolean | Boolean |
| char | Character |
| int | Integer |
| byte | Byte |
| short | Short |
| long | Long |
| float | Float |
| double | Double |

为什么需要基本数据类型？

> 为了效率。new创建的对象存放在堆区，简单的小变量如果使用对象创建的方式，效率并不会很高。因此，这类变量是直接存放在栈区，从而提高了效率。

为什么需要包装类型？

> 1. 基本类型不具备对象的特性，比如有属性和方法。包装类型是为了让基本类型具有对象的特性；
> 2. 可以方便地使用泛型。例如，我们不能直接将基本数据类放入集合类。

为了灵活地支持包装类型，Java编译器提供了自动装箱和拆箱的机制。例如：

```java
Integer x = new Integer(36);
int y = x;
int z = 36;
Integer u = new Integer(z);
```

基本类型和包装类型有一些区别，主要体现为下面几点：

1. 基本类型不能使用new关键字来创建，包装类型必须使用new关键字来创建。
2. 存储的位置不同。基本类型存储在栈区，包装类型存储在堆区。
3. 初始值不同。基本类型中的int初始值为0，boolean初始值为false。包装类型初始值都为null。因此在进行自动装拆箱时一定要避免空指针异常。
4. 使用场景不同。基本类型主要用于计算和赋值。包装类型用于泛型。
5. 基本类型是值传递，包装类型是引用传递。

>【注】：应尽量避免使用`xxx == yyy`和`xxx.equals(yyy)`的方式进行比较，而是应该通过`Objects.equals(xxx, yyy)`的方式。

## 三、包装类型常用方法

### 1. Boolean

```java
public static final Boolean TRUE = new Boolean(true);
public static final Boolean FALSE = new Boolean(false);
public static final Class<Boolean> TYPE = (Class<Boolean>) Class.getPrimitiveClass("boolean");
public Boolean(boolean value) {
    this.value = value;
}
public Boolean(String s) {
    this(parseBoolean(s));
}
public static boolean parseBoolean(String s) {
    return ((s != null) && s.equalsIgnoreCase("true"));
}
public boolean booleanValue() {
    return value;
}
public static Boolean valueOf(boolean b) {
    return (b ? TRUE : FALSE);
}
public static Boolean valueOf(String s) {
    return parseBoolean(s) ? TRUE : FALSE;
}
public static String toString(boolean b) {
    return b ? "true" : "false";
}
public String toString() {
    return value ? "true" : "false";
}
public static boolean parseBoolean(String s) {
    return ((s != null) && s.equalsIgnoreCase("true"));
}
```

### 2. Integer

```java
public static final int MIN_VALUE = 0x80000000;
public static final int MAX_VALUE = 0x7fffffff;
public static final Class<Integer> TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
public static Integer valueOf(String s);
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
public int intValue();
public static int parseInt(String s) throws NumberFormatException {
    return parseInt(s,10);
}
```

### 3. Long

```java
public static final long MIN_VALUE = 0x8000000000000000L;
public static final long MAX_VALUE = 0x7fffffffffffffffL;
public static final Class<Long> TYPE = (Class<Long>) Class.getPrimitiveClass("long");
public static Long valueOf(String s);
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
public long longValue();
public static long parseLong(String s) throws NumberFormatException {
    return parseLong(s, 10);
}
```

### 4. Double

```java
/**
 * 用于表示正无穷大
 * A constant holding the positive infinity of type
 * {@code double}. It is equal to the value returned by
 * {@code Double.longBitsToDouble(0x7ff0000000000000L)}.
 */
public static final double POSITIVE_INFINITY = 1.0 / 0.0;

/**
 * 用于表示负无穷大
 * A constant holding the negative infinity of type
 * {@code double}. It is equal to the value returned by
 * {@code Double.longBitsToDouble(0xfff0000000000000L)}.
 */
public static final double NEGATIVE_INFINITY = -1.0 / 0.0;

/**
 * 用于表示非数字
 * A constant holding a Not-a-Number (NaN) value of type
 * {@code double}. It is equivalent to the value returned by
 * {@code Double.longBitsToDouble(0x7ff8000000000000L)}.
 */
public static final double NaN = 0.0d / 0.0;

/**
 * 最大值
 * A constant holding the largest positive finite value of type
 * {@code double},
 * (2-2<sup>-52</sup>)&middot;2<sup>1023</sup>.  It is equal to
 * the hexadecimal floating-point literal
 * {@code 0x1.fffffffffffffP+1023} and also equal to
 * {@code Double.longBitsToDouble(0x7fefffffffffffffL)}.
 */
public static final double MAX_VALUE = 0x1.fffffffffffffP+1023; // 1.7976931348623157e+308

/**
 * 最小的正值
 * A constant holding the smallest positive normal value of type
 * {@code double}, 2<sup>-1022</sup>.  It is equal to the
 * hexadecimal floating-point literal {@code 0x1.0p-1022} and also
 * equal to {@code Double.longBitsToDouble(0x0010000000000000L)}.
 *
 * @since 1.6
 */
public static final double MIN_NORMAL = 0x1.0p-1022; // 2.2250738585072014E-308

/**
 * 最小值
 * A constant holding the smallest positive nonzero value of type
 * {@code double}, 2<sup>-1074</sup>. It is equal to the
 * hexadecimal floating-point literal
 * {@code 0x0.0000000000001P-1022} and also equal to
 * {@code Double.longBitsToDouble(0x1L)}.
 */
public static final double MIN_VALUE = 0x0.0000000000001P-1022; // 4.9e-324

/**
 * 最大的指数
 * Maximum exponent a finite {@code double} variable may have.
 * It is equal to the value returned by
 * {@code Math.getExponent(Double.MAX_VALUE)}.
 *
 * @since 1.6
 */
public static final int MAX_EXPONENT = 1023;

/**
 * 最小的指数
 * Minimum exponent a normalized {@code double} variable may
 * have.  It is equal to the value returned by
 * {@code Math.getExponent(Double.MIN_NORMAL)}.
 *
 * @since 1.6
 */
public static final int MIN_EXPONENT = -1022;

/**
 * double的长度
 * The number of bits used to represent a {@code double} value.
 *
 * @since 1.5
 */
public static final int SIZE = 64;

/**
 * double占用的字节数
 * The number of bytes used to represent a {@code double} value.
 *
 * @since 1.8
 */
public static final int BYTES = SIZE / Byte.SIZE;

/**
 * The {@code Class} instance representing the primitive type
 * {@code double}.
 *
 * @since JDK1.1
 */
@SuppressWarnings("unchecked")
public static final Class<Double>   TYPE = (Class<Double>) Class.getPrimitiveClass("double");
public static Double valueOf(String s);
public static Double valueOf(double d) {
    return new Double(d);
}
public double doubleValue() {
    return value;
}
public static double parseDouble(String s) throws NumberFormatException {
    return FloatingDecimal.parseDouble(s);
}
```

### 5. Float

```java
/**
 * A constant holding the positive infinity of type
 * {@code float}. It is equal to the value returned by
 * {@code Float.intBitsToFloat(0x7f800000)}.
 */
public static final float POSITIVE_INFINITY = 1.0f / 0.0f;

/**
 * A constant holding the negative infinity of type
 * {@code float}. It is equal to the value returned by
 * {@code Float.intBitsToFloat(0xff800000)}.
 */
public static final float NEGATIVE_INFINITY = -1.0f / 0.0f;

/**
 * A constant holding a Not-a-Number (NaN) value of type
 * {@code float}.  It is equivalent to the value returned by
 * {@code Float.intBitsToFloat(0x7fc00000)}.
 */
public static final float NaN = 0.0f / 0.0f;

/**
 * A constant holding the largest positive finite value of type
 * {@code float}, (2-2<sup>-23</sup>)&middot;2<sup>127</sup>.
 * It is equal to the hexadecimal floating-point literal
 * {@code 0x1.fffffeP+127f} and also equal to
 * {@code Float.intBitsToFloat(0x7f7fffff)}.
 */
public static final float MAX_VALUE = 0x1.fffffeP+127f; // 3.4028235e+38f

/**
 * A constant holding the smallest positive normal value of type
 * {@code float}, 2<sup>-126</sup>.  It is equal to the
 * hexadecimal floating-point literal {@code 0x1.0p-126f} and also
 * equal to {@code Float.intBitsToFloat(0x00800000)}.
 *
 * @since 1.6
 */
public static final float MIN_NORMAL = 0x1.0p-126f; // 1.17549435E-38f

/**
 * A constant holding the smallest positive nonzero value of type
 * {@code float}, 2<sup>-149</sup>. It is equal to the
 * hexadecimal floating-point literal {@code 0x0.000002P-126f}
 * and also equal to {@code Float.intBitsToFloat(0x1)}.
 */
public static final float MIN_VALUE = 0x0.000002P-126f; // 1.4e-45f

/**
 * Maximum exponent a finite {@code float} variable may have.  It
 * is equal to the value returned by {@code
 * Math.getExponent(Float.MAX_VALUE)}.
 *
 * @since 1.6
 */
public static final int MAX_EXPONENT = 127;

/**
 * Minimum exponent a normalized {@code float} variable may have.
 * It is equal to the value returned by {@code
 * Math.getExponent(Float.MIN_NORMAL)}.
 *
 * @since 1.6
 */
public static final int MIN_EXPONENT = -126;

/**
 * The number of bits used to represent a {@code float} value.
 *
 * @since 1.5
 */
public static final int SIZE = 32;

/**
 * The number of bytes used to represent a {@code float} value.
 *
 * @since 1.8
 */
public static final int BYTES = SIZE / Byte.SIZE;

/**
 * The {@code Class} instance representing the primitive type
 * {@code float}.
 *
 * @since JDK1.1
 */
@SuppressWarnings("unchecked")
public static final Class<Float> TYPE = (Class<Float>) Class.getPrimitiveClass("float");
public static Float valueOf(String s) throws NumberFormatException {
    return new Float(parseFloat(s));
}
public static Float valueOf(float f) {
    return new Float(f);
}
public static float parseFloat(String s) throws NumberFormatException {
    return FloatingDecimal.parseFloat(s);
}
public float floatValue() {
    return value;
}
```

### 6. Byte

```java
public static final byte MIN_VALUE = -128;
public static final byte MAX_VALUE = 127;
public static final Class<Byte>     TYPE = (Class<Byte>) Class.getPrimitiveClass("byte");
public static Byte valueOf(byte b) {
    final int offset = 128;
    return ByteCache.cache[(int)b + offset];
}
public static byte parseByte(String s) throws NumberFormatException {
    return parseByte(s, 10);
}
public static Byte valueOf(String s) throws NumberFormatException {
    return valueOf(s, 10);
}
public byte byteValue() {
    return value;
}
```

### 7. Short

```java
public static final short MIN_VALUE = -32768;
public static final short MAX_VALUE = 32767;
public static final Class<Short>    TYPE = (Class<Short>) Class.getPrimitiveClass("short");
public static short parseShort(String s) throws NumberFormatException {
    return parseShort(s, 10);
}
public static Short valueOf(String s) throws NumberFormatException {
    return valueOf(s, 10);
}
public static Short valueOf(short s) {
    final int offset = 128;
    int sAsInt = s;
    if (sAsInt >= -128 && sAsInt <= 127) { // must cache
        return ShortCache.cache[sAsInt + offset];
    }
    return new Short(s);
}
public short shortValue() {
    return value;
}
```

### 8. Character

```java
public static final Class<Character> TYPE = (Class<Character>) Class.getPrimitiveClass("char");
public static Character valueOf(char c) {
    if (c <= 127) { // must cache
        return CharacterCache.cache[(int)c];
    }
    return new Character(c);
}
public char charValue() {
    return value;
}
```

### 9. 简单总结

通过对常用方法的分析，我们看到所有的包装类型都有一些类似的字段或函数，他们主要的作用是简化我们处理基本类型和包装类型的操作。而自动装箱和拆箱正是其中比较重要的一种机制。

## 四、自动装箱和拆箱的原理

java为了简化我们使用基本类型和包装类型的复杂度，特意引入了装箱和拆箱机制。

1. 装箱机制 - 将基本数据类型转换为包装类型
2. 拆箱机制 - 将包装类型转换为基本数据类型

```java
public class Main {

    public static void main(String[] args) {
        Integer a = 32323;
        int b = a;
    }
}
```

上述代码，我们使用javap -c来反编译一下，看到如下结果：

```
C:\Users\Administrator>javap -c D:\zhangfb\ws\demo\target\classes\com\juconcurre
nt\demo\Main.class
Compiled from "Main.java"
public class com.juconcurrent.demo.Main {
  public com.juconcurrent.demo.Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":
()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: sipush        32323
       3: invokestatic  #2                  // Method java/lang/Integer.valueOf:
(I)Ljava/lang/Integer;
       6: astore_1
       7: aload_1
       8: invokevirtual #3                  // Method java/lang/Integer.intValue
:()I
      11: istore_2
      12: return
}
```

我们看到了`#2`和`#3`，他们分别对应的是装箱和拆箱。其实，装箱就是调用了`valueOf()`方法，而拆箱则调用了`intValue()`方法。与此类似，其他的类型有相同的机制。

## 总结

首先，我们引入了基本类型和包装类型，然后介绍了其异同和作用。
然后，了解了包装类型常用的字段和函数，他们的命名非常规范，函数的实现也非常简洁。
最后，我们通过自动装箱和拆箱机制，分析了这些函数在其中起到的作用。

## 参考

1. https://www.cnblogs.com/huajiezh/p/5225637.html
2. https://www.cnblogs.com/lxrm/p/6427628.html
3. https://blog.csdn.net/johntang2014/article/details/83027553