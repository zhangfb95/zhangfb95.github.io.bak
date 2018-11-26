---
layout: post
title:  YAML语言
date:   2016-09-09 09:28:14 +0800
categories: yaml
tag: yaml
---

* content
{:toc}

编程离不开数据，也离不开配置文件。配置文件如何书写得简洁、清爽、强大、不冗余？以前的方式是使用properties或者ini的方式，但是他们的组织方式比起YAML来说，就显得略有不足。我们常用的交换数据格式是JSON，其本身可以看成YAML1.2的子集。

## 简介

YAML，之所以取这个名字，是因为它表示的意思——'yet another markup language'，英文读作“家门儿”，文件后缀是`.yml`。其设计目标就是方便读写。

基本语法规则如下：

1. 大小写敏感
1. 使用缩进表示层级
1. 缩进只能使用空格，不能使用tab（跳格）
1. 缩进空格数量不限，只要同一层级元素左对齐即可
1. “#”表示注释，从当前字符到行尾，均会被解析器忽略
1. 键值之间使用“:”+“空格”来区分，键值对之间使用英文逗号“,”分隔

YAML支持的数据结构主要有3类：

1. 对象，又叫键值对、map、hash、dict等
1. 数组，按序排列的值，又叫list
1. 基本类型，包括数字（整数或浮点数）、布尔值、字符串、Null、时间和日期

## 值类型

### 数字

```yml
# 浮点数
number4Float: 13.34
# 整数
number4Int: 13
```

### 布尔值

使用`true`和`false`来表示

```yml
boolean4true: true
boolean4false: false
```

### Null

空指针用`~`表示

```yml
null4Str: ~
```

### 日期和时间

日期和时间，采用ISO8601的格式，

```yml
# 日期，如：yyyy-MM-dd
date: 1900-07-28
# 时间，如：yyyy-MM-ddTHH:mm:ss+时区
datetime: 1900-07-28T20:23:32+08:00
```

### 字符串

字符串相对于其他基本类型来说，写法就更为复杂一些。

1. 默认方式，不使用引号

```yml
str4common: config
str4chinese: 我是一个字符串，咿呀咿呀哟！
```

2. 如果字符串中有空格或其他特殊字符串，需要加引号。单引号不会对内容进行转意，双引号会对内容进行转意。

```yml
# 普通带空格
str4space: 'this is my first name'
# 单引号带空格，转换后为：first\\nsecond
s1: 'first\nsecond'
# 双引号带空格，转换后为：first\nsecond
s2: "first\nsecond"
```

3. 字符串中包含单引号，需使用两个单引号转意

```yml
# 转换后为：abc\'def
s1: 'abc''def'
```

4. 字符串可以写成多行，不过从第二行开始，必须有一个缩进的单空格。转后的字符串如`this is a man`

```yml
str: this
 is
 a man
```

5. 多行字符串，可以使用`|`保留换行符，也可以使用`>`折叠换行符。可以和`+`和`-`混合使用，`+`表示保留文字末尾的换行，`-`表示删除文字末尾的换行。

```yml
# 转换为：foo\nis\n
first: |
 foo
 is

# 转换为：foo is\n
second: >
 foo
 is

# 转换为：a\nb\n
third: |+
 a
 b

# 转换为：a\nb
four: |-
 a
 b
```

### 引用

YAML使用`&`、`<<`和'*'来描述引用特性。

```yml
defaults: &defaults
  adapter: pes
  host: 127.0.0.1

develop:
  database: develop_db
  <<: *defaults

test:
  database: test_db
  <<: *defaults
```

转换后如下：

```yml
defaults:
  adapter: pes
  host: 127.0.0.1

develop:
  database: develop_db
  adapter: pes
  host: 127.0.0.1

test:
  database: test_db
  adapter: pes
  host: 127.0.0.1
```

`&`建立锚点（`defaults`），`<<`表示合并当前引用数据，`*defaults`表示引用锚点。

数组中的引用也是类似，只是无需`<<`。

```yml
- &fet str
- first
- second
- *fet
```

转换后如下：

```yml
- str
- first
- second
- str
```

### 数组

中划线`-`开头的构成一个数组。同时，数组还可以嵌套数组

```yml
# 简单数组，转换后为：{first: ['a', 'b']}
first:
  - a
  - b

# 嵌套数组，转换后为：'arrays': ['obj': {'a': 'first', 'b': 'second'}, array2: [[{'yoyo': 'yes'}]]]
arrays:
  - obj:
    a: first
    b: second
  - array2:
    - yoyo: yes
```

### 对象

对象是以键值对的方式存在的。有两种方式表示对象中的元素：

```yml
# 简单方式
a:
  first: value1
  second: value2
# 花括号方式
a: {first: value1, second: value2}
```

### 类型强制转换

数字、布尔值、日期和时间，如果要转换成字符串，是可以进行强转的。如下：

```yml
# 转换后为：'123'
int2str: !!str 123
# 转换后未：'true'
boolean4str: !!str true
```

## 和JSON的关系

JSON和YAML都是人类可读的数据交换方式。尽管如此，它们仍然有不同的侧重点。
JSON的第一设计目标是简单和通用，因此它很容易生成和解析。基于最低限度的信息模式，使得JSON数据在任何编程语言环境，都能很容易地处理。
与之相反，YAML的第一设计目标是人为可读性和任意的本地数据序列化结构。所以，YAML更多的是作用于可读文件（如配置文件）。但是与JSON相比，它更难生成和解析。在跨编程语言时，它需要更为复杂的处理操作。

## 和XML的关系

新手常常搜索YAML和XML的关系，尽管这两种语言实际上并不存在必然的联系。YAML主要是一种数据序列化语言。XML是向后兼容于SGML，更多的是作用于文档的描述，所以XML能更多的容纳数据。

## 总结

YAML入门级实用比较简单，也很容易入门，但是涉及到一些比较深入的东西，需要参考官网。

## 参考链接

1. [阮一峰博客YAML](http://www.ruanyifeng.com/blog/2016/07/yaml.html?f=tt)
1. [YAML1.2规范](http://www.yaml.org/spec/1.2/spec.html)
1. [百度百科](http://baike.baidu.com/link?url=lERaMm6DWxX1n__yYRt9_WKvZTFJJUNE8USMVxFhe__z21l2484MRP382Dr4uwxR-AI2urXZdi9QLQjoKGzPza)
1. [ISO8601说明](http://baike.baidu.com/link?url=9He6gT8RocTq3Zwopir_avoImw7d4yjxa9ct_0xx9-1_EzoJ-3CkWov67oTd6GJrwaPZ2j0LOWa5_JQ_EFMuy_)