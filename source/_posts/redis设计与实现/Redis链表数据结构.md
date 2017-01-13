---
title: Redis链表数据结构
date: 2015-06-24
categories: Redis设计与实现
tags: Redis
---

``` bash
typeof struct listNode {
	//前置节点
	struct listNode *prev;
	//后置节点
	struct listNode *next;
	//节点的值
	void *value；
	} listNode;
	链接的数据结构：
	typeof struct list {
	//表头节点
	listNode *head;
	//表尾节点
	listNode *tail;
	//节点数量
	unsiged long len;
	//节点复制函数
	void *(*dup) (void *ptr);
	//节点释放函数
	void (*free) (void *ptr);
	//节点对比函数
	int (*match) (void *ptr, void *key);
} list;
```
