---
layout: post
title: ThreadPoolExecutor线程池详解
category: juc
tags: [juc]
lock: need
excerpt: ExecutorService（ThreadPoolExecutor的顶层接口）使用线程池中的线程执行每个提交的任务，通常我们使用Executors的工厂方法来创建ExecutorService。
---



## 为什么要使用线程池

- 减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务/

- 可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存，而把服务器累趴下(每个线程需要大约1MB内存，线程开的越多，消耗的内存也就越大，最后死机)。

- Java里面线程池的顶级接口是Executor，但是严格意义上讲Executor并不是一个线程池，而只是一个执行线程的工具。真正的线程池接口是ExecutorService。

## 线程池的状态

这里假设Integer类型是32位二进制，其中高三位用来表示线程池状态，后面29位用来记录线程池线程个数。

``` java
 //默认是RUNNING状态，线程个数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
 //线程个数
    private static final int COUNT_BITS = Integer.SIZE - 3;
//线程最大个数
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS; //-- 对应的高3位值是111。
    private static final int SHUTDOWN   =  0 << COUNT_BITS; //-- 对应的高3位值是000。
    private static final int STOP       =  1 << COUNT_BITS; //-- 对应的高3位值是001。
    private static final int TIDYING    =  2 << COUNT_BITS; //-- 对应的高3位值是010。
    private static final int TERMINATED =  3 << COUNT_BITS; //-- 对应的高3位值是011。

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

> 变量**ctl**定义为AtomicInteger ，其功能非常强大，记录了“线程池中的任务数量”和“线程池的状态”两个信息。共32位，其中高3位表示"线程池状态"，低29位表示"线程池中的任务数量"。

### 线程池状态含义：
- RUNNING：接受新任务并且处理阻塞队列里的任务；
- SHUTDOWN：拒绝新任务但是处理阻塞队列里的任务；

- STOP：拒绝新任务并且抛弃阻塞队列里的任务，同时会中断正在处理的任务；

- TIDYING：所有任务都执行完（包含阻塞队列里面任务）当前线程池活动线程为 0，将要调用 terminated 方法；

- TERMINATED：终止状态，terminated方法调用完成以后的状态。


### 线程池状态转换：
- RUNNING -> SHUTDOWN：显式调用 shutdown() 方法，或者隐式调用了 finalize()，它里面调用了 shutdown() 方法。
- RUNNING 或 SHUTDOWN -> STOP：显式调用 shutdownNow() 方法时候。
- SHUTDOWN -> TIDYING：当线程池和任务队列都为空的时候。
- STOP -> TIDYING：当线程池为空的时候。
- TIDYING -> TERMINATED：当 terminated() hook 方法执行完成时候。

 ![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191014151824.png)

### 线程池参数
- corePoolSize：线程池核心线程个数；
- workQueue：用于保存等待执行的任务的阻塞队列；比如基于数组的有界 ArrayBlockingQueue，基于链表的无界 LinkedBlockingQueue，最多只有一个元素的同步队列 SynchronousQueue，优先级队列 PriorityBlockingQueue 等。

``` java
ArrayBlockingQueue    // 数组实现的阻塞队列，数组不支持自动扩容。所以当阻塞队列已满
                      // 线程池会根据handler参数中指定的拒绝任务的策略决定如何处理后面加入的任务

LinkedBlockingQueue   // 链表实现的阻塞队列，默认容量Integer.MAX_VALUE(不限容)，
                      // 当然也可以通过构造方法限制容量

SynchronousQueue      // 零容量的同步阻塞队列，添加任务直到有线程接受该任务才返回
                      // 用于实现生产者与消费者的同步，所以被叫做同步队列

PriorityBlockingQueue // 二叉堆实现的优先级阻塞队列

DelayQueue          // 延时阻塞队列，该队列中的元素需要实现Delayed接口
                    // 底层使用PriorityQueue的二叉堆对Delayed元素排序
                    // ScheduledThreadPoolExecutor底层就用了DelayQueue的变体"DelayWorkQueue"
                    // 队列中所有的任务都会封装成ScheduledFutureTask对象(该类已实现Delayed接口)

```



- maximunPoolSize：线程池最大线程数量。

- ThreadFactory：创建线程的工厂。

- RejectedExecutionHandler：饱和策略，当队列满了并且线程个数达到 maximunPoolSize 后采取的策略，比如 AbortPolicy （抛出异常），CallerRunsPolicy（使用调用者所在线程来运行任务），DiscardOldestPolicy（调用 poll 丢弃一个任务，执行当前任务），DiscardPolicy（默默丢弃，不抛出异常）。

- keeyAliveTime：存活时间。如果当前线程池中的线程数量比核心线程数量要多，并且是闲置状态的话，这些闲置的线程能存活的最大时间。

- TimeUnit，存活时间的时间单位。


### 线程池类型

   1.newFixedThreadPool：创建一个核心线程个数和最大线程个数都为 nThreads 的线程池，并且阻塞队列长度为 `Integer.MAX_VALUE`，`keeyAliveTime=0` 说明只要线程个数比核心线程个数多并且当前空闲则回收。代码如下：

``` java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
 }
 //使用自定义线程创建工厂
 public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
 }
```

2.newSingleThreadExecutor：创建一个核心线程个数和最大线程个数都为1的线程池，并且阻塞队列长度为 `Integer.MAX_VALUE`，`keeyAliveTime=0` 说明只要线程个数比核心线程个数多并且当前空闲则回收。代码如下：

``` java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
 
    //使用自己的线程工厂
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

3.newCachedThreadPool：创建一个按需创建线程的线程池，初始线程个数为 0，最多线程个数为 Integer.MAX_VALUE，并且阻塞队列为同步队列，keeyAliveTime=60 说明只要当前线程 60s 内空闲则回收。这个特殊在于加入到同步队列的任务会被马上被执行，同步队列里面最多只有一个任务。代码如下：

``` java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
 
    //使用自定义的线程工厂
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

## 主要源码分析

- public void execute(Runnable command)

`execute` 方法是提交任务 command 到线程池进行执行，用户线程提交任务到线程池的模型图如下所示

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191014152109.jpg)

如上图可知 `ThreadPoolExecutor` 的实现实际是一个生产消费模型，其中当用户添加任务到线程池时候相当于生产者生产元素，workers 线程工作集中的线程直接执行任务或者从任务队列里面获取任务相当于消费者消费元素。

``` java
public void execute(Runnable command) {
 
    // 如果任务为null，则抛出NPE异常
    if (command == null)
        throw new NullPointerException();
 
    // 获取当前线程池的状态+线程个数变量的组合值
    int c = ctl.get();
 
    // 当前线程池线程个数是否小于corePoolSize,小于则开启新线程运行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
 
    // 如果线程池处于RUNNING状态，则添加任务到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
 
        // 二次检查
        int recheck = ctl.get();
        // 如果当前线程池状态不是RUNNING则从队列删除任务，并执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
 
        // 否则如果当前线程池线程空，则添加一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果队列满了，则新增线程，新增失败则执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

> 执行流程如下：
>
> 1. 如果线程池当前线程数小于corePoolSize，则调用addWorker创建新线程执行任务，成功返回true，失败执行步骤2。
> 2. 如果线程池处于RUNNING状态，则尝试加入阻塞队列，如果加入阻塞队列成功，则尝试进行Double Check，如果加入失败，则执行步骤3。
> 3. 如果线程池不是RUNNING状态或者加入阻塞队列失败，则尝试创建新线程直到maxPoolSize，如果失败，则调用reject()方法运行相应的拒绝策略。



- addWorker(Runnable firstTask, boolean core)

  方法用于创建线程执行任务，源码如下：

  ``` java
    private boolean addWorker(Runnable firstTask, boolean core) {
          retry:
          for (;;) {
              int c = ctl.get();
  
              // 获取当前线程状态
              int rs = runStateOf(c);
  
  
              if (rs >= SHUTDOWN &&
                      ! (rs == SHUTDOWN &&
                              firstTask == null &&
                              ! workQueue.isEmpty()))
                  return false;
  
              // 内层循环，worker + 1
              for (;;) {
                  // 线程数量
                  int wc = workerCountOf(c);
                  // 如果当前线程数大于线程最大上限CAPACITY  return false
                  // 若core == true，则与corePoolSize 比较，否则与maximumPoolSize ，大于 return false
                  if (wc >= CAPACITY ||
                          wc >= (core ? corePoolSize : maximumPoolSize))
                      return false;
                  // worker + 1,成功跳出retry循环
                  if (compareAndIncrementWorkerCount(c))
                      break retry;
  
                  // CAS add worker 失败，再次读取ctl
                  c = ctl.get();
  
                  // 如果状态不等于之前获取的state，跳出内层循环，继续去外层循环判断
                  if (runStateOf(c) != rs)
                      continue retry;
              }
          }
  
          boolean workerStarted = false;
          boolean workerAdded = false;
          Worker w = null;
          try {
  
              // 新建线程：Worker
              w = new Worker(firstTask);
              // 当前线程
              final Thread t = w.thread;
              if (t != null) {
                  // 获取主锁：mainLock
                  final ReentrantLock mainLock = this.mainLock;
                  mainLock.lock();
                  try {
  
                      // 线程状态
                      int rs = runStateOf(ctl.get());
  
                      // rs < SHUTDOWN ==> 线程处于RUNNING状态
                      // 或者线程处于SHUTDOWN状态，且firstTask == null（可能是workQueue中仍有未执行完成的任务，创建没有初始任务的worker线程执行）
                      if (rs < SHUTDOWN ||
                              (rs == SHUTDOWN && firstTask == null)) {
  
                          // 当前线程已经启动，抛出异常
                          if (t.isAlive()) // precheck that t is startable
                              throw new IllegalThreadStateException();
  
                          // workers是一个HashSet<Worker>
                          workers.add(w);
  
                          // 设置最大的池大小largestPoolSize，workerAdded设置为true
                          int s = workers.size();
                          if (s > largestPoolSize)
                              largestPoolSize = s;
                          workerAdded = true;
                      }
                  } finally {
                      // 释放锁
                      mainLock.unlock();
                  }
                  // 启动线程
                  if (workerAdded) {
                      t.start();
                      workerStarted = true;
                  }
              }
          } finally {
  
              // 线程启动失败
              if (! workerStarted)
                  addWorkerFailed(w);
          }
          return workerStarted;
      }
  ```

1. 判断当前线程是否可以添加任务，如果可以则进行下一步，否则return false；
   1. rs >= SHUTDOWN ，表示当前线程处于SHUTDOWN ，STOP、TIDYING、TERMINATED状态
   2. rs == SHUTDOWN , firstTask != null时不允许添加线程，因为线程处于SHUTDOWN 状态，不允许添加任务
   3. rs == SHUTDOWN , firstTask == null，但workQueue.isEmpty() == true，不允许添加线程，因为firstTask == null是为了添加一个没有任务的线程然后再从workQueue中获取任务的，如果workQueue == null，则说明添加的任务没有任何意义。
2. 内嵌循环，通过CAS worker + 1
3. 获取主锁mailLock，如果线程池处于RUNNING状态获取处于SHUTDOWN状态且 firstTask == null，则将任务添加到workers Queue中，然后释放主锁mainLock，然后启动线程，然后return true，如果中途失败导致workerStarted= false，则调用addWorkerFailed()方法进行处理。

在这里需要好好理论addWorker中的参数，在execute()方法中，有三处调用了该方法：

- 第一次：`workerCountOf(c) < corePoolSize ==> addWorker(command, true)`，这个很好理解，当然线程池的线程数量小于 corePoolSize ，则新建线程执行任务即可，在执行过程core == true，内部与corePoolSize比较即可。
- 第二次：加入阻塞队列进行Double Check时，`else if (workerCountOf(recheck) == 0) ==>addWorker(null, false)`。如果线程池中的线程==0，按照道理应该该任务应该新建线程执行任务，但是由于已经该任务已经添加到了阻塞队列，那么就在线程池中新建一个空线程，然后从阻塞队列中取线程即可。
- 第三次：线程池不是RUNNING状态或者加入阻塞队列失败：`else if (!addWorker(command, false))`，这里core == fase，则意味着是与maximumPoolSize比较。

## 流程图

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191014153938.jpg)

[高清图点这里](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191014153938.jpg)

## 总结

线程池巧妙地使用了一个Integer类型的原子变量来记录线程池状态和线程池中的线程个数，通过线程池状态来控制任务的执行，每个worker线程可以处理多个任务。线程池通过线程的复用减少了线程创建和销毁的开销。