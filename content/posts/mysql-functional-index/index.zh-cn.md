---
weight: 1
title: "使用MySQL函数索引优化实践"
date: 2025-11-16T10:43:36+08:00
lastmod: 2025-11-16T10:43:36+08:00
draft: false
author: "August"
authorLink: "https://augustwang.github.io"
description: "使用MySQL函数索引优化线上项目的过程."
images: []
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["content", "MySQL"]
categories: ["documentation"]

lightgallery: true
---

记录一次使用MySQL函数索引优化线上项目的过程.

<!--more-->

## 1 背景

最近在维护一个很老的线上项目时，发现有个操作响应非常慢，而之前这个操作响应是很快的，
这个项目一直在线上运行，很久没有更新过，为什么突然变慢呢？

对这种情况，很自然就想到是服务器资源不足，或者数据库数据增长引起的问题，于是就开始排查问题。

## 2 数据库问题记录

对于数据库的问题，优先要复现操作，同时查看数据库操作记录，如果可以很容易复现，那直接在数据库执行命令：
```mysql
show full processlist;
```
这样可以查看当前数据库连接操作情况，以及正在执行的SQL语句。

如果不能及时复现，或者想查看更加详细执行时间等情况，来确认问题所在，可以开启慢SQL记录：
```mysql
set global slow_query_log = 1;
set global long_query_time = 2;
set global slow_query_log_file = 'slow.log';
```

如果已经设置过，可以查看设置情况：
```mysql
show variables like '%slow%';
```

注意：比如设置慢SQL查询时间为2秒，但是查看设置结果发现并没有变化，这里只需要断开连接重新连接即可看到生效结果。

## 3 问题场景

通过慢SQL记录，查看慢SQL记录，可以发现一个SQL查询非常慢，查询时间超过2秒，这个SQL查询如下：
```sql
SELECT * FROM users WHERE mobile = 8613911112222  AND channel_id = 1 ORDER BY id DESC LIMIT 1;
```
但是使用命令行执行这条SQL，查询结果是很快的，这又是为什么呢？

### 3.1 表结构分析

查看表结构，可以发现这个表结构中是存在索引的，而且是唯一索引如下：
```sql
UNIQUE KEY `channel_mobile` (`mobile`,`channel_id`) USING BTREE,
```
这条查询在表数据才百万级规模的情况下，不应该会慢才对，所以及有可能是索引失效。

这个时候通过表字段类型，其实已经发现问题，就是 mobile 字段是varchar类型，而查询时用的是数值，**存在隐式类型转换**，所以索引失效了。
```sql
`mobile` varchar(64) DEFAULT NULL,
```

### 3.2 explain分析验证

可以使用 explain 命令查看索引情况，就可以验证是索引失效了。

使用隐式的类型转换查询的SQL结果如下：
```mysql
mysql> EXPLAIN SELECT * FROM users WHERE mobile = 8613911112222  AND channel_id = 1 ORDER BY id DESC LIMIT 1;
+----+-------------+-------+------------+-------+----------------+---------+---------+------+------+----------+----------------------------------+
| id | select_type | table | partitions | type  | possible_keys  | key     | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+-------+------------+-------+----------------+---------+---------+------+------+----------+----------------------------------+
|  1 | SIMPLE      | users | NULL       | index | channel_mobile | PRIMARY | 8       | NULL |    1 |     5.00 | Using where; Backward index scan |
+----+-------------+-------+------------+-------+----------------+---------+---------+------+------+----------+----------------------------------+
1 row in set, 3 warnings (0.01 sec)
```

没有使用隐式的类型转换查询的SQL结果如下：
```mysql
mysql> EXPLAIN SELECT * FROM users WHERE mobile = '8613911112222'  AND channel_id = 1 ORDER BY id DESC LIMIT 1;
+----+-------------+-------+------------+-------+----------------+----------------+---------+-------------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys  | key            | key_len | ref         | rows | filtered | Extra |
+----+-------------+-------+------------+-------+----------------+----------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | users | NULL       | const | channel_mobile | channel_mobile | 268     | const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+----------------+----------------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.02 sec)
```
可以看出，存在隐式类型转换时，type为index，ref为NULL，而走索引时，type和ref都为const，这种有唯一索引的，正常应该是const。

### 3.3 解决办法

这个时候最有效的解决办法，就是修改这条查询SQL，不要使用隐式的类型转换，而是使用字符串查询。
```sql
SELECT * FROM users WHERE mobile = '8613911112222'  AND channel_id = 1 ORDER BY id DESC LIMIT 1;
```
但是由于是线上项目，不能修改代码，而且是很老的项目，已经没有人来维护，也就没有后续更新，只能在数据库层面想办法解决。

上面说过，在命令行下执行这个SQL，查询结果是很快的，这说明是有连接参数可以优化，那么这个参数是什么？

但是遗憾的是，连接参数也不能修改，也不太敢修改，怕出现其他影响，所以只能通过索引来解决。

这就引出了下面函数索引优化的解决方法？

## 4 函数索引

先要了解什么是[函数索引](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-functional-key-parts)

函数索引是一种索引类型，它可以对函数结果进行索引，而不是对字段本身进行索引，从MySQL 8.0.13起，开始支持函数索引功能。

这种通过在函数表达式上创建索引，可以有效的提高查询性能。

函数索引其实是通过创建一个虚拟列（GENERATED COLUMN），该虚拟列的值是原始字段经过函数转换的结果，然后在这个虚拟列上创建索引。

虚拟列是本身不存储数据的列，而是通过实时计算得到数据结果的，但是虚拟列的函数索引是存储在磁盘上的。

生成列(GENERATED COLUMN)特性是从MySQL 5.7引入的，通过表达式自动计算列值，分为VIRTUAL（虚拟）和STORED（存储）两种类型。

**生成列核心特性**
{{< admonition note "类型对比" >}}

|类型      | 计算时机	|存储空间 |	可索引性|	适用场景|
|:---- |:---- |:---- |:---- |:---- |
|VIRTUAL |  查询时实时计算	|不占用	|不可索引	|简单计算且不频繁使用的场景|
|STORED	 |  插入/更新时计算	|占用	|可索引	|复杂计算或需要索引的场景|

{{< /admonition >}}

**注意**：
- **MySQL 版本**： 确保版本确实在 8.0.13 或更高版本。
- **存储引擎**： 函数索引通常需要 InnoDB 存储引擎。
- **双括号语法**： 确保创建索引时使用了正确的双层括号语法 ((expression))，而不是单层括号 (expression)。 

### 4.1 简单应用

使用函数索引的一个常见场景是对字符串列进行查询。比如下面SQL要查询匹配电话号码前3位:
```sql
SELECT * FROM users WHERE mobile LIKE '%139%';
```
如果没有函数索引，就只能是模糊查询，这样查询速度就会很慢，使用函数索引，可以解决这个问题。

针对这个字段，添加一个函数索引:
```sql
ALTER TABLE users ADD KEY IND_FUNCTIONAL_MOBILE_PREFIX((substr(mobile,3,3)))
```

直接使用 `substr(mobile, 3, 3) = '139'` 从第3位开始截取3位，然后进行查询，这样查询速度就会很快。
```sql
SELECT * FROM users WHERE substr(mobile, 3, 3) = '139';
```

## 5 问题解决

虽然可以创建函数索引，使用CAST() 函数将字段转换为数值类型，但是由于不能修改代码，所以执行的SQL还是数值类型的，
这里就需要依赖MySQL优化器，来处理隐式类型转换，如果优化器不能正确的找到这个索引怎么办呢？下面看看实际情况：

1. 创建虚拟列

创建一个虚拟列，然后再创建索引
```sql
ALTER TABLE users
ADD mobile_numeric BIGINT AS (CAST(mobile AS UNSIGNED)) VIRTUAL;
```

2. 创建索引

或者在 `ALTER TABLE` 语句中同时完成：
```sql
ALTER TABLE users
ADD mobile_numeric BIGINT AS (CAST(mobile AS UNSIGNED)) VIRTUAL,
ADD INDEX idx_users_mobile_numeric (mobile_numeric);
```

3. 在原有列上直接创建函数索引

从 MySQL 8.0.13 版本开始，可以直接在 CREATE INDEX 或 ALTER TABLE 语句中使用表达式来创建函数索引（Functional Index），而不需要预先创建单独的生成列。 

创建一个函数索引，将mobile字段转换为数值类型
```sql
ALTER TABLE users 
ADD INDEX idx_users_mobile_numeric ((CAST(mobile AS UNSIGNED)));
```

4. 验证和使用

验证使用函数索引，使用 `EXPLAIN` 命令查看SQL执行计划，如果使用了索引，type字段将显示为index，ref字段将显示为const。

但是很遗憾，`EXPLAIN` 分析 type为ALL, ref为NULL，MySQL优化器并没有使用函数索引，
```sql
EXPLAIN SELECT * FROM users WHERE mobile = 8613911112222; 
```
显然，优化器并不足够智能，没有识别出这里的隐式类型转换，所以没有使用函数索引。

尝试修改SQL来验证使用函数索引，使用 `CAST` 函数将mobile字段转换为数值类型
```sql
EXPLAIN SELECT * FROM users WHERE (CAST(mobile AS UNSIGNED)) = 8613911112222; 
```
显示可以用到 idx_users_mobile_numeric 索引，也就是索引是有效的，现在只需要让MySQL优化器识别到索引。 

### 5.1 优化器识别函数索引

MySQL 优化器在处理查询时，会估算使用索引的成本（读取索引，回表查询数据行）等多种条件之间的权衡，并选择最合适的执行计划。

通过上面的问题SQL可以知道，查询条件中存在多个字段，这就成为优化器识别索引的关键，MySQL优化器会根据索引的列顺序和列的个数来判断索引是否匹配查询条件。
```sql
SELECT * FROM users WHERE mobile = '8613911112222'  AND channel_id = 1 ORDER BY id DESC LIMIT 1;
```
所以完全可以把channel_id字段放在组合索引的前面，这样MySQL优化器就可能会使用到这个索引了。

所以重新创建函数索引如下：
```sql
mysql> CREATE INDEX idx_users_channel_id_mobile_numeric ON users (channel_id, ((CAST(mobile AS UNSIGNED))));
Query OK, 0 rows affected (11.57 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

然后分析SQL执行计划如下：
```mysql
mysql> EXPLAIN SELECT * FROM users WHERE mobile = 8613911112222  AND channel_id = 1 ORDER BY id DESC LIMIT 1;
+----+-------------+-------+------------+------+----------------------------------------------------+-------------------------------------+---------+-------+------+----------+-----------------------------+
| id | select_type | table | partitions | type | possible_keys                                      | key                                 | key_len | ref   | rows | filtered | Extra                       |
+----+-------------+-------+------------+------+----------------------------------------------------+-------------------------------------+---------+-------+------+----------+-----------------------------+
|  1 | SIMPLE      | users | NULL       | ref  | channel_mobile,idx_users_channel_id_mobile_numeric | idx_users_channel_id_mobile_numeric | 9       | const |  161 |    10.00 | Using where; Using filesort |
+----+-------------+-------+------------+------+----------------------------------------------------+-------------------------------------+---------+-------+------+----------+-----------------------------+
1 row in set, 3 warnings (0.02 sec)
```
虽然还是有警告，但是已经识别到索引了，说明索引是生效的。

实际情况也是，后续操作响应就变得很快了，也没有慢查询了。
