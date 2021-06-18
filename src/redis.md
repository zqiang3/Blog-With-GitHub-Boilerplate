---
layout: post
title: redis设计与实现
slug: redis
date: 2021-06-17 17:15:16
status: publish
author: 君祁
categories:
  - Maverick
tags:
  - redis
excerpt: redis
---

## 简单动态字符串(SDS)

**定义**

```c
struct sdshdr {
  int len;
  int free;
  char buf[];
};
```

**优点**
1. 常数时间获取字符串长度。
2. 杜绝缓冲区溢出。
3. 通过内存预分配和惰性空间释放减少内存重分配次数。
4. 二进制安全。
5. 兼容部分c字符串函数。

## 链表

```c
typedef struct listNode {
  struct listNode *prev;
  struct listNode *next;
  void *value;
} listNode;

typedef struct list {
  listNode *head;
  listNode *tail;
  unsigned long len;
  void *(*dup)(void *ptr);
  void *(*free)(void *ptr);
  int (*match)(void *ptr, void *key);
} list;
```

**特点**

1. 双端
2. 无环
3. 带表头和表尾指针
4. 长度计数器
5. 多态。

## 跳跃表
链接：[跳跃表的实现（全网最详细的分析）](https://www.jianshu.com/p/9d8296562806)

跳跃表的数据结构(c源码)
```c
typedef struct zskiplistNode {
  struct zskiplistNode *backward;
  double score;
  robj *obj;
  struct zskiplistLevel {
    struct skiplistNode *forward;
    unsigned int span;
  } level[];
} skiplistNode;

typedef struct zskiplist {
  struct zskiplistNode *header, *tail;
  unsigned long length;
  int level;  // 表头节点的层高不会记录
} zskiplist;
```

跳跃表由以下几部分组成：
* 表头：不存储元素，负责维护指向第一个元素各层的指针。
* 表尾：全部由NULL组成，表示每一层链表的结尾。
* 跳跃表节点：保存元素值，以及多个层。
* 层：保存指向其他层的指针。高层的指针越过的元素数量大于等于低层的指针。查询时总是从高层的先开始搜索，然后慢慢降低层次，直到找到相应的元素。

![跳跃表的数据结构](https://user-gold-cdn.xitu.io/2019/6/17/16b650948652b4ec?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

跳跃表的查询复杂度可以接近O(logN)。
跳跃表的查询效率可以与平衡树媲美，但设计与实现却比平衡树简单得多。设计思想是通过空间换时间。通过建立多层链表，高层的链表比低层的跨度大，就可以加快查询速度。

当有序集合中元素数量很多，或者元素的成员是较长的字符串，就会使用跳跃表作为底层实现。

