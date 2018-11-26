---
layout: post
title:  JSON语言
date:   2016-09-09 03:52:14 +0800
categories: json
tag: json
---

* content
{:toc}

JSON，中文叫做“江省”，是Javascript Object Notation的缩写，且完全独立于语言的文本格式。它的文法易于阅读、编写和解析，在数据传输领域得到了很好的运用。

## 语法规则

1. 数据存于键值对中
1. 数据由逗号分隔
1. 花括号保存对象
1. 方括号保存数组

## 6种取值

+ 数字（整数或浮点数）
+ 字符串（在双引号中）
+ 布尔值（true和false）
+ 数组（方括号中）
+ 对象（花括号中）
+ null

【注】：json不支持注解，它是一种纯数据文本格式。如果非要加入注释，可以使用变通的方式。参考链接：http://stackoverflow.com/questions/244777/can-comments-be-used-in-json。

```json
{
    "__comment": "this is a comment",
    "data": {
    }
}
```

## 对象和数组

对象和数组可以相互嵌套。数组意味着可以有多个，它可以包含6种取值，包括自己。对象却是一组键值对的集合。

### 数组例子

```json
{
    "array": [
        "a", 123, 123.2, false, {}, null, []
    ]
}
```

### 对象错误的例子

这不符合语法，对象只能包含键值对。

```json
{
    "object": {
        "a", 123, 123.2, false, {}, null
    }
}
```

### 对象正确的例子

```json
{
    "object": {
        "key1": {},
        "key2": "value2"
    }
}
```

## 工具推荐

1. [json格式化在线网站](http://json.cn/)
1. [java语言json解析库——jackson](https://github.com/FasterXML/jackson/)
1. [java语言json解析库——fastjson](https://github.com/alibaba/fastjson)
