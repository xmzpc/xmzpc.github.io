---
layout: post
title: ReentrantLock
category: juc
tags: [juc]
lock: need
excerpt: ReentrantLock，可重入的互斥锁，是一种递归无阻塞的同步机制。
---

## 简介

ReentrantLock，可重入锁，是一种递归无阻塞的同步机制。它可以等同于synchronized的使用，但是ReentrantLock提供了比synchronized更强大、灵活的锁机制，可以减少死锁发生的概率。

“可重入锁”概念是：自己可以再次获取自己的内部锁。比如一个线程获得了某个对象的锁，此时这个对象锁还没有释放，当其再次想要获取这个对象的锁的时候还是可以获取的，如果不可锁重入的话，就会造成死锁。同一个线程每次获取锁，锁的计数器都自增1，所以要等到锁的计数器下降为0时才能释放锁。

## 基本API介绍

### 构造方法

**ReentrantLock默认是非公平锁，** ReentrantLock还提供了公平锁也非公平锁的选择，构造方法接受一个可选的公平参数，当设置为true时，表示公平锁，否则为非公平锁。

``` java

    public ReentrantLock() {
        sync = new NonfairSync();
    }
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

公平锁与非公平锁的区别在于公平锁的锁获取是有顺序的。但是公平锁的效率往往没有非公平锁的效率高，在许多线程访问的情况下，公平锁表现出较低的吞吐量。

### 几个基本概念

- AQS（AbstractQueuedSynchronizer）：为java中管理锁的抽象类。该类为实现依赖于先进先出 (FIFO) 等待队列的阻塞锁和相关同步器（信号量、事件，等等）提供一个框架。该类提供了一个非常重要的机制，在JDK API中是这样描述的：为实现依赖于先进先出 (FIFO) 等待队列的阻塞锁和相关同步器（信号量、事件，等等）提供一个框架。此类的设计目标是成为依靠单个原子 int 值来表示状态的大多数同步器的一个有用基础。子类必须定义更改此状态的受保护方法，并定义哪种状态对于此对象意味着被获取或被释放。假定这些条件之后，此类中的其他方法就可以实现所有排队和阻塞机制。子类可以维护其他状态字段，但只是为了获得同步而只追踪使用 getState()、setState(int) 和 compareAndSetState(int, int) 方法来操作以原子方式更新的 int 值。 这么长的话用一句话概括就是：维护锁的当前状态和线程等待列表。

- CLH：AQS中“等待锁”的线程队列。我们知道在多线程环境中我们为了保护资源的安全性常使用锁将其保护起来，同一时刻只能有一个线程能够访问，其余线程则需要等待，CLH就是管理这些等待锁的队列。

- CAS（compare and swap）：比较并交换函数，它是原子操作函数，也就是说所有通过CAS操作的数据都是以原子方式进行的。

- 锁对象：其实就是ReentrantLock的实例对象
- state：用来记录同步状态，比如lock对象是自由状态则state为0，如果大于零则表示被线程持有了，当然也有重入那么state则>1
- waitStatus：仅仅是一个状态而已；ws是一个过渡状态，在不同方法里面判断ws的状态做不同的处理。（**ReentrantLock中用waitStatus来自旋两次尝试获取锁，获取不成功后再加入队列**）
- tail：队列的队尾
- head：队列的队首

## 源码分析

### 加锁（公平锁为例）

#### lock方法

```java
final void lock() { 
  acquire(1);//1------标识加锁成功之后改变的值，改变state
}
```

#### acquire方法

```java
public final void acquire(int arg) {
     //tryAcquire(arg)尝试加锁，如果加锁失败则会调用acquireQueued方法加入队列去排队，如果加锁成功则不会调用
    //加入队列之后线程会立马park，等到解锁之后会被unpark，醒来之后判断自己是否被打断了，如果被打断了则执行selfInterrupt方法
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

#### 公平锁和非公平锁加锁区别

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191018100800.png)

#### tryAcquire方法

``` java
protected final boolean tryAcquire(int acquires) {
        //当前线程
        final Thread current = Thread.currentThread();
        //获取锁状态state
        int c = getState();
        /*
         * 当c==0表示锁没有被任何线程占用，在该代码块中主要做如下几个动作：
         * 则判断“当前线程”是不是CLH队列中的第一个线程线程（hasQueuedPredecessors），
         * 若是的话，则获取该锁，设置锁的状态（compareAndSetState），
         * 并切设置锁的拥有者为“当前线程”（setExclusiveOwnerThread）。
         */
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        /*
         * 如果c != 0，表示该锁已经被线程占有，则判断该锁是否是当前线程占有，若是设置state，否则直接返回false（可重入）
         */
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

#### 重点：hasQueuedPredecessors()方法

>hasQueuedPredecessors判断是否需要排队，这里需要记住一点，整个方法如果最后返回false，则去加锁，如果返回true则不加锁。

``` java
public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

>如果是第一个线程，那么和队列无关，线程直接持有锁。并且也不会初始化队列，如果接下来的线程都是交替执行，那么永远和AQS队列无关，都是直接线程持有锁，如果发生了竞争，比如第一个线程持有锁的过程中T2来lock，那么这个时候就会初始化AQS，初始化AQS的时候会在队列的头部虚拟一个Thread为NULL的Node，因为队列当中的head永远是持有锁的那个node（除了第一次会虚拟一个，其他时候都是持有锁的那个线程锁封装的node），现在第一次的时候持有锁的线程不在队列当中所以虚拟了一个node节点，队列当中的除了head之外的所有的node都在park，当第一个线程释放锁之后unpark某个（基本是队列当中的第二个，为什么是第二个呢？前面说过head永远是持有锁的那个node，当有时候也不会是第二个，比如第二个被cancel之后，至于为什么会被cancel，不在我们讨论范围之内，cancel的条件很苛刻，基本不会发生）node之后，node被唤醒，假设node是t2，那么这个时候会首先把t2变成head（sethead），在sethead方法里面会把t2代表的node设置为head，并且把node的Thread设置为null，为什么需要设置null？其实原因很简单，现在t2已经拿到锁了，node就不要排队了，那么node对Thread的引用就没有意义了。所以队列的head里面的Thread永远为null。

队列图：

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191018102731.Png)

#### acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 方法

  addWaiter(Node.EXCLUSIVE)源码分析：

```java
private Node addWaiter(Node mode) {
    //由于AQS队列当中的元素类型为Node，故而需要把当前线程tc封装成为一个Node对象,下文我们叫做nc
    Node node = new Node(Thread.currentThread(), mode);
    //tail为对尾，赋值给pred 
    Node pred = tail;
    //判断pred是否为空，其实就是判断对尾是否有节点，其实只要队列被初始化了对尾肯定不为空，假设队列里面只有一个元素，那么对尾和对首都是这个元素
    //换言之就是判断队列有没有初始化
    //上面我们说过代码执行到这里有两种情况，1、队列没有初始化和2、队列已经初始化了
    //pred不等于空表示第二种情况，队列被初始化了，如果是第二种情况那比较简单
   //直接把当前线程封装的nc的上一个节点设置成为pred即原来的对尾
   //继而把pred的下一个节点设置为当nc，这个nc自己成为对尾了
    if (pred != null) {
        //直接把当前线程封装的nc的上一个节点设置成为pred即原来的对尾，对应 10行的注释
        node.prev = pred;
        //这里需要cas，因为防止多个线程加锁，确保nc入队的时候是原子操作
        if (compareAndSetTail(pred, node)) {
            //继而把pred的下一个节点设置为当nc，这个nc自己成为对尾了 对应第11行注释
            pred.next = node;
            //然后把nc返回出去
            return node;
        }
    }
    //如果上面的if不成了就会执行到这里，表示第一种情况队列并没有初始化---下面32行解析这个方法
    enq(node);
    //返回nc
    return node;
}




private Node enq(final Node node) {//这里的node就是当前线程封装的node也就是nc
    //死循环
    for (;;) {
        //对尾复制给t，上面已经说过队列没有初始化，故而第一次循环t==null（因为是死循环，因此强调第一次，后面可能还有第二次、第三次，每次t的情况肯定不同）
        Node t = tail;
        //第一次循环成了成立
        if (t == null) { // Must initialize
            //new Node就是实例化一个Node对象下文我们成为nn，看下面87行Node类的结构，调用无参构造方法实例化出来的Node里面三个属性都为null
            //入队操作--compareAndSetHead继而把这个nn设置成为队列当中的头部，cas防止多线程、确保原子操作；记住这个时候队列当中只有一个，即nn
            if (compareAndSetHead(new Node()))
                //这个时候AQS队列当中只有一个元素，即头部=nn，所以为了确保队列的完整，设置头部等于尾部，即nn即是头也是尾
                //然后第一次循环结束
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
//为了方便 第二次循环我再贴一次代码来对第二遍循环解释
private Node enq(final Node node) {//这里的node就是当前线程封装的node也就是nc
    //死循环
    for (;;) {
        //对尾复制给t，由于第二次循环，故而tail==nn，即new出来的那个node
        Node t = tail;
        //第二次循环不成立
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //不成立故而进入else
            //首先把nc，当前线程所代表的的node的上一个节点改变为nn，因为这个时候nc需要入队，入队的时候需要把关系维护好
            //所谓的维护关系就是形成链表，nc的上一个节点只能为nn，这个很好理解
            node.prev = t;
            //入队操作--把nc设置为对尾，对首是nn，
            if (compareAndSetTail(t, node)) {
                //上面我们说了为了维护关系把nc的上一个节点设置为nn
                //这里同样为了维护关系，把nn的下一个节点设置为nc
                t.next = node;
                //然后返回t，即nn，死循环结束，也就是24行放回了，这个返回其实就是为了终止循环，返回出去的t，没有意义，你可以查看24行代码。
                return t;
            }
        }
    }
}
```

-------------------总结：addWaiter方法就是让nc入队-并且维护队列的链表关系，但是由于情况复杂做了不同处理

-------------------主要针对队列是否有初始化，没有初始化则new一个新的Node nn作为对首，nn里面的线程为null

-------------------接下来分析acquireQueued方法

``` java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //线程中断标志位
            boolean interrupted = false;
            for (;;) {
                //上一个节点，因为node相当于当前线程，所以上一个节点表示“上一个等待锁的线程”
                final Node p = node.predecessor();
                /*
                 * 如果当前线程是head的直接后继则尝试获取锁
                 * 这里不会和等待队列中其它线程发生竞争，但会和尝试获取锁且尚未进入等待队列的线程发生竞争。这是非公平锁和公平锁的一个重要区别。
                 */
                if (p == head && tryAcquire(arg)) {
                    setHead(node);     //将当前节点设置设置为头结点
                    p.next = null; 
                    failed = false;
                    return interrupted;
                }
                /* 如果不是head直接后继或获取锁失败，则检查是否要阻塞当前线程,是则阻塞当前线程
                 * shouldParkAfterFailedAcquire:判断“当前线程”是否需要阻塞
                 * parkAndCheckInterrupt:阻塞当前线程
                 */
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);     
        }
    }
```

>当p == head时，会自旋两次尝试获取锁，这样做的目的是为了“让当前线程获取锁”。出于公平锁的考虑
>
>shouldParkAfterFailedAcquire方法中会通过改变前一个结点的waitStatus来进行两次自旋尝试获取锁。
>
>waitStatus是节点Node定义的，她是标识线程的等待状态，他主要有如下四个值：
>
>CANCELLED = 1：线程已被取消;
>
>SIGNAL = -1：当前线程的后继线程需要被unpark(唤醒);
>
>CONDITION = -2 :线程(处在Condition休眠状态)在等待Condition唤醒;
>
>PROPAGATE = –3:(共享锁)其它线程获取到“共享锁”.

## 释放锁

``` java
public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

release(1)，尝试在当前锁的锁定计数（state）值上减1。成功返回true，否则返回false。当然在release()方法中不仅仅只是将state - 1这么简单，- 1之后还需要进行一番处理，如果-1之后的新state = 0 ，则表示当前锁已经被线程释放了，同时会唤醒线程等待队列中的下一个线程，当然该锁不一定就一定会把所有权交给下一个线程。

``` java
protected final boolean tryRelease(int releases) {
        int c = getState() - releases;   //state - 1
        //判断是否为当前线程在调用，不是抛出IllegalMonitorStateException异常
        if (Thread.currentThread() != getExclusiveOwnerThread())   
            throw new IllegalMonitorStateException();
        boolean free = false;
        //c == 0,释放该锁，同时将当前所持有线程设置为null
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        //设置state
        setState(c);
        return free;
    }
```

``` java
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        //如果waitStatus < 0 则将当前节点清零
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        //若后续节点为空或已被cancel，则从尾部开始找到队列中第一个waitStatus<=0，即未被cancel的节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//唤醒
    }
```



## 附上ReentrantLock与synchronized的区别

>1、与synchronized相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
>
>2、ReentrantLock还提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock更加适合（以后会阐述Condition）。
>
>3、ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。
>
>4、ReentrantLock支持更加灵活的同步代码块，但是使用synchronized时，只能在同一个synchronized块结构中获取和释放。注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。
>
>5、ReentrantLock支持中断处理，且性能较synchronized会好些。

## 参考

[AQS系列一ReentrantLock的源码--aqs加锁过程](https://note.youdao.com/ynoteshare1/index.html?id=59cc299252a906c899d65198ab366a0f&type=note)

[【Java并发编程实战】—–“J.U.C”：ReentrantLock之二lock方法分析](http://cmsblogs.com/?p=1662)

