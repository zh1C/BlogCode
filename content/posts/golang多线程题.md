---
author: "Narcissus"
title: "Golang多线程题目"
date: "2022-02-25"
Lastmode: "2022-02-25"
description: "Go语言多线程相关的LeetCode题目。"
tags: ["Golang", "面试"]
categories: ["Golang"]
---

一般使用多线程的题目都会用到无缓冲区的Channel来进行同步，channel具有以下特性：

- 给一个nil channel发送数据，造成永远阻塞。
- 给一个nil channel接收数据，造成永远阻塞。
- 给一个已经关闭的channel发送数据，引起panic。
- 从一个已经关闭的channel接收数据，如果缓冲区中为空，则返回一个零值。
- 无缓冲区的channel是同步的，有缓冲区的channel是非同步的

可以通过口诀记忆，**空读写阻塞，写关闭异常，读关闭空零**。

## LeetCode1114 按序打印

请设计程序，以确保 `second()` 方法在 `first()` 方法之后被执行，`third()` 方法在 `second()` 方法之后被执行。三个方法都是多线程下的方法。代码如下：

```go

var chan1 = make(chan int)
var chan2 = make(chan int)


func first() {
	fmt.Println("1")
	chan1 <- 1
}

func second() {
	<- chan1
	fmt.Println("2")
	chan2 <- 1
}

func third() {
	<- chan2
	fmt.Println("3")
}

func main() {
	go first()
	go second()
	go third()

	// 防止其他线程还没结束，主线程就结束了
	time.Sleep(time.Second)
}
```

## LeetCode1195 交替打印字符串

![image-20220225092837602](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225092837602.png)

![image-20220225092852400](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225092852400.png)

代码如下：

```go
var wg sync.WaitGroup

type FizzBuzz struct {
	n int
	numChan chan struct{}
	fizzChan chan struct{}
	buzzChan chan struct{}
	fizzBuzzChan chan struct{}
	q chan struct{}
}

func (f *FizzBuzz) fizz() {
	defer wg.Done()
	for {
		select {
		case <- f.fizzChan:
			fmt.Println("fizz")
			f.numChan <- struct{}{}
		case <- f.q:
				return
		}
	}
}

func (f *FizzBuzz) buzz() {
	defer wg.Done()
	for {
		select {
		case <- f.buzzChan:
			fmt.Println("buzz")
			f.numChan <- struct{}{}
		case <- f.q:
			return
		}
	}
}

func (f *FizzBuzz) fizzBuzz() {
	defer wg.Done()
	for {
		select {
		case <- f.fizzBuzzChan:
			fmt.Println("fizzBuzz")
			f.numChan <- struct{}{}
		case <- f.q:
			return
		}
	}
}

func (f *FizzBuzz) number() {
	defer wg.Done()
	for i := 1; i <= f.n; i++ {
		<- f.numChan
		if i%3 == 0 && i%5 == 0 {
			f.fizzBuzzChan <- struct{}{}
			continue
		}
		if i%3 == 0 {
			f.fizzChan <- struct{}{}
			continue
		}
		if i%5 == 0 {
			f.buzzChan <- struct{}{}
			continue
		}
		fmt.Println(i)
		f.numChan <- struct{}{}
	}
	f.q <- struct{}{}
	f.q <- struct{}{}
	f.q <- struct{}{}
}

func NewFizzBuzz(n int) *FizzBuzz {
	return &FizzBuzz{
		n: n,
		// 使用了一个缓冲区，避免下轮循环死锁
		numChan: make(chan struct{}, 1),
		fizzChan: make(chan struct{}),
		buzzChan: make(chan struct{}),
		fizzBuzzChan: make(chan struct{}),
		// 用来通知其他 三个线程退出死循环，避免泄露
		q: make(chan struct{}, 3),
	}
}

func main() {
	fb := NewFizzBuzz(15)

	wg.Add(4)

	go fb.fizz()
	go fb.buzz()
	go fb.fizzBuzz()
	go fb.number()
	fb.numChan <- struct{}{}

	wg.Wait()
}
```

> 注意，为了防止主线程提前退出，其他4个线程还没有结束，使用了`sync.WaitGroup`，因为使用`time.Sleep()`无法设置准确时间。故使用`sync.WaitGroup`更合理。
>
> **WaitGroup用于等待一组线程的结束。父线程调用Add方法来设定应等待的线程的数量。每个被等待的线程在结束时应调用Done方法。同时，主线程里可以调用Wait方法阻塞至所有线程结束。**

- q管道的作用是当number线程结束后，会通知另外三个线程退出死循环，所以设置了3个缓冲区。

## LeetCode1115 交替打印FooBar

![image-20220225100410323](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225100410323.png)

代码如下：

```go
var wg sync.WaitGroup

// 加一个缓冲区，避免死循环
var chanFoo = make(chan struct{}, 1)
var chanBar = make(chan struct{})

func foo(n int) {
	defer wg.Done()
	for i := 0; i < n; i++ {
		<- chanFoo
		fmt.Printf("foo")
		chanBar <- struct{}{}
	}
}

func bar(n int) {
	defer wg.Done()
	for i := 0; i < n; i++ {
		<- chanBar
		fmt.Printf("bar\n")
		chanFoo <- struct{}{}
	}
}

func main() {
	wg.Add(2)

	n := 3
	go foo(n)
	go bar(n)
	chanFoo <- struct{}{}

	//time.Sleep(time.Second)
	wg.Wait()
}
```

## LeetCode1116 打印零与奇偶数

![image-20220225103006438](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220225103006438.png)

代码如下

```go
var wg sync.WaitGroup
// 加一个缓冲区，避免死循环
var chanZero = make(chan struct{}, 1)
var chanOdd = make(chan struct{})
var chanEven = make(chan struct{})

func zero(n int) {
	defer wg.Done()
	for i := 1; i <= n; i++ {
		<- chanZero
		fmt.Printf("%d", 0)
		if i%2 == 1 {
			chanOdd <- struct{}{}
		} else {
			chanEven <- struct{}{}
		}
	}
}

func odd(n int) {
	defer wg.Done()
	for i := 1; i <= n; i++ {
		if i%2 == 1 {
			<- chanOdd
			fmt.Printf("%d", i)
			chanZero <- struct{}{}
		}
	}
}

func even(n int) {
	defer wg.Done()
	for i := 1; i <= n; i++ {
		if i%2 == 0 {
			<- chanEven
			fmt.Printf("%d", i)
			chanZero <- struct{}{}
		}
	}
}

func main() {
	wg.Add(3)
	n := 10
	go zero(n)
	go odd(n)
	go even(n)
	chanZero <- struct{}{}
	wg.Wait()
	fmt.Println()
}
```

交替打印奇数和偶数，一个管道实现方式：

```go
package main

import (
	"fmt"
	"sync"
)

var flagChan  = make(chan int)
var wg = &sync.WaitGroup{}

func odd() {
	defer wg.Done()
	for i := 1; i <= 100; i++ {
		flagChan <- 1
		if i&1 == 1 {
			fmt.Println(i)
		}
	}
}

func even() {
	defer wg.Done()
	for i := 1; i <= 100; i++ {
		<- flagChan
		if i&1 == 0 {
			fmt.Println(i)
		}
	}
}

func main() {
  defer close(flageChan)
	wg.Add(2)
	go odd()
	go even()
	wg.Wait()
}

```

## 四匹马赛跑问题

四匹马同时出发，每100m打印一个输出，1000m结束。

```go
var (
	wg   = sync.WaitGroup{}
	cond = sync.NewCond(&sync.Mutex{})
)

func print(number int) {
	defer wg.Done()
	fmt.Println(number, "号已经就位")
	// 调用 Wait 会自动释放锁 c.L，并挂起调用者所在的 goroutine，
	// 因此当前协程会阻塞在 Wait 方法调用的地方。
	// 如果其他协程调用了 Signal 或 Broadcast 唤醒了该协程，
	// 那么 Wait 方法在结束阻塞时，会重新给 c.L 加锁，并且继续执行 Wait 后面的代码。
	cond.L.Lock()
	cond.Wait()
	cond.L.Unlock()
	for i := 1; i <= 10; i++ {
		time.Sleep(time.Second)
		fmt.Printf("%d号马跑了%d米\n", number, i*100)
	}
}

func main() {
	wg.Add(4)
	for i := 1; i <= 4; i++ {
		go print(i)
	}

	time.Sleep(time.Second)
	// 开始跑
	cond.Broadcast()
	wg.Wait()
}
```

