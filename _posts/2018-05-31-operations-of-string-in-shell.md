---
layout: post
title: "Linux 中字符串的操作"
date: "2018-05-31"
description: "Linux 中字符串的操作"
tag: [shell, linux]
---

**环境:** CentOS-7.5-x86_64

###

#### 获取字符串的长度
语法
```
${#str}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
len=${#str}
echo ${len} # 61
```

#### 使用 # 和 ## 获取尾部子字符串
- 使用 # 最小限度从前面截除子字符串
语法
```
${parameter#word}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str#*/}
echo ${sub_str} # /www.fengbohello.xin3e.com/blog/shell-truncating-string
```
- 使用 ## 最大限度的从前面截除子字符串
语法
```
${parameter##word}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str##*/}
echo ${sub_str} # shell-truncating-string
```

#### 使用 % 和 %% 获取头部子字符串
- 使用 % 最小限度从后面截除子字符串
语法
```
${parameter%word}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str%*/}
echo ${sub_str} # http://www.fengbohello.xin3e.com/blog
```
- 使用 %% 最大限度从后面截除子字符串
语法
```
${parameter%word}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str%%*/}
echo ${sub_str} # http:
```
#### 使用 ${str:} 模式获取子字符串
- 指定从左边第几个字符开始以及子串中的字符个数
语法
```
${str:start:len}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str:0:7}
echo ${sub_str} # http://
```
- 从左边第几个字符开始一直到结束
语法
```
${str:7}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str:7}
echo ${sub_str} # www.fengbohello.xin3e.com/blog/shell-truncating-string
```
- 从右边第几个字符开始以及字符的个数
语法
```
${str:0-start:len}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str:0-23:5}
echo ${sub_str} # shell
```
- 从右边第几个字符开始一直到结束
语法
```
${str:0-start}
```
示例
```
str="http://www.fengbohello.xin3e.com/blog/shell-truncating-string"
sub_str=${str:0-6}
echo ${sub_str} # string
```

>**参考:**
[Linux Shell 截取字符串](https://www.cnblogs.com/fengbohello/p/5954895.html)
