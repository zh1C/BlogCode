---
author: "Narcissus"
title: "Shell脚本学习"
date: "2021-12-06"
description: "学习Shell脚本基本用法，使用Bash。"
tags: ["shell"]
categories: ["技术"]
---

## Bash基本语法

### 1. echo命令

`echo`命令作用是在屏幕输出一行文本。如果想要输出多行文本，即包括换行符，需要把多行文本放在引号里面。

![image-20211206194932194](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206194932194.png)

#### 1.1 -n参数

默认情况下，`echo`输出的文本末尾会有一个回车符，`-n`参数可以取消末尾的回车符，使得下一个提示符紧跟在输出内容后面。

![image-20211206195222336](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206195222336.png)

#### 1.2 -e参数

`-e`参数会解释引号（双引号和单引号）里面的特殊字符（比如换行符`\n`）。如果不使用`-e`参数，即默认情况下，引号会让特殊字符变成普通字符，`echo`不解释它们，原样输出。

![image-20211206195422313](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206195422313.png)

### 2. 命令行格式

shell命令基本都是如下格式

![image-20211206195948967](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206195948967.png)

有些参数是命令的配置项，这些配置项一般都以一个连词线开头，比如上面的`-l`。同一个配置项往往有长和短两种形式，比如`-l`是短形式，`--list`是长形式，它们的作用完全相同。短形式便于手动输入，长形式一般用在脚本之中，可读性更好，利于解释自身的含义。

Bash 单个命令一般都是一行，用户按下回车键，就开始执行。有些命令比较长，写成多行会有利于阅读和编辑，这时可以在每一行的结尾加上反斜杠，Bash 就会将下一行跟当前行放在一起解释。

![image-20211206200039034](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206200039034.png)

### 3. 空格

Bash使用空格区分不同的参数，如果参数之间有多个空格，Bash会自动忽略多余空格。

### 4. 分号

分号是命令的结束符，使得一行可以放置多个命令。

> 注意：使用分号时，第二个命令总是接着第一个命令执行，不管第一个命令执行成功与否。

### 5. 命令 组合符&&和||

下面的命令的意思是，如果`Command1`命令运行成功，则继续运行`Command2`命令。

![image-20211206201449273](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206201449273.png)

下面的命令意思是，如果`Command1`命令运行失败，则继续运行`Command2`命令。

![image-20211206201519672](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211206201519672.png)

### 6. type命令

`type`命令用来判断命令的来源，是内部命令还是外部程序。

### 7. 快捷键

- `Ctrl + L`：清除屏幕并将当前行移到页面顶部。
- `Ctrl + C`：中止当前正在执行的命令。
- `Shift + PageUp`：向上滚动。
- `Shift + PageDown`：向下滚动。
- `Ctrl + U`：从光标位置删除到行首。
- `Ctrl + K`：从光标位置删除到行尾。
- `Ctrl + D`：关闭 Shell 会话。
- `↑`，`↓`：浏览已执行命令的历史记录。

除此之外，还有Tab键，可以自动补全命令。

