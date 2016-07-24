---
layout: post
title:  Tornado 核心源码剖析
description:  `tornado.ioloop` — Main event loop
category: blog
---
ioloop.py 共五个类，简单明了：

```python
class TimeoutError(Exception):...

class IOLoop(Configurable):...

class PollIOLoop(IOLoop):...

class _Timeout(object):...

class PeriodicCallback(object):...
```

第一个类`TimeoutError`是一个超时异常类，无需多说；
第二个类`IOLoop`是我们的应用调用时使用的类，这个类看起来比较`magic`，原因在于，它是`Configurable`的子类，重写了父类的一些方法，而它又是我们的第三个类`PollIOLoop`的父类，定义了一些需要子类`PollIOLoop`具体实现的方法。
正常情况下，我们都会使用第三个类`PollIOLoop`来作为最终对外使用类。而这里并不是这样。那么问题来了：

>我们使用`IOLoop`的话，它的子类实现的方法怎么用？

这里的`magic`在于，`Configurable`显式指定了`__new__()`。
说一下`__new__()`，在实例化类时，如果当前类里面没有定义`__new__()`，则子类会沿着父类往上寻找，直到父类定义有该方法，如果都没有显式定义，那么将一直按此规矩追溯至`object`的`__new__()`方法，因为`object`是所有新式类的基类。

在这里，我们调用IOLoop时在父类就会找到`__new__()`，那它做了什么呢？看一下代码：

```python
    def __new__(cls, *args, **kwargs):
        base = cls.configurable_base()
        init_kwargs = {}
        if cls is base:
            impl = cls.configured_class()
            if base.__impl_kwargs:
                init_kwargs.update(base.__impl_kwargs)
        else:
            impl = cls
        init_kwargs.update(kwargs)
        instance = super(Configurable, cls).__new__(impl)
        # initialize vs __init__ chosen for compatibility with AsyncHTTPClient
        # singleton magic.  If we get rid of that we can switch to __init__
        # here too.
        instance.initialize(*args, **init_kwargs)
        return instance
```
 
`__new__()` 是用来返回实例的，在这里，关键是第五行的`impl`：

```
             impl = cls.configured_class()
```

然后一路查下去，最终看到调用的是这个方法：

```python
    @classmethod
    def configurable_default(cls):
        if hasattr(select, "epoll"):
            from tornado.platform.epoll import EPollIOLoop
            return EPollIOLoop
        if hasattr(select, "kqueue"):
            # Python 2.6+ on BSD or Mac
            from tornado.platform.kqueue import KQueueIOLoop
            return KQueueIOLoop
        from tornado.platform.select import SelectIOLoop
        return SelectIOLoop
```

从这个方法里看出来，`impl`是平台相关的。在`Linux`下是`epoll`，`Mac` 或者 `BSD` 是 `KQueue`。
假设我们现在是Linux下，那么`impl`会等于`EPollIOLoop`：

```
class EPollIOLoop(PollIOLoop):
    def initialize(self, **kwargs):
        super(EPollIOLoop, self).initialize(impl=select.epoll(), **kwargs)
```
这里我们会发现，`EPollIOLoop`的父类即为我们的第三个类`PollIOLoop`。

也就是说，最终我们使用`IOLoop`的时候，通过`__new__()`方法，我们最终使用的还是第三个类`PollIOLoop`，这样得到的就会是平台相关的`IOLoop`，所以这样的设计看起来晦涩，实际还是非常巧妙的。

好了，上面说的都是类`IOLoop`在实现上的一些`magic`，接下来再具体看一下它的功能伤的实现。

`IOLoop`是用来处理网络`I/O`以及定时事件的。所以它的代码也紧紧围绕这个主题，在`IOLoop`类里面，刚开始的一些静态方法以及类方法都是用来保证当前线程只会有一个`IOLoop`在使用。它定义的一些方法的具体实现还是要看它的子类：

```python
class PollIOLoop(IOLoop):...
```

核心方法当然是`start`，总体上来讲，`start()`方法就是一个大的无限循环，在这里调度所有的网络`I/O`事件以及定时事件。所以，它使用的是一个标准的`Reactor`模式。

未完待续：

>1 waker: 使用信号 ＋ no-blocking socket

>2 事件timeout机制

>3 处理I/O事件



