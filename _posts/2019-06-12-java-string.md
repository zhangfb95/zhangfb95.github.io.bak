---
layout: post
title:  Java中的String
date:   2019-06-12 00:09:00 +0800
categories: Java基本技能
tag: java
---

* content
{:toc}

## 前言

除了上篇讲述的基本类型及其包装类型之外，字符串`String`在Java中的使用频率也相当地高。那么，`String`到底是什么，它又涉及到哪些知识点呢？楼主通过很多网上的`String`源码分析文章，加上自己对于源码的阅读理解，综合得出以下内容：

1. 概述说明
2. 实现的接口
3. 成员变量
4. 静态内部类
5. 成员方法
6. 类方法

这些内容相对独立，但又有一些关联。接下来，让我们一起来看看吧！

## 概述说明

`String`这个类代表了字符串。Java编程中的所有字符串字面值都是这个类的实例。
`String`是常量，一旦被创建，其值将不可变。`StringBuilder`和`StringBuffer`支持可变的字符串。由于`String`是不可变的，所以可以被安全地共享使用。例如：

```java
String str = "abc";
// 等价于
char data[] = {'a', 'b', 'c};
String str = new String(data);
```

我们还可以列出更多字符串使用的例子。

```java
System.out.println("abc");
String cde = "cde";
System.out.println("abc" + cde);
String c = "abc".substring(2,3);
String d = cde.substring(1, 2);
```
`String`包含了各类方法，主要体现如下：

1. 检查其内包含的字符(串)
2. 比较字符串
3. 搜索字符串
4. 提取子串
5. 创建转换为全大写或小写的字符串拷贝
6. 支持基于Unicode标准版本的映射操作，内部通过调用`Character`的方法来实现

Java语言给`String+操作符`提供了支持，这种方式可以用于将其他`Object`类型的对象转换为`String`类型，主要是通过调用其他`Object`类型对象的`toString()`方法来实现。字符串的连接和追加，还可以使用`StringBuffer`或`StringBuilder`的`append()`方法来实现。

除非特别说明，构造字符串对象时传入null将抛出空指针异常 - `NullPointerException`。

一个`String`通常还可用于代表一个UTF-16格式的字符串，主要是通过补充`Character`的方式来实现的，使用两个Character来表示。更详细的说明，可参考链接 - [java增补字符](https://blog.csdn.net/r2100/article/details/83120631)。

## 实现的接口

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
}
```
1. `java.io.Serializable`，标识这是一个可序列化的类，该接口没有任何需要实现的方法。
2. `Comparable<String>`，有且仅有一个方法，`public int compareTo(T o)`。可比较两个实例化对象的大小，常用于排序场景。
3. `CharSequence`，用于标识一个只读的字符序列。值得说明一下的是，`StringBuilder`和`StringBuffer`也实现了该接口。`CharSequence`主要有以下方法：
    + `int length();` -- 获取长度
    + `char charAt(int index);` -- 获取指定下标的字符
    + `CharSequence subSequence(int start, int end);` -- 截取指定范围的字符序列
    + `public String toString();` -- Object的方法

## 成员变量

```java
 /** The value is used for character storage. */
private final char value[];

/** Cache the hash code for the string */
private int hash; // Default to 0

/** use serialVersionUID from JDK 1.0.2 for interoperability */
private static final long serialVersionUID = -6849794470754667710L;

public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                         = new CaseInsensitiveComparator();
```

`value[]`，表示这是一个字符数组，字符串的内容正是存储在这儿，比如`abc`，其实就是将a/b/c三个字符分别存储到数组下标为1/2/3的位置。
`hash`，字符串的散列码，在字符串的`hashCode()`方法第一次调用时就会生成并缓存。之所以升级为成员变量，主要原因是为了提高效率，降低计算散列码的开销。
`serialVersionUID`，用于java自带的序列化和反序列化机制，现在已经很少使用了。
`CASE_INSENSITIVE_ORDER`，是一个全局的、线程安全的字符串比较器对象，用于忽略大小写的字符串比较。

## 静态内部类

String只有一个静态内部类。在《成员变量》一节，我们提到了`CASE_INSENSITIVE_ORDER`，其类型为`CaseInsensitiveComparator`，这个类型就是`String`的静态内部类。

```java
private static class CaseInsensitiveComparator
        implements Comparator<String>, java.io.Serializable {
    // use serialVersionUID from JDK 1.2.2 for interoperability
    private static final long serialVersionUID = 8575799808933029326L;

    public int compare(String s1, String s2) {
        int n1 = s1.length();
        int n2 = s2.length();
        int min = Math.min(n1, n2);
        for (int i = 0; i < min; i++) {
            char c1 = s1.charAt(i);
            char c2 = s2.charAt(i);
            if (c1 != c2) {
                c1 = Character.toUpperCase(c1);
                c2 = Character.toUpperCase(c2);
                if (c1 != c2) {
                    c1 = Character.toLowerCase(c1);
                    c2 = Character.toLowerCase(c2);
                    if (c1 != c2) {
                        // No overflow because of numeric promotion
                        return c1 - c2;
                    }
                }
            }
        }
        return n1 - n2;
    }

    /** Replaces the de-serialized object. */
    private Object readResolve() { return CASE_INSENSITIVE_ORDER; }
}
```

在`String`中，比较方法主要有两个 - `compareTo()`和`compareToIgnoreCase()`，分别用于不忽略大小写和忽略大小写的比较。

## 成员方法

`String`的成员方法非常多，主要围绕着其持有的字符数组来展开的。通过对这些方法的用途和实现的了解，我们可以对他们进行简单的分组，每一组代表一类操作。

#### 1. 构造方法

`String`有很多重载的构造方法，这些方法支持很多类型的对象，例如：String、char[]、byte[]、int[]和StringBuilder。归根结底，还是将接受到的参数进行传递或转换，然后赋值给成员变量`value`和`hash`。

```java
public String() {
    this.value = "".value;
}
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
```

#### 2. 字符长度

`length()`用于获取字符串的长度，`isEmpty()`用于判断字符串是否为空，而`charAt()`则用于获取指定位置的字符。

```java
public int length() {
    return value.length;
}
public boolean isEmpty() {
    return value.length == 0;
}
public char charAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return value[index];
}
```

#### 3. 代码点

```java
// 返回指定位置的代码点
public int codePointAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointAtImpl(value, index, value.length);
}

// 返回指定位置的前一个代码点
public int codePointBefore(int index) {
    int i = index - 1;
    if ((i < 0) || (i >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointBeforeImpl(value, index, 0);
}

// 返回开始到结束段内的代码点个数
public int codePointCount(int beginIndex, int endIndex) {
    if (beginIndex < 0 || endIndex > value.length || beginIndex > endIndex) {
        throw new IndexOutOfBoundsException();
    }
    return Character.codePointCountImpl(value, beginIndex, endIndex - beginIndex);
}

// 返回指定位置加上codePointOffset后得到的代码点个数
public int offsetByCodePoints(int index, int codePointOffset) {
    if (index < 0 || index > value.length) {
        throw new IndexOutOfBoundsException();
    }
    return Character.offsetByCodePointsImpl(value, 0, value.length,
            index, codePointOffset);
}
```

这些方法都和代码点有关，且底层是通过调用`Character`静态方法来实现的。这些方法平时使用得非常少，主要用于基于语言的字符集扩展。楼主对此了解得比较浅，因此，借助于网上的文章来说明代码点的意思和用途：

> 这里说明一下，16位Unicode编码的所有 65, 536 个字符并不能完全表示全世界所有正在使用或曾经使用的字符。于是，Unicode 标准已扩展到包含多达 1, 112, 064 个字符。那些超出原来的16 位限制的字符被称作增补字符。Java的char类型是固定16bits的。代码点在U+0000 — U+FFFF之内到是可以用一个char完整的表示出一个字符。但代码点在U+FFFF之外的，一个char无论如何无法表示一个完整字符。这样用char类型来获取字符串中的那些代码点在U+FFFF之外的字符就会出现问题。<br>
增补字符是代码点在 U+10000 至 U+10FFFF 范围之间的字符，也就是那些使用原始的 Unicode 的 16 位设计无法表示的字符。从 U+0000 至 U+FFFF 之间的字符集有时候被称为基本多语言面 （BMP UBasic Multilingual Plane ）。因此，每一个 Unicode 字符要么属于 BMP，要么属于增补字符。

#### 4. getChars()

```java
void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```

这两个方法用于获取指定范围的字符数组，并存放在dst字符数组中。最终都调用了`System.arraycopy()`方法，这个方法是一个native方法，用于数组的拷贝。

#### 5. getBytes()

```java
/**
 * @deprecated  This method does not properly convert characters into
 * bytes.  As of JDK&nbsp;1.1, the preferred way to do this is via the
 * {@link #getBytes()} method, which uses the platform's default charset.
 * 废弃原因，这个方法不能正确地将字符数组转换为字节数组，
 * 因此从JDK1.1版本开始，可使用其他的`getByte()`方法来转换。
 */
@Deprecated
public void getBytes(int srcBegin, int srcEnd, byte dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    Objects.requireNonNull(dst);

    int j = dstBegin;
    int n = srcEnd;
    int i = srcBegin;
    char[] val = value;   /* avoid getfield opcode */

    while (i < n) {
        dst[j++] = (byte)val[i++];
    }
}
public byte[] getBytes(String charsetName)
        throws UnsupportedEncodingException {
    if (charsetName == null) throw new NullPointerException();
    return StringCoding.encode(charsetName, value, 0, value.length);
}
public byte[] getBytes(Charset charset) {
    if (charset == null) throw new NullPointerException();
    return StringCoding.encode(charset, value, 0, value.length);
}
public byte[] getBytes() {
    return StringCoding.encode(value, 0, value.length);
}
```

这几个方法用于将字符串转换为二进制byte[]数组。归根结底，还是调用了`StringCoding.encode()`静态方法。

#### 6. equals() && hashCode()

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

`equals()`方法用于比较当前字符串对象和另一个对象（可能不是字符串类型）是否相等。其主要逻辑如下：

1. 比较两个对象引用的地址是否相同
2. 比较`anObject`的类型是否为`String`
3. 比较字符数组长度是否相同
4. 遍历字符数组，依次比较相同下标的字符是否相同

步骤3和步骤4其实可以合并为一步操作，也就是遍历字符进行比较，但是性能会比分成两步低。所以，从这一点小的细节也能看出Java开发者的良苦用心。

`hashCode()`用于生成散列码，主要用于散列表。该方法会将生成的结果缓存到hash变量中，以空间换时间，从而提高效率。该方法的计算公式如下：

> s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]

除了`equals()`之外，还有一些和比较相关的方法，这些方法主要用于`基于内容`的比较和`忽略大小写`的比较，我们来看一下。

```java
public boolean contentEquals(StringBuffer sb) {
    return contentEquals((CharSequence)sb);
}
private boolean nonSyncContentEquals(AbstractStringBuilder sb) {
    char v1[] = value;
    char v2[] = sb.getValue();
    int n = v1.length;
    if (n != sb.length()) {
        return false;
    }
    for (int i = 0; i < n; i++) {
        if (v1[i] != v2[i]) {
            return false;
        }
    }
    return true;
}
public boolean contentEquals(CharSequence cs) {
    // Argument is a StringBuffer, StringBuilder
    if (cs instanceof AbstractStringBuilder) {
        if (cs instanceof StringBuffer) {
            synchronized(cs) {
               return nonSyncContentEquals((AbstractStringBuilder)cs);
            }
        } else {
            return nonSyncContentEquals((AbstractStringBuilder)cs);
        }
    }
    // Argument is a String
    if (cs instanceof String) {
        return equals(cs);
    }
    // Argument is a generic CharSequence
    char v1[] = value;
    int n = v1.length;
    if (n != cs.length()) {
        return false;
    }
    for (int i = 0; i < n; i++) {
        if (v1[i] != cs.charAt(i)) {
            return false;
        }
    }
    return true;
}
```

在《概述说明》一节，我们提到了一点，`StringBuilder`和`StringBuffer`其实也是`CharSequence`的实现。因此，我们主要看这个方法 - `public boolean contentEquals(CharSequence cs)`。

它做了几件事情：

1. 如果对象为`AbstractStringBuilder`的实例，那么调用`nonSyncContentEquals()`来比较。这儿需要注意一点，如果对象类型为`StringBuffer`，需要将对象加锁，至于为什么，楼主还未想明白。
2. 如果对象为`String`，那么直接调用`equals()`方法进行比较。
3. 比较两个对象的长度是否一致。
4. 遍历字符进行比较。

```java
public boolean equalsIgnoreCase(String anotherString) {
    return (this == anotherString) ? true
            : (anotherString != null)
            && (anotherString.value.length == value.length)
            && regionMatches(true, 0, anotherString, 0, value.length);
}
```

这个方法用于忽略大小写的比较，除了判断引用和长度之外，额外还可能调用`regionMatches()`进行比较。

#### 7. regionMatches()

该方法有两个重载方法。阅读代码之后，我们知道其主要实现思路是通过while循环来比较每个字符是否相等。忽略大小写的范围匹配，则是通过调用`Character.toUpperCase()`的静态方法，来将当前字符转为大写字符，然后进行比较。

```java
public boolean regionMatches(boolean ignoreCase, int toffset,
        String other, int ooffset, int len) {
    char ta[] = value;
    int to = toffset;
    char pa[] = other.value;
    int po = ooffset;
    // Note: toffset, ooffset, or len might be near -1>>>1.
    if ((ooffset < 0) || (toffset < 0)
            || (toffset > (long)value.length - len)
            || (ooffset > (long)other.value.length - len)) {
        return false;
    }
    while (len-- > 0) {
        char c1 = ta[to++];
        char c2 = pa[po++];
        if (c1 == c2) {
            continue;
        }
        if (ignoreCase) {
            // If characters don't match but case may be ignored,
            // try converting both characters to uppercase.
            // If the results match, then the comparison scan should
            // continue.
            char u1 = Character.toUpperCase(c1);
            char u2 = Character.toUpperCase(c2);
            if (u1 == u2) {
                continue;
            }
            // Unfortunately, conversion to uppercase does not work properly
            // for the Georgian alphabet, which has strange rules about case
            // conversion.  So we need to make one last check before
            // exiting.
            if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                continue;
            }
        }
        return false;
    }
    return true;
}

public boolean regionMatches(int toffset, String other, int ooffset,
        int len) {
    char ta[] = value;
    int to = toffset;
    char pa[] = other.value;
    int po = ooffset;
    // Note: toffset, ooffset, or len might be near -1>>>1.
    if ((ooffset < 0) || (toffset < 0)
            || (toffset > (long)value.length - len)
            || (ooffset > (long)other.value.length - len)) {
        return false;
    }
    while (len-- > 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}
```

#### 8. startsWith() && endWith()

```java
public boolean startsWith(String prefix, int toffset) {
    char ta[] = value;
    int to = toffset;
    char pa[] = prefix.value;
    int po = 0;
    int pc = prefix.value.length;
    // Note: toffset might be near -1>>>1.
    if ((toffset < 0) || (toffset > value.length - pc)) {
        return false;
    }
    while (--pc >= 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}
public boolean startsWith(String prefix) {
    return startsWith(prefix, 0);
}
public boolean endsWith(String suffix) {
    return startsWith(suffix, value.length - suffix.value.length);
}
```

`startWith()`用于判断字符串是否以指定前缀开头。而`endsWith()`用于判断字符串是否以指定后缀结尾，其最终调用的是`startWith()`方法。

#### 9. indexOf() && lastIndexOf()

```java
public int indexOf(int ch) {
    return indexOf(ch, 0);
}
public int indexOf(int ch, int fromIndex) {
    final int max = value.length;
    if (fromIndex < 0) {
        fromIndex = 0;
    } else if (fromIndex >= max) {
        // Note: fromIndex might be near -1>>>1.
        return -1;
    }

    if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
        // handle most cases here (ch is a BMP code point or a
        // negative value (invalid code point))
        final char[] value = this.value;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == ch) {
                return i;
            }
        }
        return -1;
    } else {
        return indexOfSupplementary(ch, fromIndex);
    }
}
private int indexOfSupplementary(int ch, int fromIndex) {
    if (Character.isValidCodePoint(ch)) {
        final char[] value = this.value;
        final char hi = Character.highSurrogate(ch);
        final char lo = Character.lowSurrogate(ch);
        final int max = value.length - 1;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == hi && value[i + 1] == lo) {
                return i;
            }
        }
    }
    return -1;
}
```

`indexOf()`，这个方法的作用是，如果没有检索到相应对象，那么返回-1，否则返回指定位置的下标。

另外，该方法最关键的地方在于`if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT)`，其作用为判断是否为增补字符。如果是的话，则需要调用`indexOfSupplementary()`方法来检索增补字符。

```java
/**
 * The minimum value of a
 * <a href="http://www.unicode.org/glossary/#supplementary_code_point">
 * Unicode supplementary code point</a>, constant {@code U+10000}.
 *
 * @since 1.5
 */
public static final int MIN_SUPPLEMENTARY_CODE_POINT = 0x010000;
```

接下来，我们再来看看`lastIndexOf()`方法。这个方法的实现恰好和`indexOf()`方法相反。

```java
public int lastIndexOf(int ch) {
    return lastIndexOf(ch, value.length - 1);
}
public int lastIndexOf(int ch, int fromIndex) {
    if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
        // handle most cases here (ch is a BMP code point or a
        // negative value (invalid code point))
        final char[] value = this.value;
        int i = Math.min(fromIndex, value.length - 1);
        for (; i >= 0; i--) {
            if (value[i] == ch) {
                return i;
            }
        }
        return -1;
    } else {
        return lastIndexOfSupplementary(ch, fromIndex);
    }
}
private int lastIndexOfSupplementary(int ch, int fromIndex) {
    if (Character.isValidCodePoint(ch)) {
        final char[] value = this.value;
        char hi = Character.highSurrogate(ch);
        char lo = Character.lowSurrogate(ch);
        int i = Math.min(fromIndex, value.length - 2);
        for (; i >= 0; i--) {
            if (value[i] == hi && value[i + 1] == lo) {
                return i;
            }
        }
    }
    return -1;
}
```

另外，`indexOf()`和`lastIndexOf()`还支持`String`类型的检索。其最终实现是通过获取字符串的字符数组，然后进行while循环来实现的。限于篇幅，本文不再粘贴此部分的源码。

#### 10. substring() && subSequence()

```java
public String substring(int beginIndex, int endIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    if (endIndex > value.length) {
        throw new StringIndexOutOfBoundsException(endIndex);
    }
    int subLen = endIndex - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
            : new String(value, beginIndex, subLen);
}

public CharSequence subSequence(int beginIndex, int endIndex) {
    return this.substring(beginIndex, endIndex);
}

public String(char value[], int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```

`subSequence()`最终调用了`substring()`。而`substring()`最终通过`String`构造方法来截取字符串的范围。

而`String`的构造方法最终是通过`Arrays.copyOfRange()`这个静态方法来实现的，而这个静态方法又是通过`System.arraycopy()`来实现的。

#### 11. concat()

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}
```

`concat()`用于把参数拼接到当前字符串的后面，从而形成一个新的字符串。

#### 12. replace()

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        while (++i < len) {
            if (val[i] == oldChar) {
                break;
            }
        }
        if (i < len) {
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            return new String(buf, true);
        }
    }
    return this;
}
```

这个方法用于将第一个检索到的旧字符替换为新字符。如果旧字符和新字符一样，或者旧字符没有检索到，那么直接返回原有的字符串。

#### 13. matches()

```java
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
public String replaceFirst(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
}
public String replaceAll(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceAll(replacement);
}
public String replace(CharSequence target, CharSequence replacement) {
    return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(
            this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
}
```

这儿为啥要将`matches()`方法和`replaceXxx()`方法一起来说明呢，主要原因在于他们底层调用的方法都是`Pattern.compile()`。`matches()`用于判断字符串是否匹配指定的正则表达式。`replaceXxx()`则用于将当前字符串中匹配到的子串替换为给定的字符串或字符序列。

#### 14. contains()

```java
public boolean contains(CharSequence s) {
    return indexOf(s.toString()) > -1;
}
```

这方法底层通过`indexOf()`来实现，用于检索指定的字符序列是否存在。

#### 15. split()

```java
public String[] split(String regex, int limit) {
    /* fastpath if the regex is a
     (1)one-char String and this character is not one of the
        RegEx's meta characters ".$|()[{^?*+\\", or
     (2)two-char String and the first char is the backslash and
        the second is not the ascii digit or ascii letter.
     */
    char ch = 0;
    if (((regex.value.length == 1 &&
         ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
         (regex.length() == 2 &&
          regex.charAt(0) == '\\' &&
          (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
          ((ch-'a')|('z'-ch)) < 0 &&
          ((ch-'A')|('Z'-ch)) < 0)) &&
        (ch < Character.MIN_HIGH_SURROGATE ||
         ch > Character.MAX_LOW_SURROGATE))
    {
        int off = 0;
        int next = 0;
        boolean limited = limit > 0;
        ArrayList<String> list = new ArrayList<>();
        while ((next = indexOf(ch, off)) != -1) {
            if (!limited || list.size() < limit - 1) {
                list.add(substring(off, next));
                off = next + 1;
            } else {    // last one
                //assert (list.size() == limit - 1);
                list.add(substring(off, value.length));
                off = value.length;
                break;
            }
        }
        // If no match was found, return this
        if (off == 0)
            return new String[]{this};

        // Add remaining segment
        if (!limited || list.size() < limit)
            list.add(substring(off, value.length));

        // Construct result
        int resultSize = list.size();
        if (limit == 0) {
            while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
                resultSize--;
            }
        }
        String[] result = new String[resultSize];
        return list.subList(0, resultSize).toArray(result);
    }
    return Pattern.compile(regex).split(this, limit);
}
public String[] split(String regex) {
    return split(regex, 0);
}
```

`split()`方法比较复杂，复杂在哪儿呢？主要在于条件判断。忽略条件判断，最核心的代码在于最后一行。即：通过正则表达式去编译解析字符串，然后将其拆分。

```java
return Pattern.compile(regex).split(this, limit);
```

既然条件判断很复杂，那么，我们就将其分解来看一看。

a、如果regex只有一位，且不为特殊字符。特殊字符为：`.$|()[{^?*+\`
b、如regex有两位，第一位为转义字符且第二位不是数字或字母。`|`表示或，即只要ch小于0或者大于9任一条件成立，小于a或者大于z任一条件成立，小于A或大于Z任一条件成立
c、第三个条件是不属于utf-16之间的字符

囊括起来就是：(a || b) && c，主要作用为过滤特殊情况。其实也就是使用一个ArrayList<String>存放每一段找到分割点的字符串，不断地循环。

#### 16. trim()

```java
public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */

    while ((st < len) && (val[st] <= ' ')) {
        st++;
    }
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
```

这个方法用于去掉首尾两端的空白字符（包括空格）。两个while()循环使用得非常巧妙，既不会过多地执行命令，也不会因为遗漏而出现程序bug。

#### 17. toUpperCase() && toLowerCase()

这两个方法用于将字符串中的字符转换为大写或小写的形式。由于其涉及到增补字符，代码变得比较复杂且多，这儿不再粘贴出这部分的代码，如果读者对此感兴趣，可自行阅读源码。

#### 18. toCharArray()

```java
public char[] toCharArray() {
    // Cannot use Arrays.copyOf because of class initialization order issues
    char result[] = new char[value.length];
    System.arraycopy(value, 0, result, 0, value.length);
    return result;
}
```

这个方法用于返回字符数组，但是其内部实现又不是直接将成员变量`value`直接返回，而是通过数组拷贝的方式，目的是为了避免可能出现的字符修改，从而违反`String`的不可变特性。

#### 19. toString()

```java
public String toString() {
    return this;
}
```

返回自身。

#### 20. intern()

```java
/**
 * Returns a canonical representation for the string object.
 * <p>
 * A pool of strings, initially empty, is maintained privately by the
 * class {@code String}.
 * <p>
 * When the intern method is invoked, if the pool already contains a
 * string equal to this {@code String} object as determined by
 * the {@link #equals(Object)} method, then the string from the pool is
 * returned. Otherwise, this {@code String} object is added to the
 * pool and a reference to this {@code String} object is returned.
 * <p>
 * It follows that for any two strings {@code s} and {@code t},
 * {@code s.intern() == t.intern()} is {@code true}
 * if and only if {@code s.equals(t)} is {@code true}.
 * <p>
 * All literal strings and string-valued constant expressions are
 * interned. String literals are defined in section 3.10.5 of the
 * <cite>The Java&trade; Language Specification</cite>.
 *
 * @return  a string that has the same contents as this string, but is
 *          guaranteed to be from a pool of unique strings.
 */
public native String intern();
```

这个方法是一个本地方法， 代码是使用c语言实现的，因此，唯一能了解此方法用途的渠道就只有看javadoc。

通过字面意思，我们知道这个方法用于返回内部字符串，也就是常量池引用。

当这个方法被调用，如果池子里已经包含了一个对象，池中对象equals当前对象，那么返回的对象就是池子里面的这个对象。如果池子里面不包含这样的对象，那么当前对象就放入池子，然后返回当前对象。

两个字符串对象，s和t。`s.intern() == t.intern()`为true的充要条件是`s.equals(t)`为true。

## 类方法

#### 1. format()

```java
public static String format(String format, Object... args) {
    return new Formatter().format(format, args).toString();
}
public static String format(Locale l, String format, Object... args) {
    return new Formatter(l).format(format, args).toString();
}
```

这两个方法用于将字符串进行格式化，何为格式化呢？我们举一个例子来看看。更详细的说明，可参考链接 - [String.format()的详细用法](https://blog.csdn.net/anita9999/article/details/82346552)

```java
public static void main(String[] args) {
    String format = "this is %dst program for %s";
    System.out.println(String.format(format, 1, "hello world"));
}

// 输出结果如下：
// this is 1st program for hello world
```

#### 2. valueOf()

这个方法在`String`里面有很多个重载方法，这些重载方法的作用大体相同。也就是说，将基本数据类型和`Object`类型的对象`构造`成字符串对象。

```java
public static String valueOf(Object obj) {
    return (obj == null) ? "null" : obj.toString();
}
public static String valueOf(char data[]) {
    return new String(data);
}
public static String valueOf(char data[], int offset, int count) {
    return new String(data, offset, count);
}
public static String valueOf(boolean b) {
    return b ? "true" : "false";
}
public static String valueOf(char c) {
    char data[] = {c};
    return new String(data, true);
}
public static String valueOf(int i) {
    return Integer.toString(i);
}
public static String valueOf(long l) {
    return Long.toString(l);
}
public static String valueOf(float f) {
    return Float.toString(f);
}
public static String valueOf(double d) {
    return Double.toString(d);
}
```

## 总结

本文通过对Java中使用频率非常高的`String`进行了全面透彻的分析。通过源码，我们了解到了`String`的实现。也看到了技术大牛是如何在char数组的基础上玩出了各种花样。真是太厉害了，感谢Java开发团队！

## 参考文献

+ [Java-- String源码分析](https://www.cnblogs.com/listenfwind/p/8450241.html)
+ [【Java源码分析】Java8的String源码分析](https://blog.csdn.net/snailmann/article/details/80882719)
+ [Oracle - String api](https://docs.oracle.com/javase/8/docs/api/)
+ [JDK1.8源码(三)——java.lang.String 类](https://www.cnblogs.com/ysocean/p/8571426.html)
+ [java增补字符](https://blog.csdn.net/r2100/article/details/83120631)
+ [String：String类型为什么不可变](https://segmentfault.com/a/1190000009914328)