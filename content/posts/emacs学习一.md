---
cauthor: "Narcissus"
title: "Emacs学习"
date: "2022-10-07"
lastmod: "2022-11-15"
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

相关介绍[链接地址](https://www.emacswiki.org/emacs/AutoSave)

在`user-config`中进行如下配置，可以配置自动保存的频率：

>`auto-save-interval`与`auto-save-timeout`设置自动保存间隔。

**注意：此处的自动保存应该理解为生成备份文件，用于不小心退出来emacs后来恢复。如果想要实现文件自动保存，可以通过`SPC SPC`后输入如下命令启动`auto-save-visited mode`**

- 拼写检查层

拼写检查层默认在layers中配置：`spell-checking`,可以通过如下配置默认不打开拼写检查：

```lisp
(spell-checking :variables spell-checking-enable-by-default nil)
```

> 后续如果需要打开，可以通过`SPC t S`进行手动开启。

### 查看帮助文档

需要会查看帮助文档，有如下常用快捷键

| emacs快捷键 | spacemacs快捷键 | 描述             |
| ----------- | --------------- | ---------------- |
| `C h v`     | `SPC h d v`     | 查看变量的文档   |
| `C h f`     | `SPC h d f`     | 查看命令的文档   |
| `C h k`     | `SPC h d k`     | 查看快捷键的文档 |

**注意：emacs有官方教程，可以通过命令`SPC SPC help-with-tutorial`打开教程进行学习**

## 三、开发环境配置

### Python开发环境配置

在对应的env中，安装如下依赖包即可，然后再spacemacs中添加`python`层，

`pip install autoflake isort yapf flake8 python-lsp-server importmagic epc`

## 四、常用快捷键

### 光标移动相关操作

> 注意：Vim相关操作教程学习，可以在命令行输入`vimtutor`进行学习。
>
> **vim许多命令都有如下结构：`一个操作符  一个动作构成`，即`d motion`,而在动作前输入数字表示该动作重复次数。**
>
> `opreator [number] motion`
>
> - Opreator:操作符，代表要做的事
> - Number: 可以附加数字，代表要重复操作的次数
> - Motion:动作，代表在文本上的移动操作

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
| `SPC j i` | 跳转到函数                 |

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

| 快捷键           | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| `x`              | 删除字符(光标后)                                             |
| `X`              | 删除字符(光标前)                                             |
| `dw`             | 删除单词(光标后)                                             |
| `dd`             | 删除当前行(也相当于剪切)                                     |
| `d`配合`SPC j j` | 按d后，再跳转到任意字符，可以实现删除按d位置到跳转位置之间的所有字符 |

> 除了删除`d`可以这样使用，复制`y`也可以实现这样的操作，并且跳转支持所有的`SPC j`开头的快捷键。这样就不需要用v模式来选择再进行相关操作了。

- 复制/粘贴操作

| 快捷键  | 描述                                     |
| ------- | ---------------------------------------- |
| `yy`    | 复制当前行                               |
| `nyy`   | 复制当前行向下n行                        |
| `p`     | 粘贴到光标下一行                         |
| `P`     | 粘贴到光标上一行                         |
| `u`,`U` | 撤销上一步操作,撤销对本行的操作          |
| `C r`   | 重做上一步，例如想多复制几次，就可以使用 |

- 替换/修改命令

| 快捷键              | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| `r [替换后的字符]`  | 替换掉光标位置的字符                                         |
| `R`                 | 连续替换多个字符，**替换和插入差不多，只是替换每输入一个字符都会删除一个已有字符** |
| `c [number] motion` | 更改当前光标位置到动作指示位置之间的文本，例如`ce`修改当前位置到单词结尾，`c$`修改当前位置当行末尾。 |



### 多行编辑操作

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
>
> 搜索匹配后是高亮显示：可以输入`:noh`结束高亮

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

- **iedit mode**

spacemacs可以通过`iedit mode`来实现同时快速编辑重复项，相当于上述的Vim中的查找并替换。

进入iedit-mode模式下，就可以直接进行批量修改。

| 快捷键    | 描述                                    |
| --------- | --------------------------------------- |
| `SPC s e` | normal模式或者v模式进入到iedit-mode模式 |
| `jk`      | 从该模式退出                            |
| `Tab`     | 切换当前选项为选中、未选中              |
| `n`       | 下一个选中项                            |
| `N`       | 上一个选中项                            |
| `F`       | 限定选中项范围为当前函数内              |

> 可方便用于同时修改函数名称和该函核数的调用

### 搜索操作

搜索操作有两种方式：

- Vim的搜索

Vim的搜索上面已经介绍，相对简单。

- spacemacs扩展

spacemacs的搜索操作都是以`SPC s`开头的快捷键，有如下常用快捷键

| 快捷键                   | 描述                                          |
| ------------------------ | --------------------------------------------- |
| `SPC s g 后续操作`       | 使用grep进行相关搜索，包括搜索项目、buffers等 |
| `SPC *`                  | 智能搜索当前光标下的单词，在项目中搜索        |
| `SPC /`等同于`SPC s g p` | 在项目中搜索内容                              |

### 窗口操作

每个窗口都有一个编号，通过快捷键`SPC number`可以快速切换到对应的窗口。

| 快捷键    | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| `SPC w d` | 删除当前窗口                                                 |
| `SPC w s` | 水平分割当前窗口                                             |
| `SPC w S` | 水平分割当前窗口并聚焦到新窗口                               |
| `SPC w v` | 垂直分割当前窗口                                             |
| `SPC w V` | 垂直分割当前窗口并聚焦到新窗口                               |
| `SPC w m` | 最大化buffer，对于临时弹出一个buffer，需要返回有用           |
| `SPC w H` | 将当前窗口移到最左边，**注意：当有两个上下分屏的窗口时，可以快速调整为左右分屏** |
| `SPC w L` | 将当前窗口移动到最右边                                       |
| `SPC w J` | 将当前窗口移动到最下边                                       |
| `SPC w K` | 将当前窗口移动到最上边                                       |

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

### Dired-mode

spacemacs中文件或者文件夹相关操作可以进入`dired-mode`模式，在一个buffer中，按`SPC f j`就可以跳转到文件目录(如果是目录，则会跳转到上一层目录)，此时就是dired-mode模式，有如下相关操作：

> `SPC a d`也可以进入该模式，然后选中某个文件夹Enter进入。
>
> 该模式下可以适配Vim相关操作，移动、搜索等。

| 快捷键   | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| `C`      | 拷贝文件，然后会选择需要拷贝到的目录并输入名称即可           |
| `R`      | 重命名文件，可以改变文件路径                                 |
| `D`      | 删除文件                                                     |
| `Enter`  | 进入文件                                                     |
| `! open` | 用外部程序打开文件，open会用默认程序打开，也可以输入程序名称进行打开 |

### org-mode

#### 基本使用

可以使用org文件进行日程管理，Org 是一个用文本方式来快速高效地做笔记、维持待办事项和做项目计划的模式。

| 快捷键或代码       | 描述                                                  |
| ------------------ | ----------------------------------------------------- |
| `*   **   ***`     | 分别对应一级标题，二级标题，三级标题                  |
| `Tab`              | 可以折叠或展开当前标题下的内容                        |
| `* TODO something` | 标题后输入TODO即可变成代办事项                        |
| `C c C t` 或者`t`  | 将当前项的状态在（unmarked）->TODO->DONE 之间循环切换 |
| `, d s`            | 可以在当前TODO事项添加scheduled日期                   |

> 注意：如果org layer的变量`org-want-todo-bindings`是True，则可以使用`t`来循环当前标题的TODO状态。
>
> `T`来插入新的DODO事项。`M t`在当前子标题中插入新的TODO事项。

#### Todo

集成使用org-todos,在`user-config`中添加如下配置，

```lisp
(setq org-agenda-files (list "~/orgFiles/todos.org"))
```

> 注意：接收的文件路径应该是一个列表，即使只有一个文件路径

| 快捷键      | 描述                                        |
| ----------- | ------------------------------------------- |
| `SPC a o o` | 打开org agenda                              |
| `t`         | 显示所有的TODOs，在TODO上则可以循环TODO状态 |
| `q`         | 退出                                        |
| `SPC a o a` | this week org agenda list                   |

### Treemacs

treemacs是一个文件导航栏layer，常见如下操作

| 快捷键         | 描述                                           |
| -------------- | ---------------------------------------------- |
| `SPC 0`        | 打开或者聚焦到Treemacs buffer                  |
| `SPC f t`      | 显示或隐藏Treemacs buffer                      |
| `Tab`或`Enter` | 展开或则折叠路径，`Enter`还可以打开文件        |
| `o o`          | 打开一个文件，不进行拆分窗口而是直接使用下一个 |
| `o v`          | 打开一个文件，进行水平拆分                     |
| `o h`          | 打开一个文件，进行垂直拆分                     |
| `c d`          | 创建一个文件夹                                 |
| `c f`          | 创建一个文件                                   |
| `d`            | 删除文件或者文件夹                             |
| `R`            | 重命名文件或者文件夹                           |
| `q`或者`Q`     | 隐藏Treemacs buffer, 关闭Treemacs buffer       |

## 五、使用小tips

### Go-mode缩进异常

在编辑Golang相关文件时，输入大括号后Enter换行，缩进不会生效，并且再次按Tab缩进的时候在Message buffer中会出现如下代码

```lisp
auto trimmed 1 chars
```

发现是一个bug，需要在设置中将某个变量设置为nil

```lisp
dotspacemacs-use-clean-aindent-mode nil
```

相关Github issue讨论如下：[链接](https://github.com/syl20bnr/spacemacs/issues/6520)

### Org-mode缩进显示

默认情况下，org文件大纲没有缩进显示，可以通过`org-indent-mode`命令来手动开关，如果想要所有文件打开就采用缩进显示，则需要在user-config中进行如下配置：

```lisp
(setq org-startup-indented t)
```

