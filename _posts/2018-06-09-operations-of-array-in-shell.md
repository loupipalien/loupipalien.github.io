---
layout: post
title: "Shell 中数组的操作"
date: "2018-05-31"
description: "Shell 中数组的操作"
tag: [shell, linux]
---

### 新建数组
- 显示声明数组
```
declare -a array
array=(element_1, element_2, element_3, ...)
```
- 直接使用
```
array[0]=1
echo ${array[@]}
```
- 从字符串中切割
```
str="element_1, element_2, element_3"
array=(${str//,/ })
```

### 数组操作
- 查看数值长度
```
echo ${#array[@]}
echo ${#array[*]}
```
- 读取数组元素
```
# 所有元素
echo ${array[@]}
echo ${array[*]}
# 单个元素
echo ${array[index]}
```
- 赋值数组元素
```
array[index]=element
```
- 删除数组元素
```
# 所有元素
unset array
# 单个元素
unset array[index]
```

### 数组分片
```
# 返回字符串, 若想返回数组可以用 () 括起来, 见显示声明数组
echo ${array[@]:start:length}
echo ${array[*]:start:length}
```

>**参考:**
[Linux Shell数组常用操作详解](http://www.cnblogs.com/dlml/p/4213329.html)
