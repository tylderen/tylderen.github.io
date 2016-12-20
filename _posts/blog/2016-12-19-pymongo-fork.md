---
layout: post
title: 多进程使用fork 需要注意的问题 
description: 从 PyMongo 多进程下的使用谈起
category: blog
---

最近准备升级 [`PyMongo`](https://github.com/mongodb/mongo-python-driver)，最新的版本为`3.4.0`，目前我在使用的版本为`3.2.2`，除了其他的一些小`bug`（比如我发过一个小PR：https://github.com/mongodb/mongo-python-driver/pull/311）得到了修补外，对于在多进程环境下的使用，官方文档也做了修改，
在 [PyMongo 3.2.2](https://api.mongodb.com/python/3.2.2/faq.html#using-pymongo-with-multiprocessing)的文档里：

> On certain platforms ([defined here](https://hg.python.org/cpython/file/d2b8354e87f5/Modules/socketmodule.c#l187)) MongoClient MUST be initialized with connect=False if a MongoClient used in a child process is initialized before forking. If connect cannot be False, then MongoClient must be initialized AFTER forking.

而在 [PyMongo 3.4](https://api.mongodb.com/python/3.4.0/faq.html#using-pymongo-with-multiprocessing)里面：

> Care must be taken when using instances of MongoClient with fork(). Specifically, instances of MongoClient must not be copied from a parent process to a child process. Instead, the parent process and each child process must create their own instances of MongoClient.  

换句话说，为了避免死锁的产生，官方文档已经不推荐使用 `connect=False` 的方式了，而是推荐不同的进程使用不同的`MongoClient`。

要想搞明白这里面的原委，有必要搞清楚`Fork`的内部机制。

`Fork`是在`Unix`（或类Unix，如Linux，Minix）中用于创建子进程的系统调用。
`Fork`之后，操作系统会复制一个与父进程完全相同的子进程。
但是，和父进程比，子进程也会有它独特的地方，[Fork 文档](http://www.gnu.org/software/libc/manual/html_node/Creating-a-Process.html#Creating-a-Process)列出了几条不同：

>* 子进程会有一个全新的唯一进程ID；
>* 子进程会复制父进程的所有文件描述符。同时，父子进程会共享相同文件描述符的文件偏移量；
>* 子进程不会继承父进程的文件锁；
>* 对子进程设置的Pending状态的信号会被清除掉；

### 打开的文件会被父子进程共用

通过 `fork()`调用后，被打开的文件与父进程和子进程存在着密切的联系。这是因为子进程与父进程公用这些文件的文件指针，文件指针由系统保存，而程序中没有保存它的值，从而当子进程移动文件指时也等于移动了父进程的文件指针。
看一个例子：

```
import os

f = open('hello.txt', 'wb')
print('At first, file position', f.tell())

pid = os.fork()
if pid != 0:
    f.write('Hello')
    print('In parent process', f.tell())
else:
    f.write('Hi')
    print('In child process', f.tell())

```
我运行的结果是：

```
('At first, file position', 0)
('In parent process', 5)
('In child process', 7)
```

可以看出来，文件指针是两个进程共用的。

### Fork 正确的使用方式

`fork`并不会开始一个全新的干净的新进程。管道，文件，socket等在子进程都会复制完全一样的一份。
比较安全的做法是在`fork`之后，在子进程里面应该立马关闭来自父进程的全部的文件相关的对象，然后重新打开需要的文件。

### PyMongo 与 redis-py 在 fork 时的不同处理

回到刚开始那里，我们看一下`PyMongo`文档给出的原因：

> This is because CPython must acquire a lock before calling getaddrinfo(). A deadlock will occur if the MongoClient‘s parent process forks (on the main thread) while its monitor thread is in the getaddrinfo() system call.
PyMongo will issue a warning if there is a chance of this deadlock occurring.


又看了一下[`redis-py`](https://github.com/andymccurdy/redis-py)，并不存在这个问题，因为他在使用连接的时候，会检查当前的进程ID，
[`def get_connection()`](https://github.com/andymccurdy/redis-py/blob/a87ae0ddb5b591f15527312229a7c92284012a5b/redis/connection.py#L1047):

```
def get_connection(self, command_name, *keys, **options):
        # Make sure we haven't changed process.
        self._checkpid()
```
如果当前进程的ID改变了，说明到了一个新的进程，就会释放从父进程复制来的连接并使用新的连接。

[`self._checkpid()`](https://github.com/andymccurdy/redis-py/blob/a87ae0ddb5b591f15527312229a7c92284012a5b/redis/connection.py#L938)是这样的：

```
    def _checkpid(self):
        if self.pid != os.getpid():
            with self._check_lock:
                if self.pid == os.getpid():
                    # another thread already did the work while we waited
                    # on the lock.
                    return
                self.disconnect()
                self.reset()
```

看来`redis-py`， `PyMongo`都是 `thread safe`的，但是只有`redis-py`是`fork safe`的。
