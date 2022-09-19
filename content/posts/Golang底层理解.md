---
author: "Narcissus"
title: "Golang底层原理理解叙述"
date: "2022-09-16"
lastmod: "2022-09-16"
description: "B站up主视频学习"
tags: ["Golang"]
categories: ["Golang"]
---

## 1. Mutex

### 概述

Mutex这个名称来自于Mutual exclusion，俗称互斥锁。Go语言中Mutex的数据结构如下:

![image-20220916174715446](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220916174715446.png)

​		Mutex.state储存互斥锁的状态，加锁和解锁都通过atomic包提供的函数来原子操作该字段；Mutex.sema表示信号量，主要用作等待队列。

​		Mutex有两种模式，在正常模式下，一个尝试加锁的Goroutine会先自旋几次，尝试通过原子操作获得锁，若几次自旋操作后仍然不能获得锁，则通过信号量排队等待，所有等待者按照先入先出的顺序排队，但是当锁被释放后，**第一个等待者被唤醒后并不会直接拥有锁，而是需要后后来者竞争，也就是那些处于自旋阶段，尚未排队等待的Goroutine。** 这种情况下，后来者更加有优势，一方面，它们正在CPU上运行，自然比刚被唤醒的goroutine更有优势；另一方面，处于自旋状态的Goroutine可以有很多，而别唤醒的Goroutine每次只有一个，所以被唤醒的Goroutine有很大概率那不到锁。这种情况下它**会被重新插入到队列头部而不是尾部。** 

​		而当一个Goroutine本次加锁等待时间超过1ms后，Mutex会从正常模式切换至“饥饿模式”。在饥饿模式下，Mutex的所有权从执行Unlock的Goroutine直接传递给等待队列头部的Goroutine。后来者不会自旋也不会尝试获得锁，及时Mutex处于Unlock状态，他们会直接从队列的尾部排队等待。当一个等待者或者锁之后，会在以下两种情况将Mutex从饥饿模式切回正常模式：

- 它的等待时间小于1ms，也就是刚来不久。
- 它是最后一个等待者，等待队列为空。

​		综上所述，在正常模式下，自旋和排队同时存在，执行Lock的Goroutine会先一边自旋，尝试过几次之后如果还没有拿到锁，就需要排队等待。**这种方式能够有更高的吞吐量，因为频繁地挂起、唤醒goroutine会带来较多的开销** ，但是又不能无限制的自旋，要把自旋开销控制到较小范围内，所以在正常模式下，Mutex有更好的性能，但是可能会出现队列尾端的goroutine迟迟抢不到锁（尾端延迟）的情况。在饥饿模式下，不再尝试自旋，所有Goroutine都要排队，严格先来后到，对于防止出现尾端延迟特别重要。

### Lock和Unlock逻辑

Mutex.state的类型是int32，

- 其中第一位用作锁状态标识，置为1表示已经加锁，对应掩码常量为mutexLocked。
- 第二位用于记录是否有goroutine被唤醒了，置为1表示已唤醒，对应掩码常量为mutexWoken。
- 第三位标识mutex的工作模式，0表示正常模式，1表示饥饿模式。对应掩码常量为mutexStarving。
- 常量mutexWaiterShift=3表示除了最低三位以外，其他位用来记录有多少个等待者在排队。

![image-20220916180932068](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220916180932068.png)



代码部分，两个方法中主要通过atomic函数实现了Fast path，相应的Slow path被单独放在了lockSlow和unlocSlow中。

![image-20220916181337118](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220916181337118.png)

> 这样是为了便于编译器对Fast path进行内联优化。

​	Lock方法中的Fast path希望Mutex处于Unlock状态，没有goroutine排队，更不会饥饿，理想状态下，一个CAS操作就可以获得锁，但如果该操作没有获得锁，就需要进入Slow path，也就是lockSlow方法。

​	Unlock方法同理，先通过原子操作从state中减去mutexLocked，也就是释放锁，然后根据state的新值来判断是否需要执行Slow path。如果新值为0，就意味着没有Goroutine在排队，所以不需要执行额外操作；如果新值不为0，就需要进入Slow path，看看是否需要唤醒某个goroutine。



当一个goroutine尝试给mutex加锁时，如果其他goroutine已经加了锁还没有释放，而且当前mutex工作在正常模式下，是不是就开始自旋呢？

**不一定。** 因为如果是单核场景，自选的goroutine在等待持有锁的goroutine释放锁，而持有锁的goroutine在等待自旋的goroutine让出CPU，这种情况下是没有意义的。而且如果GOMAXPROCS=1，或者当前没有其他的P正在运行也和单核场景类似，同样不需要自旋。除此之外，如果当前P的本地队列(runq)不为空，相较于自旋而言，切换到本地goroutine更有效率，所以为了保障吞吐量也不会自旋。

​		最终**只有在多核状态下并且GOMAXPROCS>1且至少有一个其他的P正在running且当前P的本地runq为空的情况下**，才可以自旋。进入自旋的goroutine会去争抢Mutex的唤醒标识位，设置mutexWoken标识位的目的是，**正常模式下，告知持有锁的goroutine，在unlock的时候不用再唤醒其他goroutine了，已经有goroutine在这里等待** 。

自旋次数的上限是4次，而且每自旋一次都要重新判断是否可以继续自旋，如果锁被释放了，或者锁进入了饥饿模式，亦或者已经自旋了4次，都会结束自旋。

### 信号量

理解了同步问题的本质，知道可以通过锁来保护临界区的互斥性，如果尝试获得锁失败了，有两种策略：

- 第一种策略是不断尝试，知道成功获得锁或者时间片耗完，这被称为自旋锁。
- 第二种策略是让出CPU，听从操作系统安排进入到等待队列中，我们称之为调度器对象。通俗理解就是通过操作系统提供的同步原语可以实现锁或者更复杂的同步工具。

​		调度器对象与自旋锁主要不同在于等待队列，同步原语由内核提供，直接与系统的调度器交互，能够挂起和唤醒线程，这是自旋锁做不到的，但也正由于其在内核中中实现，所以应用程序中需要以**系统调用的方式来使用它**。**这会造成一定的开销，并且获取锁失败还会发生线程切换** 。所以调度器对象和自旋锁各有各的应用场景。

​		在多核环境且持有锁的时间占比比较小的情况下，往往自旋几次之后就可以拿到锁，比发生一次线程切换的代价小得多。在单核环境或者持有锁的时间比较长的情况下，一直自旋会一直占据CPU。反而得不偿失。实际业务中一般先自旋，但限制最大自旋次数，有限次数内不能加锁成功，再通过调度器对象将当前线程挂起，这样就结合了二者优点，也是现在主流的锁实现思路。



Go语言的Mutex就是这样实现的，不过这不适合协程，如果协程等待，还需要切换线程，就不太适合了。那么协程要等待一个锁，要如何休眠、等待和唤醒呢？

​		这需要runtime.semaphore来实现，这是可供协程使用的信号量。runtime内部会通过一个大小为251的semaTable来管理所有的semaphore。怎么通过固定大小来管理执行阶段数量不定的semaphore呢？

大致思路是，semaTable存储的是251棵平衡二叉树的根，平衡树中每一个节点都是一个sudog类型的对象。要使用一个信号量时，需要提供一个记录信号量数值的变量，根据地址进行计算映射到semaTable中的一棵平衡树上，找到对应节点就找到了该信号量的等待队列。

![image-20220919194845098](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919194845098.png)

## 2. GMP模型

一个Hello World程序编译成可执行文件，执行时可执行文件被加载到内存中。

​		对于进程虚拟地址空间中的代码段，我们感兴趣的是程序执行入口，他并不是我们熟悉的main.main,不同平台下的程序入口不同，再进行一系列检查与初始化等准备工作后，会以runtime.main为执行入口创建main goroutine，main goroutine执行起来后才会调用我们编写的main.main。

​		再来看数据段，这里有几个重要的全局变量，我们知道Go语言中协程对应的数据结构是runtime.g，工作线程对应的数据结构是runtime.m:

- 全局变量g0：主协程对应的g。与其他协程不同，它的栈是在主线程栈上分配的。
- 全局变量m0,就是主线程对应的m

> g0持有m0的指针，m0里也记录了g0的指针。并且一开始m0上执行的协程就是g0。

- allgs记录了所有的g
- allm记录了所有的m
- sched(调度器)，用于保存空闲的m,空闲的p,全局队列等
- allp用于保存所有的p，在程序初始化时会进行调度器初始化，同时会根据GOMAXPROCS这个环境变量决定创建多少个P

> 同时会将第一个P和m0关联起来

![image-20220919222549888](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919222549888.png)

### GM模型

最初Go语言模型调度只有G和M，所有的待执行的G都等在一处，每个M来这里获取一个G时都要加锁，多个M分担多个G的执行任务，**就会因为频繁加锁和解锁而发生等待，影响程序并发性能。** 

![image-20220919221000015](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919221000015.png)

### GMP模型

所以后面又引入了P，对应的数据结构是runtime.p，它有一个本地的runq。这样只要把一个M关联到一个P，这个M就可以从P这里直接获取待执行的G，不用每次都和众多M从一个全局队列中争抢任务了。但仍然有一个全局runq，保存在全局变量sched(调度器)中，对应数据结构是runtime.schedt,里面记录了所有空闲的m和空闲的p。

![image-20220919221707886](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919221707886.png)

​	如果P的本地队列已满，则等待执行的G会被放到全局队列里面，而M会先从关联的P的本地队列runq中获取等待执行的G，没有的话再到调度器持有的全局队列里获取一些任务，如果全局队列里面也没有G，则会从别的P哪里窃取一些G过来。



GPM关系如下，main goroutine创建之后，会被加入到当前P的本地队列里面，然后通过mstart函数开启调度循环，**这个mstart是所有工作线程的入口，主要是调用schedule函数，也就是执行调度循环。**因此对于某个活跃的M而言，不是在执行某个G，就是在执行调度程序获取某个G。因为队列里面只有main goroutine等待执行，所以m0切换到main goroutine，执行入口则是runtime.main.

Runtime.main会做很多事，包括创建监控线程，进行包初始化等，也包括调用熟悉的mian.mian。则会输出hello world。最后在main.main返回时，runtime.main会调用exit()函数结束进程。

![image-20220919223406184](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919223406184.png)



改变函数例子，使用协程来输出hello, world，这样调用过程又是什么样呢？

我们通过go关键字创建协程，会通过编译器转换为newproc函数调用，当然main goroutine也是通过newproc函数创建。创建goroutine时只负责指定入口、参数，而newproc会给goroutine构造一个栈帧，目的是让协程任务结束后，返回到goexit函数中，进行协程资源回收处理等工作。

如果设置GOMAXPROCS只创建一个P，新创建的hello goroutine被添加到当前P的本地队列runq中，然后main.main就结束返回了，再然后exit()函数被调用，进程就结束了。所以hello goroutine没有被执行。

如果想hello goroutine被执行，就要留住时间，可以使用time.Sleep函数，实际上会调用gopark函数将当前协程状态从_Grunning修改为Gwaiting，则main goroutine不会回到当前P的runq中，而是在timer中等待。继而调用schedule进行调度，hello goroutine得到调度执行。等到Sleep时间到达后，timer会把main goroutine重新 _Grunnable状态，放回到P的runq中，最后等到被调用结束程序。

![image-20220919224545995](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919224545995.png)