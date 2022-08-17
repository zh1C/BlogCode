---
author: "Narcissus"
title: "Redis(六)压缩列表"
date: "2021-09-13"
description: "Redis设计与实现----阅读笔记之压缩列表"
tags: ["Redis"]
categories: ["Redis学习"]
password: ""
---

压缩列表(ziplist)是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项(哈希键只包含少量键值对)，并且每个列表项(键值对的键和值)要么就是较小整数值，要么就是较短的字符串时，Redis就会使用压缩列表作为底层实现。

## 1、压缩列表的构成

压缩列表是Redis为节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表可以包含任意多个节点，压缩列表组成部分及说明：

![ScreenShot2021-09-13 10.20.44](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-13%2010.20.44.png)

## 2、压缩列表节点的构成

每个压缩列表节点可以保存一个字节数组或者一个整数值。每个压缩列表节点都是由三部分组成：

![ScreenShot2021-09-13 10.27.48](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-13%2010.27.48.png)

- **previous_entry_length**

该属性以字节为单位，记录了压缩列表**前一个节点的长度**。该属性长度可以是1字节或者5字节。**因为`previous_entry_length`属性记录了前一个节点的长度，所有程序可以通过指针运算，根据当前节点起始地址计算出前一个节点的起始地址**。压缩列表从表尾向表头遍历操作就是使用这一原理实现的。

- **encoding**

该属性记录了节点的`content`属性所保存数据的类型及长度。

- **content**

该属性负责保存节点的值，节点值可以是一个字节数组或者整数。由`encoding`属性决定。

## 3、连锁更新

每个节点的`previous_entry_length`属性记录了前一个节点的长度，有如下规则：

![ScreenShot2021-09-13 10.43.15](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-13%2010.43.15.png)

现有如下情况，如果e1至eN都是大小介于250字节至253字节的节点，新添加的new节点长度大于等于254字节(需要5字节的`previous_entry_length`来保存)，那么e1节点的`previous_entry_length`属性就需要从1字节扩展为5字节长。这样e1长度就介于254字节至257字节之间，引发后续的连锁更新。如下图：

![ScreenShot2021-09-13 10.50.59](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-13%2010.50.59.png)

**Redis将这种特殊情况下产生的连续多次空间扩展操作称之为“连锁更新”**。除了添加节点会引发之外，删除节点也会引发连锁更新。

![ScreenShot2021-09-13 10.52.48](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-13%2010.52.48.png)

连锁更新在最坏情况下需要对压缩列表执行N次空间重新分配操作，而每次空间重新分配最快复杂度为O(N),所以连锁更新最坏复杂度为O(N平方)。但实际中这种情况发生记录很低。所以相关命令操作的平均复杂度均为O(N)。

> 注意：因为`ziplistPush`、`ziplistInsert`、`ziplistDelete`和`ziplistDeleteRange`四个函数都有可能会引发连锁更新，所以最坏复杂度为O(N平方)。

## 重点回顾

![ScreenShot2021-09-13 10.58.01](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-13%2010.58.01.png)

