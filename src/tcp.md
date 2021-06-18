---
layout: post
title: TCP
slug: tcp
date: 2021-06-16 15:07:17
status: publish
author: 君祁
categories:
  - Maverick
tags:
  - 网络
  - TCP
excerpt: TCP协议
---

## TCP
在socket编程中，当客户端执行`connect()`时，将触发三次握手。建立TCP连接时，客户端和服务端，总共需要发送三个包。
* 第一次握手(SYN=1, seq=x)
客户端发送第一个包，SYN标志位为1，序列号为x。发送完毕后，客户端进入`SYN_SENT`状态。
  
* 第二次握手(SYN=1, ACK=1, seq=y, ACKNum=x+1)
服务端返回确认包并发送一个SYN包，SYN标志位与ACK标志位都设为1，服务端设置自己的序列号y，同时将确认序号设置为客户端的序列号加1，即x+1。发送完毕后，
服务端进入`SYN_RCVD`状态。
  
* 第三次握手(ACK=1, ACKNum=y+1)
客户端再次发送确认包。ACK标志位设置为1，确认序号设为服务端的序列号加1，即y+1。发送完毕后，客户端进入`ESTABLISHED`状态。当服务端收到这个包后，
也进入`ESTABLISHED`状态。TCP三次握手过程结束。
  
![](./images/tcp.jpg)