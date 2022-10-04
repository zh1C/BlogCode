---
author: "Narcissus"
title: "Git日常操作总结"
date: "2022-09-27"
description: "总结不常用Git操作"
tags: ["Git"]
categories: ["Git"]

---

## 1. 移除已经commit的文件

将某个文件已经提交了，这时候需要移除这个文件，并且将其添加到.gitignore中，让git不追踪该文件，可以使用如下命令：
`git rm -r --cached 文件名`移除已经提交的文件，然后再将该文件添加到gitignore中，最后再commit即可。