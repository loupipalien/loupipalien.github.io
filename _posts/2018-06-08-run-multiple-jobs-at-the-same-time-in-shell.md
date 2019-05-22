---
layout: post
title: "在 Shell 中同时跑多个任务"
date: "2018-06-08"
description: "在 Shell 中同时跑多个任务"
tag: [shell, linux]
---

### 思路
通过 for 循环控制并发作业数, 调用封装的方法并加 & 使之在当前窗口后台运行或说那个 nohup 调用另一个脚本再后台运行, 最后使用 wait 命令等待所有作业运行完成

### 伪代码
```
for item in ${items}
do
    function_name & | nohup sh another.sh &
done

wait
```

>**参考:**
[bash shell实现并发多进程操作](https://blog.csdn.net/wzy_1988/article/details/8811153)
