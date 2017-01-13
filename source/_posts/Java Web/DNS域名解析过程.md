---
title: DNS域名解析过程
date: 2016-05-11
categories: Java Web
tags: DNS 
---

当用户在浏览器输入www.abc.com，DNS解析将会有将近10个步骤。

1.	浏览器检查缓存有没有该域名对应的IP。如果有，这个解析过程就结束。浏览器缓存域名是有限制的，不仅大小有限制，时间也有限制，通过TTL属性设置域名缓存时间。
2. 浏览器缓存中没有，检查操作系统缓存中是否有解析结果。如果有，解析结束。没有的话，请求域名服务器解析该域名。
3. 网络配置有配置DNS服务器地址，操作系统把域名发送给这里的LDNS。Linux下通过以下命令查看。  
```
cat /etc/resolv.conf
```
4.	LDNS未命中，直接到Root Server域名服务器请求解析。
5. 根域名服务器返回本地域名服务器一个服务器地址(gTLD Server)。gTLD是国际顶级域名服务器，如.com、.cn、.org等，全球只有13台左右。
6. 本地域名服务器再向上一步返回的gTLD服务器发送请求。
7. gTLD服务器查找并返回此域名对应的Name Server域名服务器地址，这个Name Server通常就是你注册的域名服务器。
8. Name Server域名服务器查询存储的域名和IP映射关系，正常情况下查询到目标IP记录，连同一个TTL值返回给DNS Server域名服务器。
9. 返回该域名对应的IP和TTL值，Local DNS Server会缓存该记录。
10. 解析结果返回给用户，用户根据TTL值缓存这个记录。

实际过程中，可能不止这个10个步骤，如Name Server有多级或者有个GTM负载均衡控制，这都会影响域名解析的过程。

#	跟踪域名解析过程

##	dig
```
--- ~ » dig www.google.com

; <<>> DiG 9.8.3-P1 <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50606
;; flags: qr rd ra; QUERY: 1, ANSWER: 16, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		299	IN	A	61.238.239.35
www.google.com.		299	IN	A	61.238.239.20
www.google.com.		299	IN	A	61.238.239.24
www.google.com.		299	IN	A	61.238.239.59
www.google.com.		299	IN	A	61.238.239.29
www.google.com.		299	IN	A	61.238.239.30
www.google.com.		299	IN	A	61.238.239.45
www.google.com.		299	IN	A	61.238.239.40
www.google.com.		299	IN	A	61.238.239.55
www.google.com.		299	IN	A	61.238.239.34
www.google.com.		299	IN	A	61.238.239.44
www.google.com.		299	IN	A	61.238.239.49
www.google.com.		299	IN	A	61.238.239.54
www.google.com.		299	IN	A	61.238.239.25
www.google.com.		299	IN	A	61.238.239.39
www.google.com.		299	IN	A	61.238.239.50

;; Query time: 61 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Wed May 11 21:04:56 2016
;; MSG SIZE  rcvd: 288

--- ~ » 
```

## dig +cmd
```
--- ~ » dig +printcmd www.taobao.com
Invalid option: +printcmd
Usage:  dig [@global-server] [domain] [q-type] [q-class] {q-opt}
            {global-d-opt} host [@local-server] {local-d-opt}
            [ host [@local-server] {local-d-opt} [...]]

Use "dig -h" (or "dig -h | more") for complete list of options
--- ~ » dig +cmd www.taobao.com                                                                                                                                        1 ↵

; <<>> DiG 9.8.3-P1 <<>> +cmd www.taobao.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62882
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.taobao.com.			IN	A

;; ANSWER SECTION:
www.taobao.com.		71	IN	CNAME	www.taobao.com.danuoyi.tbcache.com.
www.taobao.com.danuoyi.tbcache.com. 599	IN A	205.204.104.227

;; Query time: 73 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Wed May 11 21:07:41 2016
;; MSG SIZE  rcvd: 93

--- ~ » 
```

##	dig +trace
```
--- ~ » dig www.google.com +trace                                                                                                                                      9 ↵

; <<>> DiG 9.8.3-P1 <<>> www.google.com +trace
;; global options: +cmd
.			5236	IN	NS	a.root-servers.net.
.			5236	IN	NS	b.root-servers.net.
.			5236	IN	NS	c.root-servers.net.
.			5236	IN	NS	d.root-servers.net.
.			5236	IN	NS	e.root-servers.net.
.			5236	IN	NS	f.root-servers.net.
.			5236	IN	NS	g.root-servers.net.
.			5236	IN	NS	h.root-servers.net.
.			5236	IN	NS	i.root-servers.net.
.			5236	IN	NS	j.root-servers.net.
.			5236	IN	NS	k.root-servers.net.
.			5236	IN	NS	l.root-servers.net.
.			5236	IN	NS	m.root-servers.net.
;; Received 228 bytes from 8.8.8.8#53(8.8.8.8) in 63 ms

com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
;; Received 504 bytes from 192.5.5.241#53(192.5.5.241) in 38 ms

;; connection timed out; no servers could be reached
--- ~ »   
```


#	几种域名解析方式
*	A记录，指定域名对应IP。
* 	MX记录，表示Mail Exchange，将某个域名下邮件服务器指向自己Mail Server。正常通过Web请求仍然解析A记录IP。
*  CNAME记录，Canonical Name(别名解析)。
*  NS记录，为某个域名指定DNS解析服务器。
*  TXT记录，为某个主机名或域名设置说明。