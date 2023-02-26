---
author: "Narcissus"
title: "Java并发相关学习"
date: "2022-02-10"
Lastmod: "2022-02-10"
description: "学习java的并发相关知识"
tags: ["Java", "面试", "面试总结"]
categories: ["Java"]
---

## Java内存模型(JMM)

### JMM概述

Java内存模型的主要目的是定义程序中各种变量的访问规则，此处的变量是指被线程共享的，包括实例字段、静态字段和构成数组对象的元素。**Java内存模型规定了所有变量都存储在主内存中，每个线程有自己的工作内存，线程在自己的工作内存中保存了需要使用的变量的主内存副本，线程对变量的所有操作不能直接作用于主内存。** 可以类比CPU的高速缓存模型。

![image-20230211134059712](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230211134059712.png)

### 重排序

编译器和处理器为了优化程序的性能会对指令序列进行重新排序。重排序必须遵循`as-if-serial`语义：**不管怎么重排序，单线程程序的执行结果不会改变。** 在单处理器中，编译器和处理器不会改变存在数据依赖关系的两个操作的顺序。

### happens-before原则

happens-before原则(先行发生原则)是对java内存模型的简化，可以帮助程序员理解并发安全。

> happens-before原则的定义：
>
> 1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
> 2. 两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照 happens-before 关系指定的顺序来执行。如果重排序之后的执行结果，与按 happens-before 关系来执行的结果一致，那么这种重排序并不非法(也就是说， JMM 允许这种重排序)。



## volatile关键字的原理

在Java中，如果一个变量被声明为volatile，java线程内存模型会确保所有线程看到这个变量的值是一致的。volatile关键字**可以保证数据的可见性，但是不能保证数据的原子性**。同时**volatile可以禁止指令重排序**。

> 如何保证数据的可见性？
>
> java线程对volatile修饰的变量的修改，会立即从工作内存中刷新到主内存中，使其他线程立即可见，并且其他线程的工作内存中缓存的该变量的值也会失效。
>
> 如何禁止指令重排序？
>
> 有volatile饰的变量，赋值后执行了一个`lock addl$0x0，(%esp)`指令操作，这个操作相当于一个内存屏障，(内存屏障指重排序时不能把后面的指令重排序到内存屏障之前的位置)。
>
> **注意：volatile修饰的变量不能保证数据的原子性，是因为虽然volatile修饰的变量没有数据一致性问题，但是Java里的运算符并非原子操作，导致volatile变量的运算在并发下一样是不安全的**。



## synchronized关键字

### 如何使用synchronized?

synchronized关键字有以下三种使用场景：

- 对于普通的同步方法，锁的是当前实例对象。
- 对于静态的同步方法，锁的是当前类。
- 对于同步的代码块，锁的是synchronized括号里配置的对象。`synchronized(object)`锁的是给定对象的锁；`synchronized(类.class)`锁的是给定的类。

> 静态`synchronized`方法与非静态`synchronized`方法之间的调用互斥吗？
>
> 不互斥，因为两类方法锁定的对象并不相同，所以不会发生互斥现象。
>
> 
>
> 构造方法可以用synchronized关键字修饰吗？
>
> 构造方法本身就是线程安全的，不同线程调用的构造器将产生不同的对象，两者之间没有关系。所以**构造方法不能使用 synchronized 关键字修饰。**

### synchronized原理

JVM基于进入和退出Monitor对象来实现方法同步和代码块同步。**同步代码块使用`monitorenter`和`monitorexit`指令实现。同步方法是在方法修饰符上加了一个 `ACC_SYNCHRONIZED` 修饰来实现的。**

`monitorenter`指令是在编译后插入到同步代码块的开始位置，而 `monitorexit` 是插入到方法结束处和异常处，JVM 要保证每个 `monitorenter` 必须有对应的 `monitorexit` 与之配 对。任何对象都有一个 monitor 与之关联，当且一个 monitor 被持有后，它将处于锁定状态。线程执行到 `monitorenter` 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。

- **对象头**

对象在内存中的布局分为三个模块：**对象头、实例数据和对象填充**。对象头中包含两部分：**MarkWord、类型指针，如果是数组，那么还包括数组长度。**

![image-20230212085424645](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230212085424645.png)

**多线程下 synchronized 的加锁就是对同一个对象的对象头中的 MarkWord 中的变量进行CAS操作。**

- **锁的升级**

锁一共有4种状态，级别从低到高分别是：**无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态**。锁的状态会随着竞争情况逐渐升级并且不能降级。**这种升级不能降级的策略，目的是为了提高获得锁和释放锁的效率。**

> 1. 偏向锁
>
> **偏向锁是针对一个线程而言的，线程获得锁之后对象头中会记录获取锁的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁或解锁，这样可以省略很多开销。** 假如有两个线程来竞争该锁话，那么偏向锁就失效了，进而升级成轻量级锁了。
>
> - 偏向锁的加锁
>
> > 如果锁标志是未偏向状态，使用CAS将MarkWord中的线程ID设置为自己的ID，如果成功，则获得偏向锁；如果失败，则进行锁升级。
> >
> > 如果锁标志是已偏向状态，MarkWord中线程ID是自己的，则获得锁；如果不是，则进行锁升级
>
> - 偏向锁的撤销
>
> 偏向锁的锁升级需要进行偏向锁的撤销。撤销偏向的操作需要在全局检查点执行。我们假设线程A曾经拥有锁（不确定是否释放锁）， 线程B来竞争锁对象，如果当线程A不在拥有锁时或者死亡时，线程B直接去尝试获得锁（根据是否 、允许重偏向（`rebiasing`），获得偏向锁或者轻量级锁）；如果线程A仍然拥有锁，那么锁升级为轻量级锁，线程B自旋请求获得锁。
>
> 
>
> 2. 轻量级锁
>
> 轻量级锁是因为只使用CAS操作来获取锁。
>
> - 轻量级锁的加锁
>
> JVM 会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的 Mark Word 复制到锁记录中，官方称为 Displaced Mark Word。然后线程尝试使用 CAS 将对象头中的 Mark Word 替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。当竞争线程的自旋次数达到界限值（`threshold`），轻量级锁将会膨胀为重量级锁。
>
> - 轻量级锁的解锁
>
> 轻量级解锁时，会使用原子的 CAS 操作将 Displaced Mark Word 替换回到对象头， 如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。
>
> 
>
> 3. 重量级锁
>
> 重量级锁，是使用操作系统互斥量（`mutex`）来实现的传统锁。 当所有对锁的优化都失效时，将退回到重量级锁。它与轻量级锁不同，竞争的线程不再通过自旋来竞争线程， 而是直接进入堵塞状态，此时不消耗CPU，然后等拥有锁的线程释放锁后，唤醒堵塞的线程， 然后线程再次竞争锁。

![image-20230212094445727](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230212094445727.png)

****

## Java线程基础知识

### wait、join、sleep、yield

#### wait()方法

`wait()`方法是父类Object的方法，当一个线程调用一个共享变量的wait()方法时， 该调用线程会被阻塞挂起， 直到发生下面几件事情之一才返回: 

1. 其他线程调用了该共享对象的 notify()或者notifyAll()方法; 
2. 其他线程调用了该线程的 interrupt()方法，该线程抛出 InterruptedException异常返回。

**注意：调用`wait()`方法的线程必须先获取对象的监视器锁，即使用`synchronized`，否则调用线程会抛出 IllegalMonitorStateException异常。**

****

#### join()方法

`join()`方法是Thread类直接提供的，一个线程内如果调用另外一个线程的`join()`方法，则当前线程停止执行，一直等到`join()`方法的线程执行完毕。

**本质上，`join()`方法内部也是调用wait()方法，当线程A中调用线程B的join方法阻塞后，会一直等到线程B执行完毕，线程B执行完毕后，会唤醒所有等待该线程B对象锁的线程**。

****

#### sleep()方法

Thread类中有一个静态的 sleep方法，当一个执行中的线程调用了 Thread的 sleep方法后，调用线程会暂时让出指定时间的执行权，也就是在这期间不参与 CPU 的调度，**但是该线程所拥有的监视器资源，比如锁还是持有不让出的 。** 指定睡眠时间截止后，线程就会处于就绪状态，参与CPU调度。

> 如果在睡眠期间其他线程调用了该线程的 interrupt()方法中断了该线程，则该线程会在调用 sleep 方法的地方抛出 IntermptedException 异常而返回 。

****

#### yield()方法

Thread类中有一个静态的 yield方法，当一个线程调用 yield方法时，实际就是在暗示线程调度器**当前线程请求让出自己 的 CPU 使用**，但是线程调度器可以无条件忽略这个暗示。

> sleep和yield方法的区别
>
> 当线程调用 sleep 方法时调用线程会被阻塞挂起指定的时间，在这期间线程调度器不会去调度该线程 。 
>
> 调用 yield 方法时，线程只是让出自己剩余的时间片，并没有被阻塞挂起，而是处于就绪状态，线程调度器下 一次调度时就有可能调度到当前线程执行 。

****

## Java锁的基础知识

### 乐观锁/悲观锁

根据线程是否要锁住同步资源，可以将锁分为：乐观锁和悲观锁。

#### 什么是悲观锁？使用场景？

对于同一数据的并发操作，悲观锁认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。

像Java中的`synchronized`关键字和`Lock`的实现类都是悲观锁。

**悲观锁通常多用于写比较多的情况下，避免频繁的失败和重试影响性能。**



#### 什么是乐观锁？使用场景？

乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。如果这个数据没有被更新，当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则根据不同的实现方式执行不同的操作（例如**报错或者自动重试**）。

乐观锁在Java中是通过使用无锁编程来实现，最常采用的是CAS算法，Java原子类中的递增操作就通过CAS自旋实现的。Git工具也是基于乐观锁来实现的。

**乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。**



#### 如何实现乐观锁？(CAS算法讲解)

乐观锁一般采用CAS算法或者版本号机制来实现。

- **版本号机制**

一般是在数据表中加上一个数据版本号 `version` 字段，表示数据被修改的次数。当数据被修改时，`version` 值会加一。当线程 A 要更新数据值时，在读取数据的同时也会读取 `version` 值，在提交更新时，若刚才读取到的 version 值为当前数据库中的 `version` 值相等时才更新，否则重试更新操作，直到更新成功。

- **CAS算法**

CAS全称 Compare And Swap（比较与交换），是一种无锁算法。

> CAS 是一个原子操作，底层依赖于一条 CPU 的原子指令。
>
> Java 语言并没有直接实现 CAS，CAS 相关的实现是通过 C++ 内联汇编的形式实现的，通过`sun.misc`包下面的`Unsafe`类提供了`compareAndSwapObject`、`compareAndSwapInt`、`compareAndSwapLong`方法来实现的对`Object`、`int`、`long`类型的 CAS 操作。

CAS算法涉及到三个操作数：

> - 需要读写的内存值V
> - 进行比较的值A
> - 要写入的新值B

当且仅当 V 的值等于 A 时，CAS通过原子方式用新值B来更新V的值（“比较+更新”整体是一个原子操作），否则不会执行任何操作。一般情况下，“更新”是一个不断重试的操作。

> CAS虽然很高效，但是也存在三大问题：
>
> 1. **ABA问题**，CAS需要在操作值的时候检查内存值是否发生变化，没有发生变化才会更新内存值。但是如果内存值原来是A，后来变成了B，然后又变成了A，那么CAS进行检查时会发现值没有发生变化，但是实际上是有变化的。**ABA问题的解决思路就是在变量前面添加版本号，每次变量更新的时候都把版本号加一**，这样变化过程就从“A－B－A”变成了“1A－2B－3A”。
> 2. **循环时间长开销大**，CAS操作如果长时间不成功，会导致其一直自旋，给CPU带来非常大的开销。
> 3. **只能保证一个共享变量的原子操作**，对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。Java从1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。



### 自旋锁/适应性自旋锁

线程尝试获取同步资源的锁失败，如果此线程不放弃CPU时间片，通过自旋等待锁的释放，称之为自旋锁。线程的上下文切换比较消耗资源，如果同步资源的锁定时间很短，就可以考虑使用自旋锁。

**适应性自旋锁意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。**如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

****

### 公平锁/非公平锁

公平锁是指多个线程按照申请锁的顺序来获取锁，**线程直接进入队列中排队**，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

**非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待**。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。

****

### 可重入锁/非可重入锁

可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，**再进入该线程的内层方法会自动获取锁**（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。

****

## Java线程安全的实现方法(线程同步的方案)

保障Java多线程安全的方案有如下三种：

- **互斥同步**，也称为阻塞同步
- **非阻塞同步**
- **无同步方案**

****

### 互斥同步(阻塞同步)

互斥同步是一种最常见也是最主要的并发正确性保障手段。保证共享数据的互斥性，即可实现多线程之间同步的正确定。

Java中互斥同步的方式有：**synchronized关键字和Lock锁。**

> 从解决问题的方式上看，互斥同步属于一种悲观的并发策略。
>
> **由于Java的线程是1:1映射到操作系统的原生内核上，因此阻塞和唤醒线程都需要操作系统的线程调度来执行，不可避免需要用户态和内核态之间的切换，这种状态转换需要更多的处理器时间。**
>
> 即对共享资源加锁的方式，会导致用户态到核心态转换、维护锁计数器和检查是否有被阻塞的线程需要被唤醒等开销。

### 非阻塞同步

非阻塞同步是一种**基于冲突检测的乐观策略**。 通俗地说就是不管风险，先进行操作，如果没有其他线程争用共享数据，那操作就直接成功了;如果共享的数据的确被争用，产生了冲突，那再进行其他的补偿措施，**最常用的补偿措施是不断地重试**， 直到出现没有竞争的共享数据为止。

这种乐观的并发策略不需要将线程阻塞挂起，因此称为非阻塞同步，这种代码也称为无锁编程。

> 但是我们**必须保证操作和冲突检测两个步骤具备原子性**， 因此需要硬件指令集的支持。这类指令常用的有：
>
> - 测试并设置(Test and Set)
> - 获取并增加(Fetch and Increment)
> - 交换(Swap)
> - 比较并交换(Compare and Swap，CAS)
> - 加载链接/条件存储(Load Linked/Store Conditional，LL/SC)
>
> ****
>
> Java暴露出来的是CAS操作，该操作由`sun.misc.Unsafe`类里面的 `compareAndSwapInt()`和`compareAndSwapLong()`等几个方法包装提供。
>
> 不过由于Unsafe类在**设计上就不是提供给用户程序调用的类**  (Unsafe::getUnsafe()的代码中限制了只有启动类加载器Bootstrap ClassLoader加载的Class才能访问它)。直到JDK 9之后，Java类库才在`VarHandle`类里开放了面向用户程序使用的CAS操作。

### 无同步方案

保证线程安全并不一定需要同步，同步只是保证了共享数据的安全，**如果不涉及共享数据，自然不需要同步** 。有如下两类：

- **可重入代码**

指可以在代码执行的任何时刻中断它，转而去执行另外一段代码(包括递归调用它本身)，而在控制权返回后，原来的程序不会出现任何错误，也不会对结果有所影响。

> 可重入代码有一些共同的特征，例如，**不依赖全局变量、存储在堆上的数据和公用的系统资源，用到的状态量都由参数中传入** ，不调用非可重入的方法等。

- **线程本地存储**

可以把共享数据的可见范围限制在同一个线程之内，那么也无需考虑同步的问题。Java中如果需要变量只要被某个线程独享，**可以通过`java.lang.ThreadLocal`类来实现线程本地存储的功能。** 

****

## ThreadLocal原理

### ThreadLocal原理

ThreadLocal提供了线程本地变量，也就是如果你创建了一个 ThreadLocal 变量 ，那么**访问这个变量的每个线程都会有这个变量的一个本地副本**  。当多个线程操作这个变量时，实际操作的是自己本地 存里面的变量 ，从而避免了线程安全问题。

> 每一个Thread类中都有两个`ThreadLocalMap`类型的变量：**`threadLocals和inheritableThreadLocals`**
>
> 在默认情况下， 每个线程中的这两个变量都为 null，只有当前线程第一次调用 ThreadLocal 的 set或者get方法时才会创建它们。因此，**本质上每个线程的本地变量不是存放在ThreadLocal变量中，而是存放在调用线程的`threadLocals`变量中**， ThreadLocal就是一个工具壳，它通过set方法把 value值放入调用线程的 threadLocals里面并存放起来， 当调用线程调用它的 get方法时，再从当前 线程的 threadLocals变量里面将其拿出来使用 。
>
> > 注意：如果调用线程不终止，那么这个本地变量会一直存在，**可能会造成内存泄漏**，因此ThreadLocal变量使用完毕后要记得调用remove方法进行删除。
> >
> > 原因分析：
> >
> > **ThreadLocal与其保存的值都是被放在ThreadLocalMap内部Entry对应的实例上。而Entry持有ThreadLocal的弱引用，当ThreadLocal只被Entry引用时，ThreadLocal对象将在GC时被无条件的回收掉。而对应的值却无法被回收，造成内存的泄漏，最终可能引起OOM。**
>
> ****
>
> Thread里面的 threadLocals为何被设计为 map结构?
>
> 因为每一个线程可以关联多个ThreadLocal变量，因此设计成map结构， 其中key为我们定义的 ThreadLocal 变量的 this 引用， value则为我们使用 set方法设置的值。

> 因为每一个线程都会有一份ThreadLocal的副本，所以同一个 ThreadLocal变量在父线程中被设置值后，在子线程中是获取不到的。即ThreadLocal不支持继承性。

****

### InheritableThreadLocal原理

为了解决ThreadLocal的不支持继承性问题，就有了`InheritableThreadLocal`，继承了ThreadLocal。而每一个Thread类中也有一个`inheritableThreadLocals`的变量用于存储值。

> InheritableThreadLocal的原理就是：子线程是通过在父线程中通过调用`new Thread()`方法来创建子线程，`Thread#init`方法在`Thread`的构造方法中被调用。在`init`方法中**会把父线程中 inheritableThreadLocals变量里面的本地变量复制一份保存到子线程的 inheritableThreadLocals 变量里面。** 
>
> > 注意：同理，Thread类中的`inheritableThreadLocals`变量也是在当前线程调用set或get方法设置变量时，创建当前线程的 inheritableThreadLocals变量。

> InheritableThreadLocal只能在父子进程之间传输，对于线程池的复用逻辑，则会存在问题，因此阿里巴巴开源了[TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)组件就可以解决该问题，源码只有约1000行，可以学习。

****

****

## AQS详解

AQS 的全称为 `AbstractQueuedSynchronizer` ，翻译过来的意思就是抽象队列同步器。他是实现同步器的基础组件，并发包中锁的底层实现就是AQS。

### AQS原理

AQS核心思想是，**如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效的工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配**。这个机制主要用的是**CLH队列的变体**实现的，将暂时获取不到锁的线程加入到队列中。

CLH：Craig、Landin and Hagersten队列，是单向链表，**AQS中的队列是CLH变体的虚拟双向队列**（FIFO，虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系），AQS是通过将每条请求共享资源的线程封装成一个节点来实现锁的分配。

![image-20230219084400333](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230219084400333.png)

**AQS使用一个Volatile的int类型的成员变量来表示同步状态，通过内置的FIFO队列来完成资源获取的排队工作，通过CAS完成对State值的修改。**

> 以 `ReentrantLock` 为例，`state` 初始值为 0，表示未锁定状态。A 线程 `lock()` 时，会调用 `tryAcquire()` 独占该锁并将 `state+1` 。此后，其他线程再 `tryAcquire()` 时就会失败，直到 A 线程 `unlock()` 到 `state=`0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A 线程自己是可以重复获取此锁的（`state` 会累加），这就是可重入的概念。但要注意，获取多少次就要释放多少次，这样才能保证 state 是能回到零态的。

****

AQS中线程共享资源的锁模式有两种：

- **Shared：** 表示线程以共享的方式等待锁

- **Exclusive：** 表示线程以独占的方式等待锁。

一般来说，自定义同步器的共享方式要么是独占，要么是共享，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但 AQS 也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。

****

## 独占锁ReentrantLock原理

ReentrantLock 是可重入的独占锁 ， 同时只能有一个线程可以获取该锁，其他获取该锁的线程会被阻塞而被放入该锁的 AQS阻塞队列里面。并且根据参数来决定其内部是一个公平还是非公平锁，默认是非公平锁。

> AQS 的 state 状态值表示**线程获取该锁的可重入次数** ， 在默认情况下， state 的值为 0表示当前锁没有被任何线程持有 。 当一个线程第一次获取该锁时会尝试使用 CAS 设置 state 的值为 1，如果 CAS 成功则当前线程获取了该锁，然后记录该锁的持有者为当前线程 。 在该线程没有释放锁的情况下第 二 次获取该锁后 ，状态值被设置为 2， 这就是可重入次数 。 在该线程释放该锁时，会尝试使用 CAS 让状态值减 1， 如果减1后状态值为 0, 则当前线程释放该锁 。

### 获取锁

下面分析`lock()`方法的源码，ReentrantLock 的 lock()委托给了 sync 类，根据创建 ReentrantLock 构 造函数选择 sync 的实现是 NonfairSync 还是 FairSync，这个锁是一个非公平锁或者公平锁 。

> ```java
> final void lock() {
>   // 1. CAS设置状态值
>   if (compareAndSetState(0, 1))
>     setExclusiveOwnerThread(Thread.currentThread());
>   else 
>     // 2. 调用AQS的acquire方法
>     acquire(1);
> }
> ```
>
> 因为默认 AQS 的状态值为 0，所以第一个调用 Lock 的线程会通过 CAS 设置状态值为 1, CAS 成功则 表示当前线程获取 到了锁， 然后 setExclusiveOwnerThread 设置该锁持有者是当前线程 。
>
> 如果这时候有其他线程调用 lock 方法企图获取该 锁， CAS 会失败，然后会调用 AQS 的 acquire 方法 。注意 ，传递参数为 1。而AQS的`acquire`方法会调用ReentrantLock自己实现的`tryAcquire`方法，如果返回false，则将线程加入AQS阻塞队列。
>
> ****
>
> 下面是非公平锁的`tryAcquire`方法：
>
> ```java
> protected final boolean tryAcquire(int acquires) {
>   return nonfairTryAcquire(acquires);
> }
> 
> final boolean nonfairTryAcquire(int acquires) {
>   final Thread current = Thread.currentThread();
>   int c = getState();
>   // 4. 当前AQS状态值为0
>   if (c == 0) {
>     if (compareAndSetState(0, acquires)) {
>       setExclusiveOwnerThread(current);
>       return true;
>     }
> 	}
>   // 5. 当前线程是该锁的持有者
>   else if (current == getExclusiveOwnerThread()) {
>     int nextc = c + acquires;
>     if (nextc < 0)  // overflow
>       throw new Error("Maximum lock count exceeded");
>     setState(nextc);
>     return true;
>   }
>   return false;
> }
> ```
>
> 首先代码4会查看当前锁的状态值是否为 0，为 0 则说明当前该锁空闲，那么就 尝试 CAS 获取该锁，将 AQS 的状态值从 0 设置为 1，并设置当前锁的持有者为当前线程然后返回true。如果当前状态值不为 0 则说明该锁已经被某个线程持有，所以代码5 查看当前线程是否是该锁的持有者，如果当前线程是该锁的持有者，则状态值加 1，然后返回 true， 这里需要注意， nextc<0 说明可重入次数溢出了。 如果当前线程不是锁的持有 者则返回 false，然后其会被放入 AQS 阻塞队列。
>
> ***
>
> 下面看看公平锁的`tryAcquire`方法：
>
> ```java
> protected final boolean tryAcquire(int acquires) {
>   final Thread current = Thread.currentThread();
>   int c = getState();
>   // 4. 当前AQS状态值为0
>   if (c == 0) {
>     // 6. 公平策略
>     if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
>       setExclusiveOwnerThread(current);
>       return true;
>     }
> 	}
>   // 5. 当前线程是该锁的持有者
>   else if (current == getExclusiveOwnerThread()) {
>     int nextc = c + acquires;
>     if (nextc < 0)  // overflow
>       throw new Error("Maximum lock count exceeded");
>     setState(nextc);
>     return true;
>   }
>   return false;
> }
> 
> ```
>
> 公平锁不同之处在于设置CAS之前**判断了当前线程节点中是否有前驱节点**。

****

### 释放锁

首先会尝试释放锁，如果当前线程持有该锁， 则调用该方法会让该线程对该线程持有的 AQS 状态值减 1， 如果减去1后当前状态值为 0，则当前线程会释放该锁 ， 否则仅仅减1而 己。下面是`unlock()`方法：

```java
public void unlock() {
  sync.release(1);
}

protected final boolean trtRelease(int releases) {
  // 1. 如果不是锁持有者调用unlock,则会抛出异常
  int c = getState() - releases;
  if (Thread.currentThread() != getExclusiveOwnerThread()) {
    throw new IllegalMonitorStateException();
   boolean free = false;
   // 2. 如果当前可重入次数为0，则清空锁持有线程
   if (c == 0) {
     free = true;
     setExclusiveOwnerThread(null);
   }
    // 3. 设置可重入次数为原始值-1
    setState(c);
    return free;
  }
}
```

****

### 生产则消费者模型

#### synchronized实现

消费者应该遵循的原则：

1. 获取对象的锁
2. 如果条件不满足，那么调用对象的wait()方法，被通知后仍要检查条件。
3. 条件满足则执行对应的逻辑。

生产者应该遵循的原则：

1. 获得对象的锁。
2. 改变条件
3. 通知所有等待在对象上的线程。

伪代码如下：

```java
// 消费者伪代码
synchronized(对象) {
  while (条件不满足) {
    // 对象.wait();
  }
  // 对应处理逻辑
}

// 生产者伪代码
synchronized(对象) {
  // 1. 改变条件
  // 2. 对象.notifyAll()
}
```

> 注意：判断条件不满足必须用`while`，而不能用`if`，用`if`会出现**虚假唤醒**的情况。

****

#### Lock版实现

```java
public class Demo {
    //共享变量
    private List<String> list = new ArrayList<>(10);
    //互斥量
    private ReentrantLock lock = new ReentrantLock();
    //条件变量[不为空]。若不符合条件，则加入到条件变量到Condition queue中
    private Condition notEmptyCondition = lock.newCondition();
    //条件变量[不为满]。若不符合条件，则加入到条件变量到Condition queue中
    private Condition notFullCondition = lock.newCondition();
    /**
     * 生产者模型
     */
    public void producer() {
      	lock.lock();
        try {
            //若list满了
            while (list.size() == 10) {
                try {
                    //线程进入，则将线程放入到不为full的条件变量中
                    //禁止再次生产。
                    notFullCondition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            list.add("元素");
            //唤醒生产者
            notEmptyCondition.signal();
        } finally {
            lock.unlock();
        }
    }

    /**
     * 消费者模型
     */
    public void consumer() {
      	lock.lock();
        try {
            //若list为空
            while (list.size() == 0) {
                try {
                    //将该线程放入到条件变量中，该条件变量为不为空
                    //禁止再次消费
                    notEmptyCondition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            list.remove(0);
            notFullCondition.signal();
        } finally {
            lock.unlock();
        }
    }
}
```

****

### synchronized与ReentrantLock的区别

- **两者都是可重入锁**
- **synchronized 依赖于 JVM 而 ReentrantLock 依赖于 API**

`synchronized` 是依赖于 JVM 实现的；`ReentrantLock` 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成）。

- **ReentrantLock相比synchronized增加了一些高级功能**
    - **等待可中断** : `ReentrantLock`提供了一种能够中断等待锁的线程的机制，通过 `lock.lockInterruptibly()` 来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。
    - **可实现公平锁** : `ReentrantLock`可以指定是公平锁还是非公平锁。而`synchronized`只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。`ReentrantLock`默认情况是非公平的，可以通过 `ReentrantLock`类的`ReentrantLock(boolean fair)`构造方法来制定是否是公平的。
    - **可实现选择性通知（锁可以绑定多个条件）**: `synchronized`关键字与`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。`ReentrantLock`类当然也可以实现，但是需要借助于`Condition`接口与`newCondition()`方法。
- Synchronized依赖于JVM，因此**即使出现异常，JVM也能保证锁的正确释放**；Lock需要确保在finally块中释放锁，否则一旦同步代码块发生异常，就可能永远不会释放锁。

****

## ReentrantReadWriteLock原理

ReentrantReadWriteLock采用读写分离的策略，允许多个线程可以同时获取读锁 。**使用AQS中 state 的高 16 位表示读状态，也就是获取到读锁的次数;使用低 16 位表示获取到写锁 的线程的可重入次数 。**

### 写锁的获取与释放

写锁的获取源代码如下：

```java
protected final boolean tryAcquire(int acquires) {
  Thread current = Thread.currentThread();
  int c = getState();
  // 写锁的可重入个数
  int w = exclusiveCount(c);
  // 1. c != 0说明读锁或者写锁已经被某线程获取
  if (c != 0) {
    // 2. w=0说明已经有线程获取了读锁，w!=0并且当前线程不是写锁拥有者，则返回false
    if (w == 0 || current != getExclusiveOwnerThread())
      return false;
    // 3. 说明当前线程获取了写锁，判断可重入次数
    if (w + exclusiveCount(acquires) > MAX_COUNT)
      throw new Error("Maximum lock count exceeded");
    // 4. 设置可重入次数
    setState(c + acquires);
    return true;
  }
  // 5. 第一个写线程获取写锁
  if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
    return false;
  setExclusiveOwnerThread(current);
  return true;
}
```

非公平锁`writerShouldBlock()`方法总是返回false，说明代码5抢占式执行CAS尝试获取写锁；公平锁`writerShouldBlock()`方法会判断是否有前驱节点。

写锁的释放过程为：尝试释放锁，如果当前线程持有该锁，调用该方法会让该线程对该线程持有的 AQS 状态值减 1，如果减去 1 后当前状态值为 0则当前线程会释放该锁， 否则仅仅减 1 而己。如果当前线程没有持有该锁而调用了该方法则会抛出 IllegalMonitorStateException异常。

****

****

## Java线程池

### 为什么要用线程池？

> 池化技术在编程中应用非常广，线程池、数据库连接池、Http 连接池等等都是对这个思想的应用。池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。

线程池是一种基于池化思想管理线程的工具，线程池可以带来如下的好处：

- **降低资源消耗**：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
- **提高响应速度**：任务到达时，无需等待线程创建即可立即执行。
- **提高线程的可管理性**：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- **提供更多更强大的功能**：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

****

线程池主要解决了以下的问题：

1. 频繁申请/销毁资源和调度资源，将带来额外的消耗，可能会非常巨大。
2. 对资源无限申请缺少抑制手段，易引发系统资源耗尽的风险。
3. 系统无法合理管理内部的资源分布，会降低系统的稳定性。

****

### 线程池的核心设计与实现

线程池在内部实际上构建了一个**生产者消费者模型，将线程和任务两者解耦**，并不直接关联，从而良好的缓冲任务，复用线程。

线程池的运行主要分成两部分：**任务管理、线程管理**。

- 任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：（1）直接申请线程执行该任务；（2）缓冲到队列中等待线程执行；（3）拒绝该任务。
- 线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

![image-20230224184609571](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230224184609571.png)

接下来，我们会按照以下三个部分去详细讲解线程池运行机制：

1. 线程池如何维护自身状态。
2. 线程池如何管理任务。
3. 线程池如何管理线程。

****

#### 生命周期管理(线程池如何维护自身状态)

线程池内部使用一个变量维护两个值：运行状态(runState)和线程数量 (workerCount)。**使用的是 AtomicInteger 类型变量，其中高 3 位用来表示线程池状态，后面 29 位用来记录线程池线程个数 。**

> 用一个变量去存储两个值，可**避免在做相关决策时，出现不一致的情况**，不必为了维护两者的一致，而占用锁资源。

ThreadPollExecutor的运行状态有5种，分别是：

![image-20230224185622498](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230224185622498.png)

其生命周期转换如下入所示：

![image-20230224185645353](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230224185645353.png)

****

#### 任务执行机制(线程池如何管理任务)

- **任务调度**

任务调度是线程池的主要入口，当用户提交了一个任务，接下来**这个任务将如何执行都是由这个阶段决定的。**

首先，所有任务的调度都是由execute方法完成的，这部分完成的工作是：**检查现在线程池的运行状态、运行线程数、运行策略，决定接下来执行的流程，是直接申请线程执行，或是缓冲到队列中执行，亦或是直接拒绝该任务。** 其执行过程如下：

> 1. 首先检测线程池运行状态，如果不是RUNNING，则直接拒绝，线程池要保证在RUNNING的状态下执行任务。
> 2. 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
> 3. 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
> 4. 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
> 5. 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

![image-20230224190333701](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230224190333701.png)

****

- **任务缓冲**

任务缓冲模块是线程池能够管理任务的核心部分。线程池中是以生产者消费者模式，**通过一个阻塞队列来实现的。阻塞队列缓存任务，工作线程从阻塞队列中获取任务。**

> 阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作是：在队列为空时，获取元素的线程会等待队列变为非空。当队列满时，存储元素的线程会等待队列可用。

****

- **任务申请**

任务的执行有两种可能：一种是**任务直接由新创建的线程执行**。 另一种是**线程从任务队列中获取任务然后执行，执行完任务的空闲线程会再次去从队列中申请任务再去执行。**

工作线程Worker会不断接收新任务去执行，如果`task`为`null`，就会调用`getTask()`方法获取任务，即上述的第二种可能。执行流程如下：

![image-20230224191931752](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230224191931752.png)

****

- **任务拒绝**

任务拒绝模块是线程池的保护部分，线程池有一个最大的容量，当线程池的任务缓存队列已满，并且线程池中的线程数目达到maximumPoolSize时，就需要拒绝掉该任务，采取任务拒绝策略，保护线程池。

****

#### 线程池如何管理线程

线程池设计了线程池内的工作线程Worker。Worker这个工作线程，**实现了Runnable接口，继承了AQS**。并持有一个线程thread，一个初始化的任务firstTask。

> thread是在调用构造方法时通过ThreadFactory来创建的线程，可以用来执行任务；firstTask用它来保存传入的第一个任务，这个任务可以有也可以为null。如果这个值是非空的，那么线程就会在启动初期立即执行这个任务，也就对应核心线程创建时的情况；如果这个值是null，那么就需要创建一个线程去执行任务列表（workQueue）中的任务，也就是非核心线程的创建。
>
> ****
>
> **Worker继承了AQS，通过AQS实现不可重入的的特性去反应线程现在的执行状态。**
>
> 1. lock方法一旦获取了独占锁，表示当前线程正在执行任务中。 
> 2. 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断。

****

**线程池使用一张Hash表去持有线程的引用，这样可以通过添加引用、移除引用这样的操作来控制线程的生命周期。**

> **线程池中线程的销毁依赖JVM自动的回收**，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。
>
> Worker被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当Worker无法获取到任务，也就是获取的任务为空时，循环会结束，Worker会主动消除自身在线程池内的引用。

****

### 线程池的创建

线程池的创建有两种方式，一是通过`Executors`工具类，另外是直接通过`ThreadPoolExecutor`类的构造器进行创建。

#### Executors工具类创建

里面提供了很多静态方法，能够根据用户选择返回不同的线程池实例。常见的线程池实例有如下三种：**FixedThreadPool、SingleThreadExecutor和CacheThreadPool。**

> Executors工具类创建的线程池实例**本质也是调用`ThreadPoolExecutor`类中的构造方法进行创建。**
>
> 阿里巴巴Java开发手册强制：线程池不允许使用`Executors`去创建，而是通过`ThreadPoolExecutor`的方式，因为`Executors`返回的线程池对象有如下弊端：
>
> 1. `FixedThreadPool`和`SingleThreadExecutor`:
>
> 这两种线程池实例的任务队列都是基于链表的无界队列，允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。
>
> 2. `CacheThreadPool`：
>
> 允许创建线程的数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。

****

#### 使用ThreadPoolExecutor方式创建

`SingleThreadExecutor`类有四种构造方法，允许传入参数最多的构造方法如下：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
  if (corePoolSize < 0 ||
      maximumPoolSize <= 0 ||
      maximumPoolSize < corePoolSize ||
      keepAliveTime < 0)
    throw new IllegalArgumentException();
  if (workQueue == null || threadFactory == null || handler == null)
    throw new NullPointerException();
  this.corePoolSize = corePoolSize;
  this.maximumPoolSize = maximumPoolSize;
  this.workQueue = workQueue;
  this.keepAliveTime = unit.toNanos(keepAliveTime);
  this.threadFactory = threadFactory;
  this.handler = handler;
}
```

7个线程池参数如下：

- **`corePoolSize` :线程池核心线程个数**，定义了最小可以同时运行的线程数量。

- **`maximumPoolSize`：线程池最大线程数量**，当队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。

- **`keepAliveTime`：存活时间** ，当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁。
- **unit：存活时间的单位**。
- **workQueue：任务队列**，当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

> 常见用于任务队列的有：基于数组的有界`ArrayBlockingQueue`、基于链表的无界 `LinkedBlockingQueue`、最多只有一个元素的同步队列 `SynchronousQueue` 及优先级队列 `PriorityBlockingQueue` 等。

- **threadFactory：创建线程的工厂，** 一般默认即可。
- **handler：饱和策略，** 当队列满并且线程个数达到`maximunPoolSize`后采取的策略，共有四种饱和策略。

> 如果当前同时运行的线程数量达到最大线程数量并且队列也已经被放满了任务时，`ThreadPoolTaskExecutor` 定义四种饱和策略：
>
> - **`ThreadPoolExecutor.AbortPolicy`** ：抛出 `RejectedExecutionException`来拒绝新任务的处理。这时线程池的默认拒绝策略，推荐比较关键的业务使用该策略，能够及时通过异常发现。
> - **`ThreadPoolExecutor.CallerRunsPolicy`** ：调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
> - **`ThreadPoolExecutor.DiscardPolicy`** ：不处理新任务，直接丢弃掉。建议一些无关紧要的业务使用该策略。
> - **`ThreadPoolExecutor.DiscardOldestPolicy`** ： 此策略将丢弃队列最前面的任务，然后重新提交被拒绝的任务。

****

![image-20230219143450101](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2023-02/image-20230219143450101.png)

****

### 常用方法对比

- **shutdown()和shutdownNow()**

调用 shutdown方法后，线程池就不会再接受新的任务了，但是工作队列里面的任务还是要执行的 。该方法会立刻返回，并不等待 队列任务完成再返回 。

调用 shutdownNow方法后， 线程池就不会再接受新的任务了，并且会丢弃工作队列 里面的任务， 正在执行的任务会被中断， 该方法会立刻返回，并不等待激活的任务执行完 成。返回值为这 时候队列里面被丢弃的任务列表。

- **execute()和submit()**

`execute()`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否。

`submit()`方法用于提交需要返回值的任务。线程池会返回一个 `Future` 类型的对象，通过这个 `Future` 对象可以判断任务是否执行成功，并且可以通过 `Future` 的 `get()`方法来获取返回值，`get()`方法会阻塞当前线程直到任务完成，而使用 `get（long timeout，TimeUnit unit）`方法的话，如果在 `timeout` 时间内任务还没有执行完，就会抛出 `java.util.concurrent.TimeoutException`。

****

### 线程池大小确定

- **CPU 密集型任务(N+1)：** 这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1，比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
- **I/O 密集型任务(2N)：** 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

