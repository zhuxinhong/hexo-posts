---
title: Redis时间事件
date: 2015-07-12
categories: Redis设计与实现
tags: Redis
---

Redis时间事件分为以下两类：  
1.定时事件：程序在指定时间执行一次。  
2.周期性事件：程序每隔指定时间执行一次。  

时间事件的属性：  
1.id：服务器为时间事件创建的全局唯一ID，ID号从小到大递增。  
2.when：毫秒精度的unix时间戳，记录时间事件的到达时间。  
3.timeProc：时间事件处理器，一个函数。当时间事件到达时，执行此函数。  

时间事件的返回值决定了时间事件类型，如返回ae.h/AE_NOMORE，表示事件为定时事件，到达一次后则删除；如返回一个非AE_NOMORE的整数，表示事件为周期性事件，当事件到达之后，服务器会根据返回值更新时间事件的when属性，并以这种方式一直更新下去。当前Redis版本中只有周期性事件，没有使用定时事件。  

实现：  
服务器将所有时间事件都存放在一个无序列表中，每当时间事件执行器执行时，它就遍历整个链表，找到所有已到达的时间事件并调用相应事件处理器。这里的的无序链表，指的是不按when属性大小排序，其实是按ID排序了，新的时间事件总是插入链表的表头。当前Redis版本中，服务器只使用serverCron一个时间事件，而在benchmark模式下，服务器也只使用2个时间事件，所以使用无序链表来保存时间事情，并不影响性能。  

事件调度伪代码：  
``` bash
def aeProcessEvents():
//获取到达时间离当前时间最接近的时间事件
time_event = aeSearchNearestTimer()
//计算最接近的时间事件距离到达还有多少毫秒
remain_ms = time_event.when - unix_ts_now()
//如果事件已到达，那么remain_ms 值可能为负数，置为0
if remain_ms < 0:
remain_ms = 0
//根据remain_ms值，创建timeval结构
timeval = create_timeval_with_ms(remain_ms)
//阻塞并等待文件事件产生，最大阻塞时间由传入的timeval结构决定
//如果remaind_ms值为0，那么aeApiPoll调用之后马上返回，不阻塞
aeApiPoll(timeval)
//处理所有已产生的文件事件
processFileEvents()
//处理所有已到达的时间事件
processTimeEvents()
```

