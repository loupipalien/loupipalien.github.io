---
layout: post
title: "Shell 中的条件判断语句"
date: "2018-06-04"
description: "Shell 中的条件判断语句"
tag: [shell, linux]
---

### 条件判断语法
```
if condition then
    do something
[
elif condition then
    do something
else
    do something
]
fi
```
#### 复杂逻辑判断符
|表达式|说明|
|--|--|
|-a|与|
|-o|或|
|!|非|

#### 字符串判断
|表达式|说明|
|--|--|
|str1 = str2|两个字符串相同时为真|
|str1 != str2|两个字符串不相同时为真|
|-n str|当串的长度不为 0 时为真 (空串)|
|-z str|当串的长度为 0 时为真 (非空串)|
|str|当串为非空时为真|

#### 数字的判断
|表达式|说明|
|--|--|
|int1 = int2|两个数相同时为真|
|int1 != int2||
|-n str|当串的长度不为 0 时为真 (空串)|
|-z str|当串的长度为 0 时为真 (非空串)|
|str|当串为非空时为真|

#### 文件的判断
|表达式|说明|
|--|--|
|-a file|file 存在则为真|
|-b file|file 存在且是一个块特殊文件则为真|
|-c file|file 存在且是一个字特殊文件则为真|
|-d file|file 存在且是一个目录则为真|
|-e file|file 存在则为真|
|-f file|file 存在且是一个普通文件则为真|
|-g file|file 存在且设置了 SGID 则为真|
|-h file|file 存在且是一个符号链接则为真|
|-k file|file 存在且设置了粘滞位则为真|
|-p file|file 存在且是一个有名管道则为真|
|-r file|file 存在且是可读的则为真|
|-s file|file 存在且大小不为 0 则为真|
|-t fd|文件描述符 fd 打开且指向一个终端则为真|
|-u file|file 存在且设置了 SUID 则为真|
|-w file|file 存在且是可写的则为真|
|-x file|file 存在且是可执行的则为真|
|-O file|file 存在且属于有效用户 ID 则为真|
|-G file|file 存在且属于有效用户组则为真|
|-L file|file 存在且是一个符号链接则为真|
|...|...|

>**参考:**
[Shell脚本IF条件判断和判断条件总结](https://www.cnblogs.com/mch0dm1n/p/5675219.html)
