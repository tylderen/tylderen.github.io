---
layout: post
title: Python 迭代器，生成器
description: 从大文件的读取看Python迭代器以及生成器的实现和作用
category: blog
---

记不清从哪偶然看到的了，这个问题大概意思是说，在Python里面如何读取一个大小约8G的文件。很多人可能第一反应，这不简单嘛，直接：

```
In [1]: f = open('file_name', 'rb')

In [2]: f.read()

```
这时候，你很可能会得到异常：

```
MemoryError: ...
```

这个仔细一想其实很好理解，这两行代码是要打开这个文件，然后将它的内容全部读取并显示出来。因为这个文件实在太大，而内存又有限，很可能直接就内存溢出了。
所以该怎么办呢？
其实办法还是挺多的，比如，我们可以指定一次读入的大小限制，

```
while True:
    data = f.read(1024)
    do_something(data)
    if not data:
        break
```

这样限制了一次读入的大小，也不会有内存问题了。
stackoverflow上正好也有一个类似问题:

 
 [   How to read large file, line by line in python ?](http://stackoverflow.com/questions/8009882/how-to-read-large-file-line-by-line-in-python)
 
 
看完之后，最简洁也最高效的做法就是：

```
with open(...) as f:
    for line in f:
        <do something with line>
```

是不是感觉：

```
    for line in f:

```

这一行有点Magic吧？到底它做了啥？

OK，让我们看一下官方文档：

[file.next()](https://docs.python.org/2/library/stdtypes.html#file.next)

> A file object is its own `iterator`, for example iter(f) returns f (unless f is closed). When a file is used as an iterator, typically in a for loop (for example, `for line in f: print line.strip()`), the `next()` method is called repeatedly. This method returns the next input line, or raises `StopIteration` when EOF is hit when the file is open for reading (behavior is undefined when the file is open for writing). In order to make a for loop the most efficient way of looping over the lines of a file (a very common operation), `the next() method uses a hidden read-ahead buffer`. As a consequence of using a read-ahead buffer, combining next() with other file methods (like readline()) does not work right. However, using seek() to reposition the file to an absolute position will flush the read-ahead buffer.


大致意思就是说，`file object` 实际上是一个 `iterator`， 所以当它用在 `for` 里面的时候，就能够重复的调用它的 `next()` 方法，这个方法会返回新的一行。这样重复下去直到遇到文件的 `EOF` 就会抛出 `StopIteration` 异常从而结束这个 `for` 循环。因为 `next()` 会自动使用文件系统本身的预读缓存以及内存管理（对文件系统的预读算法以及内存管理有兴趣的同学可以看看Linux内核架构那本书，对这一块有专门的讲解），从而无论是读取速度还是内存占用都会有相当好的表现。

到这里，我们才算把这篇文章的主角之一 `iterator` 提出来。

迭代器提供了一个类序列的接口。它还允许我们迭代非序列类型，包括我们自定义的对象。只是需要我们自定义的对象拥有 `__iter__()` 和 `next()` 两个方法，其中 `__iter__()` 返回迭代器自身，`next()` 返回下一个元素。

Python还提供了 `iter` 来使拥有可迭代能力的对象返回一个迭代器：

```
In [1]: a = [1,2,3]

In [2]: b = iter(a)

In [3]: b.__iter__() == b
Out[3]: True

In [4]: b.next()
Out[4]: 1

In [5]: b.next()
Out[5]: 2

In [6]: b.next()
Out[6]: 3

In [7]: b.next()
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
```

我们也可以实现自己的迭代器，只要实现 `__iter__` 和 `next` 方法即可:

```
class first_n(object):
     def __init__(self, n):
         self.curr = 0
         self.n = n

     def __iter__(self):
         return self

     def next(self):
         if self.curr < self.n:
             self.curr += 1
             return self.curr
         raise StopIteration()
```

使用：

```
In [1]: from iter import first_n

In [2]: for i in first_n(3):
   ...:     print i
   ...:
1
2
3
```

是不是感觉有点复杂？ 
光实现一个 `itreator` 就需要写这么多代码，别怕，让我们再请出第二个主角， `generator`。

> Generators functions allow you to declare a function that behaves like an iterator, i.e. it can be used in a for loop.

所以让我们看看 `generator` 版的 `first_n`:

```
 def first_n(n):
     curr = 0
     while curr < n:
         curr += 1
         yield curr
```

是不是简单多了？

那么他们的实际使用效果如何？

从开头的例子就能看出，`iterator` 会为我们在处理大的可迭代化对象时带来很好的性能提升，大大减少内存占用。 Python里面很多：

`xrange()` 相比  `range()`；

字典操作里的：

`dict.iteritems()` 相比 `dict.items()` 等等都是这方面的例子。


Pytohn引入的 `yield` 生成器，内部使用了协程的实现，除了在这里作为可迭代器使用，在其他很多地方都会给我们带来性能上的巨大提升，以后的文章里面我们会陆续提到。


[tylderen]:    http://tylderen.github.io  "tylderen"
