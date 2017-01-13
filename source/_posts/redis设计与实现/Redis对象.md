---
title: Redis对象
date: 2015-06-28
categories: Redis设计与实现
tags: Redis
---

Redis中每个对象都由一个redisObject结构表示：

``` bash
typeof struct redisObject {
	//类型
	unsigned type : 4;
	//编码
	unsigned encoding : 4;
	//指向底层数据结构的指针
	void *ptr;
	// . . .
} robj;
```
type  对象的类型
REDIS_STRING 字符串对象
REDIS_LIST 列表对象
REDIS_HASH 哈希对象
REDIS_SET 集合对象
REDIS_ZSET 有序集合对象

对于Redis保存的键值对来说，键总是一个字符串对象，而值可以是上述对象中的任意对象。使用TYPE命令返回的结果是数据库对应键的值对象类型。

encoding 记录对象底层数据结构的实现。使用OBJECT ENCODING 可查看一个数据库键的值对象编码。

Redis对象的内存回收采用的是类似JVM垃圾回收的方法之一  引用计数法。
