---
layout: post
title: "Shell 中的数值计算"
date: "2018-06-08"
description: "Shell 中的数值计算"
tag: [shell, linux]
---

Shell 中数学表达式与其他语言中不同, 因为 Shell 把所有变量都视为字符串, 所以 a=1+2, a 并不等于 3, 而是 1+2 的字符串; 在 Shell 计算有如下方案

### 运算符 [] 或 (())
```
a=$[1+2]
b=$((1+2))
```
中括号或两个小括号里的表达式可以视为一个变量值, $ 只是取变量值; 使用时需注意以下几点
- 此运算只支持整数, 如果有小数则会报错, 结果时小数会自动向下取整
- 此运算中的操作符和变量间可以有空格

### 使用 expr 命令
```
a=`expr 1 + 2`
b=`expr 1 - 2`
c=`expr 1 \* 2`
d=`expr 1 / 2`
e=`expr 1 % 2`
```
使用 expr 命令执行并将结果赋给变量; 使用时需注意以下几点
- 操作符合操作数之间一定要有空格间隔
- 乘号要用反斜杠进行转义
- 仅支持整数运算

### 用 bc 进行浮点数运算
bc 是一个交互式的计算器, 在命令行中可以键入 bc 进入 bc 的命令行; bc 也支持管道, 所以可以在 bc 命令行外执行计算
```
a=`echo '12.34-56.78' | bc`
```
#### 设置精度
bc 中的精度是指小数的位数, 在 bc 中进行除法运算时会发现结果是默认取整的, 即不会有小数部分, 可以通过 scale 来指定精度
```
a=`echo 'scale=3;12.34/56.78' | bc`
```

#### 进制转换
```
echo 'ibase=8;obase=2;775' | bc
```

### 自增自减
Shell 也支持变量自增自减, 以及 +=, -= 操作符; 但需要借助 let 命令实现, 在使用时要保证操作数必须是变量, 且必须是整数
```
a=1
let a++
echo ${a}
```

>**参考:**
[数值计算](https://blog.csdn.net/guodongxiaren/article/details/40370701)
