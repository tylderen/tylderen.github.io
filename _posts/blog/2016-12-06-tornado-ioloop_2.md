---
layout: post
title: Tornado 核心源码剖析(二)
description: tornado.ioloop 事件处理以及唤醒机制
category: blog
---

##事件处理

`Tornado`的`ioloop.py`主要用来处理网络I/O事件以及定时事件。
所以理所应当的，它会给我们提供接口供我们自由添加事件处理函数。

###网络I/O等事件
比如为网络I/O的`socket`等添加处理函数，一般是把`socket`的可读事件注册到`ioloop`，如果该`socket`来了请求，那么标志着该事件可读，`ioloop`就会为我们调用事件处理函数。
通过这样的方式，就监听了所有的`fds`，等`fd`可用之后就调用对应的事件处理函数。

###定时事件
[add_timeout](https://github.com/tornadoweb/tornado/blob/master/tornado/ioloop.py#L475)，[call_at](https://github.com/tornadoweb/tornado/blob/master/tornado/ioloop.py#L522)，[call_later](https://github.com/tornadoweb/tornado/blob/master/tornado/ioloop.py#L509)等函数都是用来添加定时事件。
通过把这些添加进来的事件按计划执行时间的先后顺序放到队列里面，每次都去检查队列最前面的，也就是最先执行的，看是否需要执行。如果已经到了执行时间就立即执行，否则，就把它的执行时间到当前时间的时间差设为`poll`的`timeout`。这样，即使在这段等待时间内没有其他事件打破该循环，也会因为`timeout`时间到来而中止`poll`，从而可以按时执行该定时事件。这样，就巧妙的实现了定时事件。

## Waker：轮询唤醒机制 

### 为什么使用`waker`？
`waker`在这里主要是用来唤醒`poll`。考虑这样一种情况，有些时候服务器会比较闲，没有新的事件到来，那么`poll`就会一直阻塞直到`timeout`。如果`timeout` 设了一个较大的数值，那么阻塞就可能是一个漫长的时间，所以我们有必要设立一种机制来保证可以自由的打断这个阻塞期，从而避免无休止的阻塞下去。看到这里，很自然的一个想法就是使用`signal`，但是`signal`本身就不是很可靠，而且在`Python`里面，只有主线程才能处理信号，更关键的是，程序是可以自由选择忽略信号的，所以，我们很难依靠信号来可靠的实现我们的需求。这个时候，`waker`就显得很必要了。

### waker怎样发挥作用？

将管道描述符的读端也加入事件循环检查，并设置相应的回调函数，这就是`waker`的实现。
[Waker](https://github.com/tornadoweb/tornado/blob/master/tornado/platform/posix.py#L37) 封装了对于管道 `pipe` 的操作：

``` python
def set_close_exec(fd):
    flags = fcntl.fcntl(fd, fcntl.F_GETFD)
    fcntl.fcntl(fd, fcntl.F_SETFD, flags | fcntl.FD_CLOEXEC)
    
def _set_nonblocking(fd):
    flags = fcntl.fcntl(fd, fcntl.F_GETFL)
    fcntl.fcntl(fd, fcntl.F_SETFL, flags | os.O_NONBLOCK)

class Waker(interface.Waker):
    def __init__(self):
        r, w = os.pipe()
        _set_nonblocking(r)
        _set_nonblocking(w)
        set_close_exec(r)
        set_close_exec(w)
        self.reader = os.fdopen(r, "rb", 0)
        self.writer = os.fdopen(w, "wb", 0)

    def fileno(self):
        return self.reader.fileno()

    def write_fileno(self):
        return self.writer.fileno()

    def wake(self):
        try:
            self.writer.write(b"x")
        except IOError:
            pass

    def consume(self):
        try:
            while True:
                result = self.reader.read()
                if not result:
                    break
        except IOError:
            pass

    def close(self):
        self.reader.close()
        self.writer.close()
```     
同时在[def initialize(self, impl, time_func=None, **kwargs)](https://github.com/tornadoweb/tornado/blob/master/tornado/ioloop.py#L710)里面注册了`pipe`的可读事件：

``` python
        self._waker = Waker()
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)
```

这样，只要调用了`Waker.wake(self)`方法，`self.writer.write(b"x")` 就会向管道中写入随意的字符`x`从而使管道的另一端变得可读。随后，`poll`就会收到该事件被唤醒然后跳出阻塞状态。

总结一下就是，需要唤醒阻塞中的事件循环监听函数的时候，只需要向管道写入一个字符，就可以提前结束循环，简单而又可靠。





