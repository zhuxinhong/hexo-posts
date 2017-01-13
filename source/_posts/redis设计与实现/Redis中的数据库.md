---
title: Redis中的数据库
date: 2015-07-01
categories: Redis设计与实现
tags: Redis
---

Redis服务器的所有数据库都保存在服务器状态 redis.h/redisServer结构的db数组中，db数组中每项都是一个 redis.h/redisDb结构，代表一个数据库实例。

``` bash
struct redisServer {
	//...
	int dbnum;
	//...
	redisDb *db;
	//...
} ;
```


初始化数据库实例默认是16个，可以用过select 0~15 来切换。
``` bash
typeof struct redisClient {
	//...
	//记录客户端当前正在使用的数据库
	redisDb *db;
	//...
} redisClient;
```

redisDb 结构中的dict字典保存了数据库中的所有键值对，这个字典称之为键空间。
``` bash
typeof struct redisDb {
	//...
	dict *dict
	//...
} redisDb;
```
我们在对键进行增删改查操作时候，其实就是在对dict的字典结构进行对应操作。

读写键的维护操作：
1.读取一个key后(读和写都要先读)，服务器会更新键空间命中次数和不命中次数。这2个值可以在INFO命令的status属性 keyspace_hit和keyspace_misses查看。
2.读取一个key后，服务器会更新key的LRU(最后一次使用)时间，这个值可以用于计算key的闲置时间，使用Object idletime $key 命令可以查看。
3.读取一个key时，发现该key已过期，会先删除。
4.如果有watch命令监视了某个key，服务器在对该key进行修改后，会将这个键标记为脏(dirty)，从而让此key暴露给事物。
5.服务器每修改一个key后，会对脏键计数器+1，这个计数器会触发redis的持久化以及复制操作。
6.如果服务器开启了数据库通知功能，那么在key被修改后，redis会按配置发送相应的通知。



Redis有4个命令可以设置键的过期时间：
1. expire <key> <ttl> 设置key的生存时间为ttl秒；
2. pexpire <key> <ttk> 设置key的生存时间为ttl毫秒； 
3. expireat <key> <ts> 设置key的过期时间为ts的秒数时间戳；
4. pexpireat <key> <ts> 设置key的过期时间为ts的毫秒数时间戳；

实际上 expire, pexpire, expireat 3个命令都是使用pexpireat实现的，只需要转换时间单位就行了。

redisDb结构中，有一个expires字典保存了所有key的过期时间，这个字典称之为过期字典。
typeof struct redisDb {
//...
dict *expires;
//...
} redisDb;

当一个key被设为过期后，redisDb中键空间字典和过期字典中的键重复，并不会出现重复对象，2个key指向的都是同一个对象。

key过期删除策略：
1.定时删除：设置过期键的时候，创意一个定时器；
2.惰性删除：放任过期键不管，每次访问时，判断是否过期，如过期则删除；
3.定期删除：每隔一段时间，redis会扫描过期键，发现过期则删除；

Redis实际使用的是惰性删除和定期删除2种配合使用。
定期删除函数执行步骤：
1.从一定量的数据库中去除一定数量的随机键检查，删除其中的过期键；
2.全局变量current_db会记录当前函数检查的进度，并在下一次调用时，接着上一次的进度处理。
3.随着函数不断被执行，所有过期键都会被检查到，这时将current_db变量重置为0，再开始新一轮的检查。
