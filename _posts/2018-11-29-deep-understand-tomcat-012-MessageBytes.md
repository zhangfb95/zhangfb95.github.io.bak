---
layout: post
title:  深入理解Tomcat（12）拾遗-MessageBytes
date:   2018-12-03 11:10:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

Tomcat为了提高性能，在接受到socket传入的字节之后并不会马上进行编码转换，而是保持`byte[]`的方式，在用到的时候再进行转换。在tomcat的实现中，`MessageBytes`正是`byte[]`的抽象。本节我们就来深入了解一下！

## 如何使用？

我们通过一个简单的例子来看看`MessageBytes`是如何使用的。这个例子用于提取`byte[]`里面的一个`子byte[]`，然后打印输出。

```java
public static void main(String[] args) {
    // 构造`MessageBytes`对象
    MessageBytes mb = MessageBytes.newInstance();
    // 待测试的`byte[]`对象
    byte[] bytes = "abcdefg".getBytes(Charset.defaultCharset());
    // 调用`setBytes()`对bytes进行标记
    mb.setBytes(bytes, 2, 3);
    // 转换为字符串进行控制台输出
    System.out.println(mb.toString());
}
```

我们运行一下，看到下面输出，的确和我们预料的一致。

![效果图](https://upload-images.jianshu.io/upload_images/845143-017dc970880aef63.png?jianshufrom=1)

## 源码解读

无源码无真相，我们这就来分析一下源码。在MessageBytes里面一共有四种类型，用于表示消息的类型。

1. `T_NULL`表示空消息，即消息为`null`
2. `T_STR`表示消息为字符串类型
3. `T_BYTES`表示消息为字节数组类型
4. `T_CHARS`表示消息为字符数组类型

```java
// primary type ( whatever is set as original value )
private int type = T_NULL;

public static final int T_NULL = 0;
/** getType() is T_STR if the the object used to create the MessageBytes
    was a String */
public static final int T_STR  = 1;
/** getType() is T_BYTES if the the object used to create the MessageBytes
    was a byte[] */
public static final int T_BYTES = 2;
/** getType() is T_CHARS if the the object used to create the MessageBytes
    was a char[] */
public static final int T_CHARS = 3;
```

接着我们看看构造方法，默认构造方法居然是private类型的，同时提供了工厂方法用于创建`MessageBytes`实例。

```java
/**
 * Creates a new, uninitialized MessageBytes object.
 * Use static newInstance() in order to allow
 *   future hooks.
 */
private MessageBytes() {
}

/**
 * Construct a new MessageBytes instance.
 * @return the instance
 */
public static MessageBytes newInstance() {
    return factory.newInstance();
}
```

我们接着看看关键方法`setBytes()`，该方法内部调用了`ByteChunk.setBytes()`方法，同时设置了`type字段`。

```java
// Internal objects to represent array + offset, and specific methods
private final ByteChunk byteC=new ByteChunk();
private final CharChunk charC=new CharChunk();

/**
 * Sets the content to the specified subarray of bytes.
 *
 * @param b the bytes
 * @param off the start offset of the bytes
 * @param len the length of the bytes
 */
public void setBytes(byte[] b, int off, int len) {
    byteC.setBytes( b, off, len );
    type=T_BYTES;
    hasStrValue=false;
    hasHashCode=false;
    hasLongValue=false;
}
```

我们深入去看看`ByteChunk.setBytes()`。非常简单，就是设置一下`待标识的字节数组`、`开始位置`、`结束位置`。

```java
private byte[] buff; // 待标记的字节数组
protected int start; // 开始位置
protected int end; // 结束位置
protected boolean isSet; // 是否已经设置过了
protected boolean hasHashCode = false; // 是否有hashCode

/**
 * Sets the buffer to the specified subarray of bytes.
 *
 * @param b the ascii bytes
 * @param off the start offset of the bytes
 * @param len the length of the bytes
 */
public void setBytes(byte[] b, int off, int len) {
    buff = b;
    start = off;
    end = start + len;
    isSet = true;
    hasHashCode = false;
}
```

既然字节数组已经标识过了，那什么时候用呢？我们能想到的自然就是转换为String，而转换的地方一般是调用对象的`toString()`方法，我们来看看`MessageBytes.toString()`方法。

```java
@Override
public String toString() {
    if( hasStrValue ) {
        return strValue;
    }

    switch (type) {
    case T_CHARS:
        strValue=charC.toString();
        hasStrValue=true;
        return strValue;
    case T_BYTES:
        strValue=byteC.toString();
        hasStrValue=true;
        return strValue;
    }
    return null;
}
```

首先判断是否有缓存的字符串，有的话就直接返回，这也是提高性能的一种方式。其次是根据`type`来选择不同的`*Chunk`，然后调用其`toString()`方法。那么我们这儿选择`ByteChunk.toString()`来分析。

```java
@Override
public String toString() {
    if (null == buff) {
        return null;
    } else if (end - start == 0) {
        return "";
    }
    return StringCache.toString(this);
}
```

调用了`StringCache.toString(this)`。`StringCache`，顾名思义，就是对字符串进行缓存，不过本节我们主要分析`MessageBytes`，所以会忽略其缓存的代码。下面来看看这个方法，该方法非常长，前方高能！

```java
public static String toString(ByteChunk bc) {
    // If the cache is null, then either caching is disabled, or we're
    // still training
    if (bcCache == null) {
        String value = bc.toStringInternal();
        if (byteEnabled && (value.length() < maxStringSize)) {
            // If training, everything is synced
            synchronized (bcStats) {
                // If the cache has been generated on a previous invocation
                // while waiting for the lock, just return the toString
                // value we just calculated
                if (bcCache != null) {
                    return value;
                }
                // Two cases: either we just exceeded the train count, in
                // which case the cache must be created, or we just update
                // the count for the string
                if (bcCount > trainThreshold) {
                    long t1 = System.currentTimeMillis();
                    // Sort the entries according to occurrence
                    TreeMap<Integer,ArrayList<ByteEntry>> tempMap =
                            new TreeMap<>();
                    for (Entry<ByteEntry,int[]> item : bcStats.entrySet()) {
                        ByteEntry entry = item.getKey();
                        int[] countA = item.getValue();
                        Integer count = Integer.valueOf(countA[0]);
                        // Add to the list for that count
                        ArrayList<ByteEntry> list = tempMap.get(count);
                        if (list == null) {
                            // Create list
                            list = new ArrayList<>();
                            tempMap.put(count, list);
                        }
                        list.add(entry);
                    }
                    // Allocate array of the right size
                    int size = bcStats.size();
                    if (size > cacheSize) {
                        size = cacheSize;
                    }
                    ByteEntry[] tempbcCache = new ByteEntry[size];
                    // Fill it up using an alphabetical order
                    // and a dumb insert sort
                    ByteChunk tempChunk = new ByteChunk();
                    int n = 0;
                    while (n < size) {
                        Object key = tempMap.lastKey();
                        ArrayList<ByteEntry> list = tempMap.get(key);
                        for (int i = 0; i < list.size() && n < size; i++) {
                            ByteEntry entry = list.get(i);
                            tempChunk.setBytes(entry.name, 0,
                                    entry.name.length);
                            int insertPos = findClosest(tempChunk,
                                    tempbcCache, n);
                            if (insertPos == n) {
                                tempbcCache[n + 1] = entry;
                            } else {
                                System.arraycopy(tempbcCache, insertPos + 1,
                                        tempbcCache, insertPos + 2,
                                        n - insertPos - 1);
                                tempbcCache[insertPos + 1] = entry;
                            }
                            n++;
                        }
                        tempMap.remove(key);
                    }
                    bcCount = 0;
                    bcStats.clear();
                    bcCache = tempbcCache;
                    if (log.isDebugEnabled()) {
                        long t2 = System.currentTimeMillis();
                        log.debug("ByteCache generation time: " +
                                (t2 - t1) + "ms");
                    }
                } else {
                    bcCount++;
                    // Allocate new ByteEntry for the lookup
                    ByteEntry entry = new ByteEntry();
                    entry.value = value;
                    int[] count = bcStats.get(entry);
                    if (count == null) {
                        int end = bc.getEnd();
                        int start = bc.getStart();
                        // Create byte array and copy bytes
                        entry.name = new byte[bc.getLength()];
                        System.arraycopy(bc.getBuffer(), start, entry.name,
                                0, end - start);
                        // Set encoding
                        entry.charset = bc.getCharset();
                        // Initialize occurrence count to one
                        count = new int[1];
                        count[0] = 1;
                        // Set in the stats hash map
                        bcStats.put(entry, count);
                    } else {
                        count[0] = count[0] + 1;
                    }
                }
            }
        }
        return value;
    } else {
        accessCount++;
        // Find the corresponding String
        String result = find(bc);
        if (result == null) {
            return bc.toStringInternal();
        }
        // Note: We don't care about safety for the stats
        hitCount++;
        return result;
    }
}
```

看完了，第一感觉就是该方法好长啊！但是通过层层剥减，我们发现关键的地方不多，下面我们把非关键的地方去掉看看。

1. 省略掉的缓存部分
2. 这儿其实又调回了`ByteChunk`，调用的方法为`toStringInternal()`

```java
public static String toString(ByteChunk bc) {

    // If the cache is null, then either caching is disabled, or we're
    // still training
    if (bcCache == null) {
        String value = bc.toStringInternal();
        // 1. 省略掉的缓存部分
        return value;
    } else {
        accessCount++;
        // Find the corresponding String
        String result = find(bc);
        if (result == null) {
            // 2. 这儿其实又调回了`ByteChunk`，调用的方法为`toStringInternal()`
            return bc.toStringInternal();
        }
        // Note: We don't care about safety for the stats
        hitCount++;
        return result;
    }
}
```

我们看看`ByteChunk.toStringInternal()`方法，猜想这儿才是关键的地方！

```java
public String toStringInternal() {
    if (charset == null) {
        charset = DEFAULT_CHARSET;
    }
    // new String(byte[], int, int, Charset) takes a defensive copy of the
    // entire byte array. This is expensive if only a small subset of the
    // bytes will be used. The code below is from Apache Harmony.
    CharBuffer cb = charset.decode(ByteBuffer.wrap(buff, start, end - start));
    return new String(cb.array(), cb.arrayOffset(), cb.length());
}
```

果然如此！这儿正是根据`偏移量`和`待提取长度`进行`编码提取转换`。

> 需要特别注意`toStringInternal()`的内部注释，该注释已经给出了使用`java.nio.charset.CharSet.decode()`代替直接使用`new String(byte[], int, int, Charset)`的原因。因为很多时候，我们往往只提取一个大的`byte[]`里面很小的一部分`byte[]`。如果使用`new String(byte[], int, int, Charset)`，将会对整个byte数组进行拷贝，严重影响性能。

## 总结

通过阅读tomcat中`byte[]转String`的源码，我们看到tomcat开发团队可谓是费尽心思地提高web服务器的性能。转换的思路也很简单，就是通过`打标记`+`延时提取`的方式来实现`按需使用`。通过在`编码提取转换`的时候使用了特殊的转换逻辑，让楼主大开眼界！