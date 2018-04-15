---
title: 【浅谈】【JAVA多线程】synchronized 【一】
date: 2018-04-15 12:01:13
tags: [ 多线程, 锁,java基础,网易面试题]
---
## synchronized 修饰静态方法和实例方法
**synchronized** 字段在java世界中，是较为常见的关键字，其作用简单说，是对一个资源进行操作时，增加了一个排他保护。

其性能在JDK1.6以后增加了偏向锁、自旋锁等优化后已经和**Reentrantlock** 不相上下。偏向锁和自旋锁会在以后的post中再讨论。

**synchronized**是一个锁，它要锁的对象必须要搞清楚。一般的用法是用它锁一些共享资源

```
	public void lock(){
		synchronized(共享资源对象){
			//do with 共享资源对象
		}
	}

```

除此之外，**synchronized**还可以直接修饰方法，而方法有静态和实例方法之分。那么在修改静态和实例方法时，**synchronized**锁的到底是什么呢？

```
//修饰实例方法
 public synchronized void lockInstnaceMethod(){
 	// do something...
 }
```
```
//修饰静态方法
public synchronized static void lockStaticMethod(){
	//do something...
}
```
这个点，曾经在网易面试中出现过。当然笔试题中并没有直接问你这个有什么区别，而是写了大量的代码让你判断输出结果，如果对这个点比较模糊的话，可能会被这个笔试题吓到（这个题目出现在了笔试题中的最后一个，压轴哦）。


简言而之，在修饰静态方法时，**synchronized**`锁的是这个类`，而修饰实例方法时，锁的是`这个类的实例（可以理解为锁的是这个实例this）`。

修饰实例方法，若不同的线程持有的实例不同，那么这俩个线程不会存在竞争关系。

```
public class ThreadTest {


    public synchronized static void lockStaticMethod() {
        //do something
    }

    public synchronized void lockInstanceMethod() {
        //do something
    }
    public static void main(String[] args) {

        final ThreadTest instanceA = new ThreadTest();
        final ThreadTest instanceB = new ThreadTest();

        Thread A = new Thread() {
            @Override
            public void run() {
                instanceA.lockInstanceMethod();
            }
        };
        Thread B = new Thread() {
            @Override
            public void run() {
                instanceB.lockInstanceMethod();
            }
        };
        A.start();
        B.start();
        
        

    }
}

```

线程A和线程B不存在竞争关系，因为持有的实例对象是完全两个不同的对象。

```
public class ThreadTest {


    public synchronized static void lockStaticMethod() {
        //do something
    }

    public synchronized void lockInstanceMethod() {
        //do something
    }
    public static void main(String[] args) {

        final ThreadTest instanceA = new ThreadTest();
        final ThreadTest instanceB = new ThreadTest();

        Thread A = new Thread() {
            @Override
            public void run() {
                ThreadTest.lockStaticMethod();
            }
        };
        Thread B = new Thread() {
            @Override
            public void run() {
                ThreadTest.lockStaticMethod();
            }
        };
        A.start();
        B.start();



    }


}
```
线程A和线程B存在竞争关系，因为**synchronized**锁的是ThreadTest这个类。