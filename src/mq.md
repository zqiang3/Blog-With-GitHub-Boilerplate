---
layout: post
title: 消息队列
slug: message_queue
date: 2021-06-16 15:07:17
status: publish
author: 君祁
categories:
  - Maverick
tags:
  - kafka
  - rocketmq
excerpt: kafka rocketmq
---

## kafka的rebalance机制
consumer group中的消费者与topic下的partition重新匹配的过程。

什么时候会产生rebalance？
* consumer group中的成员个数发生变化
* consumer消费超时
* group订阅的topic个数发生变化
* partition数量发生变化

kafka在进行rebalance时，不能进行读写操作，因此rebalance会影响读写性能。应尽量避免rebalance。