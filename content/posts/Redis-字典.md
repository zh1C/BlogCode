---
author: "Narcissus"
title: "Redis(三)字典"
date: "2021-09-11"
description: "Redis设计与实现----阅读笔记之字典"
tags: ["Redis"]
categories: ["Redis学习"]
password: ""
---

字典，又称为符号表、关联数组或映射，是一种用于保存键值对的抽象数据结构。字典中每个键都是独一无二的。字典在Redis中应用相当广泛，比如Redis的数据库就是使用字典作为底层实现的，除此之外，字典还是哈希建的底层实现之一。

## 1、字典的实现

Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

### 哈希表

Redis字典所用哈希表结构定义如下：

![ScreenShot2021-09-11 11.07.10](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2011.07.10.png)

`table`属性是一个数组，数组中每个元素都是一个指向`dictEntry`结构的指针，每个`dictEntry`结构保存着一个键值对。如下展示了一个大小为4的空哈希表：

![ScreenShot2021-09-11 11.45.03](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2011.45.03.png)

### 哈希表节点

哈希表节点使用`dictEntry`结构表示，每个`dictEntry`结构都保存着一个键值对，结构定义如下：

![ScreenShot2021-09-11 11.47.27](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2011.47.27.png)

`next`属性是指向另一个哈希表节点的指针，这个指针**可以将多个哈希值相同的键值对连接在一起，来解决键冲突问题**。

### 字典

Redis中的字典结构如下：

![ScreenShot2021-09-11 12.24.01](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2012.24.01.png)

- `type`属性是一个指向`dictType`结构的指针，每个`dictType`结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型的特定函数。

- `privdata`属性则保存了需要传给那些特定函数的可选参数。
- `ht`属性包含两个项的数组，每一个项就是一个`dictht`哈希表，一般情况下只是用`ht[0]`哈希表，`ht[1]`哈希表只会在对`ht[0]`哈希表进行`rehash`时使用。

## 2、哈希算法

当要将一个新的键值对添加到字典里面时，需要先根据键值对的键计算出哈希值和索引值，再根据索引值将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。

Redis计算哈希值和索引值的方法如下：

![ScreenShot2021-09-11 12.36.25](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2012.36.25.png)![ScreenShot2021-09-11 12.36.36](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2012.36.36.png)

## 3、解决键冲突

当有两个或以上数量的键被分配到哈希表数组的同一个索引上面时，称为这些键发生了冲突。**Redis的哈希表使用链地址法解决冲突**，每个哈希表节点都有一个`next`指针，多个哈希表节点可以用`next`指针构成一个单链表。

> 注意：由于`dictEntry`节点组成的链表没有指向链表尾的指针，所以为了考虑速度，程序会将新节点添加到链表的表头位置。（复杂度O(1)）。

## 4、rehash

随着不断操作，哈希表保存的键值对会逐渐增多或减少，**为了让哈希表的负载因子维持在一个合理范围**，需要对哈希表的大小进行相应的扩展或者收缩。扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成。具体步骤如下：

1. 为字典的`ht[1]`哈希表分配空间，其大小取决于要执行的操作以及`ht[0]`当前包含的键值对数量。

- 如果执行扩展操作，那么`ht[1]`的大小为第一个大于或等于`ht[0].used*2`的2的n次方幂。
- 如果执行收缩操作，那么`ht[1]`的大小为第一个大于或等于`ht[0].used`的2的n次方幂。

2. 将保存在`ht[0]`的所有键值对rehash到`ht[1]`上面：rehash指重新计算哈希值和索引值，然后将键值对放置到`ht[1]`哈希表的指定位置上。
3. 迁移完成`ht[0]`变为空表后，释放`ht[0]`，将`ht[1]`设置为`ht[0]`，并在`ht[1]`新创建一个空白的哈希表。

### 哈希表的扩展与收缩

以下条件的任意一个被满足时，会自动开始对哈希表执行扩展操作：

1. 服务器目前没有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表负载因子大于等于1。
2. 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表负载因子大于等于5。

> 注意：`load_factor = ht[0].used / ht[0].size`负载因子 = 哈希表已保存节点数量/哈希表大小

当哈希表负载因子小于0.1时，程序会自动对哈希表执行收缩操作。

## 5、渐进式rehash

为了避免rehash对服务器性能造成影响，服务器不是一次性将`ht[0]`里面所有键值对全部rehash到`ht[1]`，而是分多次、渐进式的将`ht[0]`里面的键值对漫漫地rehash到`ht[1]`。

![ScreenShot2021-09-11 13.16.48](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2013.16.48.png)

> 注意：渐进式rehash期间，新添加到字典的键值对一律会被保存到`ht[1]`里面，而`ht[0]`则不再进行任何添加操作。

## 重点回顾

![ScreenShot2021-09-11 13.19.29](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-11%2013.19.29.png)