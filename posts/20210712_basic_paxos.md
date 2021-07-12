---
author : "jihongwei"
tags : ["分布式"]
date : 2021-01-26T11:24:00Z
title : "paxos基本工作方式"
---



####  proposer：
proposer选择一个新的提案编号Mn，然后向某个acceptor集合（至少需满足majority）发送请求，要求该集合中的acceptor做出如下回应：
保证不再批准任何编号小于Mn的提案。
如果acceptor已经批准过任何提案，那么其就向proposer反馈当前该acceptor已经批准的编号小于Mn但为最大编号的那个提案的值。
将该请求成为编号Mn的提案的prepare请求。
如果proposer收到了来自半数以上（majority）的acceptor的反馈，那么有两种情况：
可以产生编号为Mn、value值为Vn的提案，其中Vn是所有响应中编号最大的提案的value值。
返回的所有反馈中，都没有批准过任何提案，即响应中不包含任何的提案，那么此时Vn值就可以由proposer任意选择。

#### acceptor：
接收到一个编号为Mn的prepare请求：
编号Mn大于该acceptor已经响应的所有prepare请求的编号，那么它就会将它已经批准过的最大编号的提案作为响应反馈给proposer，同时该acceptor承诺不会再批准任何编号小于Mn的提案。
反之，由于该acceptor已经对编号大于Mn的prepare请求做出了响应，那么此时该acceptor肯定不会批准编号Mn的提案，因此可以选择忽略该prepare请求，也可以发送错误码让proposer更快地感知到结果。
接收到 [Mn，Vn] 提案的accept请求：
只要该acceptor尚未对编号大于Mn的prepare请求做出响应，它就可以通过这个提案。

![paxos](https://demoio.cn:90/blog-image/paxos.png)


basic paxos可能会出现活锁，因此可以(multi-paxos)选举出一个leader，来执行一次prepare，多次accept。


>https://blog.csdn.net/wbin233/article/details/85258836
https://www.cnblogs.com/foxmailed/p/5487533.html