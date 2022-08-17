---
author: "Narcissus"
title: "Python版本管理"
date: "2021-10-21"
description: "介绍Python版本管理及虚拟环境。"
tags: ["技术","Python"]
categories: ["技术"]
---

## virtualenv

virtualenv 所要解决的是同一个库不同版本共存的兼容问题。例如项目 A 需要用到 requests 的 1.0 版本，项目 B 需要用到 requests 的 2.0 版本。如果不使用工具的话，一台机器只能安装其中一个版本，无法满足两个项目的需求。virtualenv 的解决方案是**为每个项目创建一个独立的虚拟环境，在每个虚拟环境中安装的库，对其他虚拟环境完全无影响**。所以就可以在一台机器的不同虚拟环境中分别安装同一个库的不同版本。

![ScreenShot2021-10-21 10.11.25](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-10/ScreenShot2021-10-21%2010.11.25.png)

## pyenv

pyenv 不是用来管理同一个库的多个版本，而是用来管理一台机器上的多个 Python 版本。主要解决开发中有的项目需要用Python 2.x，有的项目需要用Python 3.x的场景。`pyenv` 可以改变全局的 `Python` 版本，安装多个版本的 `Python`， 设置目录级别的 `Python` 版本，还能创建和管理 `virtual python environments` 。**pyenv 使用了垫片的原理，可以做到进入项目目录自动选择 Python 版本**。

### 安装

使用homebrew包管理软件进行安装，安装前先用命令`brew update`跟新新版本的 homebrew。

- 安装软件包：`brew install XXXX`
- 更新软件包：`brew upgrade XXXX`,如果不写具体的软件包，就是更新所有可更新的软件包。

使用命令`brew install pyenv`安装，效果如下：

![ScreenShot2021-10-21 09.49.21](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-10/ScreenShot2021-10-21%2009.49.21.png)

检查是否安装完成，执行命令`pyenv -v`,会显示pyenv安装的版本:

![ScreenShot2021-10-21 09.51.19](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-10/ScreenShot2021-10-21%2009.51.19.png)

> 需要配置环境变量才能生效，否则会出现`pyenv local XXXX`修改版本后查看版本，依然没有变化。
>
> ```shell
> echo 'eval "$(pyenv init --path)"' >> ~/.zprofile
> echo 'eval "$(pyenv init -)"' >> ~/.zshrc
> ```

### pyenv使用

使用`pyenv commands`可以显示所有可使用的命令

![ScreenShot2021-10-21 09.54.55](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-10/ScreenShot2021-10-21%2009.54.55.png)

还可以使用命令:`pyenv shell 版本号`指定当前shell中临使用某一个Python版本。

> 注意：优先级为：pyenv shell > pyenv local > pyenv global > system。即优先使用 pyenv shell 设置的版本，三种级别都没设置时才使用系统安装的版本。

### pyenv virtualenv

pyenv 解决的是多个 Python 的版本管理问题，virtualenv 解决的是同一个库的版本管理问题。但如果两个问题都需要解决呢？分别使用不同工具就很麻烦了，而且容易有冲突。为此，pyenv 引入了了 virtualenv 插件，可以在 pyenv 中解决同一个库的版本管理问题。

![ScreenShot2021-10-21 10.24.47](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-10/ScreenShot2021-10-21%2010.24.47.png)

## Jupyter notebook

Jupyter Notebook是基于网页的用于交互计算的应用程序。其可被应用于全过程计算：开发、文档编写、运行代码和展示结果。

使用命令`pip install jupyter `安装，后面可以添加参数`-i 镜像源`切换本次安装的镜像。
