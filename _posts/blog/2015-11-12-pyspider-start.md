---
layout: post
title: pyspider初探
description: 开始学习一个爬虫框架。
category: blog
---

最初开始关注这个项目是因为偶然在stackoverflow上看到了作者对这个项目的亮点的介绍：
> pyspider and Scrapy have the same purpose, web scraping, but a different view about doing that.

> spider should never stop till WWW dead. (information is changing, data is updating in websites, spider should have the ability and responsibility to scrape latest data. That's why pyspider has URL database, powerful scheduler, @every, age, etc..)

> pyspider is a service more than a framework. (Components are running in isolated process, lite - all version is running as service too, you needn't have a Python environment but a browser, everything about fetch or schedule is controlled by script via API not startup parameters or global configs, resources/projects is managed by pyspider, etc...)

> pyspider is a spider system. (Any components can been replaced, even developed in C/C++/Java or any language, for better performance or larger capacity)

> and

> on_start vs start_url

> token bucket traffic control vs download_delay

> return json vs class Item

> message queue vs Pipeline

> built-in url database vs set

> Persistence vs In-memory

> PyQuery + any third package you like vs built-in CSS/Xpath support

> In fact, I have not referred much from Scrapy. pyspider is really different from Scrapy.

> But, why not try it yourself? pyspider is also fast, has easy-to-use API and you can try it without install. 

这里面，最打动我的有：
> * 相比Scrapy，原生支持分布式
> * 支持使用phantomjs来解析javascript页面
> * 强大的调度能力，对每一个任务有详细的跟踪处理流程
> * 令牌桶算法实现对目标网站的访问流量控制，简单，合理
> * 异步抓取，事件机制，负责爬取的机器可以同时维持大量连接，非常适合抓取的网站种类比较多的情况

当然，不喜欢的是，Web端是爬虫启动和调试的地方。Web前端做个监控还是不错，作为代码调试和启动的地方，感觉还是不习惯和不喜欢。不过，应该是可以调整的，这样就感觉比较完美了。
