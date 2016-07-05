---
layout: post
title:  线程池（ThreadPool）
description: 从 Python ThreadPoolExecutor 源码剖析看线程池的使用和实现
category: blog
---
线程池，顾名思义，就是一个像水池一样的数据结构，里面养有一些线程，想用的时候就从里面取，不用的时候再放回去。所以从概念上讲线程池还是比较简单的。

### 线程池使用场景分析：
首先来分析一个线程的产生与销毁过程。
 
 >* T1: 创建线程时间；
 >* T2: 在线程中执行任务的时间；
 >* T3: 销毁线程时间。

线程的产生与销毁花费时间为 `T1+T3`，真正用来做任务的时间为`T2`，如果任务正好需要频繁执行而且单个任务花费时间又较短，那么就需要频繁的产生和销毁线程，总体来看`T1+T3`就会比较大，影响执行效率。

而如果采用了线程池，在任务执行之前就保存一批线程，想用的时候就拿过来用，用完放回去，就不需要线程不断的建立和销毁，也就是说节省了`T1+T3`的时间。

所以可以看出来线程池比较适合处理一些相互独立而又频繁执行的小任务。

### 线程池大小
这个还是需要考虑具体场景的任务类型。
如果我们放到线程池的任务属于` I/O-bound `，也就是大部分时间花在`I/O`上面，那么我们就可以多开一些固定线程放到线程池，因为大部分时候线程池里的任务都是`I/O`等待状态，是不会占用`CPU`的。这样，我们就可以实现较高的任务并发；

相反，如果我们的任务都是一些占用计算资源的，也就是`CPU-bound `，那么开过多的线程就不好了，因为这样频繁的线程上下文切换带来的花销并不合算。

### 使用线程池

Python里面的 [concurrent.futures](https://docs.python.org/dev/library/concurrent.futures.html) 提供了两种并发实现，一种是基于多进程，另一种基于多线程。
我们这里主要关注多线程的实现。多线程的实现使用的就是线程池；
先看一下使用方法：

```python
In [2]: from concurrent.futures import ThreadPoolExecutor

In [3]: with ThreadPoolExecutor(max_workers=1) as executor:
   ...:     future = executor.submit(pow, 323, 1235)
   ...:     print(future.result())
```

是的，非常简单，只要我们指定`max_workers`即线程池的大小，然后向线程池丢任务函数就可以了。至于`max_workers`的大小，已经说过了，可以根据具体的使用场景来设定。

### 实现线程池

上面我们在Python里使用的线程池的官方实现代码：   [ThreadPoolExecutor](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py)

一个完备的线程池至少应该有这4项组成：

>* 任务队列: 用于存放没有处理的任务。提供一种缓冲机制。
>* 工作线程: 线程池中线程
>* 线程池管理器: 用于创建并管理线程池
>* 任务接口: 每个任务必须实现的接口，以供工作线程调度任务的执行。

下面我们就来剖析一下它的源码来看看线程池的这几个构成是如何体现的。
线程池被封装成了一个类： [`class ThreadPoolExecutor(_base.Executor)`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l83)

任务队列为： [`self._work_queue = queue.Queue()`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l99)

工作线程：即为线程池里的线程，刚开始，每向线程池提交一个任务就会产生一个工作线程，直到工作线程的数量达到指定的线程池大小`max_workers`为止。以后再向线程池添加任务会继续放到任务队列里面，但不会开新的线程了。

线程池管理器：应该有这些功能，创建线程池，销毁线程池，添加新任务等；但是在代码里面这些个功能是分布在不同函数里面的，创建线程池是在 [`def _adjust_thread_count(self)`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l117)这里不断的创建新的线程并放到 `_threads_queues` 里面；销毁线程池是在 [`def shutdown(self, wait=True)`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l133)这个函数里面；添加新任务就是我们最常使用的 [`def submit(self, fn, *args, **kwargs)`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l104) 函数。
任务接口： [`class _WorkItem(object)`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l43) 提供了对任务函数的包装；[`def _worker(executor_reference, work_queue)`](https://hg.python.org/cpython/file/default/Lib/concurrent/futures/thread.py#l61) 给工作线程提供了执行入口。
