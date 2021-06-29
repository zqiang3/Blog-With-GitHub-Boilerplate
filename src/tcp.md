---
layout: post
title: TCP
slug: tcp
date: 2021-06-16 15:07:17
status: publish
author: 君祁
categories:
  - 网络
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

在socket编程中，客户端或服务端任何一方执行`close()`操作即可产生挥手动作。
* 第一次挥手(FIN=1, seq=x)

  主动关闭的一方（假设是客户端）发送FIN标志位为1的包，同时带上序列号x，表示不再发送数据，但仍然可以接收数据。发送完毕后，进行`FIN_WAIT_1`状态。
* 第二次挥手(ACK=1, ACKNum=x+1)

  被动关闭的一方（假设为服务端）确认客户端的FIN包，发送一个确认包，确认号设为x+1，服务端进入`CLOSE_WAIT`状态，客户端收到这个包后，进行`FIN_WAIT_2`状态。
* 第三次挥手(FIN=1, seq=y)

  服务端可以关闭连接了，向客户端发送关闭连接请求，将FIN标志位设为1，同时带上序列号y，发送完毕后，进行`LASK_ACK`状态。
* 第四次挥手(ACK=1, ACKNum=y+1)

  客户端收到服务端的关闭连接请求后，返回一个确认包。发送后进入`TIME_WAIT`状态。服务端收到确认包后，进行`CLOSED`状态。客户端等待2倍的MSL时间后，
没有再收到网络包也进入`CLOSED`状态。

![](./images/tcp.jpg)

## 为什么TIME_WAIT状态要保持2MSL？
MSL是报文在网络中的最大生存时间，超过这个时间报文将被丢弃。

维持TIME_WAIT状态的目的有两个
* 为了可靠地关闭TCP连接，确保当前连接的所有报文在网络上彻底消失
* 为了能正确处理延迟的重复报文，如果前后两个连接使用相同的网络四元组，避免前一个连接对后一个连接的影响。

TCP连接中的一端在发送了FIN报文后如果没有收到相应的ACK报文，则会反复多次重传FIN报文，大约持续几分钟。处于TIME_WAIT状态的一端在收到重传的FIN报文时会重新计时。

为什么需要2倍的MSL呢？
第一个MSL是为了保证客户端发送的ACK生命周期结束，第二个MSL是确保服务端发送的FIN生命周期结束。

客户端发送ACK，经过多长时间到达服务是不知道的，假设这个时间为t，则0<t<MSL。如果服务端没有收到ACK，会重传FIN，FIN也需要经过最大MSL的时间到达客户端，
因此TIME_WAIT需要维持2MSL才能收到重传的FIN报文。

## 什么是MTU
最大传输单元，1500bytes

## TCP与UDP的区别

|    |  TCP  |  UDP   |
|--- | ---| --- |
| 是否建立连接   | 面向连接，三次握手| 无连接，不需要握手   |
|    | 面向字节流 | 面向数据报 | 
| 可靠性   | 可靠：无差错，不丢失，不重复，按序到达，有确认、重传机制   | 不可靠，最大努力交付，不保证可靠性 |
|    | 端到端 | 一对一，一对多，多对多 |
|    | 有拥塞控制，流量控制 | 无握手、确认、重传、拥塞控制机制 |
| 协议头   | 首部开销大，20字节| 开销小 |
| 速度  | 慢，效率低，占用系统资源高  | UDP更快 |
| 设计复杂度 | 复杂  | 简单 |
|     | 全双工 | 全双工 |

TCP的缺点：
* 慢，效率低，占用系统资源高
* 三次握手机制，易被攻击（DOS, DDOS, CC）

UDP的缺点：
* 不可靠，不稳定，丢包

适用场景：
* TCP适用可靠传输的场景，对网络通讯要求高。
* UDP适用于对网络通讯质量要求不高的场景，如音视频传输。

## TCP内核参数优化
**建立连接阶段**
* net.ipv4.tcp_syn_retries：tcp三次握手中发送syn得不到服务端响应时重传syn的次数。
* net.ipv4.tcp_syncookies: 默认开启，建议开启，可以提升对SYN flood攻击的防护能力。
* net.ipv4.tcp_synack_retries: tcp三次握手第二步服务端发送syn+ack得不到响应时重传的次数。
* net.ipv4.tcp_max_syn_backlog: 控制半连接队列大小。服务端收到客户端的SYN包后，就会把这个连接放到半连接队列中，
  然后再向客户端发送SYN+ACK。建议调大。
* net.core.somaxconn: 全连接队列。服务端收到三次握手中第三步客户端的ACK，就会把这个连接放到全连接队列中。
  全连接队列中的连接还需要被accept()系统调用取走，服务端才开始处理客户端的请求。建议适当调大。
* net.ipv4.tcp_abort_on_overflow: 全连接队列满了后，新的连接就会被丢弃掉，服务端的默认行为是直接丢弃不通知客户端。

**数据传输阶段**
net.ipv4.tcp_wmem
net.core.wmem_max
net.ipv4.tcp_mem
net.ipv4.tcp_rmem
net.core.rmem_max
net.ipv4.tcp_window_scaling
net.ipv4.tcp_keepalive_probes
net.ipv4.tcp_keepalive_intvl
net.ipv4.tcp_keepalive_time
net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_congestion_control

**断开连接阶段**
* net.ipv4.tcp_fin_timeout：从FIN_WAIT_2到TIME_WAIT的超时时间。长时间收不到对端FIN包，大概率是对端机器有问题，
  不能及时调用close()关闭连接，建议调低，避免等待时间太长，资源开销太大。
* net.ipv4.tcp_max_tw_buckets: 系统TIME_WAIT连接的最大数量，根据实际业务需要调整，超过最大值后dmesg会有报错TCP：
  time wait bucket table overflow
* net.ipv4.tcp_tw_reuse: 允许TIME_WAIT状态的连接占用的端口用到新建连接，客户端可开启