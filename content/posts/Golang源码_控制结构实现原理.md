---
author: "Narcissus"
title: "Golang源码之常见控制结构实现原理"
date: "2022-07-16"
lastmod: "2022-07-16"
description: "Go语言专家编程书籍阅读笔记，常见控制结构的源码阅读。"
tags: ["Golang"]
categories: ["Golang"]
---

## Defer

### 1. 前言

defer语句用于延迟函数的调用，每次defer都会把一个函数压入栈中，函数返回前再把延迟的函数取出并执行。

### 2. defer规则

#### 2.1 规则一：延迟函数的参数在defer语句出现时就已经确定下来了

如下的例子：

![image-20211201200755232](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201200755232.png)

defer语句中的`fmt.Println(i)`的参数i在defer出现时就已经确定下来了，实际上是拷贝了一份。所以打印”0“。

> 注意：对于指针类型参数，规则仍然适用，只不过延迟函数的参数是一个地址，这种情况下，defer后面的语句可能会影响延迟函数。

#### 2.2 规则二：延迟函数执行按后进先出顺序执行，即先出现的defer最后执行

这个很好理解，因为定义defer类似于入栈，执行defer类似于出栈。

#### 2.3 规则三：延迟函数可能操作主函数的具名返回值

定义defer的函数，即主函数可能有返回值，返回值有没有名字没有关系，defer所作用的函数，即延迟函数可能会影响到返回值。

##### 2.3.1 函数返回过程

**关键字return不是一个原子操作，实际上只代理汇编指令ret**。比如语句`return i`,实际上分为两步，将i值存入栈中作为返回值，然后执行跳转，而defer的执行时机在跳转之前。

如下例子：

![image-20211201201812696](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201201812696.png)

该函数return语句可以拆分成下面两行：

![image-20211201201902325](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201201902325.png)

而延迟函数正是在return之前，加入defer后执行过程如下：

![image-20211201201944816](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201201944816.png)

##### 2.3.2 不同情况分析

- 主函数拥有匿名返回值，返回字面值

![image-20211201202358084](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201202358084.png)

没有影响返回1.

- 主函数拥有匿名返回值，返回变量

![image-20211201202438741](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201202438741.png)

没有影响，返回0

- 主函数拥有具名返回值

![image-20211201202518514](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201202518514.png)

有影响。

### 3. defer实现原理

defer数据结构定义在`src/runtime/runtime2.go:_defer`中：

![image-20211205180729282](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211205180729282.png)

结构体中的`link`指针用于指向另一个defer，每个goroutine数据结构中实际也有一个defer指针，指向一个defer单链表，每次声明一个defer时就将defer插入到单链表表头，执行时从表头取出执行。

![image-20211205181608529](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211205181608529.png)

一个goroutine可能连续调用多个函数，defer添加过程跟上述流程一致，进入函数时添加defer，离开函数时取出 defer。

## Select

select是Golang在语言层面提供的多路IO复用的机制，可以检测多个channel是否ready（即是否可读或可写）。

### 使用Tips

- select语句中除default外，每个case操作一个channel，要么读要么写
- select语句除default外，各case执行顺序是随机的
- select语句中如果没有default语句，则会阻塞等待任一case
- select语句中读操作要判断是否成功读取，关闭的channel也可以读取

热手题目：

下列程序会发生什么？

![image-20220213193853489](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213193853489.png)

**对于空的select语句，程序会被阻塞，准确的说是当前协程被阻塞，同时golang自带死锁检测机制，当发现当前协程再也没有机会被唤醒时，会panic。所以上述程序会panic**。

## Range

range是Golang提供的一种迭代遍历手段，可操作的类型有数组、切片、Map、channel等。

### 1. 实现原理

range对于不同类型，细节上有些差异

#### 1.1 range for slice

遍历slice过程：

![image-20220213195230771](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213195230771.png)

遍历slice前**会先获取以slice的长度len_temp作为循环次数**，循环体中，每次循环会先获取元素值，如果for-range中接收index和value的话，则**会对index和value进行一次赋值**。

**注意：由于循环开始前循环次数已经确定，所以循环过程中添加新元素是没办法遍历到的**。

#### 1.2 range for map

![image-20220213195828444](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213195828444.png)

遍历map时没有指定循环次数，循环体与遍历slice类似。由于map底层使用hash表实现，插入位置随机，所以遍历过程中新插入的数据不能保证遍历到。

#### 1.3 range for channel

![image-20220213200016146](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213200016146.png)

channel遍历是依次从channel中读取数据，读取前不知道有多少数据。**如果channel中没有元素，则会阻塞等待，如果channel被关闭，则会解除阻塞退出循环**。

**注意：用for-range 遍历channel时只能获取一个返回值**。

### 2. 总结

- 遍历过程中可以视情况放弃接收index或value，可以一定程度上提升性能
- 遍历channel时，如果channel中没有数据，可能会阻塞
- 使用index、value接收range返回值会发生一次数据拷贝

## Mutex

互斥锁是并发程序中对共享资源进行访问控制的主要手段，对此Go语言提供了非常简单易用的Mutex，Mutex为一结构体类型，对外暴露两个方法Lock()和Unlock()分别用于加锁和解锁。

### 1. Mutex数据结构

![image-20220213201015448](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213201015448.png)

- Mutex.state表示互斥锁状态，比如是否被锁定等
- Mutex.sema表示信号量，协程阻塞等待该信号量，解锁的协程释放信号量从而唤醒等待信号量的协程。

Mutex.state内部把该变量分成4份，用于记录Mutex四种状态

![image-20220213201313128](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213201313128.png)

- Locked：表示Mutex是否被锁定，0表示没有锁定；1表示已被锁定。
- Woken：表示是否有协程已被唤醒，0表示没有协程唤醒；1表示已有协程唤醒，正在加锁过程中，则此时解锁协程就不必释放信号量了。
- Starving：表示Mutex是否处于饥饿状态，0表示没有饥饿；1表示饥饿状态，说明协程阻塞超过了1ms。
- Waiter：表示阻塞等待解锁的协程个数，协程解锁时根据此值来判断是否需要释放信号量。

### 2. 加解锁过程

#### 2.1 加锁

如果只有一个协程在加锁，则加锁成功后，只是Locked位置1，其他状态为不会发生变化。

如果加锁时，锁被其他协程占用，则如下图：

![image-20220213202400325](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213202400325.png)

可以看到，当协程B对一个已经被占用的锁再次加锁时，Waiter计数器增加1，此时协程B被阻塞，直到Locked值变为0才会被唤醒。

#### 2.2 解锁

如果解锁时，没有其他协程阻塞，则只需要把Locked位置为0即可，不需要释放信号量。

如果有协程阻塞，过程如下：

![image-20220213202656878](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220213202656878.png)

协程A解锁过程分为两个步骤，一是把Locked位置0，二是查看到Waiter>0，所以释放一个信号量，唤醒一个阻塞的协程，被唤醒的协程B把Locked位置1，于是协程B获得锁。

### 3. 自旋过程

加锁时，如果当前Locked位为1，说明该锁当前由其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会持续的探测Locked位是否变为0，这个过程即为**自旋过程**。
自旋时间很短，但如果在自旋过程中发现锁已被释放，那么协程可以立即获取锁。此时即便有协程被唤醒也无法获取锁，只能再次阻塞。

**自旋的好处是，当加锁失败时不必立即转入阻塞，有一定机会获取到锁，这样可以避免协程的切换**。

#### 3.1 自旋条件

无限制的自旋会给CPU带来巨大压力，因此判断是否可以自旋就很重要。自旋必须满足以下条件：

- 自旋次数要足够小，通常为4，即最多自旋4次。
- CPU核数要大于1，否则自旋没有意义，因为此时不可能有其他协程释放锁。
- 协程调度机制中process数量要大于1。
- 协程调度机制中可运行队列必须为空，否则会延迟协程调度。

#### 3.2 自旋的优势

自旋的优势是更充分的利用CPU，尽量避免协程切换。因为当前申请加锁的协程拥有CPU，如果经过短时间的自旋可以 获得锁，当前协程可以继续运行，不必进入阻塞状态。

#### 3.3 自旋的缺点

如果自旋过程中获得锁，那么之前被阻塞的协程将无法获得锁，如果加锁的协程特别多，每次都通过自旋获得锁，那 么之前被阻塞的进程将很难获得锁，从而进入饥饿状态。**饥饿状态下不会自旋**。

### 4. Mutex模式

每个Mutex都有两个模式，称为Normal和Starving。

#### 4.1 normal模式

默认情况下，Mutex的模式，该模式下，协程加锁不成功有可能会启动自旋。

#### 4.2 Starving模式

自旋过程中能抢到锁，一定意味着同一时刻有协程释放了锁，我们知道释放锁时如果发现有阻塞等待的协程，还会释 放一个信号量来唤醒一个等待协程，被唤醒的协程得到CPU后开始运行，此时发现锁已被抢占了，自己只好再次阻塞， 不过阻塞前会判断自上次阻塞到本次阻塞经过了多长时间，如果**超过1ms的话，会将Mutex标记为”饥饿”模式，然后再阻塞**。

### 5. 为什么重复解锁要panic？

Unlock过程分为将Locked置为0，然后判断Waiter值， 如果值>0，则释放信号量。
**如果多次Unlock()，那么可能每次都释放一个信号量，这样会唤醒多个协程，多个协程唤醒后会继续在Lock()的逻辑里抢锁，势必会增加Lock()实现的复杂度，也会引起不必要的协程切换。**

## RWMutex

读写互斥锁，可以说是Mutex的一个改进版，**在读取数据频率远远大于写数据频率的场景下可以发挥更加灵活的控制能力**。

**规则是写锁与写锁、读锁都互斥；读锁与读锁间不互斥**。

### 1. 方法

RWMutex提供了4个对外的方法：

- RLock():读锁定
- RUnlock():解除读锁定
- Lock():写锁定，与MUtex完全一致
- Unlock():解除写锁定，与MUtex完全一致

### 2. 相关问题解答

#### 2.1 写操作如何阻止写操作

读写锁包含一个互斥锁，写锁定必须先获取该互斥锁。

#### 2.2 写操作如何阻止读操作

RWMutex.readerCount用于表示读者数量，是一个整型值，每次读锁定将该值加1，解除读锁该值减1。最大可支持2^30个并发读者。

**当写锁定进行时，会先将readerCount减去2^30，从而readerCount变成了负值，此时再有读锁定到来时检测到 readerCount为负值，便知道有写操作在进行，只好阻塞等待。而真实的读操作个数并不会丢失，只需要将 readerCount加上2^30即可获得。**

#### 2.3 读操作如何阻塞写操作

读锁定会先将RWMutext.readerCount加1，此时写操作到来时发现读者数量不为0，会阻塞等待所有读操作结束。

#### 2.4 为什么写锁不会被饿死

写操作获取到锁后要等待之前的读操作结束后才可以继续执行，写操作休眠等待期间可能还有新的读操作持续到来，如果写操作休眠等待所有读操作结束，很可能被饿死永远不会被唤醒。

**写操作到来时，会把RWMutex.readerCount值拷贝到RWMutex.readerWait中，用于标记排在写操作前面的读者 个数。前面的读操作结束后，除了会递减RWMutex.readerCount，还会递减RWMutex.readerWait值，当 RWMutex.readerWait值变为0时唤醒写操作。**
