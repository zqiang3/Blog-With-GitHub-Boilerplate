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
2. 杜绝缓冲区溢出。SDS在修改数据之前，会先检查空间是否足够，不够会先拓展空间，再执行操作。
3. 通过内存预分配和惰性空间释放减少内存重分配次数。c字符串修改字符串长度N次需要执行N次内存重分配，SDS最多执行N次内存重分配。
4. 二进制安全。c字符采用0字符标识末尾，因此不能保存图片，音视频，压缩文件，SDS采用len记录长度，不存在这种问题，因此可以保存各种各样的格式。
5. 兼容部分c字符串函数。

## 链表
在 Redis3.2 之前，链表采用了 ZipList 和 LinkedList 实现的，在 3.2 之后，List 底层采用了 QuickList。
**定义**

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

1. 双端：带有prev和next指针
2. 无环：表头节点的 prev 指针和表尾节点的 next 都指向NULL
3. 带表头和表尾指针
4. 保存了链表长度
5. 多态：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数。

## 跳跃表
链接：[跳跃表的实现（全网最详细的分析）](https://www.jianshu.com/p/9d8296562806)

**定义**

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

## 哈希表
**定义**
```c
// 字典
typedef struct dict {
  dictType *type;
  void *privdata;
  dictht ht[2];
  int rehashidx;
} dict;

// hashtable
typedef struct dictht {
  dictEntry **table;
  unsignd long size;
  unsigned long sizemask;
  unsigned long used;
} dictht;

// 哈希表节点
typedef struct dictEntry {
  void *key;
  union {
    void *val;
    uint64_t u64;
    int64_t s64;
  } v;
  struct dictEntry *next;
} dictEntry;
```

链地址法来解决冲突，将新节点添加到链表的表头位置。

渐进式哈希，两个哈希表，每次对字典进行添加，删除，查找或更新操作，会将rehashidx索引上的键值对rehash到`ht[1]`。当所有键值对都rehash到`ht[1]`，
设置rehashidx=-1，rehash操作完成。分而治之，避免集中式处理对redis性能影响太大。rehash期间，查找会在两个哈希表查找，新增的键值会直接添加到`ht[1]`。

## 整数集合
**定义**
```c
typedef struct intset {
  uint32_t encoding;
  uint32_t length;
  int8_t contents[];
} intset;
```

整数集合的每个元素都是contents数组的一个数据项，各个项在数据中按值的大小**从小到大**有序地排列，并且数组中**不包含任何重复项**。

contents数组的真正类型取决于encoding属性的值。 encoding为INTSET_ENC_INT16，整数集合底层实现为int16_t类型的数组，集合保存的都是int16_t类型的整数值。

**升级**

添加新元素，并且新元素的类型比整数集合现有所有元素都要长时。

升级要做的是，根据新类型的长度，以及元素的数量，对底层数组进行空间重分配。

**降级**

不支持降级。一旦对数组进行了升级，编码就会一直保持升级后的状态。

采用整数集合存储提升了灵活性，也尽可能地节约内存

## 压缩列表
压缩列表是为了节约内存而开发，由一系列特殊编码的**连续内存块**组成。每个节点可以保存一个字符数组或者一个整数值。

**数据结构**
```c
zlbytes zltail zllen entry1 entry2 ... entryN zlend
previous_entry_length encoding content
```

了解下连锁更新。

## 对象
**定义**
```c
typedef struct redisObject {
  unsigned type: 4;
  unsigned encoding: 4;
  void *ptr;
} robj;
```

**类型**

| 类型常量     | 对象名称   |
| ------------ | ---------- |
| REDIS_STRING | 字符串对象 |
| REDIS_LIST   | 列表       |
| REDIS_HASH   | 哈希       |
| REDIS_SET    | 集合       |
| REDIS_ZSET   | 有序集合   |

使用TYPE命令查看对象的类型
```bash
rpush alist 1 2 5
TYPE alist  # list
```

**数据类型与编码**

| 类型         | 编码                      | 对象                               |
| ------------ | ------------------------- | ---------------------------------- |
| REDIS_STRING | REDIS_ENCODING_INT        |                                    |
| REDIS_STRING | REDIS_ENCODING_EMBSTR     |                                    |
| REDIS_STRING | REDIS_ENCODING_RAW        |                                    |
| REDIS_LIST   | REDIS_ENCODING_ZIPLIST    | 压缩列表实现                       |
| REDIS_LIST   | REDIS_ENCODING_LINKEDLIST | 使用双端链表实现的列表对象         |
| REDIS_HASH   | REDIS_ENCODING_ZIPLIST    | 压缩列表实现                       |
| REDIS_HASH   | REDIS_ENCODING_HT         | 字典实现                           |
| REDIS_SET    | REDIS_ENCODING_INTSET     | 整数集合实现                       |
| REDIS_SET    | REDIS_ENCODING_HT         | 字典实现                           |
| REDIS_ZSET   | REDIS_ENCODING_ZIPLIST    | 压缩列表实现                       |
| REDIS_ZSET   | REDIS_ENCODING_SKIPLIST   | 使用跳跃表和字典实现的有序集合对象 |

**底层数据结构与编码常量**

| 底层数据结构                  | 编码常量                  | 命令输出   |
| ----------------------------- | ------------------------- | ---------- |
| 整数                          | REDIS_ENCODING_INT        | int        |
| embstr编码的简单动态字符串SDS | REDIS_ENCODING_EMBSTR     | embstr     |
| 简单动态字符串SDS             | REDIS_ENCODING_RAW        | raw        |
| 字典                          | REDIS_ENCODING_HT         | hashtable  |
| 双端链表                      | REDIS_ENCODING_LINKEDLIST | linkedlist |
| 压缩列表                      | REDIS_ENCODING_ZIPLIST    | ziplist    |
| 整数集合                      | REDIS_ENCODING_INTSET     | intset     |
| 跳跃表和字典                  | REDIS_ENCODING_SKIPLIST   | skiplist   |

使用OBJECT ENCODING查看对象的编码
```bash
set msg "hello world"
OBJECT ENCODING msg
```