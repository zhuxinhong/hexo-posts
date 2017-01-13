---
title: Redis字典
date: 2015-06-25
categories: Redis设计与实现
tags: Redis
---

字典就是我们熟悉的map，键值对(key-value pair)的抽象数据结构。Redis数据库就是使用字典来作为底层实现的。
先介绍下数据结构：
1.哈希表
``` bash
typeof struct dictht {
	//哈希表数组
	dictEntry   **table;
	//哈希表大小
	unsigned long size;
	//哈希表大小掩码，用于计算索引
	//总是等于size-1
	unsigned long sizemask;
	//已有节点数量
	unsigned long used;
} dictht;
```

这种数据结构很好理解，和Java里面HashMap实现差不多。哈希表里面每个节点都是一个entry对象。

2.哈希表节点
``` bash
typeof struct dictEntry {
	//键
	void *key;
	//值
	union {
	void *val;
	unit64_t u64;
	int64_t s64;
} v;

//指向下个哈希表节点，形成链表
struct dictEntry *next;
} dictEntry;
```

key属性存着键值对中的键，v属性存值。键可以是一个指针，也可以是一个unit64_t整数，或是一个int^4_t整数。C语言没了解过，这种奇怪的整数也不知道什么意思，姑且理解为Java里面的hashCode吧。
next属性指向链表中的下一个节点，这点和Java一样，当多个对象哈希值相同时，就造成hash冲突，多个对象的索引值都落在同一索引下标上，该节点的数据结构为链表，需要遍历该链表才能找到对应元素。

3.字典数据结构
``` bash
typeof struct dict {
	//类型特定函数
	dictType *type;
	//私有数据
	void *privdata;
	//哈希表
	dictht ht[2];
	//rehash索引
	//当rehash不再进行时，值为-1
	int rehashidx;
} dict;
```

type属性和privdata属性针对不同类型的键值对，为创建多态字典设置的。
type属性是一个指向dictType结构的指针，每个dictType保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数。
privdata属性保存了需要传给那些类型特定函数的可选参数。

``` bash
typeof struct dictType {
	//计算哈希值的函数
	unsigned int (*hashFunction) (const void *key);
	//复制键的函数
	void *(*keyDup) (void *privdata, const void *key);
	//复制值的函数
	void *(*valDup)(void *privdata, const void *obj);
	//对比键的函数
	int (*keyCompare) (void *privdata, const void *key1, const void *key2);
	//销毁键的函数
	void (*keyDestructor) (void *privdata, void *key);
	//销毁值的函数
	void (*valDestructor) (void *privdata, void *obj);
} dictType;
```

ht属性是一个包含2个索引的数组，每个位置都是一个dictht哈希表，一般使用ht[0]，ht[1]用于存放对ht[0]哈希表进行rehash的结果。
rehashidx记录目前rehash进度，当前没有执行rehash，值为-1。


1.Redis 字典 Hash算法
根据key计算hash值
hash = dict -> type -> hashFunction(key);
根据sizemask属性和哈希值,计算出索引,ht[x]可以是ht[0]或ht[1]
index = hash & dict -> ht[x].sizemask;
Redis使用MurmurHash2算法计算键的哈希值，优点在于即使输入的键是有规律的，算法仍能给出一个很好的随机分布性。

2.解决Hash冲突
当2个或多个不同对象的Hash值相同时，会被分配到哈希表的同一个索引上，此时就称之为哈希冲突。
Redis采用链地址法解决键冲突，每个哈希节点都有一个next指针，由此指针构成一个单向链表，新添加的元素在链表表头。

3.rehash
负载因子 = 哈希表节点数量/哈希表大小
load_factor = ht[0].used / ht[0].size
当哈希表的操作不断进行，需要把哈希表的负载因子保持在一个合理范围内，需要对哈希表进行扩容或缩容操作，2种操作都通过执行再散列操作来执行(rehash)：
a.为字典的ht[1]分配空间，如果是扩容操作，ht[1]的空间为ht[0]的节点数量2倍;如果是缩容,ht[1]的空间为ht[0]节点数量的一半。
b.将ht[0]中的所有键值对rehash到ht[1]上面：rehash需要重新计算键的哈希值和索引值，然后放在ht[1]的指定位置上。
c.当ht[0]所有键值对都迁移到ht[1]上之后，释放ht[0]，就是将ht[0]表置为空表，再将ht[1]置为ht[0],并将ht[1]新建一个大小为0的空哈希表。

4.渐进式rehash
上面讲到rehash过程中，需要将ht[0]上的键值对迁移到ht[1]上，但这个过程并不是一次性操作完成的。如果需要一次性完成，就必须对当前哈希表进行锁表操作，锁表期间会导致服务的不可用，所以rehash操作需要分多次、渐进式的完成。
a.为ht[1]分配空间，字典同时持有ht[0]和ht[1]两个哈希表
b.将rehashidx值置为0，表示rehash工作开始。
c.rehash期间，所有对字典执行的读写操作，都会在ht[0]和ht[1]两个表上操作，读操作如果在ht[0]上找到，会把结果rehash到ht[1]上，如果未找到会到ht[1]上去找，保存操作只会存在ht[1]上，当rehash工作完成之后,rehashidx++;
d.随着字典操作的不断执行，最终ht[0]上所有节点会全部rehash到ht[1]上，这时将rehashidx值置为-1，表示rehash完成。
