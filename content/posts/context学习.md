---
author: "Narcissus"
title: "Context学习"
date: "2022-09-19"
lastmod: "2022-09-19"
description: "Golang Context包相关学习"
tags: ["Golang"]
categories: ["Golang"]
password: ""

---

## 一、什么是Context

Go1.7标准库引入Context，译为“上下文”，准确说是goroutine的上下文，包含goroutine的运行状态、环境、现场等信息。**主要用来在goroutine之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v等** 。

## 二、为什么要有Context

Go常用于写后台服务，通常每一个请求都会启动若干个goroutine同时工作：有些拿数据，有些调用下游接口获取相关数据......

![image-20220919170721414](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919170721414.png)

这些goroutine需要共享这个请求的基本数据，例如登录的token，处理请求的最大超时时间（如果超过此值再返回数据，请求方因为超时接收不到）等等。当请求被取消或者处理时间太长（可能是使用者关闭了浏览器或者超过了请求方规定的超时时间），请求方直接放弃了这次请求结果。这时，所有为这个请求工作的goroutine需要快速退出，因为他们的工作不再被需要。相关goroutine退出后，系统就可以回收资源。

context包就是为了解决上述问题而开发的：**在一组goroutine之间传递共享的值、取消信号、deadline**......

![image-20220919170833903](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919170833903.png)

一句话概述：**context用来解决goroutine之间退出通知、元数据传递的功能**。

另外，像 `WithCancel`、`WithDeadline`、`WithTimeout`、`WithValue` 这些创建函数，实际上是创建了一个个的链表结点而已。我们知道，对链表的操作，通常都是 `O(n)` 复杂度的，效率不高。

那么，context 包到底解决了什么问题呢？答案是：`cancelation`。仅管它并不完美，但它确实很简洁地解决了问题。

## 三、如何使用Context

context使用非常方便，源码对外提供了一个创建根节点context的函数：`func Background() Context` 。这是一个空的context，它不能被取消，没有值，也没有超时时间。

有了根节点，又提供四个函数创建子节点context：

![image-20220919172928684](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919172928684.png)

context会在函数传递间传值。只需要在适当的时间调动cancel函数向goroutine发出取消信号或者调用Value函数取出context中的值。

官方博客有如下几点使用建议：

- 不要将Context塞到结构体中。直接将Context类型作为参数的第一参数，并且一般命名为ctx。

- 不要向函数传入一个nil的context，如果实在不知道传什么，标准库给你准备好了一个context：todo。

- 不要把本应该作为函数参数的类型塞到context中，context存储的应该是一些共同的数据。例如：登录的session、cookie等。

- 同一个context可能会被传递到多个gorourine，别担心，**context是并发安全的**。

### 3.1 传递共享的数据

对于web服务端开发，往往希望将一个请求处理的整个过程串起来，这就非常依赖于Thread Local（对于Go可理解为单个协程所独有）的变量，而Go语言没有这个概念，因此需要在函数调用的时候传递context。

![image-20220919173712130](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919173712130.png)

### 3.2 取消goroutine

有如下一个场景：打开外卖的订单页，地图上显示外卖小哥的位置，而且是每秒更新 1 次。app 端向后台发起 websocket 连接（现实中可能是轮询）请求后，后台启动一个协程，每隔 1 秒计算 1 次小哥的位置，并发送给端。如果用户退出此页面，则后台需要“取消”此过程，退出 goroutine，系统回收资源。

后端可能实现如下：

![image-20220919181506624](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919181506624.png)

> 如果需要实现“取消”功能，并且在不了解 context 功能的前提下，可能会这样做：给函数增加一个指针型的 bool 变量，在 for 语句的开始处判断 bool 变量是发由 true 变为 false，如果改变，则退出循环。

优雅的方式则是使用Context:

![image-20220919181722984](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919181722984.png)

**注意到WithTimeout函数返回的context和cancelFun是分开的，context本身没有取消函树，这样做的原因是取消函数只能由外层调用，防止子节点context调用取消函数，从而严格控制信息流向：从父节点流向子节点。**

### 3.3 防止goroutine泄露

如下例子，如果不用context取消，goroutine则会泄露：

![image-20220919182427693](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919182427693.png)

这是一个可以生成无限整数序列的协程，但如果只需要前五个序列，就会发生goroutine泄露。

![image-20220919182658308](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919182658308.png)

使用context改进的例子如下：

![image-20220919183456847](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919183456847.png)

## 四、Context底层实现原理

### 4.1 整体概述

context代码包并不长，总共500多行

![image-20220919171132847](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919171132847.png)

### 4.2 接口

#### Context

![image-20220919171526528](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919171526528.png)

`Context` 是一个接口，定义了四个方法，都是**幂等**的。也就是说连续多次调用同一个方法，得到的结果是相同的。

1. `Done()` 返回一个channel，可以表示context被取消的信号：**当这个channel被关闭时，说明context被取消了** 。

1. > 注意：这是一个只读的channel。我们知道，读一个关闭的channel会读出相应类型的零值。并且源码没有地方会向这个channel里面写入值。换句话说在子协程里读这个channel，除非被关闭，否则读不出任何东西。**当子协程从channel里面读出了零值后，就可以做一些收尾工作，尽快退出。**

2. `Err()` 返回一个错误，表示channel被关闭的原因。例如被取消还是超时。

3. `Deadline()` 返回context的截止时间，通过此时间，函数就可以决定是否进行接下来的操作，如果时间太短，就可以不往下做了，否则浪费系统资源。当然也可以用这个deadline来设置一个I/O操作的超时时间。

4. `Value()` 获取之前设置的key对应的value。

#### canceler

![image-20220919172126522](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919172126522.png)

实现了上面定义的两个方法的Context，就表明该Context是可取消的。源码中有两个类型实现了canceler接口：**`*cannelCtx`** **和`*timerCtx`** 。注意：加*表明是这两个结构体的指针实现了canceler接口。

Context接口设计成这个样子的原因：

- **“取消”操作应该是建议性，而非强制性**

caller不应该去关心、干涉callee的情况，决定如何以及何时return是callee的责任。caller只需发送“取消”信息，callee根据收到的信息来做进一步的决策，因此接口没有定义cancel方法。

- **“取消”操作应该可传递**

“取消”某个函数时，和它相关联的其它函数也应该“取消”。因此，`Done()` 方法返回一个只读的channel，所有相关函数监听此channel。一旦channel关闭，通过channel的“广播机制”，所有监听者都能收到。

### 4.3 结构体

#### emptyCtx

源代码中定义了Context后，给出了一个实现：
![image-20220919172648769](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919172648769.png)

每个函数的实现都非常简单，要么直接返回，要么返回nil。**这实际上是一个空的context，永远不会被cancel，没有存储值，也没有deadline。**它被包装成：

![image-20220919183847776](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919183847776.png)

通过下面两个导出函数对外公开：
![image-20220919183925308](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919183925308.png)

**background通常在main函数中，作为所有context的根节点。**

**todo通常用在并不知道传递什么context的情形。** 例如，调用一个需要传递context参数的函数，手头并没有其他context可以传递，就可以传递todo。这常常发生在重构进行中，给一些函数添加一个Context参数，但不知道传递什么，可以暂时用todo占一个位置。

#### cancelCtx

这是一个重要的context:
![image-20220919184031743](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919184031743.png)

这是一个可以取消的Context，实现了canceler接口。它直接将接口Context作为它的一个匿名字段，这样它就可以被看成一个Context。

- Done()方法的实现

![image-20220919184327050](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919184327050.png)

> c.done是“懒汉式”创建，只有调用了Done()方法的时候才会被创建。函数返回的是一个只读的channel，而且没有地方向这个channel里面写数据。**所以直接调用读这个channel，协程会被block住，一般通过搭配select来使用。一旦关闭，就会立即读出零值。**

- Err()方法和String()方法实现简单，推荐看源码
- cancel()方法实现

![image-20220919184922404](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919184922404.png)

> `cancel()` 方法的**功能就是关闭channel：c.done；递归地取消它的所有子节点；从父节点删除自己。**达到的效果是通过关闭channel，将取消信号传递给它的所有子节点。goroutine接收到取消信号的方式就是select语句中读c.done被选中。

- 创建一个可取消的context

![image-20220919185613461](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919185613461.png)

> 这是一个暴露给用户的方法，传入一个父Context(通常是一个`background` ,作为根节点)，返回新建的context。当`WithCancel` 函数返回的CancelFunc被调用或者父节点的done channel被关闭(父节点的CancelFunc被调用),此context的done channel也会被关闭。

> 注意传给WithCancel方法的参数，前者是true表示取消的时候，需要将自己从父节点里面删除。第二个参数则是一个固定的取消错误类型：`var Canceled = errors.New("context canceles")` 。



#### valueCtx

![image-20220919190658836](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919190658836.png)

它实现了两个方法：

![image-20220919190734093](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919190734093.png)

> 由于它直接将 Context 作为匿名字段，因此仅管它只实现了 2 个方法，其他方法继承自父 context。但它仍然是一个 Context，这是 Go 语言的一个特点。

创建valueCtx的函数：

![image-20220919190912350](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919190912350.png)

> 对 key 的要求是可比较，因为之后需要通过 key 取出 context 中的值，可比较是必须的。

通过层层传递context，最终形成一棵树：

![image-20220919191149401](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220919191149401.png)

和链表有点像，只是它的方向相反：Context 指向它的父节点，链表则指向下一个节点。通过 WithValue 函数，可以创建层层的 valueCtx，存储 goroutine 间可以共享的变量。

取值的过程，实际上是一个递归查找的过程：

```go
func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}

func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
		case *emptyCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```

> 它会顺着链路一直往上找，比较当前节点的 key 是否是要找的 key，如果是，则直接返回 value。否则，一直顺着 context 往前，最终找到根节点（一般是 emptyCtx），直接返回一个 nil。所以用 Value 方法的时候要判断结果是否为 nil。

**因为查找方向是往上走的，所以，父节点没法获取子节点存储的值，子节点却可以获取父节点的值。**

`WithValue` 创建 context 节点的过程实际上就是创建链表节点的过程。两个节点的 key 值是可以相等的，但它们是两个不同的 context 节点。查找的时候，会向上查找到最后一个挂载的 context 节点，也就是离得比较近的一个父节点 context。所以，整体上而言，用 `WithValue` 构造的其实是一个低效率的链表。

