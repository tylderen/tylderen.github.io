---
layout: post
title:  Python 多线程中信号的正确使用方式
description:  多线程使用信号，要注意
category: blog
---
之前写的一篇关于signal的文章只是讲了一些基础使用方法，本来在大部分情况下就够用了。在最后谈到了多线程下的使用，并未深入学习过，直到最近在一个项目中需要使用，所以就开始看了一下，也感到了远比想象复杂的多。

要想搞清楚本文接下来描述的问题。首先需要了解这么几个知识点：

>* 在多线程程序中，主线程结束，其他非deamon线程可以正常运行。 
>* 主线程结束，deamon线程会立刻结束。

### 在多线程的程序里，Python里面只允许主线程处理信号:

>[Signals and threads](https://docs.python.org/3.4/library/signal.html#signals-and-threads)
>Python signal handlers are always executed in the main Python thread, even if the signal was received in another thread. This means that signals can’t be used as a means of inter-thread communication. You can use the synchronization primitives from the threading module instead.
Besides, only the main thread is allowed to set a new signal handler.

这段话是说，无论信号由哪个线程获得，信号处理函数总是在主线程中执行。信号不是用于线程之间通信的手段（信号属于进程间通信的一种）。线程间的通信可以参考`threading`模块的`同步`。另外，只有主线程被允许来设置信号处理函数。
来看几个例子,在例子前面，先把这几个例子里都不会变的子线程函数以及信号处理函数写下来：

```python
# -*- coding: utf-8 -*-
import time
import threading
import signal
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
)

is_exit = False


def work():
    global is_exit
    current_thread = threading.current_thread()
    index = 0
    while not is_exit:
        if (index < 10):
            logging.info('%s:  %d', current_thread, index)
            index += 1
        else:
            break
        time.sleep(1)

    logging.info('%s complete.' , str(current_thread))


def handler(signum, frame):
    global is_exit
    is_exit = True
    current_thread = threading.current_thread()
    logging.info('%s receive a signal %d, is_exit = %d', current_thread, signum, is_exit)

```

### 1，子线程为非`deamon`线程，主线程不立刻退出

```python
if __name__ == "__main__":
    signal.signal(signal.SIGINT, handler)
    threads_num = 2
    for i in range(threads_num):
        t = threading.Thread(target=work)
        t.start()
    
    # sleep means the main thread 
    # don't quit right now.
    time.sleep(500)
```

运行一下：

```
2016-07-17 20:58:50,192 - INFO - <Thread(Thread-1, started 123145306509312)>:  0
2016-07-17 20:58:50,192 - INFO - <Thread(Thread-2, started 123145310715904)>:  0
2016-07-17 20:58:51,194 - INFO - <Thread(Thread-1, started 123145306509312)>:  1
2016-07-17 20:58:51,194 - INFO - <Thread(Thread-2, started 123145310715904)>:  1
^C2016-07-17 20:58:51,817 - INFO - <_MainThread(MainThread, started 140735284137984)> receive a signal 2, is_exit = 1
2016-07-17 20:58:52,198 - INFO - <Thread(Thread-2, started 123145310715904)> complete.
2016-07-17 20:58:52,198 - INFO - <Thread(Thread-1, started 123145306509312)> complete.
```

可以看到 按下 `Ctrl+C`后，程序马上在主线程里处理了该信号。

### 2，子线程为非`deamon`线程，主线程立刻退出:

```
if __name__ == "__main__":
    signal.signal(signal.SIGINT, handler)
    threads_num = 2
    for i in range(threads_num):
        t = threading.Thread(target=work)
        t.start()
    
    # sleep means the main thread 
    # don't quit right now.
    # time.sleep(500)
```
运行一下：

```
2016-07-17 21:05:37,691 - INFO - <Thread(Thread-1, started 123145306509312)>:  0
2016-07-17 21:05:37,692 - INFO - <Thread(Thread-2, started 123145310715904)>:  0
2016-07-17 21:05:38,697 - INFO - <Thread(Thread-1, started 123145306509312)>:  1
2016-07-17 21:05:38,697 - INFO - <Thread(Thread-2, started 123145310715904)>:  1
2016-07-17 21:05:39,699 - INFO - <Thread(Thread-1, started 123145306509312)>:  2
2016-07-17 21:05:39,699 - INFO - <Thread(Thread-2, started 123145310715904)>:  2
^C2016-07-17 21:05:40,700 - INFO - <Thread(Thread-2, started 123145310715904)>:  3
2016-07-17 21:05:40,700 - INFO - <Thread(Thread-1, started 123145306509312)>:  3
2016-07-17 21:05:41,706 - INFO - <Thread(Thread-1, started 123145306509312)> complete.
2016-07-17 21:05:41,706 - INFO - <Thread(Thread-2, started 123145310715904)> complete.
2016-07-17 21:05:41,707 - INFO - <_MainThread(MainThread, stopped 140735284137984)> receive a signal 2, is_exit = 1
```

可以看到 按下 `Ctrl+C`后，程序没有立刻处理该信号。事实上，是在所有子线程都执行完了结束了之后，此时中止了的主线程才在最后处理了该信号。也就是说，死掉的主线程是没有能力来处理信号的。这样也就达不到我们使用信号的目的。


### 3，子线程为`deamon`线程，主线程不退出:

```
if __name__ == "__main__":
    signal.signal(signal.SIGINT, handler)
    threads_num = 2
    for i in range(threads_num):
        t = threading.Thread(target=work)
        t.setDaemon(True)
        t.start()
    
    # sleep means the main thread 
    # don't quit right now.
    time.sleep(500)
```

这种情况下，

```
2016-07-17 21:27:15,228 - INFO - <Thread(Thread-1, started daemon 123145306509312)>:  0
2016-07-17 21:27:15,229 - INFO - <Thread(Thread-2, started daemon 123145310715904)>:  0
2016-07-17 21:27:16,231 - INFO - <Thread(Thread-1, started daemon 123145306509312)>:  1
2016-07-17 21:27:16,231 - INFO - <Thread(Thread-2, started daemon 123145310715904)>:  1
^C2016-07-17 21:27:16,328 - INFO - <_MainThread(MainThread, started 140735284137984)> receive a signal 2, is_exit = 1
```

可以看出来，只要主线程退出了，子线程即使没有执行完，这时候进程也会全部退出。

### 4，子线程为`deamon`线程，主线程立刻退出:
根据我们很早之前的了解，只要主线程退出了，这时候进程也会全部退出。

## 结论：处理信号靠的就是主线程，只有保证他活着，信号才能正确处理。

### Thread `join()`

主线程等待其他子线程结束，我们使用的`sleep`方法并不是一个靠谱的方式，按正常理解，一个更优雅的办法会是`join`。
线程`join`机制能让一个线程`join`到另外一个线程中.比如一个子线程`join`回主线程,那么主线程就会等待子线程运行结束.从而达到线程间等待的同步机制。
好，看起来只要把用`sleep`的地方换成`join`的方式就行了。

### 5，子线程为非`deamon`线程，主线程阻塞在`join()`:

```python
if __name__ == "__main__":
    signal.signal(signal.SIGINT, handler)
    threads_num = 2
    for i in range(threads_num):
        t = threading.Thread(target=work)
        t.setDaemon(True)
        t.start()
        t.join()
```

运行，得到：

```
2016-07-17 21:38:54,545 - INFO - <Thread(Thread-1, started 123145306509312)>:  0
2016-07-17 21:38:55,550 - INFO - <Thread(Thread-1, started 123145306509312)>:  1
^C2016-07-17 21:38:56,553 - INFO - <Thread(Thread-1, started 123145306509312)>:  2
2016-07-17 21:38:57,555 - INFO - <Thread(Thread-1, started 123145306509312)>:  3
2016-07-17 21:38:58,560 - INFO - <Thread(Thread-1, started 123145306509312)> complete.
2016-07-17 21:38:58,561 - INFO - <_MainThread(MainThread, started 140735284137984)> receive a signal 2, is_exit = 1
2016-07-17 21:38:58,561 - INFO - <Thread(Thread-2, started 123145306509312)> complete.
```

分析一下日志，按下 `Ctrl+C`后，程序没有立刻处理该信号。主线程看样子是阻塞在了`join()`，等到`join()`返回后，才立刻处理该信号，这样也导致了下一次`for`循环生成的第二个子线程由于`is_exit == True`而立刻退出。
所以，看来使用了`join`带来了新的问题，信号并不能中断`join()`调用，看来`join`在这里并不能用。
我也翻了一下源码，发现：

```python

def join(self, timeout=None):
   
    if timeout is None:
        self._wait_for_tstate_lock()
    else:
        # the behavior of a negative timeout isn't documented, but
        # historically .join(timeout=x) for x<0 has acted as if timeout=0
        self._wait_for_tstate_lock(timeout=max(timeout, 0))

def _wait_for_tstate_lock(self, block=True, timeout=-1):
    lock = self._tstate_lock
    if lock is None:  # already determined that the C code is done
        assert self._is_stopped
    elif lock.acquire(block, timeout):
        lock.release()
        self._stop()
        
```

从上面代码的倒数第三行可以看到，

```python
        elif lock.acquire(block, timeout)
```

`join`会调用`lock.acquire()`, 
也就是说，正常情况下，`join`会`wait`在一个锁上，等待子线程退出。子线程退出后，主线程获得锁才能够响应中断信号。而查阅`lock.acquire()`的文档：

>[acquire(blocking=True, timeout=-1)](https://docs.python.org/3/library/threading.html#threading.Lock.acquire)

>Acquire a lock, blocking or non-blocking.

>Changed in version 3.2: The timeout parameter is new.

>Changed in version 3.2: Lock acquires can now be interrupted by signals on POSIX.

注意最后一句，自从`Python 3.2`之后，`acquire()`才会处理信号，也就是说，在`Python 3.2`之前，对可忽略信号都是直接`ignore`的！所以，其实`join`在`Python 3.2`之后，是可以正常使用的！不信可以试试！


所以，要想写出在`Python 2` 和 `Python 3`中都能正确处理多线程信号的程序，还是要避免使用`join`而是选择其他可行的方案。。。

一种可行的方案就是不断的在主线程的最后调用`isAlive()`,同时应该把子线程们设为`deamon`，因为大多数情况下我们使用信号也主要是为了方便的去中止该进程，设为`deamon`也就会随着主线程的退出而自动退出,就不用专门去写信号的处理函数通知子线程退出。也许像这样：

```python
if __name__ == "__main__":
    signal.signal(signal.SIGINT, handler)
    threads = []
    threads_num = 2
    for i in range(threads_num):
        t = threading.Thread(target=work)
        t.setDaemon(True)
        t.start()
        threads.append(t)

    try:
        while threads:
            for t in threads:
                if not t.isAlive():
                  threads.remove(t)
            time.sleep(0.1)
    except:
        pass
```







