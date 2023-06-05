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

在有多个P的情况下， 虽然hello goroutine会默认创建到当前P的本地队列里面，但当有空闲的P的情况下，就可以启动新的线程关联到这个空闲的P，同样main.main中可以使用time.Sleep、channel、waitGroup等方式等待hello goroutine执行完成即可。

### Goroutine的创建、让出与恢复

#### 创建

接下来的一个hello world程序，给协程入口函数添加了一个入口函数。我们知道main goroutine执行起来会创建一个hello goroutine，而创建的任务则交由newproc函数来负责，我们通过函数栈帧来看一下newproc函数调用过程：

​	main 函数栈帧自然分配到main goroutine 的协程栈中，栈布局为call指令入栈的返回地址之后是调用者栈基，然后是局部变量区间以及调用其他函数时传递返回值和参数的区间。接下来会调用newproc函数（需要两个参数：第一个是传递给协程入口函数的参数占多少字节，第二个是协程入口函数对应的funcval指针）先入栈fn，也就是协程入口函数hello对应的funcval，再入栈参数大小(一个string类型参数64位下占16字节)，实际上传入的参数也会被放到第二个参数之后，所以局部变量也会被拷贝一份，就相当于给newproc函数传递了三个参数。下面就是call指令入栈的返回地址了，最后就是newproc函数的栈帧了，主要做的就是切换到g0栈去调用newproc1函数。

![image-20220920090755559](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920090755559.png)

为什么要切换到g0栈呢？简单来说是因为g0栈空间大，因为runtime中很多函数都有no-split标记，意味着这个函数不支持栈增长，也就是说编译器不会在这个函数中插入栈增长相关的检测代码，协程栈本来就比线程栈小很多，这些函数要消耗栈空间却又不支持栈增长，就可能会栈溢出。而g0的栈直接分配到线程栈上，栈空间足够大，所以直接切换到g0栈来执行这些函数。

​	调用newproc1函数时，它的任务是创建一个协程，传递了如下参数：

- Fn、siz：就是newproc自己接受的参数
- Argp：用来定位到协程入口函数的参数
- gp: 是当前协程的指针
- pc: 在newproc中调用getcallerpc函数得到的是newproc函数调用结束后的返回地址。

![image-20220920091905259](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920091905259.png)

下面是newproc1创建协程的过程，首先通过acquirem禁止当前m被抢占，

> 为什么要被禁止抢占呢？因为接下来要执行的程序中可能会把当前P保存到局部变量中，若此时M被抢占，P关联到别的M，等到再次恢复时，继续使用这个局部变量里保存的P，就会造成问题，所以为了保持数据一致性，会暂时禁止M被抢占。

接下来会尝试获取一个空闲的G，如果当前P和调度器中都没有空闲的G，就创建一个并添加到allgs中（我们记为hello goroutine），此时他的状态是 _Gdead,并且有了自己的协程栈。接下来如果协程入口函数有参数，就把参数移动到协程栈上，并且把goexit地址加一压入协程栈中。并且newproc1还会给新建的goroutine赋予一个唯一的id，赋值前会将goroutine状态变为 _Grunnable。这个状态意味着可以进入到runq中。

![image-20220920092954176](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920092954176.png)

​		接下来会把这个G放入到当前P的本地队列中，然后判断如果当前有空闲的P并没有处于spinning状态的M（即所有M都在忙），同时主协程已经开始执行了，就调用wakep函数启动一个M并把它赋值为spinning状态，最后会调用releasem函数允许当前M被抢占

![image-20220920093324104](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920093324104.png)

#### 让出与恢复

我们这一次通过channel，看一下协程如何让出又怎样恢复的。channel对应的数据结构是runtime.hchan，里面保存了相关的数据。我们创建的时无缓存的channel，当前goroutine也就是main goroutine会阻塞在ch这里等待数据。

![image-20220920093747641](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920093747641.png)

然后chanrecv通过调用gopark函数，使当前goroutine挂起让出CPU。gopark首先会调用acquirem禁止当前M被抢占，然后把main goroutine状态从 _Grunning变为 _Gwaiting，接下来调用releasem解除M的抢占禁令，最后调用mcall，主要负责保存当前协程的执行线程。最后切换到g0栈，将m.curg置为nil，则当前M正在执行的G便不是main goroutine了，最后schedule函数会寻找下一个等待执行的G。

![image-20220920094418347](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920094418347.png)

最后hello goroutine无论是得到当前M的调度还是被其他M抢了去，总之是可以被执行了。

当hello goroutine执行结束，会关闭这个channel，然后读队列中等待的main goroutine接收的数据置为零值。最终会调用goready函数结束这个G的等待状态，而goready函数会切换到g0栈，并执行runtime.ready函数，将main goroutine状态修改为 _Grunnable，表示又可以被调度执行，然后放入到当前P的本地队列中，最总main goroutine得到执行。结束进程。

![image-20220920094953817](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920094953817.png)

**这样看来 time.Sleep和channel底层都会调用gopark函数来实现协程让出，调用goready函数来将协程恢复到 _Grunnable状态并放回到runq中。**

### 协程让出、抢占、监控和调度

我们知道协程协程执行time.Sleep时，会将G状态从_Grunning 变为 _Gwaiting，并进入到对应的timer中等待。而timer中持有一个回调函数，在指定时间到达后调用这个回调函数，把等在这里的协程恢复到 _Grunnable状态，并放回到runq中。

那么谁负责在定时器时间到达时，触发定时器注册的回调函数呢？

每一个P都持有一个最小堆，存储在p.timers中，用于管理自己的timer，堆顶就是接下来要触发的那一个。每次调度时，都会调用checkTimers函数检查并执行已经到达时间的timer。不过万一所有的M都在忙，不能及时触发调度的话，可能会导致timer执行时间发生较大偏差，所以需要通过监控线程来增加一层保障。

![image-20220920100121531](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920100121531.png)

监控线程是由main goroutine创建的，这个监控线程与GPM中的工作线程不同，不需要依赖P，也不需要由GPM模型调度，会重复执行一系列任务，只不过会视情况调整自己的休眠时间。**监控线程其中一项任务就是保证timers正常执行**，监控线程检测到接下来有timer要执行时，不仅会调整自己的休眠时间，还会在空不出M时，创建新的工作线程，保障timer顺利执行。

#### 被动让出

一个协程等待timer、channel或者IO时间都属于主动让出，如何当不用等待的协程让出呢？

**监控线程的另一个任务就是公平调度**，对运行时间过长的P实行抢占调度，让运行时间超过特定阈值(10ms)的G，应该被动让出。

如何知道运行时间过长呢？

P里面有一个schedtick字段，每当调度执行一个新的G并且不继承上一个G的时间片时，就会将它自增1，通过此来判断当前G是否运行时间过长。如果真的运行时间过长，怎么通知让出呢？

不得不提栈增长，Go语言编译器一般会在函数头部插入栈增长相关代码，因此在协程不主动让出时，**是通过设置stackPreempt标识来通知他让出的**，不过这种抢占方式的缺陷就是过于依赖栈增长代码了，如果使用空的for循环，因为与栈增长无关，监控线程无法通过设置stackPreempt标识来实现抢占，最终会导致程序卡死。

在Go1.14中这一问题得到了解决，实现了异步抢占，不同平台有所不同，子Unix平台中，会向协程关联的M发送信号(sigPreempt)。目标线程会被信号中断，执行sigHandler函数，当检测到sigPreempt信号时，会执行doSigPreempt函数，该函数会向当前被打断的协程上下文中注入一个异步抢占函数调用，最终会调用runtime中的调度逻辑，最终实现被动让出。

![image-20220920102616317](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920102616317.png)

为了充分利用CPU，监控线程还会抢占处于系统调度中的P。因为一个协程要执行系统调用，需要切换到g0栈，在系统调用没执行完之前，M与G绑定在一起了，但也用不到P，所以在陷入系统调用之前，当前M会让出P，只在m.oldp中记录该P，因为P的数目有限，如果有其他协程等待执行，就会与该P关联进行执行。如果当前M从系统调用中恢复，会先检查之前的P是否被占用，没有就继续使用，否则就申请一个，没申请到就将当前G放入全局队列里面。然后当前线程就睡眠了。

#### 调度

schedule要给当前的M找到一个待执行的G，首先判断当前M是否和当前G绑定，如果绑定就不能执行其他G，需要阻塞当前G。如果没有绑定，就看GC是不是在等待执行，如果GC等待执行就执行GC。接下来检查有没有要执行的timer。调度程序还有一定几率从全局runq中获取一部分G到本地队列中，而获取下一个待执行的G时，会先去本地队列查找。

​	没有的话会调用findrunnable函数，该函数会直到获取到待运行的G才会返回，该函数依然会判断是否要执行GC，然后从本地队列获取G，没有就从全局队列获取G，如果还没有就先执行netpoll，恢复IO事件已经就绪的G，它们会被放回全局runq中，然后才会尝试从其他P哪里偷取一些任务。当调度程序获取到G后，要先判断是否有绑定的M，如果有绑定，就退还在进行调度。

![image-20220920104204844](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920104204844.png)

## 3. GC

###  Go的GC

Go语言的GC在准备阶段会为每一个P创建一个mark work协程，把对应的G指针存储到P中，这些后台的mark work协程创建后会很快进入睡眠，等待到标记阶段得到调度执行。

接下来的第一次STW。GC进入_GCMark阶段，全局变量gcphrase记录GC阶段标识，全局变量writeBarrier.enable记录是否开启写屏障，全局变量gcBlackenEnbled用于标识是否允许进行GC标记工作，置为1表示允许。STW阶段准备相关工作并开启写屏障，然后开始标记工作。后台所有mark work都可以得到调度执行，进行标注工作。

![image-20220920112004023](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920112004023.png)

当没有标记任务时，进行第二次STW，GC进入_GCMarkTermination阶段，确认标记工作完成，停止标记。将gcBlackenEnable置为0。接下来进入_GCOff阶段关闭写屏障，同时start the world。进入GC清扫阶段，（在_GCOff之前，新分配的对象会被直接标记为黑色，进入_GCOff阶段后，新分配的对象为白色）。

![image-20220920112439145](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920112439145.png)

执行清扫工作的协程由runtime.main在gcenable中创建,对应G指针在存储在全局变量sweep中，清扫阶段这个后台的sweep会被加入到runq中，被调度时执行清扫任务。因为清扫阶段也是增量执行的，所以每一轮GC前需要确认上一轮GC是否清扫完成。

![image-20220920112744064](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920112744064.png)

### 常见GC算法介绍

从进程虚拟地址空间来看，程序要执行的指令在代码块，全局变量、静态数据分配到数据段，而函数的局部变量、参数和返回值都可以在函数栈中找到。如果在编译阶段不能确定数据大小或者对象生命周期会超过当前函数，就会被分配到堆上。

> 直接分配到栈上的数据会随着函数调用栈的销毁释放自身占用的内存，而堆上的占用的内存则需要程序主动释放才能被从新使用。

有些程序语言(C、C++)需要手动释放那些不需要的、分配到堆上的数据，被称为“手动垃圾回收”。但是会出现释放过早的问题，这样释放的内存可能已经被清空、重新分配或者已经归还给操作系统了，这就是“悬挂指针”问题。而如果忘记释放，就会一直占用内存，出现“内存泄露”。所以越来越多编程语言支持自动垃圾回收，由运行时识别不需要使用的数据，并及时进行释放。不能从栈或者数据段追踪到的数据一定是用不到的数据，因此，目前主流的GC算法都是使用“可达性”近似等价于“存活性”的。

![image-20220920113646523](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920113646523.png)

**标记-清扫算法的核心思想是：将栈、数据段上的数据对象作为root，基于他们进一步追踪，把能追踪到的数据进行追踪标记，剩下的追踪不到的就是垃圾了。**

三色抽象可以可以清晰的展现追踪式垃圾回收中对象状态变化的过程。垃圾回收开始时所有数据都是白色，然后把直接追踪到的root节点标记为灰色，灰色表示基于当前节点的追踪还没有完成，完成后就会把节点标记为黑色，表示是存活数据，基于黑色节点找到的节点标记为灰色，表示还需要继续追踪。当没有灰色节点时，表示标记过程结束，此时有用数据都是黑色，垃圾都是白色。

![image-20220920132243091](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920132243091.png)

> 标记-清扫算法实现简单，但是容易造成内存碎片化，面对大量不连续的小分块内存，需要找到合适的内存分块代价很高，也会造成很多无法使用的小块内存。

上述问题可以基础BiBOP(Big Bag Of Page)的思想，把内存块划分为多种规格大小，对相同规格的内存进行统一管理，这样可以更快匹配到大小合适的空闲内存，规格类型划分合理也有助于提高内存使用率。

- **标记-整理算法**

也有人提出了通过移动数据来减少碎片化的方法，在标记-整理算法中，则就是这样的实现。标记阶段同标记-清理算法的标记阶段，完成标记工作后，移动非垃圾数据。

> 标记-整理算法有效解决了内存碎片化问题，但带来了多次扫描与移动的开销。

- **复制式回收算法**

会将堆内存划分为两个相等的空间from和to，程序执行时使用from空间，垃圾回收扫描from空间，把能追踪到的数据复制到to空间，所有有用空间复制完毕后，两个空间交换一下即可。

> 这种复制式回收也不会带来碎片化问题但是只有一半堆内存可以被使用。

为了提高堆内存使用率，通常会搭配其他垃圾回收算法使用，只在一部分堆内存中使用复制式回收。例如在分代回收的垃圾算法中就经常搭配复制式回收。

- **分代式回收**

分代式回收是基于弱分代假说：大部分对象都在年轻时死亡。也就是说新生代对象成为垃圾的概率高于老年代对象。

- **引用计数式垃圾回收**

上述都是追踪式垃圾回收，和引用计数式垃圾回收有所不同。引用计数是指一个数据对象被引用的次数，当引用计数更新到0时，就表示这个对象不会被使用。

> 高频率的更新引用计数也会造成不小的开销。并且会存在循环引用的情况。



垃圾回收就会有STW(Stop the world)，我们可以将垃圾回收工作分多次完成，即垃圾回收与用户程序交替执行，**这被称为增量式垃圾回收**，可以缩短每次暂停的时间，但是交替执行过程中，可能标记的黑色对象，立即被用户又修改了，会出现误判的情况。

**在三色抽象中，如果出现黑色对象到白色对象的引用，并且没有灰色对象指向白色对象，就会出现误判。**我们需要限制这种情况的发生：

- 强三色不变式：不允许出现黑色对象到白色对象的引用
- 弱三色不变式：允许出现黑色对象到白色对象的引用，但是该白色对象也会有灰色对象引用。

实现强、弱三色不变式通常是建立“读\写屏障”。

- 写屏障

在强三色不变式中，**插入写屏障**通过将白色对象变为灰色，或者把黑色对象变为灰色来保证强三色不变式。

在弱三色不变式中，删除灰色对象到白色对象的引用时，可以把白色对象变成灰色，这属于**删除写屏障**。

- 读屏障

非移动式垃圾回收天然不需要读屏障，而复制式回收算法，当GC和用户程序交替执行时，读数据便也不安全了。例如：回收器已经将A复制到to空间，此时B引用了陈旧对象指针，当把B移动到to空间后，B对象持有的就是陈旧指针了：

![image-20220920135936203](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920135936203.png)

> 当检测到引用对象存在新副本时，转而读取to空间的新副本即可。



多核情况下：

- 并行垃圾回收

并行情况下，同步问题是不可避免的，会出现分工不均，实现负载均衡又会增加进程间同步开销。复制式回收还需要注意重复复制的问题。

- 并发垃圾回收

指的是多核场景下，垃圾回收和用户程序并行执行的情况，用户程序和垃圾回收可能会同时使用写屏障记录集，所以并发写屏障的设计必须通过STW时间，将垃圾回收开始的消息及时准确的通知到所有线程。不会出现某些线程开启写屏障机制延迟的问题。

所以在实际应用中，在某些阶段采取STW方式，其他阶段支持并发“主体并发式垃圾回收”更容易实现。 在此基础上再支持增量式垃圾回收，**则称为“主体并发增量式垃圾回收”。**

![image-20220920141225862](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220920141225862.png)

**Go语言的垃圾回收采用标记-清理算法，支持主体并发增量式垃圾回收，使用插入和删除两种写屏障结合的混合写屏障。**

### GC触发方式

- 手动触发

入口在runtime.GC函数中

- 分配内存

分配内存时，有些情况需要检查是否需要触发GC，每次GC都会在标记结束后设置下一次触发GC的堆内存分配量，分配大对象或者从mcentral获取空闲内存时，会判断是否达到了这里设置的gc_trigger，以决定是否要触发GC。

- sysmon

监控线程的一个任务就是强制执行GC，runtime包初始化时会开启一个协程，监控线程在检测到距离上一次GC已经超过指定时间时，就会把该协程添加到runq中，等待得到调度执行。

## 全CPU采集

### 问题

假设有如下的一段代码：

```go
func main() {
	for {
		// Http request to a web service that might be slow.
		slowNetworkRequest()
		// Some heavy CPU computation.
		cpuIntensiveTask()
		// Poorly named function that you don't understand yet.
		weirdFunction()
	}
}
```

使用pprof的profile采集的火焰图如下所示：
![image-20220922090400054](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922090400054.png)

可以看到所有时间消耗都在cpuIntensiveTask任务上，而我们通过简单的统计时间方式可以得到如下效果：

![image-20220922091254594](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922091254594.png)

> profile的采集原理是，系统1s内发送100次信号SIGPROF，信号到达时，Go会捕获该信号，调用信号处理函数去获取当前活跃的Goroutine的调用栈，即信号将会中断当前线程，并记录当前线程的堆栈。
>
> 出现上述情况是因为profile的采集是On-CPU的，等待I/O所花费的时间被隐藏了。由于 Go 使用非阻塞 I/O，等待 I/O 的 Goroutines 被停放并且不在任何线程上运行。因此，它们最终对 Go 的内置 CPU 分析器基本上是不可见的。

而使用fgprof，则三个函数的消耗就可以显示：

![image-20220922092820005](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922092820005.png)

### 原理

原理是后台启动了一个goroutine，每秒99次唤醒99次调用`runtime.GoroutineProfile`函数，这个函数将返回所有当前的goroutine与他们的调用栈，不管他们是On/Off CPU状态。但是开销将会随着活动的goroutine的增加而增加，当服务小于1000goroutine的时候，无须担心，但是当goroutine数量达到10k或者更多的时候，开销将会很大。

> `func GoroutineProfile(p []StackRecord) (n int, ok bool)`
>
> GoroutineProfile 返回 n，即活动 goroutine 堆栈配置文件中的记录数。如果 len(p) >= n，GoroutineProfile 将 profile 复制到 p 并返回 n，true。如果 len(p) < n，GoroutineProfile 不会改变 p 并返回 n，false。
>
> `runtime.StackRecord`用于描述单个执行堆栈。结构为[32]uintptr，记录的是程序计数器PC的值，所以调用栈追踪深度为32，不清楚的调用栈函数为0，如下是一条StackRecord记录：
>
> `[4299802552 4299802509 4299802384 4299446240 4299614708 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]`

由于并不知道会有多少runtime.StackRecord记录数，所以采用动态增长切片长度的方式，先使用长度为0的切片去调用，然后runtime.GoroutineProfile会返回记录数，然后动态增加1.1*n去获取记录，因为这期间可能增加了goroutine。

![image-20220922100428655](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922100428655.png)

然后将每一次唤醒获取的记录进行合并，采用map的数据结构，存储到结构体wallclockProfile.stacks中。同时ignore中存储当前函数的栈帧，用户后面忽略统计函数的调用栈。

Runtime.Frame包含函数名，函数所在文件、行号等信息，如下所示：

```go
type Frame struct {
    // PC is the program counter for the location in this frame.
    // For a frame that calls another frame, this will be the
    // program counter of a call instruction. Because of inlining,
    // multiple frames may have the same PC value, but different
    // symbolic information.
    PC uintptr

    // Func is the Func value of this call frame. This may be nil
    // for non-Go code or fully inlined functions.
    Func *Func

    // Function is the package path-qualified function name of
    // this call frame. If non-empty, this string uniquely
    // identifies a single function in the program.
    // This may be the empty string if not known.
    // If Func is not nil then Function == Func.Name().
    Function string

    // File and Line are the file name and line number of the
    // location in this frame. For non-leaf frames, this will be
    // the location of a call. These may be the empty string and
    // zero, respectively, if not known.
    File string
    Line int

    // Entry point program counter for the function; may be zero
    // if not known. If Func is not nil then Entry ==
    // Func.Entry().
    Entry uintptr
    // contains filtered or unexported fields
}
```

![image-20220922102411065](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922102411065.png)

记录已经存在就count++,没有就创建一个记录，并获取这条调用栈的函数名、文件名、行号等信息存入frames的切片中。

![image-20220922185413286](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922185413286.png)

最后将wallclockProfile结构体转换成profile.Profile结构体即可展示火焰图。先将map的调用栈转换成切片形式，同时去忽略本函数栈帧。转换过程中将count*10ms（1s/hz），就表示成了消耗时间。

```go
func (p *wallclockProfile) exportPprof(hz int, startTime, endTime time.Time) *profile.Profile {
	prof := &profile.Profile{}
	m := &profile.Mapping{ID: 1, HasFunctions: true}
	prof.Period = int64(1e9 / hz) // Number of nanoseconds between samples.
	prof.TimeNanos = startTime.UnixNano()
	prof.DurationNanos = int64(endTime.Sub(startTime))
	prof.Mapping = []*profile.Mapping{m}
	prof.SampleType = []*profile.ValueType{
		{
			Type: "samples",
			Unit: "count",
		},
		{
			Type: "time",
			Unit: "nanoseconds",
		},
	}
	prof.PeriodType = &profile.ValueType{
		Type: "wallclock",
		Unit: "nanoseconds",
	}

	type functionKey struct {
		Name     string
		Filename string
	}
	funcIdx := map[functionKey]*profile.Function{}

	type locationKey struct {
		Function functionKey
		Line     int
	}
	locationIdx := map[locationKey]*profile.Location{}
	for _, ws := range p.exportStacks() {
		sample := &profile.Sample{
			Value: []int64{
				int64(ws.count),
				int64(1000 * 1000 * 1000 / hz * ws.count),
			},
		}

		for _, frame := range ws.frames {
			fnKey := functionKey{Name: frame.Function, Filename: frame.File}
			function, ok := funcIdx[fnKey]
			if !ok {
				function = &profile.Function{
					ID:         uint64(len(prof.Function)) + 1,
					Name:       frame.Function,
					SystemName: frame.Function,
					Filename:   frame.File,
				}
				funcIdx[fnKey] = function
				prof.Function = append(prof.Function, function)
			}

			locKey := locationKey{Function: fnKey, Line: frame.Line}
			location, ok := locationIdx[locKey]
			if !ok {
				location = &profile.Location{
					ID:      uint64(len(prof.Location)) + 1,
					Mapping: m,
					Line: []profile.Line{{
						Function: function,
						Line:     int64(frame.Line),
					}},
				}
				locationIdx[locKey] = location
				prof.Location = append(prof.Location, location)
			}
			sample.Location = append(sample.Location, location)
		}
		prof.Sample = append(prof.Sample, sample)
	}
	return prof
}
```

> Profile结构体中，每一条调用栈存入Sampling切片中，记录了调用次数和转换的时间，同时映射到一个Location结构体，Location中存储了函数相关信息，包括行号与Function结构体，Function中存储函数相关信息。同时profile中也有Location结构体切片。

![image-20220922185646202](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220922185646202.png)

### 缺陷

内部的C函数并不会被跟踪，因为原理是采集goroutine的情况,并不会跟踪C的函数。

Goroutine分析可以捕获的最大调用栈深度是32。如果调用栈深度超过，将会被截断，也就意味着将错过调用栈中导致采样处于活动状态的函数部分。

### goroutine采样原理

#### 1、概述

Go runtime 在一个名为[allgs](https://github.com/golang/go/blob/3a778ff50f7091b8a64875c8ed95bfaacf3d334c/src/runtime/proc.go#L500)的简单切片中保存了所有的goroutine。包含了活着和死去的goroutine。死去的goroutine被保留用于新的goroutine产生的时候复用。

Go runtime 中有提供API来追踪allgs中存活的goroutine及其堆栈等各种信息。一些API也能提供统计各种信息的功能。

尽管API之间存在差异，但是存活的goroutine都有如下[基本](https://github.com/golang/go/blob/9b955d2d3fcff6a5bc8bce7bafdc4c634a28e95b/src/runtime/mprof.go#L729)[定义](https://github.com/golang/go/blob/9b955d2d3fcff6a5bc8bce7bafdc4c634a28e95b/src/runtime/traceback.go#L931)：

- 没有[死亡](https://github.com/golang/go/blob/go1.15.6/src/runtime/runtime2.go#L65-L71)

- 不是[系统goroutine](https://github.com/golang/go/blob/9b955d2d3fcff6a5bc8bce7bafdc4c634a28e95b/src/runtime/traceback.go#L1013-L1021)也不是终结器goroutine

总结来说，正在运行的goroutine以及等待I/O、锁、通道、调度等的goroutine都被认为是存活的goroutine。

#### 2、声明

Go中所有可用的Goroutine分析都一个O(N)时间复杂度的STW阶段（其中N是分配的goroutine的数量）。一个简单的[基准测试](https://github.com/felixge/fgprof/blob/fe01e87ceec08ea5024e8168f88468af8f818b62/fgprof_test.go#L35-L78)[表明](https://github.com/felixge/fgprof/blob/master/BenchmarkProfilerGoroutines.txt)，当使用 [runtime.GoroutineProfile() ](https://pkg.go.dev/runtime#GoroutineProfile)API 时，每个 goroutine 会停止大约 1µs。但是这个数字很可能会随着程序的平均堆栈深度、死去的 goroutine 的数量等因素而波动。

因此，对延迟非常敏感并且使用活动数达数千个goroutine的程序，进行goroutine分析要谨慎。大多数不会产生大量 goroutine 并且可以容忍几毫秒的偶然额外延迟的应用程序在生产中的连续goroutine 分析应该没有问题。

#### 3、goroutine属性

Goroutines 有很多[属性](https://github.com/golang/go/blob/go1.15.6/src/runtime/runtime2.go#L406-L486)可以帮助调试 Go 应用程序。通过本文档后面描述的 API 不同程度地公开。

- `goid`：goroutine的唯一id，主goroutine的id为1。

- `atomicstatus` ：goroutine 的状态，以下之一：
    - `idle`: 刚被分配
    - `runnable` :在运行队列，等待被调度
    - `running` :在操作系统线程上执行
    - `syscall` :被系统调用阻塞
    - `waiting` :由调度程序停放，参见`g.waitreason` 
    - `dead` :刚刚退出或正在重新初始化
    - `copystack` :当前正在移动堆栈
    - `preempted` :只是抢占了自己

- `waitreason` :goroutine处于等待状态的原因，例如：Sleep、channel操作、i/o、gc等

- `waitsince` :goroutine进入等待或系统调用状态的大致时间戳，由等待开始后的第一个GC确定。

- `lables` :一组可以附加到 goroutines 的键/值[分析器标签](https://rakyll.org/profiler-labels/)。

- `stack trace` :当前正在执行的函数及其调用者。这表现为文件名、函数名和行号的纯文本输出或程序计数器地址 (pcs) 的片段。 🚧 研究更多细节，例如func/file/line 文本可以转换成 pcs 吗？

- `gopc` :导致该 goroutine 被创建的 go ... 调用的程序计数器地址 (pc)。可以转换为文件、函数名和行号。

- `lockedm` :这个 goroutine 被锁定的线程，如果有的话。

#### 4、特征矩阵

下面的功能矩阵通过各种 API 让您快速了解这些属性的当前可用性。

![img](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/(null)-20220922192607181.(null))
