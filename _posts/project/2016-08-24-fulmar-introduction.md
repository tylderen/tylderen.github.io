---
layout: post
title: fulmar 介绍
description: 发布一个分布式爬虫框架
category: project
---

之前在知乎上回答过一个关于网络爬虫的问题，里面谈到了当时对爬虫框架的一些期望:

>1.如何抓取JavaScript生成的页面？

>2.一些网站会限制你的抓取频率，过快的抓取会封禁IP，如何定量控制抓取频率？

>3.google早就实现了单台机器同时维持300个爬取任务，如何提高单台机器爬虫的工作效率？

>4.大数据背景下，单台机器不能满足数据量要求，爬虫分布式如何实现？

>5.如何对DeepWeb进行自动化挖掘？附论文： [Google’s Deep-Web Crawl](http://www.cs.cornell.edu/~lucja/publications/i03.pdf)


从 1 到 5，难度依次递增。5 更多的是搜索引擎的通用爬虫所面临的很大的挑战。
对于定向爬虫，做好前4个还是可以的。`fulmar`算是我对自己提出来的前四个期望的答案。

`fulmar`是一个专为分布式爬虫设计的项目。其目的是提供一个简单易用而又足够强大的爬虫框架。

爬虫从逻辑上来讲比较简单，无非就是去特定的网站抓取网页，然后把网页保存下来供我们提取有用数据或者提取新的URL以供再次爬取。
把这个逻辑简单概括，就是：
>* 提供原地址；
>* 抓取网页；
>* 解析网页，存储；

非常的清晰。

`fulmar`作为爬虫框架，就是想把这个逻辑里面最通用的部分做好，让使用者更多关注具体业务部分，而不用关注大量的爬虫抓取细节。

这里就不得不再次提到`pyspider`，这是一个让我看到的时候眼前一亮的项目，之前还写过一篇这个项目的介绍，[pyspider初探](http://tylderen.github.io/pyspider-start)。也从那时起开始看它的源码，对它的内部实现也有了比较充分的认识。在写`fulmar`的时候很多想法和实现也是借鉴于`pyspider`。

`fulmar`主体由两部分组成，`scheduler` 和 `worker`。
单从名字就大致可以猜出来它俩的不同作用，`scheduler`负责调度不同的抓取任务以及处理定时任务，`worker`负责具体的抓取，页面解析，数据存储。`worker`可以有多个，可以分布在不同的机器上。

`fulmar`内部使用了另外两个项目， `tornado` 和 `redis`。 `tornado`是`Python`下的一个非常优秀的异步网络库。它也非常适合作为爬虫的请求端来异步并发的请求上百个`URL`。

`redis`在这里主要作为任务队列的载体，它是连接负责爬取任务的不同`worker`之间的纽带。另外，由于`redis`本身优秀的数据结构设计和出色的性能，其在`fulmar`里也承担了更多的角色，比如抓取任务的去重，抓取速率的控制等等。接下来会有专门的文章介绍`redis`在`fulmar`里面的运用。

和其他爬虫框架相比，`fulmar`更注重并行抓取能力和机器资源的充分利用。也会在接下来的文章里专门介绍。

`fulmar`作为业余时间的一个项目，由于时间，精力有限，更重要的也是能力有限，肯定会有不尽人意的地方，欢迎到 [fulmar的github](https://github.com/tylderen/fulmar) 上提`Issue`或者直接发`PR`，将会非常感谢！



