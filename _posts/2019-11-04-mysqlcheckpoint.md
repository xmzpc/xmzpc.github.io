---
layout: post
title: MySQL-InnoDB CheckPoint技术
category: mysql
tags: [mysql]
lock: need
excerpt: checkpoint检查点主要是刷新脏页到磁盘上，使数据库达到数据状态一致性的工作。因为事务的提交只会刷新操作日志到磁盘，脏数据是异步刷新到磁盘，这个异步就是靠checkpoint。
---

## 几个概念

- 缓冲池：缓存磁盘数据，通过内存速度弥补CPU速度和磁盘速度的鸿沟。
- 脏页：LRU列表中被修改的页，和磁盘上的数据不一致
- 刷新频率：每次有脏页就刷新，开销很大。需要一种刷新机制
- 数据丢失：有脏页，未刷新到磁盘，发生宕机，可能会丢失--->引入Write Ahead Log策略（先写重做日志，再修改页）
- 数据恢复：通过重做日志恢复数据，体现事务ACID中的持久性（D）
- checkpoint：将缓冲池中的脏页刷新回到磁盘。

## Write Ahead Log策略

  缓冲池的设计目的为了协调CPU速度与磁盘速度的鸿沟。因此页的操作首先都是在缓冲池中完成的。如果一条DML语句，如Update或Delete改变了页中的记录，那么此时页是脏的，即缓冲池中的页的版本要比磁盘的新。数据库需要将新版本的页从缓冲池刷新到磁盘。

  倘若每次一个页发生变化，就将新页的版本刷新到磁盘，那么这个开销是非常大的。若热点数据集中在某几个页中，那么数据库的性能将变得非常差。同时，如果在从缓冲池将页的新版本刷新到磁盘时发生了岩机，那么数据就不能恢复了。为了避免发生数据丢失的问题，当前事务数据库系统普遍都采用了**Write Ahead Log**策略，**即当事务提交时，先写重做日志，再修改页。**当由于发生岩机而导致数据丢失时，通过重做日志来完成数据的恢复。这也是事务ACID中D（Durability持久性）的要求。

## CheckPoint的目的

### 缩短数据库的恢复时间

- 当数据库宕机时，数据库不需要重做所有日志，因为CheckPoint之前的页都已经刷新回磁盘。只需对CheckPoint后的重做日志进行恢复，从而缩短恢复时间

### 缓冲池不够用时，将脏页刷新到磁盘

- 当缓存池不够用时，LRU算法会溢出最近最少使用的页，若此页为**脏页**，会**强制执行CheckPoint**，将该脏页刷回磁盘

### 重做日志不可用时，刷新脏页

- 不可用是因为对重做日志的设计是**循环使用**的。重做日志可以被重用的部分，是指当数据库进行恢复操作时不需要的部分。若此时这部分重做日志还有用，将**强制执行CheckPoint**，将缓冲池的页至少刷新到当前重做日志的位置

## CheckPoint的实现

通过LSN实现
实例恢复时，假如checkpointLSN=1000,而redoLSN=1200，则LSN=[1001,1200]均需要重做

LSN：

Log Sequence Number:日志序列号，一个不断增大的数字,表示redo log的增量（字节数）。
用途：类似oracle的SCN，用来标识数据库变更版本，保证磁盘上的redo和data文件处于一致状态。实例恢复时，比较page的LSN和redo里记录的该page的LSN是否一致。若小于，则需要将redo重做到page里。
存在于：redo log中每个page的头部

## CheckPoint的种类

### Sharp CheckPoint

Sharp Checkpoint发生在**数据库关闭时**将所有的脏页都刷新回磁盘，这是默认的工作方式，即参数innodb fast shutdown-1。

### Fuzzy CheckPoint

为提高性能，数据库运行时使用Fuzzy CheckPoint进行页的刷新，即**只刷新一部分脏页**

#### 何时发生Fuzzy CheckPoint

Master Thread CheckPoint

- 差不多以**每秒或每十秒**的速度，从缓存池脏页列表中刷新**一定比例**的页，且此过程是**异步**的，因此不会阻塞其他操作

FLUSH_LRU_LIST CheckPoint

- 因为InnoDB需要保证LRU列表中有**一定数量的空闲页**可使用，倘若不满足该条件，则会将LRU列表尾端的页移除，若这些页中有脏页，则会进行CheckPoint
- 该检查被放在一个单独的**Page Cleaner线程**中进行
- 用户可以通过**innodb_lru_scan_depth**控制LRU列表的可用页数量，默认为1024

Async/Sync Flush CheckPoint

- 当**重做日志文件不可用**的情况下，会强制将一些页刷回磁盘
- Async/Sync Flush CheckPoint是为了重做日志的循环使用的可用性
- 简单来说，Async发生在要刷回磁盘的**脏页较少**的情况下，Sync发生在要刷回磁盘的**脏页很多**时。具体公式略过
- 这部分操作放入到了**Page Cleaner线程**中执行，不会阻塞用户操作

Dirty Page too much CheckPoint

- 是指当**脏页比例太多**，会导致InnoDB存储引擎强制执行CheckPoint
- 目的根本上还是为了保证缓冲池中有足够可用的页
- 比例可由参数**innodb_max_dirty_pages_pct**控制。若该值为75，表示当缓冲池中脏页占据75%时，强制CheckPoint

## 参考

[MySQL技术内幕：InnoDB存储引擎]()