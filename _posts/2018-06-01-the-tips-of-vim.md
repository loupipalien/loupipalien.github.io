---
layout: post
title: "Vim 的小技巧"
date: "2018-06-01"
description: "Vim 的小技巧"
tag: [vim, linux]
---

**环境:** CentOS-7.5-x86_64

### 多文件操作

#### 同时打开多个文件并切换编辑
- 使用 vim 打开文件, 假定为 file.txt
- 使用 : 进入底行模式, 输入 sp 横向切分一个窗口, 或者使用 vsp 纵向切分一个窗口
- 在底行模式下输入 e another_file.txt 在其中一个窗口打开另一个文件
- 在命令模式下按 ctrl + w, 再按 w 可以切换到另一个窗口

#### Vim 跨文件复制
- 使用 vim 打开文件, 假定为 file.txt
- 在命令模式下输入 "a3yy, 表示从当前向下复制 3 行到剪贴板 a 中, 剪贴板名可以是 26 个字母中任意一个, 即意味着可以有多个剪贴板
- 退出 file.txt, 再使用 vim 打开 another_file.txt
- 在命令模式下输入 "ap 即可粘贴剪贴板中的内容

>**参考:**
[Vim多文件操作及复制到系统剪贴板](https://blog.csdn.net/I_Moo/article/details/48653107)
