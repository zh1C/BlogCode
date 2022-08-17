---
author: "Narcissus"
title: "Redis(七)对象"
date: "2022-02-26"
description: "Redis设计与实现----阅读笔记之对象"
tags: ["Redis"]
categories: ["Redis学习"]
password: ""
---

Redis用到的主要数据结构有简单动态字符串(SDS)、双端链表、字典、跳跃表、压缩列表、整数集合等。但是Redis并没有直接使用这些数据结构来实现键值对数据库，而是**基于这些数据结构创建了一个对象系统**。包含**字符串对象、列表对象、哈希对象、集合对象和有序集合对象**。每种对象至少用到了一种前面的数据结构。

## 1、对象类型及编码

Redis数据库中新创建一个键值对时，至少会创建两个对象，分别用作键对象和值对象。

对象的结构体定义如下：

![ScreenShot2021-09-17 12.43.15](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2012.43.15.png)

### 类型

对象的`type`属性记录了对象的类型，属性值如下表：

![ScreenShot2021-09-17 12.51.34](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2012.51.34.png)

> 键总是字符串对象，则值可以为上述所有对象。

对数据库键执行`TYPE`命令时，返回的结果是值对象的类型，不同类型值对象的`TYPE`命令输出如下表：

![ScreenShot2021-09-17 12.57.58](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2012.57.58.png)

### 编码和底层实现

对象的`ptr`指针指向对象的底层实现数据结构，而数据结构由对象的`encoding`属性决定。`encoding`属性记录了对象所使用的编码，即对象使用了什么数据结构作为对象的底层实现，每种类型的对象都至少使用了两种不同的编码：

![ScreenShot2021-09-17 13.29.41](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2013.29.41.png)



使用`OBJECT ENCODING`命令可以查看数据库键的值对象编码。

![ScreenShot2021-09-17 13.50.36](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2013.50.36.png)![ScreenShot2021-09-17 13.50.54](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2013.50.54.png)

通过`encoding`属性来设定对象所使用的编码，而不是为特定类型对象关联一种固定编码，极大提升了Redis的灵活性和效率。

## 2、字符串对象

**字符串对象的编码可以是`int、raw`或者`embstr`。即底层实现是整数、embstr编码的SDS和SDS。

- 如果字符串对象保存的是整数值，并且该整数值可以用`long`类型表示，那么字符串对象会将整数值保存在字符串对象结构的`ptr`属性里面(将`void*`转换为`long`)，并将字符串对象编码设置为`int`。
- 如果字符串对象保存的是字符串值，并且长度大于32字节，将设置为简单动态字符串(SDS)，编码设置为`raw`。
- 如果字符串对象保存的是字符串值，并且长度小于等于32字节，将设置为`embstr`编码方式。

> `embstr`编码专门用于保存短字符串的优化编码方式。和`raw`编码方式一样，都使用`redisObject`结构和`sdshdr`结构；**不同的是`raw`编码会调用两次内存分配函数分别创建`redisObject`结构和`sdshdr`结构，`embstr`编码只会调用一次内存分配函数来分配一块连续的空间，空间依次包含`redisObject`和`sdshdr`两个结构**。

`embstr`编码的字符串对象在执行命令时，产生的效果和`raw`编码的一样。但有以下好处：

![ScreenShot2021-09-17 14.39.38](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2014.39.38.png)

执行以下命令：

![ScreenShot2021-09-17 14.47.16](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2014.47.16.png)![ScreenShot2021-09-17 14.47.26](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2014.47.26.png)

> 注意：可以用`long double`类型表示的浮点数在Redis中也是作为字符串值来保存的。

![ScreenShot2021-09-17 14.53.29](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2014.53.29.png)

### 编码的转换

`int`编码的字符串对象和`embstr`编码的字符串对象在条件满足的情况下，会被转换为`raw`编码的字符串对象。

- 对于`int`编码的字符串，如果对对象执行一些命令，使得这个对象保存的不再是整数值，而是一个字符串值，编码则会变为`raw`。例如`APPEND`命令。
- 因为Redis没有为`embstr`编码的字符串对象编写任何修改程序，所以`embstr`编码的字符串对象实际上是只读的。执行任何修改命令时，编码就会改为`raw`。

### 字符串命令的实现

部分字符串命令及在不同编码下的实现方法：

![ScreenShot2021-09-17 15.06.00](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2015.06.00.png)

## 3、列表对象

**列表对象的编码可以是`ziplist`或者`linkedlist`。即底层实现是压缩列表和双端链表**。

`ziplist`编码的列表对象使用压缩列表作为底层实现，每个压缩列表节点保存了一个列表元素。如下图：

![ScreenShot2021-09-17 16.16.36](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2016.16.36.png)

`linkedlist`编码的列表对象使用双端链表作为底层实现，每个双端链表节点都保存了一个字符串对象，而每个字符串对象都保存了一个列表元素。

![ScreenShot2021-09-17 16.16.47](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2016.16.47.png)

> 注意：
>
> 1. 双端链表结构中嵌套了字符串对象，字符串对象是Redis五种类型的对象中唯一一种会被其他对象嵌套的对象。
> 2. 上图是简化的字符串对象表示，真实情况如下图：
>
> ![ScreenShot2021-09-17 18.21.43](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2018.21.43.png)

### 编码转换

当列表对象满足以下两个条件时，列表对象使用`ziplist`编码：

- 列表对象保存的所有字符串元素的长度都小于64字节；
- 列表对象保存的元素数量小于512个。

不能同时满足条件的列表对象需要使用`linkedlist`编码。

> 注意：以上两个条件的上限值可以修改。

对于使用`ziplist`编码的列表对象来说，当以上任意一个条件不能被满足时，对象编码转换操作就会被执行。

### 列表命令的实现

下表列出了列表对象常见命令：

![ScreenShot2021-09-17 19.18.02](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-09/ScreenShot2021-09-17%2019.18.02.png)

## 4、哈希对象

**哈希对象的编码可以是`ziplist`和`hashtable`。即底层实现是字典和压缩列表**。

压缩列表作为底层实现时，有以下特点：

- 保存了同一键值对的两个节点总是紧挨在一起，保存键的节点在前，值的节点在后。
- 先添加的键值对会被放在压缩列表表头方向，后添加的在表尾方向。

![image-20220226162938047](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226162938047.png)

字典作为底层实现时，有以下特点：

- 字典的每个键都是一个字符串对象，保存了键值对中的键。
- 字典的每个值都是一个字符串对象，保存了键值对中的值。

![image-20220226163203569](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226163203569.png)

### 编码转换

当哈希表对象同时满足以下两个条件时，使用`ziplist`编码：

- 哈希对象保存的键值对中键和值的字符串长度都小于64字节。
- 键值对数量小于512个。

### 哈希表命令实现：

![image-20220226163452498](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226163452498.png)

## 5、集合对象

**集合对象的编码可以是`intset`或者`hashtable`。即底层整数集合或者字典**。

`inset`编码的集合使用整数集合作为底层是实现：

![image-20220226163718584](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226163718584.png)

`hashtable`编码的集合使用字典作为底层对象，字典的每一个键都是一个字符串对象，包含一个集合元素，字典的值全部被设置为NULL。

![image-20220226163838352](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226163838352.png)

### 编码转换

当集合对象同时满足以下两个条件是，使用`intset`编码：

- 集合对象所有元素都是整数值
- 元素数量不超过512个。

### 集合命令的实现

![image-20220226164008388](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226164008388.png)

![image-20220226164020065](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226164020065.png)

## 6、有序集合对象

**有序集合的编码可以是`ziplist`或者`skiplist`。即有序集合的底层实现是压缩列表或者跳表和字典**。

`ziplist`编码的有序集合对象使用压缩列表作为底层实现。每个集合元素使用两个紧挨一起的压缩列表结点保存，第一个节点保存元素成员，第二个节点保存元素的分值。

**压缩列表内集合元素按分值从小到大进行排序**。

![image-20220226164523407](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226164523407.png)

`skiplist`编码的有序集合对象使用`zset`结构作为底层实现，**一个`zset`结构同时包含一个字典和一个跳表**：

![image-20220226164701912](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226164701912.png)

![image-20220226164710676](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226164710676.png)

`zet`结构中跳表按分值从小到大保存了所有集合元素：跳表节点的object属性保存了元素的成员，score属性保存了元素的分值。

`zset`结构中的 字典为有序集合创建了一个从成员到分值的映射：字典的键保存了元素的成员，字典的值保存了元素的分值。即通过字典可**以用O(1)复杂度查找给定成员的分值**。

**虽然`zset`结构同时使用跳表和字典保存有序集合的元素，但这两种数据结构通过指针共享相同元素的成员和分值，所以不会造成额外的内存浪费**。

![image-20220226165329367](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226165329367.png)

### 编码转换

当有序集合对象同时满足以下两个条件时，使用`ziplist`编码：

- 有序集合元素小于128个。
- 有序结合保存的元素成员长度都小于64字节。

### 有序集合的命令实现

![image-20220226165549191](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-02/image-20220226165549191.png)

