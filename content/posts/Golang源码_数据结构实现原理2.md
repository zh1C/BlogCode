---
author: "Narcissus"
title: "Golang源码之常见数据结构实现原理下"
date: "2021-12-01"
description: "Go语言专家编程书籍阅读笔记，常见数据结构的源码阅读，包括struct、iota、string。"
tags: ["Golang"]
categories: ["Golang"]
---

## Struct的Tag

### 1. 前言

Go的struct声明允许字段附带`Tag`来对字段做一些标记。该Tag不仅仅是一个字符串，写法也需要遵循一定的规则。

### 2. Tag的本质

#### 2.1 Tag规则

`Tag`本身是一个字符串，规则必须满足**以空格分隔的key:value对**。

- `key`：必须是非空字符串，不能包含控制符号、空格、引号、冒号。
- `value`：以双引号标记的字符串。
- 注意：冒号前后不能有空格。

#### 2.2 Tag是struct的一部分

`Tag`只有在反射场景中才有用，反射包提供了操作`Tag`的方法，以下是`reflect`包中的类型声明，省略了部分与无关字段。

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2017.27.27.png)

描述一个结构体成员的结构中包含了`StructTag`，其本身是一个string。**也就是说，`Tag`是结构体字段的一个组成部分**。

### 3. Tag常见用法

常见的tag用法主要是JSON数据解析，ORM映射等。

## Iota

### 规则

很多博客上描述的规则是：

- iota在const关键字出现时被重置为0
- const声明块中每新增一行iota值自增1

但实际上规则就一条：

- **iota代表了const声明块的行索引(下标从0开始)**

这样理解更贴近编译器实现逻辑，也更准确。此外，**const声明还有一个特点，即第一个常量必须指定一个表达式，后续的常量如果没有表达式，则继承上面的表达式**。

如下面的代码，每一个常量值为多少：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2018.52.49.png)

- 第0行表达式展开即`bit0, mask0 = 1<<0, 1<<0 - 1`，所以bit0 = 1,mask0 = 0。
- 第1行没有指定表达式继承上一行，即`bit1, mask1 = 1<<1, 1<<1 - 1`，所以bit1 = 2,maks1 = 1。
- 第2行没有定义常量
- 第3行没有指定表达式继承第一行，即`bit3, mask3 = 1<<3, 1<<3 - 1`，所以bit3 = 8, mask3 = 7。

### 2. 编译原理

const块中每一行在Go中使用`spec`数据结构描述，声明如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.04.39.png)

只关注`ValueSpec.Names`，这个切片中保存了一行中定义的常量，如果一行中定义了N个常量，那么`ValueSpec.Names`切片长度为N。const块实际上是spec类型的切片，用于表示const中的多行。

编译期间构造常量时的伪算法如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.07.42.png)

**可以清晰看到iota实际上就是遍历const块的索引，所以每行即便多次使用iota，其值也不会递增**。

## String

### 1. string标准概念

Go标准库`builtin`给出了所有内置类型的定义。源码位于`src/builtin/builtin.go`，关于string描述如下：

![ScreenShot2021-12-01 19.11.41](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.11.41.png)

string是8比特字节的集合，通常但并不一定是UTF-8编码的文本，**string可以为空（长度为0），但不会是nil；string对象不可以修改**。

### 2. string数据结构

源码`src/runtime/string.go`中定义了string的数据结构：

![ScreenShot2021-12-01 19.15.23](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.15.23.png)

- `stringStruct.str`：字符串的首地址；
- `stringStruct.len`：字符串长度。

### 3. string操作

#### 3.1 声明

如下声明一个string变量并赋予初值：

![ScreenShot2021-12-01 19.19.13](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.19.13.png)

字符串构建过程是先根据字符串构建`stringStruct`，再转换成string。转换源代码如下：

![ScreenShot2021-12-01 19.21.54](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.21.54.png)

#### 3.2 []byte转string

byte切片可以很方便的转换为string，代码如下：

![](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.23.29.png)

**注意：这种转换需要一次内存拷贝**。如下图：

![ScreenShot2021-12-01 19.26.50](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/ScreenShot2021-12-01%2019.26.50.png)

#### 3.3 string转[]byte

string也可以方便的转成byte切片，如下代码所示：

![image-20211201193136572](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201193136572.png)

**也需要一次内存拷贝**，如下图示：

![image-20211201193223624](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201193223624.png)

#### 3.4 字符串拼接

字符串可以很方便进行拼接，代码如下：

![image-20211201193358528](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211201193358528.png)

即便非常多的字符串需要拼接，性能上也有比较好的保证，因为新字符串的内存空间是一次分配完成的，所以性能消耗主要在拷贝数据上。**一个拼接语句的字符串编译时都会被存放到一个切片中，拼接过程需要遍历两次切片，第一次遍历获取总的字符串长度，据此申请内存，第二次遍历会把字符串逐个拷贝过去**。

### 4. []byte转换成string一定会拷贝内存吗？

byte切片转换成string的场景很多，**为了性能上的考虑，有时候只是临时需要字符串的场景下，byte切片转换成 string时并不会拷贝内存，而是直接返回一个string，这个string的指针(string.str)指向切片的内存**。

比如以下临时场景：

- 使用`m[string(b)]`来查找map（map是string为key，临时把切片b转成string）； 
- 字符串拼接，如`”<”	+	“string(b)”	+	“>”`； 
- 字符串比较：`string(b)	==	“foo”`。

