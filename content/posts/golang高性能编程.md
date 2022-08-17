---
author: "Narcissus"
title: "Golang高性能编程"
date: "2022-03-10"
description: "学习Golang的benchmark基准测试，并测试一些编程技巧的性能差异。"
tags: ["Golang"]
categories: ["Golang"]
---

## 1. Benchmark 基准测试

Go 语言标准库内置的 testing 测试框架提供了基准测试(benchmark)的能力，能让我们很容易地对某一段代码进行性能测试。

### 1.1 benchmark的使用

benchmark和普通单元测试用例一样，都位于`xxx_test.go`文件中。单元测试函数名以`Test`开头，参数是`t *testing.T`；**基准测试函数名以`Benchmark`开头，参数是`b *testing.B`。**

如下一个测试计算斐波那契数：

```go
// fib.go
package main

func fib(n int) int {
	if n == 0 || n == 1 {
		return n
	}
	return fib(n-2) + fib(n-1)
}

// fib_test.go
package main

import "testing"

func BenchmarkFib(b *testing.B) {
	for n := 0; n < b.N; n++ {
		fib(30) // run fib(30) b.N times
	}
}
```

- **运行用例**

`go test <module name>/<package name>`用来运行某个package内所有测试用例。

- 运行当前package内的用例：	`go test <module name>`或者`go test .`
- 运行子package内的用例：`go test <module name>/<package name>`或者`go test ./<package name>`
- 递归测试当前目录下所有package：`go test ./...` 或者`go test <module name>/...`

> 注意：**`go test`命令默认不运行benchmark用例，如果想运行，则必须加上`-bench`参数，并且支持正则表达式，匹配到额用例才会得到执行**。

例如：`go test -bench='Fib$'`表示只运行以FIb结尾的benchmark用例。`go test -bench=.`表示执行package下所有测试用例。

### 1.2 benchmark如何工作的

benchmark用例的参数`b *testing.B`，有个属性`b.N`表示这个用例需要运行的次数，对于每个用例都不一样。**`b.N`从1开始，如果该用例能在1s内完成，该值便会增加，再次执行；大概以1，2，3，5，1，,20，30，50，100这样的序列增加，越到后面增加越快**。

上述例子输出结果:

![image-20220310204333362](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220310204333362.png)

BenchmarkFib-8 中的 `-8` 即 `GOMAXPROCS`，默认等于 CPU 核数。

- 可以通过 `-cpu` 参数改变 `GOMAXPROCS`，`-cpu` 支持传入一个列表作为参数。则会在不同CPU核数下运行。
- `-benchtime`可以指定执行时间或者执行执行具体次数。

`-benchtime=30x`表示指定执行次数是30次；`-benchtime=5s`表示指定执行时间是5s。

- `-count`参数可以用来设置benchmark的轮数。
- `-benchmem` 参数可以看到内存分配次数和内存分配量。内存分配次数和性能也是息息相关的，例如不合理的切片容量，将导致内存重新分配，带来不必要的开销。

### 1.3 注意事项

- 如果在 benchmark 开始前，需要一些准备工作，如果准备工作比较耗时，则需要将这部分代码的耗时忽略掉。可以使用`ResetTimer`方法。

![image-20220310211148703](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220310211148703.png)

- 还有一种情况，每次函数调用前后需要一些准备工作和清理工作，我们可以使用 `StopTimer` 暂停计时以及使用 `StartTimer` 开始计时。

## 2. 字符串拼接性能及原理

在 Go 语言中，字符串(string) 是不可变的，拼接字符串事实上是创建了一个新的字符串对象。

### 2.1 字符串拼接方式

Go中常见有如下5中字符串拼接方式：

- 使用`+`
- 使用 `fmt.Sprintf`
- 使用 `strings.Builder`

![image-20220310213936632](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220310213936632.png)

- 使用 `bytes.Buffer`

![image-20220310214123143](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220310214123143.png)

- 使用`[]byte`

### 2.2 性能测试

每个 benchmark 用例中，生成了一个长度为 10 的字符串，并拼接 1w 次。测试结果如下：

```sh
$ go test -bench=. -benchmem             
goos: darwin
goarch: arm64
pkg: example
BenchmarkPlusConcat-8                 43          27403990 ns/op        530995869 B/op     10005 allocs/op
BenchmarkSprintfConcat-8              24          47286842 ns/op        833492883 B/op     37317 allocs/op
BenchmarkBuilderConcat-8           18522             64941 ns/op          505841 B/op         24 allocs/op
BenchmarkBufferConcat-8            20043             59793 ns/op          423537 B/op         13 allocs/op
BenchmarkByteConcat-8              21460             55752 ns/op          612337 B/op         25 allocs/op
BenchmarkPreByteConcat-8           38476             31120 ns/op          212992 B/op          2 allocs/op
PASS
ok      example 10.637s
```

可以看到使用`+`和`fmt.Sprintf`效率最低。`strings.Builder`、`bytes.Buffer`和`[]byte`的性能差距不大，且内存消耗十分接近。性能最好的是采用预分配内存的`[]byte`。因为该过程不需要发生内存拷贝和重新分配内存。

**综合建议，一般推荐使用`strings.Builder`来拼接字符串**。官方描述是：A Builder is used to efficiently build a string using Write methods. It minimizes memory copying. 

> 注意：`strings.Builder`也提供了预分配内存的方式。

![image-20220311141350330](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311141350330.png)

性能对比结果如下：

```sh
BenchmarkPreByteConcat-8           38613             30964 ns/op          212992 B/op          2 allocs/op
BenchmarkPreBuilderConcat-8        29694             40361 ns/op          106496 B/op          1 allocs/op
```

可以看到与预分配内存的`[]byte`相比，因为省去了`[]byte`和`string`之间的转换，内存分配次数减少了一次，内存消耗也减半。

### 2.3 性能背后的原理

`strings.Builder` 和 `+` 性能和内存消耗差距如此巨大，是因为两者的内存分配方式不一样。

字符串在Go语言中是不可变类型，占用内存大小是固定的，使用`+`拼接两个字符串时，需要开辟一段空间大小是两个字符串大小之和的新空间。

**而 `strings.Builder`，`bytes.Buffer`，包括切片 `[]byte` 的内存是以倍数申请的。**

### 2.4 比较strings.Builder和bytes.Buffer

`strings.Builder` 和 `bytes.Buffer` 底层都是 `[]byte` 数组，但 `strings.Builder` 性能比 `bytes.Buffer` 略快约 10% 。一个比较重要的区别在于，`bytes.Buffer` 转化为字符串时重新申请了一块空间，存放生成的字符串变量，而 `strings.Builder` 直接将底层的 `[]byte` 转换成了字符串类型返回了回来。

![image-20220311143845317](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311143845317.png)

## 3. 切片性能及陷阱

### 3.1 切片操作常见的操作技巧图示

- **Copy**

![image-20220311145459113](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311145459113.png)

- **Append**

![image-20220311145619636](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311145619636.png)

- **Delete**

切片底层是数组，删除意味着后面的元素需要逐个向前移动，复杂度是O(N)，因此切片不适合大量随机删除场景。

![image-20220311145850745](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311145850745.png)

- **Delete(GC)**

删除后将空余位置置空，有助于垃圾回收

![image-20220311150251335](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311150251335.png)

- **Filter**

当原切片不会再使用时，就地filter方式比较推荐，可以节省内存空间。

![image-20220311150622602](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-03/image-20220311150622602.png)

### 3.2 性能陷阱

在已有切片的基础上进行切片，不会创建新的底层数组。因为原来的底层数组没有发生变化，**内存会一直占用，直到没有变量引用该数组**。因此很可能出现这么一种情况，原切片由大量的元素构成，但是我们在原切片的基础上切片，虽然只使用了很小一段，但底层数组在内存中仍然占据了大量空间，得不到释放。比较推荐的做法，使用 `copy` 替代 `re-slice`。

