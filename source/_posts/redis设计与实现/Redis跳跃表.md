---
title: Redis跳跃表
date: 2015-06-27
categories: Redis设计与实现
tags: Redis
---

Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量较多，或者有序集合中成员是较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。
和链表、字典等数据结构在Redis内部的广泛应用不同，Redis只在实现有序集合键和集群节点中用到跳跃表。

跳跃表节点 zskiplistNode
``` bash
typeof struct zskiplistNode {
	//层
	struct zskipzlistLevel{
	//前进指针
	struct zskiplistNode *forward;
	//跨度
	unsigned int span;
	} level[];
	
	//后退指针
	struct zskiplistNode *backward;
	//分值
	double score;
	//成员对象
	rojb *obj;
} zskiplistNode;

```


跳跃表 zskiplist
``` bash
typeof struct zskiplist {
	//表头节点和表尾节点
	structz skiplistNode *header, *tail;
	//表中节点数量
	unsigned long length;
	//表中层数最大节点的层数
	int level;
} zskiplist;
```
