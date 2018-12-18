---
layout: post
title:  关于char数组的三个算法题
date:   2018-12-18 18:08:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

## 前言

针对char数组我们可以创造很多算法题，有一些是有现成的算法来实现（比如排序、查找等等），有一些却是需要我们具有机灵和应变的能力。

这里，我们罗列几个关于char数组的题目，然后给出实现思路和代码。希望通过对这些题目的分析，将我们固有的、僵化的思想给稍微活络一下，以纪念我们逝去的青春。

## 1. 给定一个char数组，找出重复次数最多的char元素

这个题目有很多实现的方式，比如我们可以通过jdk集合类来进行实现，但是效率并不会很高。

这里给出一种不需要额外集合和类库的实现方式。

```java
/**
 * 给定一个char数组，找出重复次数最多的char元素
 */
private static void getCharWhichCountIsMax() {
    // 输入char数组
    char[] chars = "acbcbacccdaaaaaaaaaaaaaaaaaaaaaaaaccccccccccccccebb".toCharArray();
    char tmp;

    // 将char数组进行排序
    for (int i = 0; i < chars.length; i++) {
        for (int j = i + 1; j < chars.length; j++) {
            if (chars[i] > chars[j]) {
                tmp = chars[j];
                chars[j] = chars[i];
                chars[i] = tmp;
            }
        }
    }

    // 对排序后的char数组进行长度最大的获取
    char a = ' ';
    int aSize = 0;
    char b = ' ';
    int bSize = 0;
    for (int i = 0; i < chars.length - 1; i++) {
        if (chars[i] == chars[i + 1]) {
            b = chars[i];
            bSize++;
        } else {
            if (bSize > aSize) {
                a = b;
                aSize = bSize;
            }
            b = chars[i + 1];
            bSize = 0;
        }
    }
    System.out.println(a);
}
```

## 2. 给定一个char数组，去掉里面的重复元素

去掉重复元素，首先需要判断哪些元素是属于重复的。怎么判断呢？我首先想到的是能否借鉴jdk集合类库来辅助我们完成功能呢？答案是肯定的，但是它的效率（时间和空间）是值得思考的。

这儿仍然给出一种比较好的实现方式。

```java
/**
 * 给定一个char数组，去掉里面的重复元素
 */
private static void dropRepeatableChar() {
    char[] chars = "hello world this is myself".toCharArray();
    // 实现的思路为，遍历整个char数组，将不重复的数据进行交换和前移
    int unRepeatIdx = 0;
    for (int i = 0; i < chars.length; i++) {
        boolean isRepeat = false;
        for (int j = 0; j < unRepeatIdx + 1; j++) {
            if (chars[j] == chars[i]) {
                isRepeat = true;
                break;
            }
        }
        if (!isRepeat) {
            unRepeatIdx++;
            chars[unRepeatIdx] = chars[i];
        }
    }

    // 将不重复的char进行输出
    for (int i = 0; i < unRepeatIdx; i++) {
        System.out.print(chars[i]);
    }
    System.out.println();
}
```

## 3. 给定一个char数组，里面的单词使用空格进行分离，将空格分离的单词进行反向输出

这个题目就稍微难一点，咋一看，貌似需要对每个数组元素进行移动，但是怎么个移动法会使移动的效率更高呢？

最简单的方式是先将整个char数组进行反转，再将char数组里面的每个单词进行反转，就能达到我们想要的效果了。思路很简单，那么怎么实现这个算法呢？我们来看一看java代码。

```java
/**
 * 给定一个char数组，里面的单词使用空格进行分离，将空格分离的单词进行反向输出
 * 比如char数组为：hello world this is myself
 * 我们需要得到新的char数组为：myself is this world hello
 */
private static void letWordsInverseOutput() {
    char[] chars = "hello world this is myself".toCharArray();
    char tmp;

    // 先将整个数组进行反转
    for (int i = 0; i < chars.length / 2; i++) {
        tmp = chars[i];
        chars[i] = chars[chars.length - i - 1];
        chars[chars.length - i - 1] = tmp;
    }

    // 再将数组里面的每个单词进行反转
    int start = 0;
    int end;
    for (int i = 0; i <= chars.length; i++) {
        if (i == chars.length || chars[i] == ' ') {
            end = i - 1;
            if (end < 0) {
                continue;
            }
            int k = 0;
            for (int j = start; j <= (start + end) / 2; j++) {
                tmp = chars[j];
                chars[j] = chars[end - k];
                chars[end - k] = tmp;
                k++;
            }
            start = i + 1;
        }
    }
    System.out.println(chars);
}
```

## 总结

在实现之前，别人问我这些问题时，我需要绞尽脑汁地想实现方式，但想出的思路不一定是一个好的方式，甚至有时候想不出一个能解决问题的方式，这就很尴尬了。但是一旦我们实现了这些问题，再回过头来看，我们发现这些实现竟然如此简单，甚至还非常高效！

因此，当看一个问题时，我们需要从一个不同的、多维度的视角来看，往往能看到和常人不一样的东西，做人和做事也是一样。

谨以此文献给思路狭隘的自己！