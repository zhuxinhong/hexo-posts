---
title: Redis哈希对象
date: 2015-06-28
categories: Redis设计与实现
tags: Redis
---

哈希对象编码可以是ziplist或者hashtable。

ziplist编码的哈希对象使用压缩列表作为底层实现。有新的键值对要保存时，程序会先将键值对的键推到列表表尾，再将键值对的值推到列表表尾。

hashtable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都是用一个字典键值对来保存。

当哈希对象满足下面2个条件时，哈希对象使用ziplist编码，其余都使用hashtable编码：
a.哈希对象保存所有键值对的键和值字符长度都小于64字节；
b.哈希对象保存的键值对数量小于512个；

上述2个条件的阀值是可以修改的，在配置文件中的 hash-max-ziplist-value 和 hash-max-ziplist-entries 选项中。
