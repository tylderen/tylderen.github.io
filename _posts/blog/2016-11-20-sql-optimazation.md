---
layout: post
title: 数据库性能优化(二)
description: 理解查询语句的执行过程并优化是关键
category: blog
---

### select 只选取需要的字段

看一个例子：
当我只查找`crn`时:

```
explain select crn  from company where crn = '123';
```

分析结果，只需要 `Index Only Scan`：

```
Index Only Scan using company_pkey on company  (cost=0.43..8.45 rows=1 width=16)
  Index Cond: (crn = '123'::text)
```

如果执行：

```
explain select *  from company where crn = '123';
```

那么，结果是 `Index Scan`：

```
Index Scan using company_pkey on company  (cost=0.43..8.45 rows=1 width=87)
  Index Cond: ((crn)::text = '123'::text)
```

也就是说，如果我们需要的字段正好是`index`里的，那么我们在执行完`index scan`之后就可以完成查询返回了，否则，我们还得通过这次`index scan`得到的该行的地址去表里面再查找一次，数据量小时这样不会有太大的问题，但是一旦数据量巨大，这样的开销也会不小。

### 限制表的大小

先考虑一下表的结构。表是由行和列组成。
所以表的增长存在这两个方向上。从行的角度来说，刚开始可能也就几千行，这时候查询是没啥问题的，随着数据的插入，表可以变得很长，一旦到达几千万甚至上亿条数据，即使使用了索引，查询速度也会慢下来。这个时候就要考虑水平分表。比如，可以根据时间来分，每一个月一张表等等；
也有可能会有很多列，这就涉及到表的结构设计。这个时候需要拆分字段出来而不是统统放到一张表里。

### 优化查询语句

如果说换成SSD属于硬件上提升数据库性能办法的话，查询语句的调优就属于软件层面的不二之选。
否则，即使你硬件性能再好，也会有不够用的一天。

#### 使用 `join` 或者 `from + 多个表` 连接多个表

很多时候都会推荐使用`join`而不是`from + 多个表`的方式进行表间连接，其中一个原因就是`from 多个表`会进行表间的笛卡尔积，事实上 查询优化器并不会傻傻的这么做，笛卡尔积的方式对机器的资源消耗是非常大的，现在的优化器都会使用和`join`一样的更聪名的办法。
比如： 

```
explain analyze select *  from company c, company d where c.crn = d.crn and c.crn > '123';

explain analyze select *  from company c join company d on  c.crn = d.crn where c.crn > '123';

```
这两种方式都会得到相同的查询计划：

```
Hash Join  (cost=160512.87..371335.65 rows=1566775 width=174) (actual time=1107.595..3651.412 rows=1567019 loops=1)
  Hash Cond: ((c.crn)::text = (d.crn)::text)
  ->  Seq Scan on company c  (cost=0.00..122557.15 rows=1566775 width=87) (actual time=0.005..651.716 rows=1567019 loops=1)
        Filter: ((crn)::text > '123'::text)
        Rows Removed by Filter: 44107
  ->  Hash  (cost=118548.72..118548.72 rows=1603372 width=87) (actual time=1101.064..1101.064 rows=1611126 loops=1)
        Buckets: 32768  Batches: 64  Memory Usage: 2883kB
        ->  Seq Scan on company d  (cost=0.00..118548.72 rows=1603372 width=87) (actual time=0.001..585.521 rows=1611126 loops=1)

Planning time: 1.911 ms
Execution time: 3732.015 ms
```

待续------









