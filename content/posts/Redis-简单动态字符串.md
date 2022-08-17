---
author: "Narcissus"
title: "Redis(一)简单动态字符串"
date: "2021-09-09"
description: "Redis设计与实现----阅读笔记之简单动态字符串"
tags: ["Redis"]
categories: ["Redis学习"]
password: ""
---

## 概述

简单来说 **Redis 就是一个使用 C 语言开发的数据库**， 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储键和五种不同类型的值之间的映射。键的类型只能为字符串，值支持五种数据类型：字符串、列表、集合、散列表、有序集合。与传统数据库不同的是 **Redis 的数据是存在内存中的** ，也就是它是内存数据库，所以读写速度非常快，因此 Redis 被广泛应用于缓存方向。另外，**Redis 除了做缓存之外，也经常用来做分布式锁，甚至是消息队列。**

## 简单动态字符串

Redis没有直接使用C语言传统的字符串表示(以空字符结尾的字符数组)，而是自己构建了一种名为简单动态字符串(simple dynamic string，SDS)的抽象类型，并作为默认字符串表示。

**在Redis的数据库里面，包含字符串值的键值对在底层都是由SDS实现的。除此之外还被用作缓冲区。**

### 1、SDS定义

SDS结构体定义如下：

![ScreenShot2021-09-09 12.58.51](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2012.58.51.png)

**SDS底层对字符数组结尾的空字符作了透明处理。方便**SDS直接重用C字符串函数库里面的函数。

### 2、SDS与C字符串的区别

#### 2.1 常数复杂度获取字符串长度

对于C字符串，因为不记录自身长度信息，所以每一次获取一个字符串长度都需要O(N)时间复杂度来遍历字符串获取长度；而对于SDS，因为`len`属性记录了SDS本身的长度，因此**获取一个SDS长度的复杂度仅为O(1)。**因此即使对一个非常长的字符串键反复执行STRLRN命令，也不会对系统性能造成任何影响。

#### 2.2 杜绝缓冲区溢出

C字符串不记录自身长度带来的另一个问题就是容易造成缓冲区溢出(buffer overflow)。SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性：**需要对SDS进行修改时，会自动先检查SDS空间是否满足修改所需的要求，并自动扩容到所需大小。**

#### 2.3 减少修改字符串时带来的内存重分配次数

内存重分配涉及复杂的算法，通常是一个非常耗时的操作。由于C字符串的长度和底层数组的长度之间的关联性(N个字符底层是N+1个字符长)，会造成频繁的内存重分配，**SDS通过未使用空间解除了字符串长度和底层数组长度之间的关联**:在SDS中，buf数组里面包含未使用的字节，这些字节的数量由`free`属性记录。

通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略。

- **空间预分配**

用于优化SDS的字符串增长操作：需要对SDS进行空间扩容时，会分配额外未使用的空间。分配策略如下：

1. 如果对SDS修改后，SDS长度(即`len`属性的值)将小于1MB，程序分配和`len`属性同样大小的未使用空间，这时`len`属性值将和`free`属性值相同。
2. 如果对SDS修改后，SDS长度大于等于1MB，程序会分配1MB未使用空间。

通过空间预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数。**将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次。**

举例，对下图执行`sdscat(s, "Cluster");`操作：

![ScreenShot2021-09-09 13.57.26](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2013.57.26.png)

此时SDS的buf数组的实际长度为13+13+1=27字节(额外1字节用于保存空字符)。

![ScreenShot2021-09-09 13.57.39](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2013.57.39.png)

- **惰性空间释放**

用于优化SDS的字符串缩短操作：**当SDS需要缩短保存的字符串时，程序不立即使用内存分配来回收缩短后多出的字节，而是用`free`属性将这些字节的数量记录起来，并等待将来使用**。举例对下图执行`sdstrim(s, "XY"); // 移除SDS字符串中所有'X'和'Y'`操作：

![ScreenShot2021-09-09 14.05.35](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2014.05.35.png)

执行后多出来的8字节作为未使用空间保留在了SDS里面。

![ScreenShot2021-09-09 14.05.43](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2014.05.43.png)

#### 2.4 二进制安全

C字符串不能包含空字符，否则会被程序误认为是结尾符，因此不能保存像图片、音频、视频等二进制数据。为了确保Redis可以用于各种不同场景，SDS的API都是二进制安全的(binary-safe)。所以将SDS的`buf`属性称为字节数组。**因为SDS使用`len`属性的值而不是空字符来判断字符串是否结束**。如下图所示：

![ScreenShot2021-09-09 14.12.55](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2014.12.55.png)

#### 2.5 兼容部分C字符串函数

虽然SDS的API都是二进制安全的，但它们一样遵循C字符串以空字符结尾的惯例。所以可以重用一部分`<string.h>`库定义的函数。

## 总结

C字符串和SDS之间的区别如下表：

![ScreenShot2021-09-09 14.16.11](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2014.16.11.png)

重点回顾

![ScreenShot2021-09-09 14.17.29](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2014.17.29.png)

