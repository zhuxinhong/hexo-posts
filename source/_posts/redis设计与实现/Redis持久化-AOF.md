---
title: Redis持久化-AOF
date: 2015-07-11
categories: Redis设计与实现
tags: Redis
---

Redis提供了AOF(Append Only File)持久化功能，AOF通过保存服务器所执行命令来记录数据库状态的，和MySQL记录增删改的log有点像哈。

AOF持久化功能实现分为命令追加(append)、文件写入(write)和文件同步(sync)三个步骤。

1.AOF命令追加  
当开启AOF持久化功能时，服务器会在执行完一个写命令后，会以协议格式将执行的写命令追加到服务器状态的aof_buf缓冲区末尾。

2.AOF文件写入和同步  
Redis服务进程就是一个事件循环，循环中的文件事件负责接受客户端的命令请求，以及向客户端发送命令回复，而时间事件则负责执行像serverCron函数这样需要定时运行的函数。
``` bash
def eventLoop():
while True:
	//处理文件事件，接收命令以及发送命令
	//处理命令时可能会有新内容被追加到aof_buf中
	processFileEvents()
	//处理时间事件
	processTimeEvents()
	//考虑是否将aof_buf中内容写入和保存到aof文件
	flushAppendOnlyFile()
```

flushAppendOnlyFile函数的行为由服务器配置的appendfsync选项值来决定。  
appendfsync配置：  
always : 服务器在每个事件循环都要将aof_buf缓冲区的所有内容写到AOF文件，并同步，效率最慢，但最安全。  
everysec : 服务器在每个事件循环都将aof_bug缓冲区的所有文件写到AOF文件，并且每隔1s就在子线程中对AOF文件做同步，效率快。  
no : 服务器在每个事件循环都将aof_bug缓冲区的所有文件写到AOF文件，此模式下的flushAppendOnlyFile无须执行同步操作，同步AOF文件由操作系统决定，写入速度最快。

3.AOF重写  
随着服务器运行时间的流逝，AOF文件中的内容会越来越多，可能会对Redis服务器和宿主计算机造成影响。  
虽然Redis将生成新AOF文件替换旧AOF文件功能命名为AOF文件重写，但实际上，AOF文件重写并不需要对现有AOF文件进行任何读取、分析和写入操作，这个功能是通过读取服务器当前数据库状态来实现的。
实际操作中，Redis依次遍历数据库中所有key，并根据key类型的不同，取出相应值并生成对应类型的写入语句。同时，为了避免在执行命令时造成客户端输入缓冲区溢出，重写程序在处理列表、哈希表、集合、有序集合这4种可能会带有多个元素的键时，会先检查键所包含元素的数量，如果数量超过了redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD常量的值，那重写程序会使用多条命令来记录键的值。当前版本中，REDIS_AOF_REWRITE_ITEMS_PER_CMD常量值为64，所以每条语句最多会写入64个元素。

4.AOF后台重写  
上面介绍的AOF重写可以很好的完成创建一个新AOF文件的任务，但是，这个操作会进行大量的写入操作，所以调用这个函数的线程将被长时间阻塞，因为Redis服务器使用单个线程来处理命令请求，所以如果由服务器调用AOF重写，那么在重写期间，服务器将无法处理客户端发来的请求。
Redis并不希望AOF的重写操作造成服务器无法处理请求，所以需要将重写放到子进程里执行，这样做可以达到两个目的：  
1)子进程执行AOF重写期间，服务器可以继续处理客户端的命令请求。  
2)子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据安全性。

不过这里仍然存在一个问题：在子进程执行AOF重写期间，服务器处理客户端的命令请求时，对现有数据库状态进行修改操作，会使得服务器当前数据库状态和重写后的AOF文件所保存的数据库状态不一致。
为了解决这个问题，Redis服务器设置了一个AOF重写缓冲区，这个缓冲区在服务器创建子进程之后开始使用，当Redis执行完一个写命令后，它会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区。
以上就是AOF后台重写，即BGREWRITEAOF命令的实现原理。
