---
title: Redis压缩列表
date: 2015-06-27
categories: Redis设计与实现
tags: Redis
---

压缩列表（ziplist）是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每项都是小整数或短字符串时，Redis就会使用压缩列表作为列表键的底层实现。

ziplis组成：
zlbytes : 记录列表占用内存字节数：在重新分配内存或计算zlend时使用
zltail : 记录压缩列表表尾节点距离列表的起始地址有多少字节
zllen : 记录压缩列表包含节点数量
entryX : 各个节点
zlend : 标记压缩列表末端

压缩列表节点构成 entryX :
previous_entry_length : 记录前1个节点长度，单位为字节，程序可通过指针运算，根据当前节点的起始地址计算出前一个节点的起始地址。如前一节点长度小于254字节，则值为1个字节；反之若前一节点长度大于等于254字节，值为5字节，其中第一字节为0xFE(十进制254)，之后4个字节用于保存前1节点长度。
encoding : 记录节点的content属性所保存数据的类型及长度
content : 节点值

连锁更新
例如在一个压缩列表中，有多个连续的长度为250-253的节点e1至eN，现在在表头插入一个长度大于254的节点。
想像一下，新节点插入后，e1需要修改previous_entry_length的值从1个字节扩展到5个字节，并且该变化会递归到eN。
因此连锁更新在最坏的情况下需要对压缩列表执行N次空间重分配，每次空间重分配的最坏复杂度为O(N)，所以连锁更新的最坏复杂度为O(N平方)。
但尽管该操作复杂度较高，但实际环境中，存在多个连续的250-253字节长度之间的节点并不多，所以我们可以不必担心连锁更新对性能的影响。
