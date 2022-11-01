---
author: "Narcissus"
title: "Emacs学习"
date: "2022-10-07"
lastmod: "2022-10-08"
description: "emacs上手学习，包括安装、配置等基础使用"
tags: ["emacs"]
categories: ["emacs"]
password: ""

---

Emacs是一个文本编辑器，和vim类似，可以配置很多插件类优化编程过程。我选择从emacs入手。使用[spacemacs](https://github.com/syl20bnr/spacemacs)的配置入门。

## 一、准备工作

安装spacemacs之前，需要一些准备工作，如下：

- 一个包管理器
- spacemacs是emacs的扩展，因此需要先安装emacs，需要27.1或者更新的版本。
- Git。
- Tar打包软件，GUN Tar或者BSD Tar都可以。
- （可选），spacemacs默认使用的是[Source Code Pro](https://adobe-fonts.github.io/source-code-pro/)字体，如果想使用，就需要提前下载。
- （可选），需要安装一个搜索程序，一般系统中都有grep搜索，但是建议使用[ripgrep](https://github.com/BurntSushi/ripgrep)，因为他更加的快速。

## 二、安装过程

### macOS

macOS安装过程如下：

- macOS 最流行的包管理器是homebrew，可以使用如下命令安装：

    `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

- macos安装emacs有如下几种方式：

    1. Emacs Plus在基础的emacs之上还有一些额外的功能。

    ```shell
    brew tap d12frosted/emacs-plus
    
    # install latest stable release, with Spacemacs icon and native compilation
    brew install emacs-plus --with-spacemacs-icon --with-native-comp
    ```

    2. [Emacs for Max OS X ](https://emacsformacosx.com/)是最基本的二进制安装的，没有任何额外的功能。

    `brew install --cask emacs`

- git也可以使用homeBrew进行安装

- macOS 附带 BSD Tar，因此您无需安装它
- (可选)安装字体：

```shell
brew tap homebrew/cask-fonts
brew install --cask font-source-code-pro
```

- (可选)安装ripgrep:

`brew install ripgrep`

至此，准备工作已经完成，下面可以安装spacemacs了。选择vim模式、spacemacs模式安装即可，即默认模式安装。

### 改键

由于emacs中control键使用频繁，为了方便，建议将中/英切换键修改为control键

![image-20220927133910018](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-2022-09/image-20220927133910018.png)

### github管理个人配置

spacemacs的个人配置都在HOME目录下的`.spacemacs`文件中，所有的个人配置都在此文件中进行修改。如果想要使用Github管理此文件，则修改在HOME目录下建立`.spacemacs.d`目录，并且将`.spacemacs`文件移到该目录中，同时重命名为`init.el`即可。然后使用Git管理此文件。

使用命令SPE g i 创建一个git仓库，然后按SPE g s可以看到文件的状态，然后使用SPE g c命令(后续还有操作，按提示操作即可)commit所有文件。

### 常用配置修改

- 显示行号

按 SPC f e d 可以快速打开 .spacemacs 文件，然后使用`SPC s s`进行搜索，搜索到`dotspacemacs-line-numbers`,修改为t表示显示行号。**可以修改为 `'relactive`表示显示相对行号，方便上下移动多少行命令。**

- 显示时间

在.spacemacs文件中搜索`dotspacemacs/user-config`选项，添加`(display-time-mode t)`表示显示时间配置。

- 修改退出编辑快捷键

使用vim风格时，快捷键`i` 是进入编辑状态，快捷键`esc`则是退出编辑，但是esc快捷键有一点麻烦，可以通过`evil-escape`来进行修改，在`.spacemacs`文件中(快捷键`SPC f e d`快速打开，`SPE f e R`重载配置)进行如下配置：

```lisp
(defun dotspacemacs/user-config ()
  ;; ...
  ;; Set escape keybinding to "jk"
  (setq-default evil-escape-key-sequence "jk"))
```

- 关闭滚动条

默认spacemacs打开了滚动条，通过如下设置关闭滚动条：
`dotspacemacs-scroll-bar-while-scrolling t`

- 高亮显示80行以后的字符

目的是防止编程超过80行，在`user-config`中进行如下配置：
`spacesmacs/toggle-highlight-long-lines-globally-on`

- 自动保存

在`user-config`中进行如下配置，可以配置自动保存的频率：

>`auto-save-interval`与`auto-save-timeout`设置自动保存间隔。

## 三、开发环境配置

### Python开发环境配置

在对应的env中，安装如下依赖包即可，然后再spacemacs中添加`python`层，

`pip install autoflake isort yapf flake8 python-lsp-server importmagic epc`

## 四、常用快捷键

### 光标移动快捷键

**Vim在编辑模式下有时候需要输入一个命令，但是来回切换有点麻烦，可以按`C o`来临时切换到命令模式，然后输入一个命令。**

> 例如：编辑模式下频繁需要从当前行中跳到下一行继续输入，`C o o`就相对高效。

| 快捷键         | 描述                                             |
| -------------- | :----------------------------------------------- |
| `h`            | 左移光标                                         |
| `j`            | 下移光标                                         |
| `k`            | 上移光标                                         |
| `l`            | 右移光标                                         |
| `H`            | 移动光标到顶部                                   |
| `L`            | 移动光标到底部                                   |
| `0`或`SPC j 0` | 移动光标到行开始位置(并在前一个位置设置一个标记) |
| `$`或`SPC j $` | 移动光标到行末尾位置(并在前一个位置设置一个标记) |
| `数字 j`       | 向下移动number行                                 |
| `数字 k`       | 向上移动number行                                 |
| `C d`、`C f`   | 向下翻页半屏，向下翻页全屏                       |
| `C u`、`C b`   | 向上翻页半屏，向上翻页全屏                       |
| `-`            | 光标移动到上一行第一个非空白字符                 |
| `Enter`        | 光标移动到下一行第一个非空白字符                 |
| `w`            | 后移动一个单词，光标停留在单词开头               |
| `e`            | 后移动一个单词，光标停留在单词末尾               |
| `b`            | 前移动一个单词，光标停留在单词开头               |
| `g g`、`G`     | 分别移动光标到文件头与文件末尾                   |

  如果安装了`ivy`层，可以使用如下跳跃，很有用：

| 快捷键    | 描述                       |
| --------- | :------------------------- |
| `SPC j b` | 返回到上一个位置(跳跃之后) |
| `SPC j j` | 跳跃到指定字符             |
| `SPC j l` | 跳跃到指定行               |
| `SPC j w` | 跳跃到指定单词             |
| `SPC j d` | 跳转到当前目录列表         |
| `SPC j D` | 跳转到当前目录列表(新窗口) |

- 跳转到配对为止

在vim模式下， 非编辑模式，可以按`%`跳转到配对为止，例如`()[]{}`等配对位置。

- 搜索操作

快捷键`SPC s s`可以在当前文件中进行搜索，后面可以加上需要选择的行号。

- 选择区域

很多时候需要选中区域，然后操作(复制/删除/加注释等)，按`v`则可以进入选择模式，然后跳转到任意结束位置即可。

1. 选中单词：

`M f`前进一个单词；`M b`回退一个单词；`y w`复制单词

2. 选中函数

`C M h`选中当前函数

3. 选中行

`$`跳到行尾；`^`跳到行首；`y y`复制行

- 编辑操作

| 快捷键 | 描述                               |
| ------ | ---------------------------------- |
| `i`    | 光标前插入                         |
| `I`    | 行首插入                           |
| `a`    | 光标后插入                         |
| `A`    | 行尾插入                           |
| `o`    | 当前行下面另起一行，并变成插入模式 |
| `O`    | 当前行上面另起一行，并变成插入模式 |

- 删除操作

| 快捷键 | 描述                     |
| ------ | ------------------------ |
| `x`    | 删除字符(光标后)         |
| `X`    | 删除字符(光标前)         |
| `dw`   | 删除单词(光标后)         |
| `dd`   | 删除当前行(也相当于剪切) |

- 复制/粘贴操作

| 快捷键 | 描述                                     |
| ------ | ---------------------------------------- |
| `yy`   | 复制当前行                               |
| `nyy`  | 复制当前行向下n行                        |
| `p`    | 粘贴到光标下一行                         |
| `P`    | 粘贴到光标上一行                         |
| `u`    | 撤销上一步操作                           |
| `C r`  | 重做上一步，例如想多复制几次，就可以使用 |

- **单行/多行缩进、缩出操作**

单行缩进/缩出：`>>  或  <<`即可，即连续按两次<或>。

多行缩进/缩出：V模式选中多行情况下，按`< 或 >`实现。

- **Vim可视模式**

为了方便选中文本，Vim引入了可视模式，有如下三种可视模式：

1. 用`v`进入字符可视化模式，文本选择以字符为单位。
2. 用`Shift v`进入行可视化模式，文本选择以行为单位。
3. 用`Ctrl v`进入块可视化模式，可以选择一个矩形内的文字。

- **Vim多行注释和解注释**

多行注释：

1. `Ctrl` + `v` 进入 *VISUAL BLOCK* 区块选择模式
2. 选择需要注释的行（*hjkl* 移动区块进行选择）
3. `Shift` + `i`（大写I) 进入插入模式，光标跳至行首
4. 键入注释符（如`#`）
5. `Esc`退出插入模式
6. **短暂延迟**后，所有被选中的行首都被添加了注释符

多行解注释：

1. `Ctrl` + `v` 进入 *VISUAL BLOCK* 区块选择模式
2. 选择行首注释符（*hjkl* 移动区块进行选择）
3. `x`或`d`删除注释符

- **Vim查找与替换**

在normal模式下，按`/`进入查找模式，输入需要查找的字符串，回车则跳转到第一个匹配位置，`n`跳转下一个匹配位置，`N`跳转到上一个。

> 默认是大小写不敏感的，输入字符后输入`\C`则大小写敏感

替换如下语法`:{作用范围}s/{目标}/{替换}/替换标志`

> 作用范围有当前行、全局、V模式选中区域三种：
>
> - 当前行：输入s即可
> - 全局：输入%s
> - 选中区域：V模式选中区域后，按`:`会自动补全，后续输入s即可

替换标志：

> `g`: 表示替换所有出现的
>
> 空替换标志：即不输入`/g`表示只替换光标后第一个匹配项
>
> `I`: 表示大小写敏
>
> 加入字符`c`：表示需要一个一个确认，自动跳转到第一个替换项。

### 窗口操作

每个窗口都有一个编号，通过快捷键`SPC number`可以快速切换到对应的窗口。

| 快捷键    | 描述                                               |
| --------- | -------------------------------------------------- |
| `SPC w d` | 删除当前窗口                                       |
| `SPC w s` | 水平分割当前窗口                                   |
| `SPC w S` | 水平分割当前窗口并聚焦到新窗口                     |
| `SPC w v` | 垂直分割当前窗口                                   |
| `SPC w V` | 垂直分割当前窗口并聚焦到新窗口                     |
| `SPC w m` | 最大化buffer，对于临时弹出一个buffer，需要返回有用 |

### 编程语言相关操作

`,`可以看到有关编程语言的一些操作，例如格式化，执行等操作

| 快捷键           | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| `, c c`          | 执行当前文件代码(针对Python，Go运行则是`, x x`采用go run运行main函数) |
| `C o`            | 回到跳转前位置或者buffer                                     |
| `, g g`或`, g G` | 跳转到函数定义，或者在新窗口跳转到函数定义                   |

> 注：
>
> - python可以使用`SPC SPC `输入flycheck list打开errors buffer，然后在里面通过j k选择某一个错误，代码buffer中光标会跳转到错误位置
> - python环境中，可以使用`SPC SPC`然后输入yapf选择格式化buffer或者region(V模式选中的块)来进行格式化。如果在python层中配置了`python-formatter 'yapf`，则可以使用`, =`来格式化。



