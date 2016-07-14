---
layout: post
title:  Linux系列： epoll  高并发的核心
description:  Linux epoll 以及 基于epoll的Reactor模式
category: blog
---
众所周知， `epoll`是`Linux`从`Kernel 2.6`开始引入的处理`I/O`多路复用的新方法。其相比以前的`select`，`poll`，
在处理大量开放连接的性能上有了非常大的提高。

我甚至觉得只要谈到和网络相关的高并发，在`Linux`下各种软件的底层都会优先使用`epoll`。列举几个我知道的：

>鼎鼎大名的 `nginx`，无需介绍；

>事件驱动的`Node.js`语言；

>近几年来越来越火热的高性能`key-value`内存数据库`redis`； 

>使用`Python`开发的全栈式`Web`框架和异步网络库， `tornado`；

在`Linux`下，它们在处理网络`I/O`事件时都无一例外的使用了`epoll`。那么`epoll`到底是什么？它的高并发
能力又是如何实现的？

推荐一篇文章，讲的比较详细：
[`linux`下`epoll`如何实现高效处理百万句柄的](http://blog.csdn.net/russell_tao/article/details/7160071)

我们知道， `epoll`相关的系统调用很简洁，只有三个：

```c
int epoll_create(int size);  

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  

int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```

`epoll_create`是用来创建一个 `epoll`对象，完成一些需要的数据结构的建立；

`epoll_ctl`用来注册`socket`等到`epoll`，或者从`epoll`中移除正在被监控的`socket`等；

`epoll_wait`将注册在epoll的准备就绪的句柄返回然后就可以进行处理。

大致了解了`epoll`，再来说说处理并发`I/O`比较常见的一种模式，`Reactor`模式。
中心思想是将所有要处理的`I/O`事件注册到一个中心`I/O`多路复用器上,同时主线程阻塞在多路复用器上；
一旦有`I/O`事件到来或是准备就绪，多路复用器返回并将相应`I/O`事件分发到对应的处理器中。

所以，从代码层面，`Reactor`模式的核心，是一个大的无限循环，在这个循环里，调用`epoll_ctl`,
将`socket`等注册到`epoll`，再调用`epoll_wait`,等待`epoll_wait`返回，然后遍历`epoll_wait`返回
的结果，并用对应的`epoll_ctl`注册的回调函数进行处理，最后回到循环的开始, 周而复始。

一些基于`Reactor`模式的软件或库不光能处理`I/O`事件，还在事件循环里加入了对定时事件的处理。

定时事件的处理，很巧妙的利用了`epoll_wait`的一个参数`timeout`，每次调用`epoll_wait`时，
其参数`timeout`就等于下次定时事件的到期时间，`epoll_wait`如果在`timeout`内有`I/O`事件
返回，就继续处理这些事件，如果等到`timeout`没有`I/O`事件，那么也会因为`timeout`到期而
返回，此时也表示有定期事件到期了，就会在下一次调用`epoll_wait`之前处理到期事件。

这样，我们就有能力去执行一些在将来某个时间点发生的事件，或者是一些需要周期执行的事件。

下一篇会分析一下`tornado`的`ioloop`源码，看一下`tornado`对上面这个`Reactor`模式的实现。
