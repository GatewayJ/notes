---
author: jihongwei
tags:
  - linux
date: 2021-03-09T11:24:00.000Z
title: epoll数据结构
---

# epoll数据结构

**流程**

1. 当进程调用epoll\_create时 在内核高速cache区创建一个epoll结构体，该结构体由红黑树`rbtree`(可以快速的区分出是否添加过重复事件，这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层,就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的内存。通过这棵树来管理用户进程下添加进来的所有socket连接。)和双向链表`rdlist`(就绪的描述符的链表。当有的连接就绪的时候，内核会把就绪的连接放到rdllist链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树),`wq`(等待队列链表。软中断数据就绪的时候会通过wq来找到阻塞在epoll对象上的用户进程。)组成。红黑树存储着所有需要监听的事件，`rdlist`链表中存贮着将要通过epoll\_wait返回给用户满足条件的事件.`wq`等待队列链表,软中断数据就绪的时候会通过wq来找到阻塞在epoll对象上的用户进程
2. 所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫ep\_poll\_callback,它会将发生的事件添加到rdlist双链表中。当我们执行epoll\_ctl时，除了把socket放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后就来把socket插入到准备就绪链表里了。

***

1. 进程通过 epoll\_create 创建 eventpoll 对象。
2. 进程通过 epoll\_ctl 添加关注 listen socket 的 EPOLLIN 可读事件。
3. 接步骤 2，epoll\_ctl 还将 epoll 的 socket 唤醒等待函数回调（唤醒函数：ep\_poll\_callback）通过 add\_wait\_queue 函数添加到 socket.wq 等待队列。

> 当 listen socket 有链接资源时，内核通过 \_\_wake\_up\_common 调用 epoll 的 ep\_poll\_callback 唤醒函数，唤醒进程。

1. 进程通过 epoll\_wait 等待就绪事件，往 eventpoll.wq 等待队列中添加当前进程。当 epoll\_ctl 监控的 socket 产生对应的事件时，被唤醒返回。
2. 客户端通过 tcp connect 链接服务端，三次握手成功，第三次握手在服务端进程产生新的链接资源。
3. 服务端进程根据 socket.wq 等待队列，唤醒正在等待资源的进程处理。例如 nginx 的惊群现象，\_\_wake\_up\_common 唤醒等待队列上的两个等待进程，调用 ep\_poll\_callback 去唤醒 epoll\_wait 阻塞等待的进程。 t. ep\_poll\_callback 唤醒回调会检查 listen socket 的完全队列是否为空，如果不为空，那么就将 epoll\_ctl 监控的 listen socket 的节点 epi 添加到 就绪队列：eventpoll.rdllist，然后唤醒 eventpoll.wq 里通过 epoll\_wait 等待的进程，处理 eventpoll.rdllist 上的事件数据。
4. 睡眠在内核的 epoll\_wait 被唤醒后，内核通过 ep\_send\_events 将就绪事件数据，从内核空间拷贝到用户空间，然后进程从内核空间返回到用户空间。
5. epoll\_wait 被唤醒，返回用户空间，读取 listen socket 返回的 EPOLLIN 事件，然后 accept listen socket 完全队列上的链接资源。

在epoll中，对于每一个事件，都会建立一个epitem结构体

```
struct epitem{
    struct rb_node  rbn;//红黑树节点
    struct list_head    rdllink;//双向链表节点
    struct epoll_filefd  ffd;  //事件句柄信息
    struct eventpoll *ep;    //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

struct eventpoll对象的详细结构 ![clickhouse](../image/epollstruct.webp)

1. 当调用epoll\_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数两返回给用户，使用mmap减少复制开销

![基本数据结构](../image/rdlist.jpg)

![工作流程](../image/epollflow.webp)

**触发方式**

![普片](../image/epoll.png)

1. 水平触发的时机

* 对于读操作，只要缓冲内容不为空，LT模式返回读就绪。
* 对于写操作，只要缓冲区还不满，LT模式会返回写就绪。

当被监控的文件描述符上有可读写事件发生时，epoll\_wait()会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 epoll\_wait()时，它还会通知你在上没读写完的文件描述符上继续读写，当然如果你一直不去读写，它会一直通知你。如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率。

1. 边缘触发的时机

* 对于读操作：

当缓冲区由不可读变为可读的时候，即缓冲区由空变为不空的时候。

当有新数据到达时，即缓冲区中的待读数据变多的时候。

当缓冲区有数据可读，且应用进程对相应的描述符进行EPOLL\_CTL\_MOD 修改EPOLLIN事件时。

* 对于写操作：

当缓冲区由不可写变为可写时。

当有旧数据被发送走，即缓冲区中的内容变少的时候。

当缓冲区有空间可写，且应用进程对相应的描述符进行EPOLL\_CTL\_MOD 修改EPOLLOUT事件时。

当被监控的文件描述符上有可读写事件发生时，epoll\_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll\_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。

在ET模式下， 缓冲区从不可读变成可读，会唤醒应用进程，缓冲区数据变少的情况，则不会再唤醒应用进程。

> https://mp.weixin.qq.com/s?\_\_biz=MzUyNzgyNzAwNg==\&mid=2247483925\&idx=1\&sn=1ac3e863594745c7466b0e88a688b203\&scene=21#wechat\_redirect https://new.qq.com/omn/20220329/20220329A09C7900.html
