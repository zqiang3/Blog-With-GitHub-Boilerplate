---
layout: post
title: redis高可用
slug: redis_ha
date: 2021-06-24 01:00:23
status: publish
author: 君祁
categories:
  - Maverick
tags:
  - redis
  - ha
excerpt: redis高可用
---

## 过期删除策略
[链接](https://zhuanlan.zhihu.com/p/139423463)

* 定时删除
  
在设置键的过期时间的同时，创建一个定时器，定时器在键过期时，立即执行对键的删除操作。定时删除可以保证键尽快地被删除，释放占用的空间。

对内存友好，对cpu非常不友好。创建大量的定时器消耗大量的时间处理键的过期删除，服务器处理命令请求的性能就会降低。目前redis没有使用定时删除策略。

* 惰性删除
  
当访问键时，才去检查键是否过期，如果过期就删除键，如果没有过期就返回该键。

* 定期删除

每隔一段时间，就对数据库进行一次检查，查看哪些键过期，再根据具体策略决定删除哪些键。

定期删除策略每隔一段时间对键执行过期删除，并通过限制执行删除的时长和频率来减少对cpu的性能影响。

### redis使用的过期删除策略
redis使用的是惰性删除和定期删除策略。

定期删除的流程为，周期性地执行定期删除函数，每次随机从数据库取出一定数量的键进行检查，并删除其中的过期键。随机删除key的算法，可能是LRU算法。

### RDB对过期键的处理
生成RDB文件时，不会包含过期的键。执行SAVE或BGSAVE命令时，会对键进行检查，已过期的键不会被保存在新创建的RDB文件中。

载入RDB文件时，主服务器中已过期的键不会被载入内存，从服务器会将所有键载入内存。

### AOF对过期键的处理
如果服务器开启了AOF持久化功能，当键过期被删除时，redis会追加一条DEL命令到AOF文件中，显示标记该键已被删除。

当执行AOF重写时，已过期的键不会被保存在重写后的AOF文件中。

### 主从复制对过期键的处理
从服务器不会触发键过期，而是会等待主节点触发键过期。主节点键过期时，会同步del命令给所有的从节点。
从节点不处理过期键，但也不会返回已过期的数据，会通过独有的逻辑来标记一个键已经过期。


## 主从模式
**主从复制的过程**
1. 主节点(master)开启一个子进程，生成rdb快照文件，同时将新的命令写到缓冲区
2. 向从节点(slave)发送rdb快照文件
3. 从节点收到rdb文件后，加载数据到内存
4. 主节点再将缓冲区命令发到从节点，加载后与主节点达到一致状态
5. 之后主节点每收到一个新的写命令，都会同步给从节点

主从复制开始时，副本从主机全量同步数据，发送psync命令，主机执行bgsave，在后台开启一个子进程产生rdb文件，同时将后面新的写命令写到一个缓冲区，然后把rdb文件发送给从机，从机收到后load到内存，这个步骤完成后主机再将缓冲区的数据发给从机，再之后主机收到新的写命令，都会发给从机

master如果发现有多个slave node都来重新连接，仅仅会启动一个rdb save操作，用一份数据服务所有slave node。

增量同步条件：主机会维护一个复制积压缓冲区，如果从节点缺失的数据在复制积压缓冲区内，可以执行增量同步，否则执行全量同步。

复制积压缓冲区是一个固定长度的先进先出队列，默认大小为1M。

**缺点**
1. 主库执行全量备份会造成毫秒或秒级的卡顿
2. COW机制，极端情况下主库内存溢出，程序异常退出或宕机
3. 发送数GB大小的备份文件导致服务器出口带宽暴增，阻塞请求。
4. 需要人为手工操作，需要通知业务方做变更配置，整个过程需要人为干预，较为烦琐

在主-从模式里，如果主服务器宕机了，虽然可以由从机来代替主机继续服务，但这需要人工把从机切换成主机，需要人工干预，还会造成一段时间服务不可用。

**异步复制**
Redis的主从复制是异步复制，异步分为两个方面，一个是master服务器在将数据同步到slave是异步的，一个是slave在接收同步数据时也是异步的。

主从复制的模式是异步复制的，主向从发送数据后就返回了，并不等待从成功写入数据。异步复制会导致数据的一致性较差，可能会产生数据丢失。

## sentinel
Sentinel是Redis的高可用性解决方案。

**故障转移的过程**
1. 每个sentinel根据心跳响应超时判断主节点是否主观下线，根据配置判定是否客观下线。sentinel判定下线的标准是写在配置里的，包括多久超时，其他sentinel认为下线的数量。
2. 选举领头sentinel，在每个纪元周期，每个sentinel都可以发送广播让其他sentinel设置自己为领头sentinel；每个sentinel有一次机会设置另一个sentinel为leader；
当某个sentinel获得了半数以上的选票，就成为leader，并发送广播告诉其他sentinel，选举阶段结束。
3. 领头sentinel选择一台从服务器，将其升级为主服务器。选取的标准有几个。
4. 向其他从服务器发送命令，让它们复制新的主服务器
5. 当旧的主服务器重新连线后，将成为新的主服务器的从服务器

选取从服务器升级为主服务器的标准：
1. 删除所有下线或断线的从服务器
2. 删除最近5秒没有回复info命令的从服务器
3. 删除与主服务器断开超过一定时间的从服务器，保证数据是相对较新的
4. 选择优先级最高的
5. 选择复制偏移量最大的

**sentinel的启动过程**
* 每个sentinel都与主服务器建立命令连接和订阅连接。命令连接可以发送info命令，发送publish命令，订阅连接可以接收hello频道的信息
* 与主服务器建立连接后，可以向主服务器发送info命令，可以得到从服务器的信息。sentinel就可以无需用户提供从服务器的信息，可以通过主服务器发现从服务器
* 发现从服务器后，创建与从服务器的命令连接和订阅连接，也可以向从服务器发送info命令了
* sentinel通过hello频道的信息可以自动发现其他sentinel，相互之间建立命令连接，不建议订阅连接


## Redis Cluster
Redis Cluster是Redis提供的**分布式集群**解决方案，集群通过分片来进行数据共享，并提供复制和故障转移功能。

通过Gossip协议进行通信。

最小配置6个节点以上（3主3从），主节点提供读写操作，从节点作为备用节点，不提供服务，只作为故障转移使用。节点只使用0号数据库

关键词：节点、槽指派、命令执行、重新分片、转向、故障转移、消息。

**故障转移**

集群中的每个节点都会定期地向其他节点发送ping消息，以此来检测对方是否在线。如果接收ping消息的节点没有在规定的时间之内回复pong，就会被标记为**疑似下线**。节点的状态包括：在线，疑似下线，已下线。

1. 如果集群里半数以上的负责处理槽的主节点将某个主节点报告为疑似下线，则该节点将被标记为**已下线**。
2. 第一个标记节点为已下线的主节点将向集群广播一条消息，所有收到消息的节点也会立即标记该节点已下线。
3. 从已下线的主节点下选中一个从节点，使其成为新的主节点。
4. 新的主节点接管所有旧的主节点的槽，指派给自己。
5. 向集群广播一条消息，告诉其他节点自己已成为新的主节点，并接管了所有的槽。
6. 新的主节点开始接收命令和处理自己负责的槽，故障转移完成。

**选举新的主节点的过程**
1. 所有的主节点有投票权，从节点没有投票权
2. 在某个主节点已下线后，其下的从节点会向集群广播一条消息，要求收到消息的主节点向自己投票。
3. 每个在线的主节点都有一次投票机会，会选择最先向自己要求投票的从节点。
4. 当某个从节点获得了半数以上的投票，则选举成功；否则，任何一个从节点都没有获得足够多的选票，会在新的纪元周期发起选举。

**读写分离的讨论**

官方默认cluster不进行读写分离。在cluster架构下，默认的，一般redis-master用于接收读写，而redis-slave则用于备份，当有请求是向slave发起时，会直接重定向到对应key的master来处理。

可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离。

不过，通过对master进行水平扩展就可以了，扩容master，跟之前扩容slave并进行读写分离，效果是一样的或者更好。