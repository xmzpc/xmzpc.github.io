---
layout: post
title: Spring如何解决循环依赖
category: spring
tags: [spring]
lock: need
excerpt: 循环依赖其实就是循环引用，也就是两个或则两个以上的bean互相持有对方，最终形成闭环。
---

## 什么是循环依赖

循环依赖其实就是循环引用，也就是两个或则两个以上的bean互相持有对方，最终形成闭环。比如A依赖于B，B依赖于C，C又依赖于A。如下图：

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/202001/20200201144927.png)

如何理解“依赖”呢，在Spring中有：

- 构造器循环依赖
- 属性注入循环依赖

直接上代码：

**构造器循环依赖**

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/202001/20200201145340.png)

>  结果：项目启动失败，发现了一个cycle

**属性注入循环依赖**

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/202001/20200201145516.png)

>  结果：项目启动成功

**属性注入循环依赖（prototype）**

![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/202001/20200201145653.png)