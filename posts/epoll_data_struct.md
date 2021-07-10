---
author : "jihongwei"
tags : ["linux"]
date : 2021-03-09T11:24:00Z
title : "epoll数据结构"
---

1. 当进程调用epoll_create时 创建一个epoll结构体，该结构体由红黑树`rbr`和双向链表`rdlist`组成。红黑树存储着所有需要监听的事件，链表中存贮这将要通过epoll_wait返回给用户满足条件的事件。


2. 所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系，也就是说，当相应的事件发生时会调用这个回调方法。这个回调方法在内核中叫ep_poll_callback,它会将发生的事件添加到rdlist双链表中。


在epoll中，对于每一个事件，都会建立一个epitem结构体
```c++
struct epitem{
    struct rb_node  rbn;//红黑树节点
    struct list_head    rdllink;//双向链表节点
    struct epoll_filefd  ffd;  //事件句柄信息
    struct eventpoll *ep;    //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

3. 当调用epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可。如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数两返回给用户


![clickhouse](https://demoio.cn:90/blog-image/epoll.jpg)


