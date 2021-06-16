---
layout: post
title: MySQL
slug: mysql
date: 2021-06-16 15:07:17
status: publish
author: 君祁
categories:
  - Maverick 
tags:
  - 数据库
  - MySQL
excerpt: MySQL原理
---

## 聚簇索引与非聚簇索引
聚簇索引(clustered index)：聚簇索引的叶节点存储了整行数据。

非聚簇索引：索引树的叶子节点只存储索引字段和主键的值，不存放整行数据。非聚簇索引也称为二级索引。

聚簇索引与非聚簇索引的区别是：是否存放完整的整行记录。

InnoDB主键使用的是聚簇索引，MyISAM所有的索引（包括主键索引）使用的都是非聚簇索引。

## MySQL的聚簇索引一定是主键吗？
在InnoDB中，聚集索引不一定是主键，但主键一定是聚集索引。如果没有为表定义主键，聚集索引会被设为第一个不为NULL的唯一索引，如果也没有这样的唯一索引，
InnoDB 会选择内置 6 字节长的 ROWID 作为隐含的聚集索引。

## InnoDB 事务的四种隔离级别是如何实现的？

## COUNT(*) COUNT(1) COUNT(id) COUNT(COLUMN)有区别吗？
先说结论，在高版本的MySQL（5.5及以后），count(1)比count(*)是没有区别的。

COUNT(COLUMN)在统计某个列值的数量时，不会统计NULL。当COUNT内的表达式确定不为NULL时，实际上是在统计所有行数。

如果表只有一个主键索引，没有其他二级索引，那么`COUNT(*)`与`COUNT(1)`都是通过主键索引统计行数的。如果该表有二级索引，`COUNT(*)`与`COUNT(1)`
会用**占用空间最小**的二级索引字段作统计。

为什么：InnoDB这样操作的原因是为了减少IO次数，因为数据库的瓶颈在IO，通过选择占用空间最小的非NULL索引来统计所有行数，尽可能地减小IO。

补充：MyISAM在统计表的总行数时会很快，原因是MyISAM对于表的行数作了优化，具体做法有一个变量用于存储表的总行数。但是，如果加了WHERE查询条件，该优化将不起作用。

## 辟谣：MySQL中使用IS NULL, IS NOT NULL, != 不能使用索引？胡扯！
NULL值是怎么在记录中存储的：在MySQL中，每一条记录都有固定的格式，以compact格式为会例，在compact格式下，会有一部分记录NULL值列表。NULL值列表是怎么表示的呢？
首先会统计表中所有可以为NULL的列，然后每个允许为NULL的列对应一个二进制位，逆序排列。如果二进制位为1，表示该列的值为NULL,如果为0表示该列的值不为NULL。另外，
二进制位的个数是整数个字节。

键值为NULL的记录怎么在B+树中存放的？答案是放在B+树的最左边。原因是InnoDB有这样的规定：
>We define the SQL null to be the smallest possible value of a field.