---
author: "Narcissus"
title: "Emacs学习"
date: "2022-09-25"
lastmod: "2022-09-25"
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

### 常用配置修改

- 显示行号

按 SPC f e d 可以快速打开 .spacemacs 文件，然后使用C-s进行搜索，搜索到`dotspacemacs-line-numbers`,修改为t表示显示行号。:wq保存并退出。

- 显示时间

在.spacemacs文件中搜索`dotspacemacs/user-config`选项，添加`(display-time-mode t)`表示显示时间配置。