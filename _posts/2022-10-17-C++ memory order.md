---
layout: post
title: "C++ memory order"
date: 2022-10-17 16:03:15 +0800
categories: 计算机 C++
math: false
typora-root-url: ..

---

### memory_order_relaxed

没有同步或顺序制约，仅对此操作要求原子性

### memory_order_release

1. 对写操作施加 release 语义，在代码中这条语句前面的所有读写操作都无法被重排到这个操作之后
2. 当前线程内的所有写操作，对于其他对这个原子变量进行 acquire 的线程可见
3. 当前线程内的与这块内存有关的所有写操作，对于其他对这个原子变量进行 consume 的线程可见

### memory_order_consume

1. 对当前要读取的内存施加 consume 语义，在代码中这条语句后面所有与这块内存有关的读写操作都无法被重排到这个操作之前
2. 在这个原子变量上施加 release 语义的操作发生之后，consume 可以保证读到所有在 release 前发生的并且与这块内存有关的写入

### memory_order_acquire

1. 对读操作施加 acquire 语义，在代码中这条语句后面所有读写操作都无法重排到这个操作之前
2. 在这个原子变量上施加 release 语义的操作发生之后，acquire 可以保证读到所有在 release 前发生的写入

### memory_order_acq_rel

作用在 read-modify-write 操作上，同时具备 memory_order_acquire、memory_order_release 的语义

### memory_order_seq_cst

同时具备 memory_order_acquire、memory_order_release、memory_order_acq_rel 的语义，额外保证所有 memory_order_seq_cst 作用的变量具备全局序
