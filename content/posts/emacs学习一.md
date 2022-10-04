---
author: "Narcissus"
title: "Emacs学习"
date: "2022-09-25"
lastmod: "2022-10-04"
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

按 SPC f e d 可以快速打开 .spacemacs 文件，然后使用C-s进行搜索，搜索到`dotspacemacs-line-numbers`,修改为t表示显示行号。:wq保存并退出。

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

## 三、常用快捷键

### 光标移动快捷键

| 快捷键    | 描述                                             |
| --------- | :----------------------------------------------- |
| `h`       | 左移光标                                         |
| `j`       | 下移光标                                         |
| `k`       | 上移光标                                         |
| `l`       | 右移光标                                         |
| `H`       | 移动光标到顶部                                   |
| `L`       | 移动光标到底部                                   |
| `SPC j 0` | 移动光标到行开始位置(并在前一个位置设置一个标记) |
| `SPC j $` | 移动光标到行末尾位置(并在前一个位置设置一个标记) |

  如果安装了`ivy`层，可以使用如下跳跃，很有用：

| 快捷键    | 描述                       |
| --------- | :------------------------- |
| `SPC j b` | 返回到上一个位置(跳跃之后) |
| `SPC j j` | 跳跃到指定字符             |
| `SPC j l` | 跳跃到指定行               |
| `SPC j w` | 跳跃到指定单词             |
