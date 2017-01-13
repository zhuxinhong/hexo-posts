---
title: Redis类型检查和命令多态
date: 2015-07-01
categories: Redis设计与实现
tags: Redis
---

Redis中用于操作键的命令基本可以分为2类。

其中一种是可以对任何类型的键执行，比如DEL,EXPIRE,RENAME,TYPE,OBJECT,TTL等命令。

另一种只能对特定的键执行，比如：
SET,GET,APPEND,STRLEN 等仅限用于字符串的键类型；
HDEL,HSET,HGET,HLEN 等仅限用于哈希键类型；
RPUSH,LPOP,LINSERT,LLEN 等仅限用于列表键类型；
SADD,SPOP,SINTER,SCARD 等仅限用于集合键类型；
ZADD,ZCARD,ZRANK,ZSCORE 等仅限用于有序集合键类型；

在执行一个类型特定命令之前，Redis会先检查输入键类型是否正确，然后再决定是否执行命令。
如果键名称正确，再检查键对象是否是执行命令所需的类型，不是的话就返回一个类型错误。

多态命令的实现除了检查键类型和命令是否匹配外，还会检查键的至对象所使用的编码。例如，LLEN命令。
如果列表对象编码为ziplist,，说明对象为压缩列表，程序使用ziplistlen作为底层实现返回列表长度。
如果列表对象编码为linkedlist，说明对象为双端链表，程序使用listLength作为底层实现返回链表长度。
