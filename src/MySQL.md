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

## 聚簇索引与二级索引
聚簇索引：整行数据与索引值放在一块，像InnoDB就是这样做的，聚簇索引的叶子节点存储了整行数据。

二级索引：除聚簇索引之外的所有索引都称为非聚簇索引（也称为二级索引），叶子节点存储的内容是主键的值。

聚簇索引与非聚簇索引的区别是：叶节点是否存放完整的整行记录。

InnoDB主键使用的是聚簇索引，MyISAM所有的索引（包括主键索引）使用的都是非聚簇索引。

```bash
CREATE INDEX [index name] ON [table name]([column name]);
```
或者
```bash
ALTER TABLE [table name] ADD INDEX [index name]([column name]);
```
在MySQL中，`CREATE INDEX` 操作被映射为 `ALTER TABLE ADD_INDEX`。

## InnoDB 事务的四种隔离级别是如何实现的？


## COUNT(*) COUNT(id) COUNT(1)有区别吗？
先说结论，这俩在高版本的MySQL（5.5及以后）是没有区别的，没有count(1)比count(*)更快这一说了。

如果表只有一个主键索引，没有其他二级索引，那么COUNT(*)与COUNT(1)都是通过主键索引统计行数的。如果该表有二级索引，COUNT(*)与COUNT(1)会用**占用空间最小**的二级索引字段作统计。

补充：MyISAM在统计表的总行数时会很快，原因是MyISAM对于表的行数作了优化，具体做法有一个变量用于存储表的总行数。但是，如果加了WHERE查询条件，该优化将不起作用。
