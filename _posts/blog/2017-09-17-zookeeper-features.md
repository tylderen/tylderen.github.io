---
layout: post
title:  分布式系统的管理员 ZooKeeper (二) 深入
description:  Zookeep特性及其工作原理分析
category: blog
---

这篇文章主要是思考Zookeeper作为一个分布式集群部署时是如何对外提供服务的。

Zookeep作为一个分布式系统对外提供服务时，同样面临分布式系统面临的经典问题：CAP问题 。
CAP的具体含义如下：

- consistency：一致性，保持数据同步更新

- availability：可用性，良好的响应性能

- partition tolerance：分区容错性，可靠性

定理：任何分布式系统只可同时满足二点，没法三者兼顾。

一般来说三种特性不会同时满足，而是应该取舍与折中。这同样也是Zookeeper的哲学。 Zookeeper提供最终一致性，高可用性；


先来看它提供的特性：

## 最终一致性

Client不论连接到哪个Server，得到的结果都是一致的；

客户端可以连接到Zookeeper集群中的任意一台。对于读请求，直接返回本地Znode数据。写操作则转换为一个事务，并转发到集群的Leader处理。Zookeeper提交事务保证写操作(更新)对于Zookeeper集群所有机器都是一致的。

那么Zookeeper这种一致性是如何实现的呢？
我觉得可以一句话概括就是，在Zookeeper集群中，只同时允许有一个Leader存在，无论用户连接到哪一个Server，但最后所有的决议都是Leader发起的，Leader决策通过的，这是Zookeeper最终一致性的根本保障。

详细来说，Zookeeper使用了一种称为Zab（Zookeeper Atomic Broadcast）的协议（附论文：http://www.cs.cornell.edu/courses/cs6452/2012sp/papers/zab-ieee.pdf）作为其一致性复制的核心。Zab就是大名鼎鼎的Paxos算法的一种简化形式 。 甚至可以说，所有分布式一致性算法其实都是Paxos算法的简化或者变种。

在Zookeeper集群中，同一时刻存在一个Leader节点，其他节点称为Follower，如果是更新请求，如果客户端连接到Leader节点，则由Leader节点执行其请求；如果连接到Follower节点，则需转发请求到Leader节点执行。但对读请求，Client可以直接从Follower上读取数据，如果需要读到最新数据，则需要从Leader节点进行。

Leader通过一个简化版的二段提交模式向其他Follower发送请求，但与二段提交有两个明显的不同之处：

- 因为只有一个Leader，Leader提交到Follower的请求一定会被接；

- 不需要所有的Follower都响应成功，只要一个多数派即可；


也就是说，如果有2n+1个节点，允许n个节点失败。因为任何两个多数派必有一个交集，当Leader切换时，通过这些交集节点可以获得当前系统的最新状态。如果没有一个多数派存在（存活节点数小于n+1）那么就会停止对外服务。

## 顺序性

client的updates请求都会根据它发出的顺序被顺序的处理；
一是；二是ZooKeeper Server。

为了保证状态的一致性，Zookeeper提出了两个安全属性（Safety Property）:


- 全序（Total order）：如果消息a在消息b之前发送，则所有Server应该看到相同的结果

- 因果顺序（Causal order）：如果消息a在消息b之前发生（a导致了b），并被一起发送，则a始终在b之前被执行。

为了保证上述两个安全属性，Zookeeper使用了TCP协议和Leader。

- 通过使用TCP协议保证了消息的全序特性（先发先到），ZooKeeper Client与Server之间的网络通信是基于TCP，TCP保证了Client/Server之间传输包的顺序;

- 通过Leader解决了因果顺序问题，执行客户端请求也是严格按照FIFO顺序的，先到Leader的先执行。

因为有了Leader，Zookeeper的架构就变为：Master-Slave模式，但在该模式中Master（Leader）会Crash，因此，Zookeeper引入了Leader选举算法（以后会谈），以保证系统的健壮性。


## 原子性

一个update操作要么成功要么失败，没有其他可能的结果；

## 高可用

ZooKeeper的主要功能是维护一个高可用且一致的数据库，数据库内容复制在多个节点上，总共2n+1个节点中只要不超过n个失效，系统就可用。

## 近实时性

Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。 但由于网络延时等原因，Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。
