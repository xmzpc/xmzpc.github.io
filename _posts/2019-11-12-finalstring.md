---
layout: post
title: String的不可变性
category: java
tags: [java]
lock: need
excerpt: 不可变类只是其实例不能被修改的类。每个实例中包含的所有信息都必须在创建该实例的时候就提供，并且在对象的整个生命周期内固定不变。
---

## 前言

## 什么是不可变类？

不可变类只是其实例不能被修改的类。每个实例中包含的所有信息都必须在创建该实例的时候就提供，并且在对象的整个生命周期内固定不变。为了使类不可变，要遵循下面五条规则：

1. 不要提供任何会修改对象状态的方法。
2. 保证类不会被扩展。 一般的做法是让这个类称为 final 的，防止子类化，破坏该类的不可变行为。
3. 使所有的域都是 final 的。
4. 使所有的域都成为私有的。 防止客户端获得访问被域引用的可变对象的权限，并防止客户端直接修改这些对象。
5. 确保对于任何可变性组件的互斥访问。 如果类具有指向可变对象的域，则必须确保该类的客户端无法获得指向这些对象的引用。

## String是如何实现不可变的

``` java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;

    /**
     * Class String is special cased within the Serialization Stream Protocol.
     *
     * A String instance is written into an ObjectOutputStream according to
     * <a href="{@docRoot}/../platform/serialization/spec/output.html">
     * Object Serialization Specification, Section 6.2, "Stream Elements"</a>
     */
    private static final ObjectStreamField[] serialPersistentFields =
        new ObjectStreamField[0];

```

String 类是 final 修饰的，满足第二条原则：保证类不会被扩展。 分析一下它的几个域：

- `private final char value[]` : 可以看到 Java 还是使用字节数组来实现字符串的，并且用 final 修饰，保证其不可变性。这就是为什么 String 实例不可变的原因。

- `private int hash` : String的哈希值缓存

- `private static final long serialVersionUID = -6849794470754667710L `： String对象的 serialVersionUID

- `private static final ObjectStreamField[] serialPersistentFields = new ObjectStreamField[0] `： 序列化时使用

在Java中，String是被设计成一个不可变(immutable)类，一旦创建完后，字符串本身是无法通过正常手段被修改的。例如 `substring` `concat` `replace` `replaceAll` 等等，这些函数是否会修改类中的 `value` 域呢？我们看一下 `substring` ()` 函数的内部实现：

``` java
 public String substring(int beginIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        int subLen = value.length - beginIndex;
        if (subLen < 0) {
            throw new StringIndexOutOfBoundsException(subLen);
        }
        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
    }

```

如果你操作后的内容会和目前String中的内容不一致的话，那么都是重新创建一个新的String类返还，不会让你去修改内部的内容。

## String 对象在内存中的位置

对与创建String的两种写法：

``` java
  String str1 = "123";
  String str2 = new String("123");
  System.out.println(str1 == str2);
```

结果为`false`, 对于str1会直接在字符串常量池中查找是否存在值 123，若存在直接返回这个值的引用，若不存在创建一个值为 123 的 String 对象并存入字符串常量池中。而使用 new 关键字，则会直接在堆上生成一个新的 String 对象，并不会理会常量池中是否有这个值。所以本质上 str1 和 str2 指向的内存地址是不一样的。


![](https://raw.githubusercontent.com/xmzpc/PicBed/master/img/201911/20191113091742.jpg)

那么，使用 `new` 关键字生成的 String 对象可以进入字符串常量池吗？答案是肯定的，String 类提供了一个 native 方法 `intern()` 用来将这个对象加入字符串常量池：

``` java
  String str1 = "123";
  String str2 = new String("123");
  str2=str2.intern();
  System.out.println(str1 == str2);

```

打印结果为 true。str2 调用 intern() 函数后，首先在字符串常量池中寻找是否存在值为 123 的对象，若存在直接返回该对象的引用，若不存在，加入 str2 并返回。上述代码中，常量池中已经存在值为 123 的 str1 对象，则直接返回 str1 的引用地址，使得 str1 和 str2 指向同一个内存地址。

## 不可变的好处

- **不可变类比较简单。**
- **不可变对象本质上是线程安全的，它们不要求同步。不可变对象可以被自由地共享。**
- **不仅可以共享不可变对象，甚至可以共享它们的内部信息。**
- **不可变对象为其他对象提供了大量的构建。**
- **不可变类真正唯一的缺点是，对于每个不同的值都需要一个单独的对象。**

- **性能，String大量运用在哈希的处理中，由于String的不可变性，可以只计算一次哈希值，然后缓存在内部**。