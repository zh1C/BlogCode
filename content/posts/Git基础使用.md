---
author: "Narcissus"
title: "Git基础使用"
date: "2021-09-03"
description: "学习Git命令行的基本使用。"
tags: ["Git"]
categories: ["Git"]
---

## 一、Git简介

Git是分布式版本控制系统。

### 集中式VS分布式

- 集中式版本控制系统，版本库集中存放在中央服务器，干活的时候先从中央服务器获取最新版本；干完活再把更新推送到中央服务器。**最大缺点就是必须联网才能工作。**
- 分布式版本控制系统，没有”中央服务器“，每个人的电脑都是一个完整地版本库。

实际使用分布式版本控制系统的时候，很少在两个个人电脑之间推送版本修改。通常也有一台充当”中央服务器“的电脑，作用是方便”交换“大家的修改。

## 二、使用Git

安装完Git后，需要设置名字和email

> `git config --global user.name "your name"`
>
> `git config --global user.email "email@example.com"`
>
> **注意`--global`参数，表示这台计算机上所有Git仓库都使用这个配置。**

### 创建版本库

在相应目录下使用`git init`命令即可把这个目录变成Git可以管理的仓库。这样目录下会多出**一个`.git`的目录，用来跟踪管理版本库**的。

1. 将文件添加到暂存区：

> `git add [file1] [file2] ...` 添加一个或多个文件到暂存区
>
> `git add [dir]` 添加指定目录(包括子目录)到暂存区
>
> `git add .`添加当前目录下所有文件到暂存区

2. 将暂存区文件提交到仓库：

> `git commit -m [message]` 提交暂存区到本地仓库，message可以是一些备注信息
>
> `git commit [file1] [file2] ... -m [message]` 提交指定文件

### 远程仓库使用

首先在GitHub上创建一个仓库，然后将本地仓库与远程仓库关联，最后可以把本地仓库内容推送到GitHub仓库。

1. 关联远程仓库GitHub

> `git remote add origin https://github.com/用户名/github远程仓库名 ` 添加后，远程仓库名字就是`origin`,**这个名字是Git默认叫法，可以更改**
>
> `git remote -v`显示所有远程仓库
>
> `git remote show [remote]`显示某个远程仓库的信息
>
> `git remote rm name`删除远程仓库
>
> `git remote rename old_name new_name`修改仓库名

2. 把本地仓库内容推送到远程仓库

> `git push -u origin master`把当前分支`master`推送到远程，第一次推送需要加`-u`参数，此时Git不但会把本地`master`分支内容推送到远程新的`master`分支，还会把两者关联起来，后续推送或拉取可以简化命令。
>
> `git push <远程仓库名> <本地分支名>:<远程分支名>`如果本地分支名和远程分支名相同，则可以简写`git push <远程仓库名> <本地分支名>`

- 从远程仓库克隆

> `git clone -b 分支名 [url] [name]`默认情况下，Git会按照提供的url所指向的项目名称创建本地项目目录，但也可以修改项目名称([name]中修改，默认的话不写)。

## 三、仓库管理

- 查看仓库当前状态，显示有变更的文件

> `git status` 查看挡墙仓库状态，输出的命令很详细，但有些繁琐
>
> `git status -s`获取简短的输出结果
>
> > 新添加的未跟踪文件前面有`??`标记
> >
> > 新添加到暂存区中的文件前面有`A`标记
> >
> > 修改过的文件前面有`M`标记，出现在右边的`M`表示文件被修改了但是还没放入暂存区，出现在靠左边表示文件被修改了并放入了暂存区。

- 比较文件的不同

> `git diff [file]` 显示暂存区和工作区的差异 
>
> `git diff --cached [file]` 显示暂存区和上一次提交到仓库的文件的差异
>
> **注意：比较后按q退出**

### 版本回退

- 查看历史提交记录

> `git log` ,`--oneline`选项来查看历史记录的简介版本
>
> `git blame <file>`查看自定文件的修改记录

- 回退版本

> `git reset [--soft | --mixed | --hard] [HEAD]`
>
> **注意：HEAD表示当前版本，HEAD~1表示上一个版本，以此类推。**
>
> 可以使用`git reflog`查看命令历史。

### 工作区/暂存区/本地仓库

- 工作区(working directory):就是电脑能看到的目录

- 版本库(repository):工作区有一个隐藏目录`.git`，就是Git版本库。Git的版本库存放了很多东西，最重要的是stage的暂存区，还有Git为我们自动创建的第一个分支`master`,以及指向`master`的一个指针叫`HEAD`。

  > ![ScreenShot2021-08-29 15.18.00](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025430.png)
  >
  > Git常用的六个命令：**git clone、git push、git add 、git commit、git chechout、git pull**如下：
  >
  > ![ScreenShot2021-08-29 14.18.17](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025431.png)

**注意：Git管理的是修改，而不是管理的文件。**

我们对一个文件进行以下操作：第一次修改->`git add`->第二次修改->`git commit`。可以发现只有第一次修改被提交了，第二次修改没有被提交。

### 撤销修改

- 如果修改了工作区某个文件，还没添加到缓存区，想直接丢弃工作区修改时，用命令`git checkout -- file`。 **注意：`--`很重要，没有就会变成”切换到另一个分支“的命令。**
- 如果修改了工作区某个文件，还添加到了暂存区，想丢弃修改，分两步：第一步用`git reset HEAD <file>`回到上一个情景，然后使用上一个情景的操作。
- 如果修改后已经提交到了本地仓库，想撤销本次提交，采用版本回退。

### 删除文件

如果工作区和本地仓库同步的时候，删除了工作区某个文件，`git status`可以看到哪些文件被删除了。此时有两个选择：

1. 确实要从版本库中删除该文件，就用`git rm`命令删除，并且`git commit`,这样文件就从版本库中删除了。

   > `git rm <file>`将从工作区和暂存区中删除文件。如果删除之前修改过并且已经放到了暂存区，则必须用强制删除选项`-f`。

2. 另一种情况就是删错了，版本库里面还有，所以可以轻松把误删文件恢复到最新版本。

使用命令`git checkout -- <file>`。**但是这种方式会丢失最近一次提交后你修改的内容。**

## 四、分支管理

### 创建与合并分支

在Git里，主分支是`master`，严格来说`HEAD`指向当前分支，然后`master`才指向提交。

> ![ScreenShot2021-08-29 14.19.24](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025433.png)
>
> 当我们创建新分支时，例如`dev`,Git新建了一个指针叫`dev`,指向`master`相同的提交，再把`HEAD`指向`dev`，就表示当前分支在`dev`上：
>
> ![ScreenShot2021-08-29 15.21.23](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025433-1.png)
>
> **现在开始，对工作区的修改和提交就针对`dev`分支了，**比如新提交一次，`dev`指针往前移动一步，而`master`指针不变：
>
> ![ScreenShot2021-08-29 15.21.30](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025435.png)
>
> 在`dev`分支上得工作完成了，就可以把`dev`合并到`master`上，最简单的方式就是直接把`master`指向`dev`的当前提交，就完成了合并：
>
> ![ScreenShot2021-08-29 15.21.36](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025437.png)
>
> 合并完成后，可以删除`dev`分支。删除分支就是把对应指针删除掉。

#### 命令实现

- 创建、切换分支

> `git branch <name>`创建分支
>
> `git branch` 查看分支(会列出所有分支，当前分支前面会标*号)

> `git checkout <name>`或者`git switch <name>`切换分支

> `git checkout -b <name>`或者`git switch -c <name>` 创建并切换分支
>
> **推荐使用git switch命令**

- 合并分支

> `git merge <name>`合并某指定分支到当前分支

但如果两个分支各自有自己的提交，Git无法自动合并分支时，必须首先解决冲突。

**解决冲突就是把Git合并失败的文件手动编辑为我们希望的内容，再提交。**

通常，合并分支时，如果可能Git会用`Fast forward`模式。但这样删除分支后，会丢掉分支信息，使用`--no-ff`会强制禁用`Fast forward`模式

> ![ScreenShot2021-08-29 15.22.22](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/blog/025438.png)
>
> `git merge --no-ff -m [描述] <name>`因为本次合并要创建新的commit,所以要加上`-m`参数，把commit描述写上去。

- 删除分支

> `git branch -d <name>`删除指定分支

### 多人协作

多人协作时，大家都会向分支上推送各自的修改，这时**会出现远程分支比本地更新，就需要先用`git pull`抓取远程新提交并试图合并。**如果有冲突，要先处理冲突。

命令格式如下：

`git pull -rebase <远程主机名> <远程分支名>:<本地分支名>`如果远程分支与当前分支合并，则冒号后面的部分可以省略不写。

> 注意：rebase操作可以把本地未push的分叉提交历史整理成直线，目的是使得我们在查看历史提交的变化时更容易。

## 五、标签管理

标签(tag)就是一个让人容易记住的有意义的名字，他跟某个commit绑定在一起。**如果这个commit即出现在master分支上，又出现在dev分支，那么在这两个分支上都可以看到这个标签。**

> 命令`git tag <tagname>`用于新建一个标签，默认为`HEAD`，也可以指定一个commit id。
>
> 命令`git tag -a <tagname> -m "blabla..."`可以指定标签信息。
>
> 命令`git tag`可以查看所有标签。

- 删除标签

> `git tag -d <tagname>`可以删除一个本地标签
>
> `git push origin <tagname>`可以推送一个本地标签到远程仓库
>
> `git push origin --tags`推送全部未推送的标签到远程仓库
>
> `git push origin :refs/tags/<tagname>`删除远程仓库中某一个标签

## 六、特殊使用

### 忽略特殊文件

有时候需要忽略Git工作目录中的一些文件，但每次`git status`时都会显示`Untracked files...`，可以使用对应方法解决。**在Git工作区的根目录下创建一个特殊的`.gitignore`文件，然后把要忽略的文件名填进去，Git则会自动忽视这些文件。**

GitHub提供了各种配置文件的模板，可以任意组合使用。[https://github.com/github/gitignore](https://github.com/github/gitignore)

- `.gitignore`文件本身要放到版本库里，并且可以对`.gitignore`做版本管理。
