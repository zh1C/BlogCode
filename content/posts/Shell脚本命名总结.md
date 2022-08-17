---
author: "Narcissus"
title: "Shell脚本常用命令总结"
date: "2022-08-17"
lastmod: "2022-08-17"
description: "总结一些好用的Linux命令。"
tags: ["shell"]
categories: ["技术"]
---

### 1. 统计代码

```shell
# 统计当前目录下以.go文件有多少行代码
find . -name "*.go" | xargs cat | wc -l
# 统计当前目录下go的测试文件有多少行代码
find . -name "*_test.go" | xargs cat | wc -l
```

> find命令用于在指定目录查找文件，`.` 表示当前目录，`-name filename` 表示文件名称符合name的文件。
>
> cat命令用于连接文件并打印到标准输出上
>
> wc命令用于计算字数，`-l` 显示行数。

### 2. 查找文件并删除

```shell
# 用于查找当前目录下指定文件名的所有文件并删除
find . -name [文件名] | xargs rm -f
```

### **xargs命令介绍** 

可以通过管道将多个命令串联起来，前提是管道后面的命令要支持从标准输入中读取数据，比如的 `grep` 命令。然而有些命令并不支持从标准输入中读取，比如这样写是无效的：

```shell
# 想要实现删除file_name文件
echo 'file_name' | rm
```

此时，我们可以借助xargs命令实现上述目的：

```shell
echo 'file_name' | xargs rm 
```

此命令的原理就是，**xargs会把换行符、空格、制表符、EOF等符号作为分隔符，把输入的内容切分为一个数组，并把数组中的每一个元素作为参数，放到后面的命令中执行。** 伪代码如下：

```shell
for arg in read_input; do
    rm arg
done
```

很常见的一个坑就是，如果文件名带有空格，比如 **hello world** 就会被 `xargs` 截断为两个参数，显然不符合预期。不过一般对内容或者文件进行过滤时，我们都会使用 `grep` 或 `find`，这两个命令都有办法配合 `xargs`。

- 用 `grep` 的话会繁琐一些，需要用 `tr` 命令把换行符转换成特殊字符 `\0`，再利用 `xargs` 的 `-0` 参数，根据文档所述，这个参数会把分隔符指定为 `-0`，从而避免了文件名中含有空格的影响。

```shell
ls | grep 'a' | tr "\n" "\0" | xargs -0 rm
```

- 用 `find` 也是类似的原理，只不过它自带了 `-print0`选项，写法更简单。

```shell
find . -print0 | xargs -0 rm
```

