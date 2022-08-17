---
author: "Narcissus"
title: "Golang源码之内存管理"
date: "2022-02-22"
description: "Go语言专家编程书籍阅读笔记，内存分配、垃圾回收原理学习。"
tags: ["Golang"]
categories: ["Golang"]

---

## 1. Go V1.3之前的标记-清除(mark and sweep)算法

该算法主要有两个步骤：

- 标记
- 清除

第一步先暂停程序业务逻辑，分类出可达和不可达对象，然后做上标记。

![image-20220222143032326](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222143032326.png)

> 对象1、2、3、4、7可达，对象5、6不可达。

第二步清除未标记的对象。

该算法有如下缺点：

1. **STW(stop the world)，让程序暂停，程序出现卡顿**。

![image-20220222143506376](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222143506376.png)

2. 标记需要扫描整个heap。

3. 清除数据会产生heap碎片。

## 2. Go v1.5的三色并发标记法

Golang中的垃圾回收主要应用三色标记法，GC过程和其他用户goroutine可并发运行，但需要一定时间的**STW(stop the world)**，所谓**三色标记法**实际上就是通过三个阶段的标记来确定清楚的对象都有哪些？

- 程序起初创建，全部标记为白色，将所有对象放入白色集合中。

![image-20220222143944933](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222143944933.png)

- 每次GC回收开始，会从根节点开始遍历所有对象，把遍历到的对象从白色集合放入灰色集合中。

![image-20220222144115746](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222144115746.png)

> 注意：**本次遍历是一次遍历，非递归形式**。

- 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合。

![image-20220222144308943](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222144308943.png)

- 重复第三步，直到灰色中无任何对象。

![image-20220222144354902](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222144354902.png)

- 最后，回收所有白色标记的对象。

虽然上述过程可以扫描并发流程，但为了GC过程中保证数据安全，开始扫描前仍会加上STW，扫描确定黑白对象后再放开STW。

## 3. 屏障机制

在GC过程中，如果不考虑STW，则可能会出现下面的情况：

- 一个白色对象被黑色对象引用；
- 灰色对象与该白色对象的可达关系被破坏。

如果同时出现上述两个条件，就会出现对象丢失现象。为了防止这种现象发生，最简单的方式就是STW，那么如何减少STW时间呢？

### 3.1 强-弱三色不变式

GC回收过程中，满足强、弱三色不变式之一，就可保证对象不丢失。

- **强三色不变式**

强制性不允许黑色对象引用白色对象，这样就不会出现白色对象误删情况。

- **弱三色不变式**

所有被黑色对象引用的白色对象都处于**灰色保护状态**。

![image-20220222145721779](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222145721779.png)

> 黑色对象可以引用白色对象，但要求白色对象存在其他灰色对象对他的引用，或者可达他的链路上游存在灰色对象。

### 3.2 插入屏障

在黑色A对象引用白色B对象的时候，白色B对象被标记为灰色，这样就满足强三色不变式了。（即**不存在黑色对象引用白色对象的情况，因为白色对象会强制变成灰色**）。

> 黑色对象的内存槽分为了堆和栈两种。栈空间特点是容量小但要求响应速度快，因为函数调用弹出频繁使用，**所以栈空间的对象操作不使用插入屏障**。

- 最开始，全部为白色对象。

![image-20220222151111298](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222151111298.png)

- 遍历标记过程中，由于并发特性，外界向黑色对象4添加白色对象8、黑色对象1添加白色对象9，因为黑色对象4在堆区，即将触发插入屏障机制，黑色对象1不触发。

![image-20220222151337088](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222151337088.png)

- 由于插入屏障，白色对象8变为灰色，白色对象9依然是白色。

![image-20220222151452702](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222151452702.png)

> 由于栈上没有启动插入屏障。所以扫描结束后，要对栈重新进行三色标记扫描，并且启动STW直到扫描结束。

![image-20220222152232281](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222152232281.png)

最后将堆栈空间扫描后剩余的白色节点回收。

### 3.3 删除屏障

被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。满足了弱三色不变式，**(即保护灰色对象到白色对象的路径不会断)**。

- 初始时，全部为白色对象

![image-20220222152616841](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222152616841.png)

- 灰色对象1删除白色对象5，如果不触发删除屏障，白色对象5、2、3与主链路断开，最后均会被清除。

![image-20220222152737943](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222152737943.png)

- 触发删除屏障，被删除的白色对象5，自身被标记为灰色。

![image-20220222152827393](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222152827393.png)

- 继续遍历，直到没有灰色对象。

![image-20220222152914297](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222152914297.png)

**这种方式回收精度低，一个对象即使被删除了，最后一个指向它的指针依旧可以活过这一轮，在下一轮GC中被清理掉**。

## 4. Go V1.8的混合写屏障机制

插入写屏障和删除写屏障的缺陷：

- 插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活。
- 删除写屏障：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。

混合写屏障机制如下：

1. **GC开始时扫描栈上全部可达对象并标记为黑色**（之后不再进行第二次重复扫描，无需STW）
2. **GC期间，任何在栈上创建的新对象，均为黑色**，不使用屏障机制。
3. **堆空间使用屏障机制，添加、删除引用的对象标记为灰色。**

满足变形的弱三色不变式。

- GC开始时，全部都是白色；**优先扫描全部栈对象，并标记全部可达对象为黑色**。

![image-20220222154408804](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222154408804.png)

### 4.1 场景1

对象被一个堆对象删除引用，称为栈对象的下游

- 将白色对象7添加到黑色对象1下游，因为栈不启动写屏障，所以直接挂下面。
- 灰色对象4删除白色对象7的引用关系，因为灰色对象4在堆区，触发写屏障，标记被删除白色对象7为灰色。

![image-20220222154721232](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222154721232.png)

### 4.2 场景2

对象被一个栈对象删除引用，称为另一个栈对象的下游

- 新创建一个对象9在栈上，混合写屏障模式下， 会被直接标记为黑色。
- 黑色对象9添加下游引用栈黑色对象3，（直接添加，栈不启动屏障机制）
- 黑色对象2删除黑色对象3的引用（直接删除，栈不启动屏障机制）

![image-20220222155148356](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222155148356.png)

### 4.3 场景3

对象被一个堆对象删除引用，成为另一个堆对象的下游

- 堆中对象10已经被扫描标注为黑色（黑色更特殊，其他颜色不讨论）
- 堆中黑色对象10添加下游引用堆中白色对象7，触发屏障机制，白色对象7变为灰色
- 堆中灰色对象4删除下游引用堆中白色对象7，触发屏障机制，被删除对象变为灰色

![image-20220222155840274](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222155840274.png)

### 4.4 场景4

对象从一个栈对象删除引用，成为另一个堆对象的下游

- 栈中对象1删除对栈中对象2的引用，栈空间不触发屏障机制

![image-20220222160052617](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222160052617.png)

- 堆中灰色对象4将之前引用的白色对象7的关系，转移到黑色对象2；在删除对象7的引用时，触发屏障机制，对象7变为灰色。

![image-20220222160242480](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220222160242480.png)

## 5. GC触发的条件

GC触发分为**手动触发**和**系统触发**两种方式。

Golang内部所有GC都是通过`gcStart()`函数，然后指定一个`gcTrigger`参数开始的，手动触发指定的条件值为`gcTriggerCycle`。另外还有两种值分别为`gcTriggerHeap`和`gcTriggerTime`。

- `gcTriggerHeap` 当前分配的内存达到一定阈值时触发，这个阈值在每次GC过后都会根据堆内存的增长情况和CPU占用率来调整。
- `gcTriggerTime` 自从上次GC后间隔时间达到了`runtime.forcegcperiod` 时间（默认为2分钟），将启动GC；
- `gcTriggerCycle` 如果当前没有开启垃圾收集，则启动GC；

满足上诉三个条件之一就会触发GC。

