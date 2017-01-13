---
title: Redis持久化-RDB
date: 2015-07-01
categories: Redis设计与实现
tags: Redis
---

RDB持久化可以自动，也可以手动，这个功能将某个时间点上的数据库状态保存到一个经过压缩的二进制文件中。

SAVE命令会阻塞服务，执行完成前客户端的所有请求都会拒绝。
BGSAVE命令会新建一个子进程执行持久化操作，不阻塞服务。

服务器状态维护了一个dirty计数器以及一个lastsave属性，供RDB持久化策略使用。
dirty计数器记录距离上一次成功执行SAVE或BGSAVE操作后，服务器中所有数据库进行了多少次修改操作（增删改）；
lastsave属性是一个unix时间戳，记录上一次RDB持久化的时间。
