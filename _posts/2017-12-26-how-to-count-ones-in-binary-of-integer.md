---
layout: post
title: "如何统计整数二进制中一的个数"
date: "2017-12-16"
description: "如何统计整数二进制中一的个数"
tag: [algorithm]
---

### 题目
对任意非负整数, 统计其二进制展开中数位1的总数

#### 版本一
思路: 将整数 n 的二进制与 1 做与运算, 最低位为 1 则结果为 1, 反之则为 0 , 然后将 n 右移一位, 循环到 0 为止.
```
```