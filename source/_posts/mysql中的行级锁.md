---
title: mysql中的行级锁
date: 2019-02-25 17:28:37
tags: [mysql,innodb,官方文档,gap locks,next-key locks,mysql 5.7]
---
## gap锁和next-key的意义
读过一些关于mysql的博文，都没有搞清楚什么是gap锁，什么next-key锁。直到看到一些官方文档，才对这些锁有了更清楚的认识。[官方直通门](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)
### gap locks
引用官方文档
>A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record

这句是一个定义，我们能了解到，gap锁是作用在 index records即索引数据上。但对于实际应用还不是很透彻，但后面的文档举了一个例子
>For example, SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE; prevents other transactions from inserting a value of 15 into column t.c1, whether or not there was already any such value in the column, because the gaps between all existing values in the range are locked.

比较重要的一部分是 ***prevents other transactions from inserting a value of 15 into column t.c1***

其中***inserting*** 这个动词，表明了gap锁的意义。gap locks是为了阻止其他事务的insert操作。

### next-key locks

同样引用官方文档
>A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

通过上面的文档，我们能理解到，next-key locks实际上是两种锁的组合。表明它既有record lock的特性，也有 gap lock 的特性。
那它到底是如何起作用的呢?

> A next-key lock on an index record also affects the “gap” before that index record


>that is, a next-key lock is an index-record lock plus a gap lock on the gap preceding the index record.

首先 next-key是针对索引数据的key，其次，它还对该索引记录的之前的数据有 gap locks的作用。

怎么理解呢？官方文档是这么说的
>If one session has a shared or exclusive lock on record R in an index, another session cannot insert a new index record in the gap immediately before R in the index order.

举个🌰，假设有表child，其表结构为：
```
CREATE TABLE `child` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(22) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `age` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT DEFAULT CHARSET=utf8mb4;
```

假设有如下数据


|id | name | age|
|:------------- |:---------------:| -------------:|
|1| jack| 23|
|2| rose| 19|


我们age列上是有普通索引的。那么根据next-key locks的定义，其实我们会有如下的 next-key locks锁

|next-key locks|
|:------------- |
|(negative infinity, 19]| 
|(19, 23]|
|(23, positive infinity)|

所以，当有两个事务A、B同时执行，但A执行语句在B之前。
假设A为
```
update child set name="Lucy" where age=19;
```

B为

```
insert child values(null,"Lily",8);
```

那么根据定义，mysql会锁住 (negative infinity, 19]这块的数据，那么就会导致B事务堵塞。因为 8 在 (negative infinity, 19]中。但是如果是
```
insert child values(null,"Lily",23);
```
就可以。

我们再看下官方文档
>InnoDB uses next-key locks for searches and index scans, which prevents phantom rows

通过上面我们能理解到next-key实际是为了解决***幻读***问题，
那么实际依然是阻止了其他事务的***insert***操作。

***大家注意，以上例子，都是在普通索引上的操作，如果你操作的列是有unique索引或者是主键索引，那么都不会有gap locks的特性，以及next-key locks***

>Gap locking is not needed for statements that lock rows using a unique index to search for a unique row. (This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur.) 


所以，合适索引，不仅能保持查询效率，也可以保证行级别的精确性。


