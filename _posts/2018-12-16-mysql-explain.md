---
layout: post
title:  理解MySQL中的explain
date:   2018-12-16 18:54:00 +0800
categories: 工具
tag: 工具
---

* content
{:toc}

## 前言

我们可以使用`explain`命令来查看MySQL查询优化器的执行计划是怎么来优化查询的。通过结果反馈，我们能更好地选择索引，同时也能写出更好的查询语句。

> 【注】：本节我们使用的MySQL版本为5.7.11。

## 语法

`explain`命令可作用于具体的表。也可作用于DML语句，包括`SELECT`、`DELETE`、`INSERT`、`REPLACE`和`UPDATE`。

当作用于具体表时，和`desc`命令效果相同，用于描述表的信息，结果信息包括：字段名称、字段类型、是否为null、是否主键、默认值等等。

```sql
-- explain作用于表
explain `{tableName}`;
```

当作用于DML语句时，可用于查看执行计划。

1. 因为`REPLACE`和`INSERT`语句相对来说较为简单，本节不会额外分析他们。
2. `DELETE`、`UPDATE`和`SELECT`有相似之处，都可以写`WHERE条件`，所以本节仅以`SELECT`为例来进行分析。

```sql
-- explain作用于DML
explain DML语句;
```

## 列信息详解

我们先给出最简单的例子，然后对例子中的每一列进行详细分析。所有的分析都借鉴了[官方说明](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html)。

### 最简单的例子

我们创建一个`user`表，其DDL如下。

```
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `username` varchar(20) NOT NULL COMMENT '用户名',
  `password` varchar(20) NOT NULL COMMENT '密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

执行`explain`之后，我们得到如下结果。

```
mysql> explain select * from user where id = 1;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

### 1. id

id列可以是一个正整数或`null`，且满足下面的规则。

1. id大的先执行
2. id相同的，按照顺序从上往下执行
3. id为`null`表示是一个结果集，不是一个查询

### 2. select_type

用于表示查询中每个SELECT子句的类型，可能出现的枚举值如下

```
1. SIMPLE
2. PRIMARY
3. UNION
4. DEPENDENT UNION
5. UNION RESULT
6. SUBQUERY
7. DEPENDENT SUBQUERY
8. DERIVED
9. UNCACHEABLE SUBQUERY
10. UNCACHEABLE UNION
```

#### 2.1.SIMPLE

简单查询，子查询和UNION查询之外的其他查询。

```
mysql> explain select * from user where id = 1;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | user  | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

#### 2.2. PRIMARY

含有子查询或者UNION查询时，最外层或者最远的查询。

```
mysql> explain select username from user a where id = 1 union select username from user b  where id = 3;
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
| id | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra           |
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY      | a          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
|  2 | UNION        | b          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
3 rows in set, 1 warning (0.00 sec)
```

#### 2.3. UNION

UNION语句中的第二个或更后面的查询子句，第一个为`PRIMARY`。

```
mysql> explain select username from user a where id = 1 union select username from user b  where id = 3;
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
| id | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra           |
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY      | a          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
|  2 | UNION        | b          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
3 rows in set, 1 warning (0.00 sec)
```

#### 2.4. DEPENDENT UNION

和UNION类似，只是表示外部查询需要对其进行依赖。比如下面的查询，会先查询id列值为2和3的记录，然后纳入`select * from user`的查询条件中。

```
mysql> explain select * from user where username in (select username from user a where id = 1 union select username from user b  where id = 3);
+----+--------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
| id | select_type        | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra           |
+----+--------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY            | user       | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    3 |   100.00 | Using where     |
|  2 | DEPENDENT SUBQUERY | a          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
|  3 | DEPENDENT UNION    | b          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
| NULL | UNION RESULT       | <union2,3> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
4 rows in set, 1 warning (0.00 sec)
```

#### 2.5. UNION RESULT

用于表示UNION的结果，因为不需要参与查询，所以id列为null。

```java
mysql> explain select * from user where username in (select username from user a where id = 1 union select username from user b  where id = 3);
+----+--------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
| id | select_type        | table      | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra           |
+----+--------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
|  1 | PRIMARY            | user       | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    3 |   100.00 | Using where     |
|  2 | DEPENDENT SUBQUERY | a          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
|  3 | DEPENDENT UNION    | b          | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL            |
| NULL | UNION RESULT       | <union2,3> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  | NULL |     NULL | Using temporary |
+----+--------------------+------------+------------+-------+---------------+---------+---------+-------+------+----------+-----------------+
4 rows in set, 1 warning (0.00 sec)
```

#### 2.6. SUBQUERY

子查询里面的第一个SELECT子句。

```
mysql> explain select password from user a where id = (select id from user b  where id = 3);
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | PRIMARY     | a     | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | NULL        |
|  2 | SUBQUERY    | b     | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```

#### 2.7. DEPENDENT SUBQUERY

子查询内部的第一个查询子句，同时外层查询依赖子查询。

```
mysql> explain select update_time from user_extra as ue where exists (select id from user where user.id = ue.user_id and id = 1);
+----+--------------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type        | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+--------------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | PRIMARY            | ue    | NULL       | ALL   | NULL          | NULL    | NULL    | NULL  |    1 |   100.00 | Using where |
|  2 | DEPENDENT SUBQUERY | user  | NULL       | const | PRIMARY       | PRIMARY | 8       | const |    1 |   100.00 | Using index |
+----+--------------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------------+
2 rows in set, 2 warnings (0.00 sec)
```

#### 2.8. DERIVED

表示派生关系，在FROM子句里面有子查询，且子查询带分组时，会出现此种情况。

```
mysql> explain SELECT id FROM (SELECT id FROM user GROUP BY id) AS tmp;
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL |    3 |   100.00 | NULL        |
|  2 | DERIVED     | user       | NULL       | index | PRIMARY       | PRIMARY | 8       | NULL |    3 |   100.00 | Using index |
+----+-------------+------------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)
```

#### 2.9. UNCACHEABLE SUBQUERY和UNCACHEABLE UNION

主要出现在子查询有随机函数和UNION有随机函数的场景

### 3. table

该列表示被查询的表名，可能出现下面的值

1. <unionM,N> // select_type为union，且union关联的id为M和N
2. <derivedN> // select_type为derived，且是从id为N的结果导出的
3. <subqueryN> // select_type为subquery，且是id为N的子查询

### 4. type

该列表示MySQL查询到具体数据行的方式，通常我们将其叫做`访问类型`，其值从优到差分别如下：

system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL

1. system & const

当用到主键索引或唯一索引时其值为const。system是const的特例，当查询系统表，且系统表里面只有一行数据时出现。

```sql
explain select * from user;
```

2. eq_ref

主键索引或唯一索引作为两个表的连接方式时，通常和`=`一起使用。

```sql
explain select * from user, user_extra where user.id = user_extra.user_id;
```

3. ref

用到了索引，但不是主键索引和唯一索引。

```sql
explain select * from user where uuid = 'x';
```

4. ref_or_null

类似于ref，但是可以查询包含null的行，通常用在子查询里面。

5. index_merge

当使用多个条件进行交集或并集时出现。

```sql

```

6. range

 对索引列进行范围查找时出现，比如`between`、`in`、`>`、`<`等

```sql
explain select * from user where id > 1;
```

7. index

全表扫描，但是只查询索引列的值，因此只需要扫描索引树即可。

```sql
explain select id from user;
```

8. ALL

全表扫描，在服务端进行逻辑处理。性能最差的一种方式。

```sql
explain select * from user;
```

### 5. possible_keys

指出MySQL可能用到的索引字段，它会按照顺序，把所有可能涉及到的索引都列出来，但并不是每个索引都会用到。

### 6. keys

指出MySQL实际用到的索引字段。

### 7. key_len

表示使用的索引最大长度。在组合索引的情况下，该字段使用`,`罗列出每个索引字段的长度。需要注意的是，`key_len`只是表结构定义的翻译。在同样的条件下，`key_len`越短，查询的效率越高。

### 8. ref

`ref`显示哪些列或者常量被用于和索引列`key`进行对比。如果值为`func`，则表示此值是通过函数计算得出。

### 9. rows

Mysql认为需要查询的行数。

### 10. filtered

该列给出了一个百分比的值，这个百分比值和`rows列`的值一起使用，可以估计出那些将要和QEP中的前一个表进行连接的行的数目。前一个表就是指id列的值比当前表的id小的表。

### 11. extra

该列包含对于MySQL查询的附加描述信息，当该列出现`Using filesort`或者`Using temporary`时，我们必须对我们的查询语句进行优化。

## 参考链接

+ https://dev.mysql.com/doc/refman/5.7/en/execution-plan-information.html
+ https://www.jianshu.com/p/1c04f199501c
+ https://www.jianshu.com/p/95c589b6cb94