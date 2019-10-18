---
layout: post
title: 【置顶】Java后端开发常见面试题（持续更新...）
category: others
tags: [others]
lock: need
excerpt: 总结自己理解不深刻的知识点（持续更新...）
---

## Java相关

###  J.U.C包下的类

- 原子操作类
- 并发集合类
- 并发锁
- 并发队列
- 线程池
- 线程同步器

## 数据库相关

### 数据库连接池

#### 早期进行数据库操作

下面以访问MySQL为例，执行一个SQL命令，如果不使用连接池，需要经过哪些流程。

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191016093245.png)

不使用数据库连接池的步骤：

1. TCP建立连接的三次握手
2. MySQL认证的三次握手
3. 真正的SQL执行
4. MySQL的关闭
5. TCP的四次握手关闭

可以看到，为了执行一条SQL，却多了非常多网络交互。

优点：

- 实现简单

缺点：

- 网络IO较多
- 数据库的负载较高
- 响应时间较长及QPS较低
- 应用频繁的创建连接和关闭连接，导致临时对象较多，GC频繁
- 在关闭连接后，会出现大量TIME_WAIT 的TCP状态（在2个MSL之后关闭）

#### 使用数据库连接池

使用数据库连接池的步骤

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191016093348.png)

第一次访问的时候，需要建立连接。 但是之后的访问，均会复用之前创建的连接，直接执行SQL语句。

优点：

- 较少了网络开销
- 系统的性能会有一个实质的提升
- 没了麻烦的TIME_WAIT状态

### 数据库连接池的工作原理

连接池的工作原理主要由三部分组成，分别为

- 连接池的建立
- 连接池中连接的使用管理
- 连接池的关闭

#### 第一、连接池的建立。

　　一般在系统初始化时，连接池会根据系统配置建立，并在池中创建了几个连接对象，以便使用时能从连接池中获取。连接池中的连接不能随意创建和关闭，这样避免了连接随意建立和关闭造成的系统开销。

Java中提供了很多容器类可以方便的构建连接池，例如Vector、Stack等。

#### 第二、连接池的管理。

　　连接池管理策略是连接池机制的核心，连接池内连接的分配和释放对系统的性能有很大的影响。其管理策略是：

当客户请求数据库连接时，首先查看连接池中是否有空闲连接，如果存在空闲连接，则将连接分配给客户使用；如果没有空闲连接，则查看当前所开的连接数是否已经达到最大连接数，如果没达到就重新创建一个连接给请求的客户；如果达到就按设定的最大等待时间进行等待，如果超出最大等待时间，则抛出异常给客户。

当客户释放数据库连接时，先判断该连接的引用次数是否超过了规定值，如果超过就从连接池中删除该连接，否则保留为其他客户服务。

该策略保证了数据库连接的有效复用，避免频繁的建立、释放连接所带来的系统资源开销。

#### 第三、连接池的关闭。

当应用程序退出时，关闭连接池中所有的连接，释放连接池相关的资源，该过程正好与创建相反。

### MVCC

多版本控制: 指的是一种提高并发的技术。最早的数据库系统，只有读读之间可以并发，读写，写读，写写都要阻塞。引入多版本之后，**只有写写之间相互阻塞，其他三种操作都可以并行**，这样大幅度提高了InnoDB的并发度。在内部实现中，与Postgres在数据行上实现多版本不同，InnoDB是在undolog中实现的，通过undolog可以找回数据的历史版本。找回的数据历史版本可以提供给用户读(按照隔离级别的定义，有些读请求只能看到比较老的数据版本)，也可以在回滚的时候覆盖数据页上的数据。在InnoDB内部中，会记录一个全局的活跃读写事务数组，其主要用来判断事务的可见性。

**MVCC是通过在每行记录后面保存两个隐藏的列来实现的。这两个列，一个保存了行的创建时间，一个保存行的过期时间（或删除时间）。当然存储的并不是实际的时间值，而是系统版本号（system version number)。每开始一个新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询到的每行记录的版本号进行比较。**

1. 一般我们认为MVCC有下面几个特点：
   - 每行数据都存在一个版本，每次数据更新时都更新该版本
   - 修改时Copy出当前版本, 然后随意修改，各个事务之间无干扰
   - 保存时比较版本号，如果成功(commit)，则覆盖原记录, 失败则放弃copy(rollback)
   - 就是每行都有版本号，保存时根据版本号决定是否成功，**听起来含有乐观锁的味道, 因为这看起来正是，在提交的时候才能知道到底能否提交成功**
2. 而InnoDB实现MVCC的方式是:
   - 事务以排他锁的形式修改原始数据
   - 把修改前的数据存放于undo log，通过回滚指针与主数据关联
   - 修改成功（commit）啥都不做，失败则恢复undo log中的数据（rollback）

### Redis和Mysql一致性如何保证

在并发不高的情况下，读操作优先读取redis，不存在的话就去访问MySQL，并把读到的数据写回Redis中；

写操作的话，直接写MySQL，成功后再写入Redis(可以在MySQL端定义CRUD触发器，在触发CRUD操作后写数据到Redis，也可以在Redis端解析binlog，再做相应的操作)

在并发高的情况下，读操作和上面一样，写操作是异步写，写入Redis后直接返回，然后定期写入MySQL

## WEB相关

### servlet的生命周期

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191016095141.png)

## 框架相关

### SpringMVC请求流程

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191016103205.png)

具体步骤：

第一步：发起请求到前端控制器(DispatcherServlet)

第二步：前端控制器请求HandlerMapping查找 Handler （可以根据xml配置、注解进行查找）

第三步：处理器映射器HandlerMapping向前端控制器返回Handler，HandlerMapping会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象，多个HandlerInterceptor拦截器对象），通过这种策略模式，很容易添加新的映射策略

第四步：前端控制器调用处理器适配器去执行Handler

第五步：处理器适配器HandlerAdapter将会根据适配的结果去执行Handler

第六步：Handler执行完成给适配器返回ModelAndView

第七步：处理器适配器向前端控制器返回ModelAndView （ModelAndView是springmvc框架的一个底层对象，包括 Model和view）

第八步：前端控制器请求视图解析器去进行视图解析 （根据逻辑视图名解析成真正的视图(jsp)），通过这种策略很容易更换其他视图技术，只需要更改视图解析器即可

第九步：视图解析器向前端控制器返回View

第十步：前端控制器进行视图渲染 （视图渲染将模型数据(在ModelAndView对象中)填充到request域）

第十一步：前端控制器向用户响应结果

### Spring常用注解

1、@Controller：用于标注控制器层组件

2、@Service：用于标注业务层组件

3、@Component : 用于标注这是一个受 Spring 管理的组件，组件引用名称是类名，第一个字母小写。可以使用@Component(“beanID”) 指定组件的名称

4、@Repository：用于标注数据访问组件，即DAO组件

5、@Bean：方法级别的注解，主要用在@Configuration和@Component注解的类里，@Bean注解的方法会产生一个Bean对象，该对象由Spring管理并放到IoC容器中。引用名称是方法名，也可以用@Bean(name = "beanID")指定组件名

6、@Scope("prototype")：将组件的范围设置为原型的（即多例）。保证每一个请求有一个单独的action来处理，避免action的线程问题。

由于Spring默认是单例的，只会创建一个action对象，每次访问都是同一个对象，容易产生并发问题，数据不安全。

7、@Autowired：默认按类型进行自动装配。在容器查找匹配的Bean，当有且仅有一个匹配的Bean时，Spring将其注入@Autowired标注的变量中。

8、@Resource：默认按名称进行自动装配，当找不到与名称匹配的Bean时会按类型装配。

## 中间件相关

### redis缓存雪崩

#### 1、什么是缓存雪崩？你有什么解决方案来防止缓存雪崩？

如果缓存集中在一段时间内失效，发生大量的缓存穿透，所有的查询都落在数据库上，造成了缓存雪崩。
由于原有缓存失效，新缓存未到期间所有原本应该访问缓存的请求都去查询数据库了，而对数据库CPU 和内存造成巨大压力，严重的会造成数据库宕机。

#### 2、你有什么解决方案来防止缓存雪崩？

1、数据预热

缓存预热就是系统上线后，将相关的缓存数据直接加载到缓存系统。这样就可以避免在用户请求的时候，先查询数据库，然后再将数据缓存的问题!用户直接查询事先被预热的缓存数据!可以通过缓存reload机制，预先去更新缓存，再即将发生大并发访问前手动触发加载缓存不同的key。

2、双层缓存策略

C1为原始缓存，C2为拷贝缓存，C1失效时，可以访问C2，C1缓存失效时间设置为短期，C2设置为长期

3、定时更新缓存策略

失效性要求不高的缓存，容器启动初始化加载，采用定时任务更新或移除缓存

5、设置不同的过期时间，让缓存失效的时间点尽量均匀

### redis的缓存穿透

#### 什么是缓存穿透？

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201910/20191016105103.png)

缓存穿透是指查询一个一定不存在的数据，由于缓存是不命中时被动写的，并且出于容错考虑，如果从存储层查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到存储层去查询，失去了缓存的意义。在流量大时，可能DB就挂掉了，要是有人利用不存在的key频繁攻击我们的应用，这就是漏洞。

#### 防止缓存穿透的解决方案

- 缓存空值

如果一个查询返回的数据为空(不管是数据不 存在，还是系统故障)我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。 通过这个直接设置的默认值存放到缓存，这样第二次到缓冲中获取就有值了，而不会继续访问数据库。

- 采用布隆过滤器BloomFilter->优势占用内存空间很小，bit存储。性能特别高。

将所有可能存在的数据哈希到一个足够大的 bitmap 中，一个一定不存在的数据会被这个bitmap 拦截掉，从而避免了对底层存储系统的查询压力

### MySQL里有2000w数据，Redis中只存20w的数据，如何保证Redis中的数据都是热点数据（redis有哪些数据淘汰策略？？？）

相关知识：redis 内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略（回收策略）。redis 提供 6种数据淘汰策略：

1. **volatile-lru**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru**：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-enviction**（驱逐）：禁止驱逐数据

### Redis两种持久化的方式

Redis为持久化提供了两种方式：

- RDB：在指定的时间间隔能对你的数据进行快照存储。
- AOF：记录每次对服务器写的操作,当服务器重启的时候会重新执行这些命令来恢复原始的数据。