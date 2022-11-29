---
title: "org mode基本使用"
description: "org mode的一些基本操作及对应快捷键。"
date: 2022-11-19T00:00:00+08:00
lastmod: 2022-11-19T14:50:41+08:00
tags: ["emacs"]
categories: ["emacs"]
draft: false
author: "Narcissus"
---

## 日程管理 {#日程管理}

可以使用org进行日程管理，org mode是一个用文本方式来快速高效的做笔记、维持代办事项和做项目计划的模式。

| 快捷键            | 描述                                         |
|----------------|--------------------------------------------|
| \* **\* \*\***    | 分别对应几级标题，和markdown类似             |
| TAB               | 可以折叠或展开当前标题下的内容               |
| \* TODO something | 标题后输入TODO即可变成代办事项               |
| C-c C-t或者t      | 将当前项的状态在(unmark) -&gt; TODO -&gt; DONE之间循环切换 |
| , d s             | 可以在当前TODO事项添加scheduled日期          |

注意: **如果需要用t来快速切换代办事项状态，需要在org layers中设置 `org-want-todo-bindings` 为t**,
后续可以使用T来插入新的TODO事项，M-t在当前子标题中插入新的TODO事项。

集成使用org TODO,在 `user-config` 中添加如下配置：

```emacs-lisp
(setq org-agenda-files (list "filePath"))
```

注意: **接收的文件路径应该是一个列表，即使只有一个文件路径。**
常用快捷键操作:

| 快捷键    | 描述         |
|--------|------------|
| SPC a o o | 打开org agenda |
| t         | TODO事项状态切换 |
| q         | 退出         |
| SPC a o a | 这一周的agenda列表 |


## eval-org-mode {#eval-org-mode}

eval-org-mode可以使org的一些操作像vim一样，org layer中已经自动添加了这个package。可以使用如下快捷键在标题之间移动:

| 快捷键 | 描述         |
|-----|------------|
| gj/gk | 下一个/上一个标题或元素 |
| gh/gl | 父/子标题或元素 |
| gH    | 根标题       |


## 标签 {#标签}

org mode 的层次结构支持标签，标签具有继承属性，即子结构会自动继承上层的标签。标签的语法为 **:tags:**, 同时可以定义多个标签。
Spacemacs中创建或修改标签的快捷键是，在光标所在当前标题使用快捷键 **, i t** 。


## org table 中英文无法对齐问题 {#org-table-中英文无法对齐问题}

org mode的表格中，中英文混合使用会无法对其，有一个package可以解决该问题，就是valign，现在不需要单独安装该package,org layer中可以直接通过变量配置：

```emacs-lisp
(org :variables
     org-enable-valign t)
```


## org mode缩进显示 {#org-mode缩进显示}

默认情况下，org文件大纲没有缩进显示，可以通过 `org-indent-mode` 命令来手动切换开关，如果想要所有文件打开就采用缩进显示，则需要在 `user-config` 中进行如下配置:

```emacs-lisp
(setq org-startup-indented t)
```


## visual-line-mode模式 {#visual-line-mode模式}

在org mode中如果一行过长没有办法在完全显示，可以手动开启 `visual-line-mode` 模式，该模式会自动将一行以多行的形式进行展示。
