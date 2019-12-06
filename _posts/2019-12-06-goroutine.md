---
layout: post
title: 千帆竞发-Go语言中的协程
category: go
tags: [go]
lock: need
excerpt: 协程和通道是 Go 语言作为并发编程语言最为重要的特色之一，初学者可以完全将协程理解为线程，但是用起来比线程更加简单，占用的资源也更少。
---

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206173910.jpg)

## 协程

协程（Coroutine）本质上是一种**用户态线程**，不需要操作系统来进行抢占式调度，且在真正的实现中寄存于线程中，因此，系统开销极小，可以有效提高线程的任务并发性，而避免多线程的缺点。使用协程的优点是编程简单，结构清晰；缺点是需要语言的支持，如果不支持，则需要用户在程序中自行实现调度器。目前，原生支持协程的语言还很少。

执行体是个抽象的概念，在操作系统层面有多个概念与之对应，比如操作系统自己掌管的进程（process）、进程内的线程（thread）以及进程内的协程（coroutine，也叫轻量级线程）。与传统的系统级线程和进程相比，协程的最大优势在于其“轻量级”，可以轻松创建上**百万个**而不会导致系统资源衰竭，而线程和进程通常最多也不能超过1万个。这也是协程也叫轻量级线程的原因。

Go 语言在语言级别支持轻量级线程，叫goroutine。Go 语言标准库提供的所有系统调用操作（当然也包括所有同步 IO 操作），都会出让 CPU 给其他goroutine。这让事情变得非常简单，让轻量级线程的切换管理不依赖于系统的线程和进程，也不依赖于CPU的核心数量。

如果把协程比喻成小岛，那通道就是岛屿之间的交流桥梁，数据搭乘通道从一个协程流转到另一个协程。通道是并发安全的数据结构，它类似于内存消息队列，允许很多的协程并发对通道进行读写。下面的图来自《Go IN ACTION》，很形象的说明了通道：

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206174052.jpg)

## 启动协程

Go 程序中使用 **go** 关键字为一个函数创建一个 goroutine。一个函数可以被创建多个 goroutine，一个 goroutine 必定对应一个函数。函数调用将成为这个协程的入口。

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206174448.png)

> main 函数运行在主协程(main goroutine)里面，上面的例子中我们在主协程里面启动了一个子协程，子协程又启动了一个孙子协程，孙子协程又启动了一个曾孙子协程。这些协程之间似乎形成了父子、子孙、关系，但是实际上协程之间并不存在这么多的层级关系，在 Go 语言里只有一个主协程，其它都是它的子协程，子协程之间是平行关系。

### 发生异常怎么办？

在使用子协程时一定要特别注意保护好每个子协程，确保它们正常安全的运行。因为子协程的异常退出会将异常传播到主协程，直接会导致主协程也跟着挂掉，然后整个程序就崩溃了。

``` go
panic("error")
```

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206175014.png)

我们看到主协程最后一句打印语句没能运行就挂掉了，主协程在异常退出时会打印堆栈信息。从堆栈信息中可以了解到是哪行代码引发了程序崩溃。

为了保护子协程的安全，通常我们会在协程的入口函数开头增加 recover() 语句来恢复协程内部发生的异常，阻断它传播到主协程导致程序崩溃。recover 语句必须写在 defer 语句里面。

``` go
go func() {
  defer func() {
    if err := recover(); err != nil {
      // log error
    }
  }()
  // do something
}()

```

## 协程的原理

### 调度模型

groutine能拥有强大的并发实现是通过GPM调度模型实现，下面就来解释下goroutine的调度模型。

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206181049.Png)

Go的调度器内部有四个重要的结构：M，P，S，Sched，如上图所示（Sched未给出）

- M:M代表内核级线程，一个M就是一个线程，goroutine就是跑在M之上的；M是一个很大的结构，里面维护小对象内存cache（mcache）、当前执行的goroutine、随机数发生器等等非常多的信息
- G：代表一个goroutine，它有自己的栈，instruction pointer和其他信息（正在等待的channel等等），用于调度。
- P:P全称是Processor，处理器，它的主要用途就是用来执行goroutine的，所以它也维护了一个goroutine队列，里面存储了所有需要它来执行的goroutine Sched：代表调度器，它维护有存储M和G的队列以及调度器的一些状态信息等。

- Sched：代表调度器，它维护有存储M和G的队列以及调度器的一些状态信息等。

### 调度实现

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206182715.Png)

从上图中看，有2个物理线程M，每一个M都拥有一个处理器P，每一个也都有一个正在运行的goroutine。
P的数量可以通过GOMAXPROCS）来设置，它其实也就代表了真正的并发度，即有多少个goroutine可以同时运行。

图中灰色的那些goroutine并没有运行，而是出于ready的就绪态，正在等待被调度。P维护着这个队列（称之为runqueue），Go语言里，启动一个goroutine很容易：go function 就行，所以每有一个go语句被执行，runqueue队列就在其末尾加入一个goroutine，在下一个调度点，就从runqueue中取出（如何决定取哪个goroutine？）一个goroutine执行。



当一个OS线程MO陷入阻塞时（如下图），P转而在运行M1，图中的M1可能是正被创建，或者从线程缓存中取出。

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206185752.Png)

当M0返回时，它必须尝试取得一个P来运行goroutine，一般情况下，它会从其他的OS线程那里拿一个P过来，如果没有拿到的话，它就把goroutine放在一个global runqueue里，然后自己睡眠（放入线程缓存里）。所有的P也会周期性的检查global runqueue并运行其中的goroutine，否则global runqueue上的goroutine永远无法执行。

另一种情况是P所分配的任务G很快就执行完了（分配不均），这就导致了这个处理器P很忙，但是其他的P还有任务，此时如果global runqueue没有任务G了，那么P不得不从其他的P里拿一些G来执行。一般来说，如果P从其他的P那里要拿任务的话，一般就拿run queue的一半，这就确保了每个OS线程都能充分的使用，如下图：

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201912/20191206191535.Png)

## 参考

[The Go scheduler](http://morsmachine.dk/go-scheduler)