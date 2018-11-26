---
layout: post
title:  mysql中IF语句用法
date:   2016-12-14 09:25:51 +0800
categories: mysql
tag: mysql
---

* content
{:toc}

在进行sql查询的时候有时我们需要对null值进行额外的处理，比如：设置默认值。这时使用`IF语句`会让代码看起更简洁。

## IF表达式

```sql
# IF(condition, true_expr, false_expr)
```

如果`condition`为TRUE（condition <> 0 && condition <> NULL），则`IF()`的返回值为`true_expr`；否则返回值为`false_expr`。
`IF()`返回值可以是字符串或数字，视具体语境而定。

```sql
select u.*, if(sex=1, "男", "女") as sex_desc from user u where id =?;
```

## CASE WHEN表达式

`IF()`表达式可以使用`CASE WHEN`表达式来替代。语法如下：

```sql
CASE real_key WHEN first_key THEN first_value WHEN second_key THEN second_value ELSE other_value END
CASE real_key WHEN first_key THEN first_value ELSE other_value END
```

`real_key`表示真实的值，也就是待比较的值。`first_key`和`second_key`表示可能值，`first_value`和`second_value`表示可能被替换的值。
`ELSE`后面的`other_value`表示其他情况下可被替换的值。

```sql
# 这条sql返回的结果就是列的值为'one'的记录
SELECT CASE 1 WHEN 1 THEN 'one' WHEN 2 THEN 'two' ELSE 'more' END AS testCol;
# 这条sql语句可用以替换IF语句
SELECT u.*, CASE sex WHEN 1 THEN '男' WHEN 2 THEN '女' ELSE '其他' END AS sex_desc
FROM user u
WHERE id = ?
```

## IFNULL表达式

此语句用以简写IF()语句，语法如下

```sql
IFNULL(expr1, expr2)
```

如果`expr1`不为NULL，则其返回值为`expr1`，否则其返回值为`expr2`。同样的，IFNULL的返回值可能是数字或字符串。

```sql
mysql> SELECT IFNULL(1,0);
-> 1

mysql> SELECT IFNULL(NULL,10);
-> 10

mysql> SELECT IFNULL(1/0,10);
-> 10

mysql> SELECT IFNULL(1/0,'yes');
-> 'yes'
```