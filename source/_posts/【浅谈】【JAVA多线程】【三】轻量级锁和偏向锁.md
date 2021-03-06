---
title: 【浅谈】【JAVA多线程】【三】轻量级锁和偏向锁
date: 2018-04-21 10:01:44
tags: [java多线程,CAS,轻量级锁,重度锁,Mark Word]
---
## 轻量级锁和偏向锁
> 轻量级锁

为什么叫轻量级锁？
这其实是对比的状态，轻量级是相对于系统使用**互斥量**来实现的传统锁而言。它并不是用来代替重量锁，而是在某些情境下，比如对于一个同步块，不会存在多余2个线程的竞争，那么轻量级锁性能优势就会比传统锁（也可以叫重量锁）更好点。
而它的实现原理是通过**CAS操作**来获取到锁，而不是互斥量。
> 偏向锁

为什么叫偏向锁？
偏向锁是在JDK1.6中引入的一项锁优化，它是一种轻量级锁进一步升华。相对比轻量级锁的CAS操作，偏向锁甚至连CAS也不使用，直接在```无竞争```的情况下，把整个同步都消除。

>Mark Word

对于JVM（HotSpot）而言，实现轻量级锁和偏向锁使用的是 Mark Word。在JVM中，对象头部分（Object Head）分为两部分，一部分是用于存储对象自身的运行时数据，如哈希码（Hash Code）、GC分代年龄（Generational GC Age）等，这部分数据的长度在32位和64位的虚拟机中分别占32bit和64bit，官方称为“Mark Word”。

在32位的HotSpot虚拟机中对象未锁定的状态下，Mark Word的32bit空间中的25bit用于储存对象的哈希码（Hashcode），4bit用于储存对象分代年龄，2bit用于储存锁标识位，1bit固定为0。
#### HotSpot虚拟机对象头 Mark Word



| 储存内容  | 标志位  | 状态 |
|:------------- |:---------------:| -------------:|
| 对象哈希码，对象分代年龄      | 01 |         未锁定 |
| 指向锁记录的指针      		 | 00|           轻量级锁|
| 指向重量级锁的指针 			 | 10 |  膨胀（重量级锁定） |
| 空，不需要记录信息			 |11|			GC标记|
|偏向线程ID，偏向时间戳			 |01|			可偏向|

简单来说，当线程A去进入同步代码块时，若此时同步对象并没有被锁住（01），那么该线程就可以进入同步块，同时修改修改同步对象的Mark Word，若此时偏向锁可用，则使用偏向锁，若此时线程B也进行获取锁，那么偏向锁就会消除，锁膨胀为轻量级锁，并修改Mark Word，将标示为修改为00,若还有线程C来进行获取锁，那么锁将进一步膨胀，变为重度锁（10）



不论是轻量级锁还是偏向锁，都是根据具体场景而提出的优化，若同步块的竞争较为激烈，那么轻量级锁或者偏向锁反而会降低性能（多出了一步CAS操作）。有时可以通过 ```-XX: -UseBiasedLocking ```来关闭偏向锁（在JDK1.6	默认是开启的）。




## 参考
[《深入理解java虚拟机》](http://item.jd.com/11252778.html)