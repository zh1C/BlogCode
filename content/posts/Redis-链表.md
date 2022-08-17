---
author: "Narcissus"
title: "Redis(二)链表"
date: "2021-09-09"
description: "Redis设计与实现----阅读笔记之链表"
tags: ["Redis"]
categories: ["Redis学习"]
password: ""
---

链表提供了高效的节点重排能力，以及排序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

链表在Redis中应用非常广泛，比如列表键的底层实现之一就是链表。除此之外，发布与订阅、慢查询、监视器等功能也用到了链表。

## 1、链表和链表节点的实现

链表节点结构体定义如下：

![ScreenShot2021-09-09 16.38.01](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2016.38.01.png)

多个`listNode`可以通过`prev`和`next`指针组成双端链表。使用`list`来持有链表，操作会方便一些：

![ScreenShot2021-09-09 16.40.13](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2016.40.13.png)

Redis的链表实现的特性可以总结如下：

- 双端：链表节点带有`prev`和`next`指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)。
- 无环：表头节点的`prev`指针和表尾节点的`next`指针都指向NULL，对链表的访问都以NULL为终点。
- 带表头指针和表尾指针：通过`list`结构的`head`指针和`tail`指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)。
- 带链表长度计数器：程序使用`list`结构的`len`属性来对`list`持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O(1)。
- 多态：链表节点使用`void*`指针来保存节点值，并且可以通过`list`结构的`dup`、`free`、`match`三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。

![ScreenShot2021-09-09 16.55.45](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2016.55.45.png)

## 2、总结

![ScreenShot2021-09-09 16.58.10](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-09%2016.58.10.png)

