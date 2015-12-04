---
layout: post
title: HTTP Cookie,Session
description: 无状态协议的状态记录者
category: blog
---

  众所周知， HTTP是无状态协议，也就是说，每个请求都是与服务器的独立连接，下一次请求到来的时候协议不用维护这次请求中客户端与服务器的交互信息；
  而有状态协议是指以前的请求导致协议所处的状态会影响后续的状态，协议会根据上一个状态创建上下文。比如说TCP协议，它的每一步都跟上一个链接有关，例如著名的的三次握手，四次挥手。
  那么问题来了，没有状态的保持，我们浏览网页就只能进行简单的静态页面的读取。想在网上买东西？看看就好了，不能买的，因为简单的购物车程序也要知道用户到底在之前选择了什么商品。交互是需要承前启后的，没有状态的记录，我们啥都干不了。
  这时候，两种用于保持HTTP连接状态的技术就应运而生了，一个是Cookie，而另一个则是Session。

##Cookie：客户端状态 
可以在客户端存储状态，这也是Cookie的初衷。Cookie是保存在用户浏览器端的，并在发出http请求时会默认携带的一段文本片段。
正如所说：
> This simple mechanism provides a powerful new tool which enables a host of new types of applications to be written for web-based environments. Shopping applications can now store information about the currently selected items, for fee services can send back registration information and free the client from retyping a user-id on next connection, sites can store per-user preferences on the client, and have the client supply those preferences every time that site is connected to.

所以，还是很有用的。

##Session：在服务端记录状态

正如所说：
> A session can be defined as a server-side storage of information that is desired to persist throughout the user's interaction with the web site or web application. 
Instead of storing large and constantly changing information via cookies in the user's browser, only a unique identifier is stored on the client side (called a "session id"). This session id is passed to the web server every time the browser makes an HTTP request (ie a page link or AJAX request). The web application pairs this session id with it's internal database and retrieves the stored variables for use by the requested page.


所以，Cookie和Session不同之处在于，在链接的两端分别去记录状态，自然会有各自比较合适的场景。当然，它俩很多时候也是交织在一起的了，比如session id会放在本地的Cookie里面，然后每次请求Cookie里面就有这个session id了。


