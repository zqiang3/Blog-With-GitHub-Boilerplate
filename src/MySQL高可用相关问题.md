---
layout: post
title: MySQL高可用
slug: mysql_ha
date: 2021-06-18 16:00:26
status: publish
author: 君祁
categories:
  - 数据库
tags:
  - 数据库
  - MySQL
excerpt: MySQL高可用
---

## 主从复制的过程
1. master开启binlog功能。
2. master开启io线程，slave开启io线程，sql线程。
3. master通过io线程将binlog日志发送到slave，slave通过io线程接收binlog数据，将其写入relay-log（中继日志）。
4. slave的sql线程实时监测relay-log的内容是否有更新，如果有更新，将里面的sql语句重新执行生成相应的数据。