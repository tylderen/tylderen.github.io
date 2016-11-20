---
layout: post
title: 数据库性能优化(-)
description: 合适的索引，合理的表结构是基础
category: blog
---

数据库在业务系统里面的重要性是不言而喻的。
然而随着系统的复杂性提高，在数据库中存放的数据越来越多，数据表以及它们之间的耦合也越来越多，查询次数以及查询的复杂度也相应提高，这个时候，大多数情况下，数据库都会成为系统的瓶颈，造成系统整体的响应速度越来越慢。

所以，数据库的性能优化就显得非常必要。
说白了，就是要想办法从数据库里面读的更快，往里面写得更快。

这里总结了一些我感觉比较重要的方面：

>* 提高磁盘I/O效率
>* 建立合适的索引
>* 区分几种`scan`方式
>* select 只选取需要的字段
>* 限制表的大小
>* 优化查询语句
>* 读写分离

### 提高磁盘I/O效率

单从硬件的角度讲，磁盘I/O的效率低下是造成数据库性能出现瓶颈的最重要原因。SSD硬盘的速度大概是普通磁盘的1000倍，所以，有些时候，单单把磁盘换成SSD硬盘，很可能就能解决大部分的性能问题。不过也有可能有些云主机会根据你的硬盘容量分配磁盘I/O速率，硬盘越大I/O速率也越高。这时候，你不光要换成SSD，还要保证换成较大的SSD。。。

###建立合适的索引

建立合适的索引是优化数据库查询时间（读操作）的最重要的办法之一。
索引是为查询而生的。
索引是一个特定的数据结构，使得数据更容易查找。

现在主流的关系数据库索引使用的数据结构默认都会是 `BTree`。
那么为什么会使用`BTree`而不是其他的数据结构呢？

这篇文章写的非常到位：

[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

(注意：不光是`MySQL`，其原理基本适用于所有的关系数据库。)

总结一下，就是：

>* 索引采用了树形的存储结构，进行查询时不会全表扫描，时间复杂度为O(logn)而不是O(n)，大大减少了访问磁盘页面的数量，在表数据量大时其优势更明显。
>* 由于索引本身也可能很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘`I/O`消耗，相对于内存存取，`I/O`存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘`I/O`操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘`I/O`的存取次数。而`BTree`的独特结构完美适配了以上要求，每个节点都会存储正好一次页面请求的数据大小，这样能够最大限度的减少磁盘`I/O`的次数，从而花费较少的时间和资源完成查询任务。

这样的话，SQL查询语句，`where`和`join`中用到的所有字段都应该提前加好索引。

#### 使用`explain`查询计划

`postgres` 里面可以在正常执行的语句前面加上`explain`关键字，那么这条查询语句不会真正去执行，而是把要执行的查询计划返回回来。 如果加上 `explain analyze`， 那么这条语句就不仅返回查询计划，而且会执行。
这样就可以很方便的查看将要执行的过程，便于分析。

#### `NULL`可以使用索引

很多人都会告诉你`NULL`值不参与建索引的过程。也就是说，不支持对`NULL`建索引。如果查询的`where`语句写成`where something is NULL`或者是`where something is NOT NULL`时，数据库就不得不去全表扫描从而拖慢查询速度。
不过，`Postgresql`  从`8.3`开始支持对`NULL`进行索引，也就是说不用担心这个问题了。 比如：

```
explain select *  from company where crn is  NULL;
```

可以看到：

```
Index Scan using company_pkey on company  (cost=0.43..8.21 rows=1 width=87) (actual time=2.581..2.581 rows=0 loops=1)
  Index Cond: (crn IS NULL)

Planning time: 1.698 ms
Execution time: 2.719 ms
```

### 区分几种`scan`方式

#### Seq Scan 
将从表的第一行开始顺序扫描，一直扫描到最后满足查询条件的记录。如果目标行很多的话，查询计划器就会全表扫描，并不会走索引。

#### Index Scan/Index Only Scan

在索引的列上进行查找，通常效率会比较高。

#### Bitmap heap scan  
当`PostgreSQL`需要合并索引访问的结果子集时 会用到这种方式 ，通常情况是在用到`or`，`and` 时会出现`Bitmap heap scan`。
比如：
有一张表叫做 `company`，里面的`primary key` 为 `crn`， 用来唯一标识一家公司，

```
explain analyze select *  from company where crn < '234' and   crn > '123';
```

会看到：

```
Bitmap Heap Scan on company  (cost=8185.39..116858.19 rows=172191 width=87) (actual time=28.454..65.314 rows=166909 loops=1)
  Recheck Cond: (((crn)::text < '234'::text) AND ((crn)::text > '123'::text))
  Heap Blocks: exact=10104
  ->  Bitmap Index Scan on company_pkey  (cost=0.00..8142.34 rows=172191 width=0) (actual time=27.018..27.018 rows=166909 loops=1)
        Index Cond: (((crn)::text < '234'::text) AND ((crn)::text > '123'::text))

Planning time: 0.087 ms
Execution time: 73.451 ms
```
普通的索引扫描(`Index scan`)一次只读一条索引项，那么
一个`page`有可能被多次访问；而`bitmap scan` 一次性将满足条件的索引项全部取
出，并在内存中进行排序, 然后根据取出的索引项顺序访问表数据。顺序查找的效率要比无序的页面查找效率高。
