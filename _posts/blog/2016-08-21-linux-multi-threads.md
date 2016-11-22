---
layout: post
title: Linux系列： 线程同步机制
description: 各种线程同步方法
category: blog
---

在多线程编程中，永远绕不开的话题就是线程安全以及线程之间的同步问题。

在我看来，这些问题的根源都是因为，在多线程环境下，程序执行顺序的不确定性。也就是说，在程序运行中，我们永远不知道在下一个时间点哪个线程会执行，以及每个线程执行到哪了。这是造成多线程编程变得异常复杂的根本原因。

试想，在单线程的环境下，我们只需要按照自己的思路一步一步写代码，在写的过程中，就可以推测出这一行执行完，下一步会执行哪一行。这样，代码写完了，程序在脑海中也执行完了。

在多线程编程中，除了需要理清单个线程内的运行逻辑，又要考虑不同线程之间的交互（也许这些交互并不是你想要的）。这些交互的地方，从代码的角度来看，也许是某一个公共变量，也许是某一个函数，等等。

多线程的问题是伴随着它的使用天生而来的，对付它的办法看起来也不少，像原子操作，锁，信号量，读写锁等等。

## 原子操作

原子（atom）本意是“不能被进一步分割的最小粒子”，而原子操作（atomic operation）意为"不可被中断的一个或一系列操作"。
从概念上来看，就可以知道：

>* 它是最小的执行单位，不可能有比它更小的执行单位；
>* 操作不会在执行完毕前被任何其他任务或事件打断；

原子操作在多线程里，意味着这个操作一旦开始，就不会被打断，更不会被其它线程抢断。所以，如果我们知道某个操作是原子性的，就大可放心，不用额外的添加锁。所以也是很省心的。

`Python`虚拟机里面的全局锁能保证`Python bytecode`的原子性，所以，我们可以认为，一个`bytecode`就是一个原子操作。
来看看我们最常用的赋值和加1操作分别需要几个`bytecode`：

```
In [1]: import dis

In [2]: def test():
   ...:     a = 1
   ...:     a += 1
   ...:

In [3]: dis.dis(test)
  2           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)

  3           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               1 (1)
             12 INPLACE_ADD
             13 STORE_FAST               0 (a)

             16 LOAD_CONST               0 (None)
             19 RETURN_VALUE
```

能看出来，`Python`里简单，基本的赋值操作，需要两个`bytecode`，

```
  2           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (a)
```

但是我们依然认为它是原子的，因为真正的赋值语句其实是第二行。

而加1的操作，需要四个`bytecode`，

```
  3           6 LOAD_FAST                0 (a)
              9 LOAD_CONST               1 (1)
             12 INPLACE_ADD                    # 先进行加一操作
             13 STORE_FAST               0 (a) # 然后进行赋值操作 
```

即： `read, update and write`; 

如果当前线程读完a的值之后，又有其他线程对a进行了加1操作，然后切换到当前线程，那么还是在原来的基础上加1，
相当于两次加1操作仍然得到的是一次加1的结果。所以可以看出来，加1操作并不是原子操作。

事实上，进行多线程编程时，也别太指望某个操作是原子操作，老老实实的考虑使用显式的锁才是最可靠的办法。
所以原则就是：

> When in doubt, use explicit locks.

## 锁

原子操作大多数时候都会是一种奢望。使用锁基本上就成了不二选择。

在Python官方文档对 [Lock Object](https://docs.python.org/2/library/threading.html#lock-objects) 的介绍里面，有这么一句话：

>A primitive lock is a synchronization primitive that is not owned by a particular thread when locked. In Python, it is currently the lowest level synchronization primitive available, implemented directly by the thread extension module.

使用也比较简单，就两个方法：

```
Lock.acquire([blocking])

Lock.release()
```
事实上，我们接下来谈到的所有多线程同步方法，实现起来都会用到这个锁。

## 可重入锁

是指同一个线程可以多次获得同一个锁，其它线程只有在这个锁释放了才能 `acquire()`,否则会一直阻塞在这里。

> RLock Objects

>A reentrant lock is a synchronization primitive that may be acquired multiple times by the same thread. Internally, it uses the concepts of “owning thread” and “recursion level” in addition to the locked/unlocked state used by primitive locks. In the locked state, some thread owns the lock; in the unlocked state, no thread owns it.

它的实现比较简单，只是在上一个锁的基础上，加入了计数器，判断当前线程获取锁的次数，当次数为0时释放锁。所以需要注意，每次都要保证获取的锁能够释放掉。

## Condition

相比于上面的两个锁， Condition提供了线程之间的通知功能。
实现这个功能，需要两个方法：

> wait([timeout])

> Wait until notified or until a timeout occurs. 
> This method releases the underlying lock, and then blocks until it is awakened by a notify() or notifyAll() call for the same condition variable in another thread, or until the optional timeout occurs. Once awakened or timed out, it re-acquires the lock and returns.


> notify(n=1)

> By default, wake up one thread waiting on this condition, if any.
> Note: an awakened thread does not actually return from its wait() call until it can reacquire the lock. Since notify() does not release the lock, its caller should.

这种线程间的通知机制非常容易让人想起协程，正常情况下都是主动放弃当前的资源让给另一个线程去运行。

## 信号量 Semaphore

信号量和可重入锁在实现上有点像，都是在锁的基础上增加计数器，只不过信号量的计数器是在做减法。

> A semaphore manages an internal counter which is decremented by each acquire() call and incremented by each release() call. The counter can never go below zero; when acquire() finds that it is zero, it blocks, waiting until some other thread calls release().

它是刚开始定义好一个计数器，每次 `acquire()`， 计数器都会减1，直到为0就会阻塞，除非其它地方调用了`release()`。所以信号量比较适合预先设定对某个共享资源的最大同时访问数量。

下一篇会用代码示例介绍一下各种锁的使用。

