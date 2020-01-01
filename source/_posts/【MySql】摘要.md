---
title: 【MySql】摘要
date: 2020-01-01 16:54:34
tags:
---

```
原文 https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html

```

### 摘要

* 要知道在范围查询时，加锁是一条记录一条记录挨个加锁的，所以虽然只有一条 SQL 语句，如果两条 SQL 语句的加锁顺序不一样，也会导致死锁。

* 日志锁类型
	1. 记录锁（**LOCK_REC_NOT_GAP**）: lock_mode X locks rec but not gap
	2. 间隙锁（**LOCK_GAP**）: lock_mode X locks gap before rec
	3. Next-key 锁（**LOCK_ORNIDARY**）: lock_mode X
	4. 插入意向锁（**LOCK_INSERT_INTENTION**）: lock_mode X locks gap before rec insert intention
	
这里有一点要注意的是，并不是在日志里看到 lock_mode X 就认为这是 Next-key 锁，因为还有一个例外：如果在 ***supremum record*** 上加锁，*locks gap before rec* 会省略掉，间隙锁会显示成 lock_mode X，插入意向锁会显示成 *lock_mode X insert intention*。譬如下面这个
```
RECORD LOCKS space id 0 page no 307 n bits 72 index `PRIMARY` of table `test`.`test` trx id 50F lock_mode X 
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
```