---
author : "jihongwei"
tags : ["linux"]
date : 2018-04-18T11:24:00Z
title : "零拷贝 MMAP"
---


https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy2/index.html


***linux中的直接io***

![direct-io](https://demoio.cn:90/blog-image/零拷贝-直接io.png)


&nbsp;&nbsp;针对数据传输不需要经过应用程序地址空间的零拷贝技术

1. 利用 mmap()

&nbsp;&nbsp;首先，应用程序调用了 mmap() 之后，数据会先通过 DMA 拷贝到操作系统内核的缓冲区中去。接着，应用程序跟操作系统共享这个缓冲区，这样，操作系统内核和应用程序存储空间就不需要再进行任何的数据拷贝操作。应用程序调用了 write() 之后，操作系统内核将数据从原来的内核缓冲区中拷贝到与 socket 相关的内核缓冲区中。接下来，数据从内核 socket 缓冲区拷贝到协议引擎中去，这是第三次数据拷贝操作。


![mmap](https://demoio.cn:90/blog-image/零拷贝-mmap.png)


2. sendfile()

&nbsp;&nbsp;为了简化用户接口，同时还要继续保留 mmap()/write() 技术的优点：减少 CPU 的拷贝次数，Linux 在版本 2.1 中引入了 sendfile() 这个系统调用。

&nbsp;&nbsp;sendfile() 不仅减少了数据拷贝操作，它也减少了上下文切换。首先：sendfile() 系统调用利用 DMA 引擎将文件中的数据拷贝到操作系统内核缓冲区中，然后数据被拷贝到与 socket 相关的内核缓冲区中去。接下来，DMA 引擎将数据从内核 socket 缓冲区中拷贝到协议引擎中去。如果在用户调用 sendfile () 系统调用进行数据传输的过程中有其他进程截断了该文件，那么 sendfile () 系统调用会简单地返回给用户应用程序中断前所传输的字节数，errno 会被设置为 success。如果在调用 sendfile() 之前操作系统对文件加上了租借锁，那么 sendfile() 的操作和返回状态将会和 mmap()/write () 一样。


![sendfile](https://demoio.cn:90/blog-image/零拷贝-sendfile.png)

3. 带有 DMA 收集拷贝功能的 sendfile()

&nbsp;&nbsp;上小节介绍的 sendfile() 技术在进行数据传输仍然还需要一次多余的数据拷贝操作，通过引入一点硬件上的帮助，这仅有的一次数据拷贝操作也可以避免。为了避免操作系统内核造成的数据副本，需要用到一个支持收集操作的网络接口，这也就是说，待传输的数据可以分散在存储的不同位置上，而不需要在连续存储中存放。这样一来，从文件中读出的数据就根本不需要被拷贝到 socket 缓冲区中去，而只是需要将缓冲区描述符传到网络协议栈中去，之后其在缓冲区中建立起数据包的相关结构，然后通过 DMA 收集拷贝功能将所有的数据结合成一个网络数据包。网卡的 DMA 引擎会在一次操作中从多个位置读取包头和数据。Linux 2.4 版本中的 socket 缓冲区就可以满足这种条件，这也就是用于 Linux 中的众所周知的零拷贝技术，这种方法不但减少了因为多次上下文切换所带来开销，同时也减少了处理器造成的数据副本的个数。对于用户应用程序来说，代码没有任何改变。首先，sendfile() 系统调用利用 DMA 引擎将文件内容拷贝到内核缓冲区去；然后，将带有文件位置和长度信息的缓冲区描述符添加到 socket 缓冲区中去，此过程不需要将数据从操作系统内核缓冲区拷贝到 socket 缓冲区中，DMA 引擎会将数据直接从内核缓冲区拷贝到协议引擎中去，这样就避免了最后一次数据拷贝。

![dma-sendfile](https://demoio.cn:90/blog-image/零拷贝-dma-sendfile.png)

&nbsp;&nbsp;零拷贝是尽量避免将数据从内核空间拷贝到用户空间，同时尽量减少数据的copy



