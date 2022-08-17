---
author: "Narcissus"
title: "Golang源码之常见数据结构实现原理上"
date: "2021-11-30"
description: "Go语言专家编程书籍阅读笔记，常见数据结构的源码阅读，包括channel、slice、map。"
tags: ["Golang"]
categories: ["Golang"]
---

## Channel

### 1. 前言

channel是Golang提供的goroutine间通信方式，主要用于进程内各goroutine间通信。源代码位于`runtime/chan.go`中。

### 2. chan数据结构

`src/runtime/chan.go:hchan`中定义了channel的数据结构：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2014.47.26.png)

#### 2.1 环形队列

chan内部实现了一个环形队列作为其缓冲区，队列长度是创建chan时指定。如下图：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2014.50.40.png)

- dataqsiz指示了队列长度为6，即可缓存6个元素
- qcount表示队列中还有2个元素
- sendx表示后续写入的数据存储的位置，取值[0, 6)
- recvx表示从该位置读取数据，取值[0, 6)

#### 2.2 等待队列

从channel读数据，如果channel缓冲区为空或者没有缓存区；向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine均会被阻塞，被挂在channel的等待队列中。

- 因读阻塞的goroutine会被向channel写入数据的gorooutine唤醒；
- 因写阻塞的goroutine会被从channel读出数据的gorooutine唤醒；

下图展示了一个没有缓冲区的channel，有几个因读阻塞的goroutine：

![ScreenShot2021-11-30 15.01.56](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2015.01.56.png)

> 注意：一般情况下，recvq和sendq至少有一个为空。但有一个例外就是同一个goroutine使用select语句向channel一边写一边读。

#### 2.3 锁

一个channel同时仅允许被一个goroutine读写，因此有锁。

### 3. channel读写

#### 3.1 创建channel

创建channel的过程实际上就是初始化`hchan`结构，由`runtime/chan.go:makechan()`函数实现，类型信息和缓冲区长度由make语句传入，`buf`的大小由元素类型和缓冲区长度共同决定。伪代码如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2015.09.01.png)

#### 3.2 向channel写数据

向一个channel中写数据过程如下：

1. 如果等待接收队列`recvq`不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从`recvq`中取出G，并把数据写入唤醒的G，结束写过程；
2. 如果缓冲区有空余位置，将数据写入缓冲区，结束写过程；
3. 如果缓冲区没有空余位置，将待写数据写入G，并将当前G加入`sendq`，进入睡眠；等待读goroutine唤醒；

流程图如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2015.22.25.png)

#### 3.3 从channel读数据

从一个channel读数据过程如下：

1. 如果等待发送队列`sendq`不为空并且没有缓冲区，直接取出G，把数据读出并唤醒G，结束读过程；
2. 如果等待发送队列`sendq`不为空，说明缓冲区已满，先从缓冲区首部读出数据，然后把G中数据写入缓冲区尾部并唤醒G,结束读过程；
3. 如果缓冲区有数据，从缓冲区读取数据，解说读过程；
4. 将当前读的goroutine加入`recvq`，进入睡眠，等待被写goroutine唤醒。

流程图如下：

![ScreenShot2021-11-30 15.29.03](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2015.29.03.png)

#### 3.4 关闭channel

关闭channel时会把`recvq`中的G全部唤醒，本该写入G的数据位置为nil；把`sendq`中的G全部唤醒，但这些G会panic。

除此之外，还会panic的场景有：

- 关闭值为nil的channel
- 关闭已经关闭的channel
- 向已经关闭的channel写数据

### 常见用法

#### 4.1 单向channel

单向channel指只能用于发送或接收数据，实际上没有单向channel，只是对channel的一种使用限制。

简单例子：

![ScreenShot2021-11-30 15.54.00](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2015.54.00.png)

mychan是一个正常的channel，`func readChan(chanName <-chan int)`通过形参限定函数内部只能从channel中读取数据；`func writeChan(chanName chan<- int)`通过形参限定函数内部只能向channel中写入数据。

#### 4.2 select

使用select可以监控多个channel，例如监控多个channel，当某一个channel有数据时，就从其读出数据，代码如下：

```go
func addNumberToChan(chanName chan int) {
	for {
		chanName <- 1
		time.Sleep(1 * time.Second)
	}
}

func main() {
	chan1 := make(chan int, 10)
	chan2 := make(chan int, 10)
	go addNumberToChan(chan1)
	go addNumberToChan(chan2)

	for {
		select {
		case e := <- chan1:
			fmt.Printf("Get element from chan1:%d\n", e)
		case e := <- chan2:
			fmt.Printf("Get element from chan2:%d\n", e)
		default:
			fmt.Printf("No element from chan1 and chan2\n")
			time.Sleep(1 * time.Second)
		}
	}
}
```

代码结果如下：

![ScreenShot2021-11-30 16.06.30](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2016.06.30.png)

可以看出，select语句的多个case执行顺序是随机的。

> 通过示例可知:**select的case语句读channel不会阻塞，尽管channel中没有数据。这时由于case语句编译后调用读channel时会明确传入不阻塞的参数，此时读不到数据不会将当前goroutine加入等待队列，而是直接返回**。

#### 4.3 range

通过range可以持续从channel中读取数据，好像在遍历一个数组一样，**当channel中没有数据时会阻塞当前goroutine，与读channel时阻塞处理机制一样**。

![ScreenShot2021-11-30 16.14.49](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2016.14.49.png)

> 注意：如果此时channel写数据的goroutine退出时，系统检测到这种情况会panic，否则range将会永久阻塞。

## Slice

### 1. 前言

Slice又称动态数组，依托数组实现，可以方便进行扩容、传递等，实际使用中比数组更灵活。

### 2. 实现原理

Slice依托数组实现，底层数组对用户屏蔽，在底层数组容量不足时可以自动重分配并生成新的Slice。源代码在`src/runtime/slice.go`中。

#### 2.1 Slice数据结构

源码包中`src/runtime/slice.go:slice`定义了Slice的数据结构：

![ScreenShot2021-11-30 16.34.23](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2016.34.23.png)

array指针指向底层数组，len表示切片长度，cap表示**底层数组容量**。

#### 2.2 使用make创建Slice

使用make创建Slice时，可以同时指定长度和容量，创建时底层会分配一个数组，**数组的长度即容量**。例如，`slice := make([]int, 5, 10)`结构如下图：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2016.37.37.png)

该Slice长度为5，即可以使用下标slice[0]~slice[4]来操作里面的元素。容量为10，表示后续向Slice添加新元素时可以不必重新分配内存，直接使用 预留内存即可。

> 注意：若不指定容量，则表示容量等于长度。

#### 2.3 使用数组创建Slice

使用数组来创建Slice时，Slice将与原数组共用一部分内存。例如，`slice := array[5:7]`如下图所示：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2016.44.21.png)

> 注意：原数组后面的内容都作为切片的预留内存，数组和切片操作都可能作用于同一块内存。

#### 2.4 Slice扩容

使用append向Slice追加元素时，如果Slice空间不足，将会触发扩容。实质是重新分配一块更大的内存，将原Slice中元素拷贝进新Slice中，然后再追加新元素。如下图：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2016.49.11.png)

扩容操作只关心容量，会把原Slice数据拷贝到新Slice中，追加数据由append在扩容结束后完成。上图可以看到，扩容后长度仍然是5。

> 扩容容量选择规则如下：
>
> - 如果原Slice容量小于1024，则新Slice容量扩大为原来的2倍。
> - 如果原Slice容量大于或等于1024，则新Slice容量扩大为原来的1.25倍。
>
> **但最新有小小修改**:
>
> 1. `threshold = 256`,门槛由1024变成了256；
> 2. `newcap += (newcap + 3*threshold) / 4`,而不是之前的`newcap += newcap / 4`

程序代码片段如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2017.16.47.png)

#### 2.5 Slice copy

使用内置函数copy()拷贝两个切片时，会将源切片数据逐个拷贝到目标切片的数组中，**拷贝数量取两个切片长度的最小值**。即长度为10的切片拷贝到长度为5的切片，将会拷贝5个元素。

> copy过程中不会发生扩容

#### 2.6 特殊切片

根据数组或切片生成新切片一般使用`slice := array[start:end]`方式，这种方式没有指定切片容量，**实际上新切片容量就是从start开始直至array的结束**。

例如下面的例子，长度和容量都是一致，共用内存：

![ScreenShot2021-11-30 17.31.00](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2017.31.00.png)

但还有另一种写法，即切片同时也指定容量，`slice[start:end:cap]`，其中`cap`即为新切片的容量，**当然容量不能超过原切片实际值(会做语法检测，编译不通过)**，如下：

![ScreenShot2021-11-30 17.33.03](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2017.33.03.png)

### 3. 编程Tips

- 创建切片时可根据实际需求预分配容量，尽量避免追加过程中扩容操作，利于提升性能；
- 切片拷贝时需要判断实际拷贝的元素个数；
- 谨慎使用多个切片操作同一数组，以防读写冲突。

#### 题目

下列程序输出什么？

![ScreenShot2021-11-30 17.41.44](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-30%2017.41.44.png)

参考答案：append函数执行时会判断切片容量是否能够存放新增元素，如果不能，则会重新申请存储空间，新存储空间将是原来的2倍或1.25倍（取决于扩展原空间大小），本例中实际执行了两次append操作，第一次空间增长到4， 所以第二次append不会再扩容，所以新旧两个切片将共用一块存储空间。程序会输出true。

## Map

map的设计被称作”The dictionary problem“, 它的任务是设计一种数据结构用来维护一个集合的数据，并且可以对集合进行增删查改操作。最主要的数据结构有两种：**哈希查找表、搜索树**。

- 哈希查找表

用一个哈希函数将key分配到不同的桶（bucket,也就是数组不同的index）。一般会存在”碰撞“问题（即不同的key被哈希到同一个bucket）。一般有两种应对方法：**链表法和开放地址法**。

> 链表法将一个bucket实现成一个链表，落在同一个bucket中的key都会插入这个链表。
>
> 开放地址法则是发生碰撞后，通过一定的规律，在数组后面挑选空位，用来存放新的key。

- 搜索树

搜索树一般采用自平衡搜索树，包括：AVL树，红黑树。

**Go语言采用的是哈希查找表，并且使用链表解决哈希冲突**。

### 1. map数据结构

源码位于`src/runtime/map.go`中，表示map的结构体是`hmap`：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2015.02.23.png)

buckets是一个指针，指向bucket结构体的数组，bucket结构体`bmap`如下：

![ScreenShot2021-12-01 15.07.08](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2015.07.08.png)

> 注意：
>
> 1. 结构体中只显示定义了tophash,data和overflow并不是在结构体中显示定义的，而是直接通过指针运算进行访问的。
> 2. tophash是个长度为8的数组，哈希值相同的键（准确的说是哈希值低位相同的键）存入当前bucket时会将哈希值的高位存储在该数组中，以方便后续匹配
> 3. data区存放的是key-value数据，存放顺序是key/key/key/…value/value/value，如此存放**是为了节省字节对齐带来的空间浪费**。 
> 4. 每个bucket最多只能存放8个key-value对，overflow指针指向的是下一个bucket，据此将所有冲突的键连接起来。

如下图展示了存放8个键值对的示意图：

![ScreenShot2021-12-01 15.15.02](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2015.15.02.png)



### 2. 哈希冲突

当有两个或以上数量的键被哈希到同一个bucket时，就称发生了冲突。由于每个bucket可以存放8个键值对，所以同一个bucket存放超过8个键值对时就会再创建一个bucket，通过overflow指针连接起来。

**因为Go的bucket可能存放8个键值对，所以能容忍更高的负载因子(`负载因子=键数量/bucket数量`)，在负载因子达到6.5时才会触发rehash**。

### 3. key定位过程

key经过哈希计算后得到哈希值，共64个bit位（对于64位机），计算它到底落在哪个桶，只会用到最后B个bit位。如果B=5，那么桶数量，也就是buckets数组长度就是2^5=32。

例如，有一个key经过哈希后得到如下结果：

![ScreenShot2021-12-01 15.37.12](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2015.37.12.png)

用最后5个bit位，也就是`01010`，值为10，就是10号桶。再用哈希值高8位，找到此key在bucket中的位置。

下面具体说明定位过程，如图：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2015.39.27.png)

> 上图中，B=5,所以buckets总数是32.首先计算出key的哈希值，用最低5位`00110`，确定6号桶。使用高8位`10010111`，对应十进制151，在6号bucket中寻找`tophash`值（HOB hash）为151的key,找到了2号槽位，然后去keys数组的对应位置查找到key2，如果该key2与key相同，则返回value2，结束。
>
> 如果在bucket中没有找到，并且overflow不为空，则还要继续去overflow bucket中寻找，最后没找到**则返回对应值类型的0值**。

### 4. 渐进式扩容

#### 4.1 扩容的前提条件

为了保证访问效率，当新元素将要添加进map时，都会检查是否需要扩容，扩容实际上是以空间换时间的手段。扩容的条件有如下两个：

- 装载因子超过阈值，阈值定义为6.5。
- overflow的bucket数量过多，当B 小于15（也就是bucket总数小于2^15时），如果overflow的bucket数量超过2^B;当B 大于等于15（也就是bucket总数大于等于2^15时），如果overflow的bucket数量超过2^15。

#### 4.2 增量式扩容

增量式扩容主要用于元素太多而bucket太少的情况。一次性搬迁非常影响性能，**Go每次最多搬迁2个bucket，搬迁前会分配新的buckets(长度是原来的2倍),并将老的buckets挂到oldbuckets字段上，后续插入、删除、修改key的时候，都会尝试搬迁工作，直到搬迁完成（即oldbuckets为nil）**。

由于新的buckets是老的buckets的2倍，所以可以按序号来搬，比如原来在0号buckets，到新的地方后，仍然放到0号buckets。

#### 4.3 等量扩容

在overflow的bucket数量增多，但负载因子又不高的情况下，但大多数overflow的buckets是空的，则需要进行等量扩容。**即buckets数量不变， 经过重新组织后overflow的bucket数量会减少，即节省了空间又会提高访问效率**。

对于这种情况，就需要重新计算key的哈希值，称为`rehash`。比如B原来为5，扩容后B变成了6,就需要多看一位。

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2016.18.11.png)

> **为什么遍历map是无序的？**
>
> map在扩容后，会发生key的搬迁，原来落在同一个bucket中的key，搬迁后，某些key就会远走高飞了。而遍历过程是按顺序遍历bucket，再按顺序遍历bucket中的key。搬迁后key位置发生了重大变化，遍历结果就不可能有序了。
>
> **但有一个hard code的map，并且不会有插入删除操作，按道理每次遍历结果应该有序呢？**
>
> 的确是这样，但Go杜绝了这种情况，go1.0加入了”迭代map的结果是无须的“这个特性。遍历map时，并不是从固定的0号bucket开始遍历，每次都从一个随机值序号开始遍历。

### 5. map进阶

#### 可以边遍历边删除吗？

**map并不是一个线程安全的数据结构**。同时读写一个map是未定义的行为，被检测到会直接panic。可以通过读写锁`sync.RWMutex`来解决。另外，`sync.Map`是线程安全的map,可以使用。

#### key可以是float型吗？

语法上是可以的。Go语言中只要是可比较的类型都可以作为key。

> **除开slice、map、function，其他类型都可以**。具体包括：布尔值、数字、字符串、指针、通道、接口类型、结构体、只包含上述类型的数组。结构体需要它们的字段值都相等才会被认为是相同的key。
>
> **任何类型都可以作为value,包括map类型**。

float型可以作为key，但由于精度的问题，会导致一些诡异的问题。

