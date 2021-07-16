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

## 异步复制、全同步复制与半同步复制
[MySQL的异步复制、全同步复制与半同步复制](https://database.51cto.com/art/201911/606556.htm)

异步复制：主库在执行完客户端提交的事务后会立即将结果返给客户端，并不关心从库是否已经接收并处理。这样会存在问题，主如果crash掉，
此时主已经提交的事务可能并没有传到从库上，如果将从提升为主，就产生了数据丢失。

全同步复制：当主库执行完一个事务，所有的从库都执行了该事务才返回给客户端。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能受到严重的影响。

半同步复制：介于全同步复制与全异步复制之间，主库只需要等待一个从库节点收到并将binlog写到relay log文件即可，不需要等待从库解析binlog写到数据库。
相比于异步复制，半同步复制提高了数据的安全性，同时造成了一定程度的延迟，这个延迟至少是一个TCP/IP往返的时间。

master将每个事务写入binlog，传递数据到slave同时主库提交事务(commit)，slave收到并刷新到磁盘(sync_relay=1)，slave反馈收到relay log向主响应ack，
master向client响应ok。

mysql主从模式默认是异步复制的，而MySQL Cluster是同步复制的。

在2010年MySQL 5.5版本之前，一直采用的是异步复制的方式。主库的事务执行不会管备库的同步进度，如果备库落后，主库不幸crash，就会导致数据丢失。

在5.5版本中引入了半同步复制，主库在应答客户端提交的事务前需要保证至少一个从库接收并写到relay log中。

2016年，MySQL在5.7.17中引入了一个全新的技术，称之为InnoDB Group Replication。基于Group Replication的全同步技术已经出现，
全同步技术带来了更多的数据一致性保障