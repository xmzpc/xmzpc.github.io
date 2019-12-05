---
layout: post
title: Thread源码分析-线程状态
category: java
tags: [java]
lock: need
excerpt: 这篇博文讲介绍线程的生命周期及各个状态。
---

## 线程的生命周期

Java 语言中， 多线程是由 Thread 的核心概念驱动的。因为多线程中的每一个线程都相互独立，有着自己的生命周期和状态转换。

我们先来看一张草图，这图描述了 Java 线程的各种状态和转换过程。

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191205192422.Png)

## 源码

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191205193012.png)

建议了解AQS、synchronized基础再阅读。

（1）为了方便讲解，我们把锁分成两大类，一类是synchronized锁，一类是基于AQS的锁（我们拿重入锁举例），也就是内部使用了LockSupport.park()/parkNanos()/parkUntil()几个方法的锁；

（2）不管是synchronized锁还是基于AQS的锁，内部都是分成两个队列，一个是同步队列（AQS的队列），一个是等待队列（Condition的队列）；

（3）对于内部调用了object.wait()/wait(timeout)或者condition.await()/await(timeout)方法，线程都是先进入等待队列，被notify()/signal()或者超时后，才会进入同步队列；

（4）明确声明，BLOCKED状态只有线程处于synchronized的同步队列的时候才会有这个状态，其它任何情况都跟这个状态无关；

（5）对于synchronized，线程执行synchronized的时候，如果立即获得了锁（没有进入同步队列），线程处于RUNNABLE状态；

（6）对于synchronized，线程执行synchronized的时候，如果无法获得锁（直接进入同步队列），线程处于BLOCKED状态；

（5）对于synchronized内部，调用了object.wait()之后线程处于WAITING状态（进入等待队列）；

（6）对于synchronized内部，调用了object.wait(timeout)之后线程处于TIMED_WAITING状态（进入等待队列）；

（7）对于synchronized内部，调用了object.wait()之后且被notify()了，如果线程立即获得了锁（也就是没有进入同步队列），线程处于RUNNABLE状态；

（8）对于synchronized内部，调用了object.wait(timeout)之后且被notify()了，如果线程立即获得了锁（也就是没有进入同步队列），线程处于RUNNABLE状态；

（9）对于synchronized内部，调用了object.wait(timeout)之后且超时了，这时如果线程正好立即获得了锁（也就是没有进入同步队列），线程处于RUNNABLE状态；

（10）对于synchronized内部，调用了object.wait()之后且被notify()了，如果线程无法获得锁（也就是进入了同步队列），线程处于BLOCKED状态；

（11）对于synchronized内部，调用了object.wait(timeout)之后且被notify()了或者超时了，如果线程无法获得锁（也就是进入了同步队列），线程处于BLOCKED状态；

（12）对于重入锁，线程执行lock.lock()的时候，如果立即获得了锁（没有进入同步队列），线程处于RUNNABLE状态；

（13）对于重入锁，线程执行lock.lock()的时候，如果无法获得锁（直接进入同步队列），线程处于WAITING状态；

（14）对于重入锁内部，调用了condition.await()之后线程处于WAITING状态（进入等待队列）；

（15）对于重入锁内部，调用了condition.await(timeout)之后线程处于TIMED_WAITING状态（进入等待队列）；

（16）对于重入锁内部，调用了condition.await()之后且被signal()了，如果线程立即获得了锁（也就是没有进入同步队列），线程处于RUNNABLE状态；

（17）对于重入锁内部，调用了condition.await(timeout)之后且被signal()了，如果线程立即获得了锁（也就是没有进入同步队列），线程处于RUNNABLE状态；

（18）对于重入锁内部，调用了condition.await(timeout)之后且超时了，这时如果线程正好立即获得了锁（也就是没有进入同步队列），线程处于RUNNABLE状态；

（19）对于重入锁内部，调用了condition.await()之后且被signal()了，如果线程无法获得锁（也就是进入了同步队列），线程处于WAITING状态；

（20）对于重入锁内部，调用了condition.await(timeout)之后且被signal()了或者超时了，如果线程无法获得锁（也就是进入了同步队列），线程处于WAITING状态；

（21）对于重入锁，如果内部调用了condition.await()之后且被signal()之后依然无法获取锁的，其实经历了两次WAITING状态的切换，一次是在等待队列，一次是在同步队列；

（22）对于重入锁，如果内部调用了condition.await(timeout)之后且被signal()或超时了的，状态会有一个从TIMED_WAITING切换到WAITING的过程，也就是从等待队列进入到同步队列；

## 参考

[死磕 java线程系列之线程的生命周期](https://www.cnblogs.com/tong-yuan/p/11701358.html)