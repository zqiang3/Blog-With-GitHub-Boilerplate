---
layout: post
title: 系统设计
slug: system_design
date: 2021-06-16 15:07:17
status: publish
author: 君祁
categories:
  - Maverick
tags:
  - system design
excerpt: system design
---

## 秒杀系统

## 短链接生成系统如何设计？
[短链生成](https://blog.csdn.net/Alen_xiaoxin/article/details/105258644)

## Feed 流系统设计
[Feed流系统设计](https://www.infoq.cn/article/t0qlhfk7uxxzwo0uo*9s)
[如何打造千万级Feed流系统](https://help.aliyun.com/document_detail/143555.html?spm=a2c4g.11174283.6.686.66f930afTL8fnv)

## 如何设计一个亿级消息量的 IM 系统
[如何设计一个亿级消息量的 IM 系统](https://xie.infoq.cn/article/19e95a78e2f5389588debfb1c)

要点
* 信箱
* 读扩散与写扩散
* 唯一ID
* 消息ID递增，全局递增 vs 用户级别递增（微信） vs 会话级别递增(QQ)，连续递增(QQ) vs 单调递增
* 推模式 vs 拉模式 vs 推拉结合模式
* 微信：写扩散 + 推拉结合

[从0到1：微信后台系统的演进之路](https://mp.weixin.qq.com/s/fMF_FjcdLiXc_JVmf4fl0w)

[企业级IM王者——钉钉在后端架构上的过人之处](http://www.52im.net/thread-2848-1-1.html)

[如何优化高并发IM系统架构](https://help.aliyun.com/document_detail/143555.html?spm=a2c4g.11174283.6.686.66f930afTL8fnv)

## 分布式唯一 ID
[分布式唯一 ID](https://xie.infoq.cn/article/6673de15fabf57e46017c12c4)

[万亿级调用系统：微信序列号生成器架构设计及演变](https://mp.weixin.qq.com/s/JqIJupVKUNuQYIDDxRtfqA)

[主键列自增](https://help.aliyun.com/document_detail/47731.html?spm=5176.10695662.1996646101.searchclickresult.6bc54277FuOVpL)

[Snowflake](https://developer.twitter.com/en/docs/twitter-ids)