---
title: RedisRDB文件结构
date: 2015-07-04
categories: Redis设计与实现
tags: Redis
---

Redis RDB文件保存的是二进制数据，结构包括5部分:
REDIS | db_version | databases | EOF | check_sum

db_version长度为4个字节，它的值是一个字符串表示的整数，记录RDB文件的版本号。

databases记录数据库实例，和各个数据库实例的键值对数据，如果redis-server中所有db都为空，那这个值也为空，长度为0字节。

EOF常量长度为1字节，标志着RDB文件正文内容结束。

check_sum是一个8字节长的无符号整数，保存着一个校验和，这个校验和是程序通过对REDIS、db_version、databases、EOF四个部分内容计算而来。Redis在载入RDB文件时，会将载入数据的检验和与该值对比，以此来检查RDB文件是否出错或损坏。

1.databases  
每个非空数据库在RDB文件中的都保存为SELECTDB、db_number、key_value_pairs三个部分。  
SELECTDB | db_number | key_value_pairs  
SELECTDB常量长度为1字节，标识后面的数据库号码。  
db_number保存数据库号码，根据号码大小不同，长度为1字节、2字节或5字节。当程序读入db_number部分后，服务器啊会调用SELECT命令，根据读入的数据库号码进行数据库切换。
key_value_pairs保存了数据库中所有键值对数据，如果键值对带有过期时间，那么过期时间也会和键值对保存在一起。

2.key_value_pairs  
不带过期时间的键值对结构
TYPE | key | value
带过期时间的键值对结构
EXPIRETIME_MS | ms | TYPE | key | value
其中，TYPE记录value的类型，是以下常量中任意一个：
REDIS_RDB_TYPE_STRING  
REDIS_RDB_TYPE_LIST  
REDIS_RDB_TYPE_SET  
REDIS_RDB_TYPE_ZSET  
REDIS_RDB_TYPE_HASH  
REDIS_RDB_TYPE_LIST_ZIPLIST  
REDIS_RDB_TYPE_SET_INSERT  
REDIS_RDB_TYPE_ZSET_ZIPLIST  
REDIS_RDB_TYPE_HASH_ZIPLIST  
key总是一个字符串对象，编码和REDIS_RDB_TYPE_STRING类型的value一样。  
value根据TYPE类型不同，结构和长度也会不同。  
EXPIRETIME_MS常量长度为1字节，当程序读入该常量时，会表示下一个读入的将是一个以毫秒为单位的过期时间。  
ms是一个8字节长的带符号整数，记录了一个以毫秒为单位的unix时间戳，即过期时间。

3.value编码  
a.字符串对象  
TYPE是REDIS_RDB_TYPE_STRING，字符串对象编码可以是REDIS_ENCODING_INT或者REDIS_ENCODING_RAW，对应保存的值类型是整数和字符串。  
如果字符串对象编码为REDIS_ENCODING_RAW，字符串长度若大于20字节，那么这个字符串会被压缩保存，否则原样保存。  
无压缩字符串结构  
len | string  
压缩后结构  
REDIS_RDB_ENC_LZF | compressed_len | origin_len | compressed_string  
其中REDIS_RDB_ENC_LZF常量标志字符串已被LZF算法压缩，程序在读入过程中，碰到这个常量时，会根据之后的compressed_len和orgin_len和compressed_string 三部分对字符串进行解压。

b.列表对象  
TYPE为REDIS_RDB_TYPE_LIST，value保存的是一个REDIS_ENCODING_LINKEDLIST编码的列表对象，结构如下  
list_length | item1| item2 | item3 | ... | itemN  
list_length记录了列表的长度，item部分代表列表的项，每项都是字符串对象。

c.集合对象  
TYPE为REDIS_RDB_TYPE_SET，value保存的是一个REDIS_ENCODING_HT编码的对象集合，结构如下  
set_size | elem1 | elem2 | elem3 | ... | elemN  
set_size记录集合大小，elem开头部分代表元素集合，每个元素都是一个字符串对象。

d.哈希表对象  
TYPE为REDIS_RDB_TYPE_HASH，value保存的是一个REDIS_ENCODING_HT编码的集合对象，结构如下  
hash_size | key_value_pair 1 | key_value_pair 2 | key_value_pair 3 | ... | key_value_pair N  
hash_size记录了哈希表的大小。key_value_pair部分代表键值对，结构中的每个键值对都以键紧挨着值的方式排列。结构如下  
key1 | value1 | key2 | value2 | key3 | value3 | ....

e.有序集合对象  
TYPE为REDIS_RDB_TYPE_ZSET，value保存的是一个REDIS_ENCODING_SKIPLIST编码的有序集合对象。结构如下  
sorted_set_size | elem1 | elem2 | elem3 | ... | elemN  
sorted_set_size记录有序集合大小，elem开头部分代表有序集合中的元素，每个元素又分为成员 (member)和分值(score)两部分，成员是一个字符串对象，分值则是一个double类型的浮点数，程序在保存RDB文件时会先将分支转换成字符串对象，再用保存字符串的方法将分值保存起来。  
有序集合中每个元素都是以成员紧挨着分值方式排列，结构如下  
sorted_set_size | member1 | score 1 | member2 | score2 | ... | memberN | scoreN

f.INTSET编码集合  
TYPE为REDIS_RDB_TYPE_SET_INTSET，value保存的是一个整数集合对象，RDB保存这种对象方法是先将整数集合转换为字符串对象，再保存到RDB文件中。读取是相反。

g.ZIPLIST编码的列表、哈希表或有序集合  
TYPE为REDIS_RDB_TYPE_LIST_ZIPLIST、REDIS_RDB_TYPE_HASH_ZIPLIST或者REDIS_RDB_TYPE_ZSET_ZIPLIST，那么value保存的是一个压缩列表对象，RDB文件保存方法是:
1)将压缩列表转换成一个字符串对象；
2)将转换所得的字符串对象保存到RDB文件。
读取时做对应类型转换即可。
