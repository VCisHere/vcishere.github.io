---
layout: post
title: "InnoDB-BufferPool-LRU"
date: 2022-10-17 16:03:15 +0800
categories: 计算机 MySQL
math: false
typora-root-url: ..

---
BufferPool

Innodb为了解决磁盘上磁盘速度和CPU速度不一致的问题，在操作磁盘上的数据时，先将数据加载至内存中，在内存中对数据页进行操作。

Mysql在启动的时候，会向内存申请一块连续的空间，这块空间名为Bufffer Pool，也就是缓冲池，默认情况下Buffer Pool只有128M。

简单的LRU：
1. 新数据插入到链表头部；
2. 每当缓存命中（即缓存数据被访问），则将数据移到链表头部；
3. 当链表满的时候，将链表尾部的数据丢弃。

不能用简单的 LRU 有以下原因：
- 假如有一张表被全表扫，就会迅速清空之前其他查询留下来的高频数据页，相当于 LRU list 被刷了一遍。
- InnoDB 有预读机制
- - 线性预读：当一个区中有连续 56 个（默认值）页面被加载到 BufferPool 中，会将这个区中所有页面都加载到 BufferPool 中。默认一个区 64 个页。
- - 随机预读：当一个区中随机 13 个（默认值）页面被加载到 BufferPool 中，会将这个区中的所有页面都加载到 BufferPool 中。

InnoDB 的 LRU

将链表分为两个部分，一个 old 区，一个 young 区。
- young 区在链表的头部，理解为热数据
- old 区在链表的尾部，理解为冷数据
- 当数据页第一次被加载进 BufferPool 时在 old 区头部
- 如果 old 区的数据页存活超过 1s(innodb_old_blocks_time) ，就把它移动到 young 区

young 区的后 3/4 被命中时会被移动到链表头部，前 1/4 被访问了不会进行移动

