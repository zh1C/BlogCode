---
author: "Narcissus"
title: "Golang面试题"
date: "2022-02-24"
description: "Go语言面试题总结。"
tags: ["Golang", "面试"]
categories: ["Golang"]


---

## 1、nil切片和空切片

### 问题

![ScreenShot2021-08-31 10.08.07](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032331.png)

nil切片和空切片指向的地址一样吗？代码会输出什么？

### 回答

- **nil切片和空切片指向的地址不一样。nil空切片引用数组指针地址为0（无指向任何实际地址）**
- **空切片的引用数组指针地址是有的，且固定为一个值**

![ScreenShot2021-08-31 10.15.01](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032332.png)

两个空切片指向的数组地址一样。

### 解释

切片的数据结构为如下：

![ScreenShot2021-08-31 10.18.08](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032333.png)

- nil切片和空切片最大的区别在于**指向的数组引用地址是不一样的。**

![ScreenShot2021-08-31 10.20.25](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032334.png)

- **所有的空切片指向的数组引用地址都是一样的。**

![ScreenShot2021-08-31 10.21.40](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032335.png)

- len,cap和append等功能在nil切片上同样可以正常工作。

![IMG_95E147EAF13C-1](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032337.jpg)

如果非常在意性能和内存使用情况，初始化一个空切片可能不如使用nil切片理想。

## 2、字符串转成byte数组，会发生内存拷贝吗？

### 回答

字符串转成切片，会产生拷贝。严格来说，只要是发生类型强转都会产生内存拷贝。

有没有什么办法可以在字符串转成切片的时候不用发生拷贝呢？

### 解释

- **StringHeader是字符串在go底层结构**

![ScreenShot2021-08-31 10.43.06](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032339.png)

- **SliceHeader是切片在go底层结构**

![ScreenShot2021-08-31 10.43.17](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/032339-1.png)

可以发现StringHeader与SliceHeader的结构体内存是对齐的，除了SliceHeader最后多了一个Cap字段。

### 高效的[]byte转string

> **unsafe.Pointer**
>
> Pointer类型用于表示任意类型的指针。任意类型的指针可以转换为一个Pointer类型值，一个Pointer类型值可以转换为任意类型的指针。

**直接转换指针类型即可，忽略cap字段**。

![image-20220225173122810](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225173122810.png)

### 高效的string转[]byte

**注意：string高效转[]byte虽然依然可以直接转，但是会出现没有生成cap字段，对读取操作没有影响，但如果要append元素可能会出问题**：

> 因为这样做的cap属性是随机的，可能是大于len的值，那么append时就不会新开辟一段内存存放元素，而是在原数组后面追加，如果后面的内存不可写就会panic

因此转换时需要想办法构建出cap字段。

方式一：

![image-20220225173502992](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225173502992.png)

方式二：

**unsafe.Pointer, int, uintpr这三种类型占用内存大小相同**

![image-20220225182143376](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225182143376.png)

因此从底层结构上看，string可以看做[2]uintptr，[]byte类型可以看做[3]uintptr。

![image-20220225182305106](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225182305106.png)

代码如下：

![image-20220225182326189](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225182326189.png)

## 3、new和make有什么区别？

### 基本特性

- **make**

在Go中，内置函数make仅支持`slice、map、channel`三种数据类型的内存创建，**其返回值是所创建类型的本身，而不是新的指针引用**。

函数签名为：`func make(t Type, size ...IntegerType) Type`

- **new**

内置函数new可以对类型进行内存创建和初始化。**其返回值是所创建类型的指针引用**,函数签名为：`func new(Type) *Type`

> 注意：new分配内存的同时会把分配的内存置为零，也就是类型的零值。

![image-20220219134945366](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220219134945366.png)

上述代码会报错，因为**对于引用类型的变量，我们不仅要声明它，还需要为其分配内存空间**，否则值放到哪里去呢？

因此，可以使用new来分配内存空间：

![image-20220219135159687](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220219135159687.png)

但实际代码中，我们不常用new这个内置函数，我们通常都是采用短语句声明以及结构体的字面量达到我们的目的，比如：

![image-20220219135705377](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220219135705377.png)

### 区别

- make

1. 能够分配并初始化类型所需的内存空间和结构，返回引用类型的本身。
2. 具有使用范围的局限性，仅支持channel、map、slice三种类型。
3. 具有独特的优势，会对三种结构的内部数据结构（长度，容量等）赋值。

- new

1. 能够分配并初始化类型所需的内存空间，返回指针应用(指向内存的指针)。
2. 可被替代，能够通过字面值快速初始化。

## 4、结构体和结构体指针调用有什么区别？

如下例子：

```go
type MyStruct struct {
    Name string
}

func (s MyStruct) SetName1(name string) {
    s.Name = name
}

func (s *MyStruct) SetName2(name string) {
    s.Name = name
}
```

该程序声明了一个`User`结构体，其包含两个结构体方法，分别是`SetName1`和`SetName2`方法，两者之间的差异就是**引用的方式不同**。

### 两者区别

当在一个类型上定义一个方法时，接收器的行为就像它是方法的一个参数一样。相当于：

```go
func SetName1(s MyStruct, name string){
    u.Name = name
 }

 func SetName2(s *MyStruct,name string){
    u.Name = name
 }
```

因此结构体方法将接收器定义成值还是指针。本质上与函数参数应该是值还是指针是同一个问题。

### 如何选择：

考虑因素有如下，按重要程度排序：

- 在使用上的考虑：方法是否需要修改接收器？如果需要，接收器必须是一个指针。
- 在效率上的考虑：如果接收器很大，比如一个大的结构，使用指针接收器会好很多。
- 在一致性上的考虑：如果类型的某些方法必须有指针接收器，那么其余的方法也应该有指针接收器，所以无论类型如何使用，方法集都是一致的。

## 5、Go的map、slice非线性安全？

### 5.1 非线性安全的例子

在Go中，map与slice不支持并发读写，也就是非线性安全的，如下两个并发操作的例子：

```go
func main() {
 var s []string
 for i := 0; i < 9999; i++ {
  go func() {
   s = append(s, "脑子进煎鱼了")
  }()
 }

 fmt.Printf("进了 %d 只煎鱼", len(s))
}
```

输出结果：

![image-20220224205634739](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220224205634739.png)

产生这样的原因是因为并发模式下，产生了覆盖写入情况。

同样针对map，也使用这种方式：

```go
func main() {
 s := make(map[string]string)
 for i := 0; i < 99; i++ {
  go func() {
   s["煎鱼"] = "吸鱼"
  }()
 }

 fmt.Printf("进了 %d 只煎鱼", len(s))
}
```

输出结果：

![image-20220224205956322](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220224205956322.png)

程序直接报错。并发写 map 会导致 `fatal error: concurrent map writes` 错误提示。

### 5.2 如何支持并发读写

- #### Map+Mutex 

Go官方提供了一种简单又便利的方式来实现并发读写：

```go
var counter = struct{
    sync.RWMutex
    m map[string]int
}{m: make(map[string]int)}
```

**这条语句声明了一个变量，它是一个匿名结构体，包含一个原生map和一个嵌入式读写锁**。

从变量中读出数据，则调用读锁：

![image-20220224210440475](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220224210440475.png)

从变量中写数据，则调用写锁：

![image-20220224210509208](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220224210509208.png)

- #### sync.Map

虽然有Map+Mutex方案，但如果map的数据量非常大时，只有一把锁，**一把锁会导致大量的争夺锁，导致各种冲突和性能低下**。

Go语言的`sync.Map`支持并发读写Map,**采用空间换时间的机制，冗余了两个数据结构，分别是:read和dirty，减少加锁对性能的影响，适合读多写少的场景**。

![image-20220224210910249](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220224210910249.png)

**若出现写多/并发多的场景，会导致 read map 缓存失效，需要加锁，冲突变多，性能急剧下降。这是他的重大缺点。**

### 5.3 为什么不支持？

Go Slice 的话，主要还是索引位覆写问题，这个就不需要纠结了，势必是程序逻辑在编写上有明显缺陷，自行改之就好。

Go Map为什么不支持并发读写，原因如下：

- 典型使用场景：map 的典型使用场景是不需要从多个 goroutine 中进行安全访问。
- 非典型场景（需要原子操作）：map 可能是一些更大的数据结构或已经同步的计算的一部分。
- 性能场景考虑：若是只是为少数程序增加安全性，导致 map 所有的操作都要处理 mutex，将会降低大多数程序的性能。

汇总来讲，就是 Go 官方在经过了长时间的讨论后，认为 Go map 更应适配典型使用场景，而不是为了小部分情况，导致大部分程序付出代价（性能），决定了不支持。

## 6、单核 CPU，开两个 Goroutine，其中一个死循环，会怎么样？

针对这个问题，需要把问题拆开，其具有以下几个元素：

- 运行Go程序的计算机只有一个单核CPU。

- 两个Goroutine在运行。
- 一个Goroutine死循环。

#### 单核CPU

单核CPU对GMP模型中影响最大的就是P， 因为P的数量默认与CPU核数(GOMAXPROCS)保持一致。

#### Goroutine受限

两个Goroutine，一个死循环，一个正常运行。可以理解为 Main Goroutine + 起了一个新 Goroutine 跑着死循环。

需要注意，Goroutine 里跑着死循环，也就是时时刻刻在运行着 “业务逻辑”。这块需要与单核 CPU 关联起来，**考虑是否会一直阻塞住，把整个 Go 进程运行给 hang 住了**？面试时需要问清楚。

#### Go版本问题

针对不同的Go版本，结果可能不一样

#### 实战演练

构造如下例子：

```go
// Main Goroutine 
func main() {
    // 模拟单核 CPU
    runtime.GOMAXPROCS(1)
    
    // 模拟 Goroutine 死循环
    go func() {
        for {
        }
    }()

    time.Sleep(time.Millisecond)
    fmt.Println("脑子进煎鱼了")
}
```

答案是：

- 在 Go1.14 前，不会输出任何结果。
- 在 Go1.14 及之后，能够正常输出结果。

在Go1.14前，这段程序是有一个 Goroutine 是正在执行死循环，也就是说他肯定无法被抢占。即主程序永远没有机会被调度，因此永远无法执行完毕。

在Go1.14版本及之后版本，会正常输出。因为**在 Go1.14 实现了基于信号的抢占式调度**，以此来解决上述一些仍然无法被抢占解决的场景。相应方法会检测符合场景的 P，当满足下述两个场景之一时，就会发送信号给 M。M 收到信号后将会休眠正在阻塞的 Goroutine，调用绑定的信号方法，并进行重新调度。以此来解决这个问题。

> 1.抢占阻塞在系统调用上的 P。
>
> 2.抢占运行时间过长的 G。

## 7、Golang常用的并发模型

控制并发有三种经典的方式，一种是通过channel通知实现并发控制，一种是WaitGroup，另外一种是Context。

### 7.1 使用channel通知实现并发控制

**无缓冲通道**，指的是通道大小为0，也就是说，这种类型的通道在接收前没有能力保存任何值，**要求发送`goroutine`和接收`goroutine`同时准备好**，才可以完成发送和接收操作。因此发送与接受的goroutine必须是同步的，如果没有同时准备好的话，先执行的操作就会阻塞等待，直到另一个相对应的操作准备好为止。这种无缓冲的通道我们也称之为同步通道。

![image-20220225140618506](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225140618506.png)

### 7.2 通过sync.WaitGroup实现并发控制

WaitGroup用于等待一组线程的结束。父线程调用Add方法来设定应等待的线程的数量。每个被等待的线程在结束时应调用Done方法。同时，主线程里可以调用Wait方法阻塞至所有线程结束。

**注意：在`WaitGroup`第一次使用后，不能被拷贝**，如下代码则会报死锁：

![image-20220225141213285](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225141213285.png)

这是因为`wg`给拷贝传递到了`goroutine`中，导致只有Add操作，其实Done操作都是在`wg`的副本执行的，因此Wait就死锁了。

- 改进方法1：将匿名函数中`wg`的传入类型改为`*sync.WaitGroup`。
- 改进方法2：将匿名函数中的`wg`的传入参数去掉，因为Go支持闭包操作，匿名函数中可以直接使用外面的`wg`变量。

### 7.3 Context上下文，实现并发控制

Go将一个程序的运行环境、现场和快照等封装在一个`Context`里，再将它传给要执行的`goroutine`。`context`包主要用来处理多个`goroutine`之间共享数据，及多个`goroutine`的管理。

## 8、进程、线程、协程、goroutine区别

### 8.1 概念理解

- **进程**

进程是具有一定独立功能的程序关于某个数据集合上的一次运行活动,进程是系统进行资源分配和调度的一个独立单位。每个进程都有自己的独立内存空间，拥有自己独立的堆和栈，既不共享堆，亦不共享栈，进程由操作系统调度。不同进程通过进程间通信来通信。由于进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全。

- **线程**

线程是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位.线程自己基本上不拥有系统资源,而拥有自己独立的栈和共享的堆，共享堆，不共享栈，线程也由操作系统调度(标准线程是这样的)。只拥有一点在运行中必不可少的资源(如程序计数器,一组寄存器和栈),但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源。线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据。

- **协程**

 **协程是一种用户态的轻量级线程**，协程的调度完全由用户控制。协程和线程一样共享堆，不共享栈，协程由程序员在协程的代码里显示调度。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。

### 8.2 理解区分

- **进程、线程(内核级线程)、协程(用户级线程)之间区别**

对于 进程、线程，都是有内核进行调度，有 CPU 时间片的概念，进行 **抢占式调度**。

对于 **协程**(用户级线程)，这是对内核透明的，也就是系统并不知道有协程的存在，是完全由用户自己的程序进行调度的，因此很难做到抢占式调度那样强制CPU切换，通常只能进行**协作式调度**。

- **goroutine和协程的区别**

**本质上，goroutine就是协程**，不同的是，Golang在runtime、系统调用等多方面对goroutine调度进行了封装和处理。协程只能进行协作式调度，而Goroutine是基于信号的抢占式调度。

## 9、Golang有几种锁

有Mutex和RWMutex。然后可以介绍两种锁的底层。

## 10、高效拼接字符串

Go语言中，字符串是只读的，意味着每次修改操作都会创建一个新的字符串。**如果需要拼接多次，应该使用`strings.Builder`，最小化内存拷贝次数**。

![image-20220225195236803](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225195236803.png)

## 11、init()函数什么时候执行？

同一个包，甚至是同一个源文件可以有多个 `init()` 函数。`init()` 函数没有入参和返回值，不能被其他函数调用，同一个包内多个 `init()` 函数的执行顺序不作保证。

一句话总结： import –> const –> var –> `init()` –> `main()`。

## 12、Go语言局部变量分配在栈上还是堆上？

由编译器决定。Go语言编译器会自动决定把一个变量放到栈还是放到堆，编译器会做逃逸分析，**当发现变量作用域没有超出函数范围，就可以在栈上，反之必须分配在堆上。**

![image-20220225200055087](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225200055087.png)

> `foo()`函数中，如果v分配在栈上，foo函数返回时，&v就不存在了，但上述代码可以正常运行，main函数中仍然能够正常访问该值，因为Go 编译器发现 v 的引用脱离了 foo 的作用域，会将其分配在堆上。

> **注意**：函数返回局部变量的指针是安全的，因为Go编译器会对每个局部变量做逃逸分析。如果发现局部变量作用域超出函数，则不会将内存分配到栈上而分配到堆上。

## 13、什么是协程泄露？

协程泄露是指协程创建后，长时间得不到释放，并且还在不断地创建新的协程，最终导致内存耗尽，程序崩溃。常见的导致协程泄露的场景有以下几种：

- **缺少接收器，导致发送阻塞**

![image-20220225201541046](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225201541046.png)

如上例子：每执行一次query，则启动1000个协程向信道ch发送数字0，但只接收了一次，导致999个协程被阻塞，不能退出。

- **缺少发送器，导致接收阻塞**

同样的，如果启动 1000 个协程接收信道的信息，但信道并不会发送那么多次的信息，也会导致接收协程被阻塞，不能退出。

- **死锁**

两个或两个以上的协程在执行过程中，由于竞争资源或者由于彼此通信而造成阻塞，这种情况下，也会导致协程被阻塞，不能退出。

- **无线循环**

为了避免网络等问题，采用了无限重试的方式，发送 HTTP 请求，直到获取到数据。那如果 HTTP 服务宕机，永远不可达，导致协程不能退出，发生泄漏。

## 14、为什么Go语言函数能返回多个参数值？

### C语言为什么只能返回一个参数值？

C语言中函数调用时，参数都是通过寄存器和栈传递的：

- 六个即六个以下的参数会按顺序分别使用edi、esi、edx、dcx、r8d和r9d六个寄存器传递；
- 六个以上的参数会使用栈传递，函数的参数会以从右到左的顺序依次存入栈中。

C语言函数返回值通过eax寄存器进行传递，**由于只使用一个寄存器存储返回值，所以C语言的函数不能同时返回多个值**。

### Go语言函数调用惯例

如下一个简单的片段代码：

![image-20220323093936689](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220323093936689.png)

可以使用`go tool compile -S -N -l file`命令编译上述代码获得汇编指令。

> 注意：如果不使用-N -l参数，编译器会对汇编代码进行优化，编译结果会有较大差别。

**通过分析Go语言编译后的汇编指令，我们发现Go语言使用栈传递参数和接收返回值，所以它只需要你在栈上多分配一些内存就可以返回多个值**。

### 对比

C语言同时使用寄存器和栈传递参数，使用eax寄存器传递返回值；Go语言使用栈传递参数和返回值。

- C语言的方式能够极大的减少函数调用的额外开销，但增加了实现的复杂度：
  - CPU访问栈的开销比访问寄存器高几十倍，
  - 需要单独处理函数参数过多的情况。
- Go语言的方式能够降低实现的复杂度并支持多返回值，但是牺牲了函数调用的性能：
  - 不需要考虑超过寄存器数量的参数应该如何传递；
  - 不需要考虑不同架构上的寄存器差异；
  - 函数入参和出参的内存空间需要在栈上进行分配。

