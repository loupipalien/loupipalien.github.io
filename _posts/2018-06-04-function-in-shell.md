---
layout: post
title: "Shell 中的函数"
date: "2018-06-04"
description: "Shell 中的函数"
tag: [shell, linux]
---

### 函数定义语法
```
[ function ] funname [()] {
    action;
    [return int;]
}
```
- 可以带 function fun() 定义, 也可以直接 fun() 定义, 不带任何参数
- 参数返回, 可以使用 return 明确返回 [ 0 - 255 ], 如果不指定则返回函数中最后一条命令的结果作为返回值

### 函数参数
|参数形式|说明|
|--|--|
|$0|当前脚本的文件名|
|$n|在函数内部获取参数值, n 代表第几个参数, $10 不能获取第 10 个参数, 需要用 ${10} 来获取|
|$#|传递到脚本的参数个数|
|$*|传递给脚本函数的所有参数, 使用时加引号, 会以一个单字符串显示所有向脚本传递的参数|
|$@|与 $* 相同, 但是使用时加引号, 仍然在引号中返回每个参数, 可遍历|
|$$|脚本运行的当前进程 ID 号|
|$!|后台运行的最后一个进程的 ID 号|
|$?|上个命令的退出状态, 或函数的返回值|
|$-|显示 Shell 使用的当前选项 (himBH), 与 set 命令功能相同|

>**参考:**
[Shell 函数](http://www.runoob.com/linux/linux-shell-func.html)  
[Shell特殊变量](https://blog.csdn.net/u011341352/article/details/53215180)
