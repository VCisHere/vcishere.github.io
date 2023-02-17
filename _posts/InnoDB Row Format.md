---
layout: post
title: "InnoDB-Row Format"
date: 2022-10-17 16:03:15 +0800
categories: 计算机 MySQL
math: false
typora-root-url: ..

---

InnoDB 数据记录的存储格式：Row Format 行格式

在 MySQL 中，所谓 Row Format 行格式是指数据记录(或者称之为行)在磁盘中的物理存储方式。具体地，对于 InnoDB 存储引擎而言，常见的行格式类型有 Compact、Redundant、Dynamic 和 Compressed。

- Compact

其大体分为两部分：记录的额外信息、记录的数据内容。

记录的额外数据：变长字段的长度列表，NULL 值标识位，记录头信息

记录的数据内容：列 1 数据，列 2 数据，...，列 n 数据

变长字段的长度列表：
1. 变长字段的长度列表不存储值为 NULL 的长度信息
2. 变长字段的长度列表不是一定存在的
一方面，表中可能没有变长类型的字段；另一方面，该记录中所有的变长字段值可能均为 NULL
3. 变长字段的长度列表中各字段长度信息是按列的顺序逆序排列的
4. char 类型字段的长度信息是否需要存储在变长字段的长度列表中，取决于其所使用的字符集是否为变长字符集

NULL 值标志位：
1. 记录中的 NULL 值使用 bitmap 存储  

bitmap 中不存储 NOT NULL 的字段

2. NULL 值标志位不是一定存在的

可能所有字段均为 NOT NULL

3. bitmap 是按列的顺序逆序排列的

记录头信息：

记录头信息用于描述该条记录，其固定为 5 个字节，即 40 位。其定义如下：

1 bit：预留位1

1 bit：预留位2：暂未使用

1 bit：delete_mask：当前记录被删除的标志位

1 bit：min_rec_mask：B+ 树的每层非叶子节点中的最小记录的标志位

4 bits：n_owned：当前记录拥有的记录数

13 bits：heap_no：当前记录在记录堆中的位置

3 bits：record_type：当前记录类型。具体地，0: 普通记录；1: B+树非叶子节点记录（即所谓的目录项记录）；2: 最小记录；3: 最大记录

16 bits：next_record：下一条记录的相对位置

记录的数据内容：

对于我们所定义的数据表，还会默认的插入一些其他列(字段)，即所谓的隐藏列。

DB_ROW_ID：该字段占 6 个字节，用于标识一条记录

DB_TRX_ID：该字段占 6 个字节，其值为事务ID

DB_ROLL_PTR：该字段占 7 个字节，其值为回滚指针

上述 3 个字段，除了 DB_ROW_ID 字段，其余两个字段均一定会被添加到数据表中。一般地，当用户未指定主键时，MySQL 会选择非 NULL 的 Unique 键作为主键。如果没有非 NULL 的 Unique 键的话，MySQL 才会添加 DB_ROW_ID 字段作为主键。

行溢出

众所周知，InnoDB 中内存与硬盘交互的基本单位是页，一般页大小为16KB。MySQL 规定一个页中至少需要存放两条记录。而所谓的行溢出是指：当某个记录的某个字段（varchar、text、blob 等类型）的值长度过长、数据量过大，会导致一个页中放不下一条记录，为此在 compact、redundant 行格式中，如果该记录某字段中数据量过多时，则在该记录的数据内容的相应字段处只存储该字段值前 768 个字节的数据和一个指向存储剩余数据的其他页（即所谓的溢出页）的地址，该地址通常占用 20 个字节。

- Dynamic 和 Compressed

Dynamic、Compressed 和 Compact 比较相似，不同之处在于对待行溢出的处理。Dynamic、Compressed 会把溢出的字段值全部存储到溢出页中，而不会在原相应字段存储前 768 个字节。而 Compressed 相比于 Dynamic 来说，Compressed 会对所有页面进行压缩以减少存储占用。
