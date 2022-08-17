---
author: "Narcissus"
title: "sync包学习"
date: "2021-11-04"
description: "Golang并发编程：sync包学习使用。"
tags: ["Golang学习","并发"]
categories: ["Golang"]
---

## 读写锁和互斥锁

Go语言标准库`sync`提供了2中锁，互斥锁（sync.Mutex）和读写锁（sync.RWMutex）。

### 互斥锁(sync.Mutex)

即使用了互斥锁的代码片段互相排斥，当一个 goroutine 获得了这个锁的拥有权后， 其它请求锁的 goroutine 就会阻塞在 `Lock` 方法的调用上，直到锁被释放。sync.Mutex提供了两个方法：

- Lock加锁
- Unlock 释放锁

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-11/ScreenShot2021-11-04%2016.30.19.png)

可以在代码前调用Lock方法，代码后调用Unlock方法来保证一段代码互斥执行(用`defer`语句可以保证互斥锁一定会被释放)。在一个 Go 协程调用 Lock 方法获得锁后，其他请求锁的协程都会阻塞在 Lock 方法，直到锁被释放。

### 读写锁(sync.RWMutex)

允许多个只读操作并行执行，但写操作完全互斥。这种锁称为`多读单写锁`，简称读写锁，读写锁分为读锁和写锁，读锁允许同时执行，但写锁互斥。有如下情况：

- 读锁之间不互斥，没有写锁情况下，读锁无阻塞，多个协程可以同时获得读锁。
- 写锁之间互斥，存在写锁，其他写锁阻塞。
- 写锁与读锁之间互斥，如果存在读锁，写锁阻塞；如果存在写锁，读锁阻塞。

sync.RWMutex提供了四个方法：

- Lock 加写锁
- Unlock 释放写锁
- RLock 加读锁
- RUnlock 释放读锁

> 读写锁的存在是为了解决读多写少时的性能问题，读场景较多时，读写锁可以有效减少锁阻塞的时间。

### 互斥锁如何实现公平

Mutex有如下性质：

> 互斥锁有两种状态：正常状态和饥饿状态。
>
> 在正常状态下，所有等待锁的goroutine按照**`FIFO`**顺序等待。唤醒的goroutine不会直接拥有锁，而是会和新请求锁的goroutine竞争锁的拥有。新请求锁的goroutine具有优势：它正在CPU上执行，而且可能有好几个，所以刚刚唤醒的goroutine有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的goroutine会加入到等待队列的前面。 如果一个等待的goroutine超过1ms没有获取锁，那么它将会把锁转变为饥饿模式。
>
> 在饥饿模式下，锁的所有权将从unlock的gorutine直接交给交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，即使锁看起来是unlock状态, 也不会去尝试自旋操作，而是放在等待队列的尾部。
>
> 如果一个等待的goroutine获取了锁，并且满足一以下其中的任何一个条件：(1)它是队列中的最后一个；(2)它等待的时候小于1ms。它会将锁的状态转换为正常状态。
>
> 正常状态有很好的性能表现，饥饿模式也是非常重要的，因为它能阻止尾部延迟的现象。

## Sync.Cond

### sync.Cond的使用场景

> `sync.Cond`条件变量用来协调想要访问共享资源的那些goroutine，当共享资源的状态发生变化的时候，可以用来通知被互斥锁阻塞的goroutine。

`sync.Cond`常用在多个goroutine等待，一个goroutine通知的场景。如果是一个通知，一个等待，使用互斥锁或channel就可以。

### sync.Cond的方法

每个 Cond 实例都会关联一个锁 L（互斥锁 *Mutex，或读写锁 *RWMutex），当修改条件或者调用 Wait 方法时，必须加锁。

- NewCond 创建实例：`func NewCond(l Locker) *Cond`

- Broadcast 广播唤醒所有：`func (c *Cond) Broadcast()`

Broadcast唤醒所有等待调价变量c的goroutine，无需锁保护。

- Signal唤醒一个协程：`func (c *COnd) Signal()`

Signal 只唤醒任意 1 个等待条件变量 c 的 goroutine，无需锁保护。

- Wait 等待：`func (c *COnd) Wait()`

调用 Wait 会自动释放锁 c.L，并挂起调用者所在的 goroutine，因此当前协程会阻塞在 Wait 方法调用的地方。如果其他协程调用了 Signal 或 Broadcast 唤醒了该协程，那么 Wait 方法在结束阻塞时，会重新给 c.L 加锁，并且继续执行 Wait 后面的代码。

### 使用示例

三个协程调用`wait()`等待，另一个协程调用`Broadcast()`唤醒所有等待的协程。

```go
var done = false

func read(name string, c *sync.Cond) {
	c.L.Lock()
	for !done {
		c.Wait()
	}
	log.Println(name, "starts reading")
	c.L.Unlock()
}

func write(name string, c *sync.Cond) {
	log.Println(name, "starts writing")
	time.Sleep(time.Second)
	c.L.Lock()
	done = true
	c.L.Unlock()
	log.Println(name, "wakes all")
	c.Broadcast()
}

func main() {
	cond := sync.NewCond(&sync.Mutex{})

	go read("reader1", cond)
	go read("reader2", cond)
	go read("reader3", cond)
	write("writer", cond)

	time.Sleep(time.Second * 3)
}
```

- `done` 即互斥锁需要保护的条件变量。
- `read()` 调用 `Wait()` 等待通知，直到 done 为 true。
- `write()` 接收数据，接收完成后，将 done 置为 true，调用 `Broadcast()` 通知所有等待的协程。
- `write()` 中的暂停了 1s，一方面是模拟耗时，另一方面是确保前面的 3 个 read 协程都执行到 `Wait()`，处于等待状态。main 函数最后暂停了 3s，确保所有操作执行完毕。

执行结果

```shell
$ go run main.go        
2021/11/14 12:58:00 writer starts writing
2021/11/14 12:58:01 writer wakes all
2021/11/14 12:58:01 reader3 starts reading
2021/11/14 12:58:01 reader1 starts reading
2021/11/14 12:58:01 reader2 starts reading
```

