
---
title: 【JVM】高级GC算法之分代回收算法
date: 2017-1-5 13:26
tag: JVM
category: JVM
---

## 参考资料

* 深入理解Java虚拟机

* [SegmentFault - [译]GC专家系列1：理解Java垃圾回收](https://segmentfault.com/a/1190000004233812#articleHeader3)

* [SegmentFault - GC的三大高级算法](https://segmentfault.com/a/1190000004674180)

* [Android梦想特工队 - JAVA垃圾回收机制](http://www.wxtlife.com/2016/04/25/java-jvm-gc/?hmsr=toutiao.io) 


<!-- more -->

## 概述

分代回收算法实际上不是一种新的算法，而是根据不同的内存区域（年代划分）采取不同的**GC算法（标记清除、复制、标记压缩）**



## JVM中的年代划分

年代划分主要分为以下三种：

* Young   Generation          新生代

* Old Generation                 老年代

* Permanent Generation     永久代



一般其中新生代与老年代的区域划分为：2:1

###  新生代概述

内存区域分为：一个Eden区、两个Survivor区

比例大小为：8:1:1

流程简述：

* 所有对象都在Eden区分配

* 当Eden区满时，还存活的对象会被复制到Survivor1区

* 当Survivor1区满时，还存活的对象会被复制到Survivor2区；（此时Survivor1为空）

* 当Survivor2区满时，还存活的对象会被复制到Survivor1区；（此时Survivor2为空）

* 经过数次在两个Survivor区移动，还存活的对象会被移动到老年代Old Generation



### 老年代概述

* 存活在新生代中但未变为不可达的对象会被复制到老年代

* 一般来说老年代的内存空间比新生代大，所以在老年代GC发生的频率较新生代低一些



### 永久代概述

* Permanent Generation称为方法区，其中存储着类和接口的元信息等等

* 这一区域并不是为老年代中存活下来的对象所定义的持久区



## 分代内存分配

* 新生代：存放所有的临时变量和new出来的对象

* 老年代：存放来自新生代存活下来的对象，或者存放当新生代已经满时创建出来的对象（直接放入）

* 永久代：存放的是类、接口、常量、方法等等元信息（不存放来自老年代的对象）

![内存分配图](http://i1.piimg.com/567571/dae8b96c6c08c2d2.png)

## 分代回收算法

各代回收：

* 新生代：当对象从新生代移除时，我们称之为"minor GC"

* 老年代：当对象从老年代被移除时，我们称之为"major GC"(或者full GC)

* 永久代：永久代回收也是“major GC”



GC类型：

* Minor GC：

    * 指发生在新生代的垃圾收集动作，因为Java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。

    * 遍历新生代中所有的对象，通过复制回收算法进行GC（复制到的目标空间可以是新生代中survivor区或老年代空间）

    * 如果新生代对象被老年代引用怎么办呢？在老年代中设计了"索引表(card table)"，是一个512字节的数据块。不管何时老年代需要持有新生代对象的引用时，都会记录到此表中。当新生代中需要执行GC时，通过搜索此表决定新生代的对象是否为GC的目标对象，从而降低遍历所有老年代对象进行检查的代价。该索引表使用写栅栏(write barrier)进行管理。wite barrier是一个允许高性能执行minor GC的设备。尽管它会引入一定的开销，却能带来总体GC时间的大幅降低。

![](http://p1.bqimg.com/567571/7cd577ba822d9572.png)

* Major GC:指发生在老年代的GC，出现了Major GC，经常会伴随至少一次的Minor GC（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行Major GC的策略选择过程）。Major GC的速度一般会比Minor GC慢10倍以上。



**-Hans**